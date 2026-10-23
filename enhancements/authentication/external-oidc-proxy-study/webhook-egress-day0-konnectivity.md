# External OIDC webhook egress on HyperShift: the day-0 / no-worker-nodes question

**Last updated:** 2026-10-16
**Trigger:** [openshift/enhancements#2097](https://github.com/openshift/enhancements/pull/2097) review by `csrwng`
([review](https://github.com/openshift/enhancements/pull/2097#pullrequestreview-5511255219),
[inline comment](https://github.com/openshift/enhancements/pull/2097#discussion_r4265136675))
**EP under review:** `enhancements/authentication/proxy-support-for-external-oidc-auth-stack.md`

> **2026-10-16 update.** This document originally concluded that the external-OIDC webhook
> component did not exist in HyperShift and that its egress was unspecified. A HyperShift
> branch implementing it (`external-oidc-as-webhook`) has since been reviewed locally. It
> **does** add a konnectivity sidecar, **and** it enables the day-0 management-DNS
> fallback. That materially changes the conclusions — see §3. Day-0 login is therefore
> already mitigated; what EP 2097 must resolve is that the sidecar mode it assumes
> (`konnectivity-https-proxy`) is not the one shipped (socks5), and only one of the two
> carries the fallback. The recommended resolution is in §7.

All code references are to `openshift/hypershift` on branch `external-oidc-as-webhook`
unless noted.

---

## 1. Context

EP 2097 extends component-scoped proxy support to the External OIDC authentication stack.
On HyperShift it specifies that the External OIDC webhook (an OAuth API server running as a
TokenReview webhook authenticator in the HCP namespace) egresses to the identity provider
**through konnectivity into the guest/worker-node network**, with a
`konnectivity-https-proxy` sidecar applying the component proxy.

Relevant EP text:

| Lines | Content |
| --- | --- |
| 652-684 | Webhook egress diagram: HCP-side webhook → sidecar → konnectivity tunnel → guest network → component proxy → IdP |
| 688-690 | Requirement: the proxy "must be reachable and DNS-resolvable **from the guest/worker-node network**" |
| 714 | Component inventory lists the sidecar as **new** ("new, or an HTTPS/Dual extension of the existing socks5 injection") |
| 839-842 | Asserts the env "**already** points at the konnectivity sidecar"; CPO configures `konnectivity-https-proxy` with `--https-proxy`, `--http-proxy`, `--no-proxy` |
| 1002-1006 | Risk: webhook requests depend on konnectivity, guest DNS, worker-network availability. "**Do not fall back to direct management-cluster egress when the guest-network path fails.**" |

### How external OIDC works on HyperShift *before* the webhook change

The hosted kube-apiserver validates tokens itself via structured authentication config and
reaches the issuer **directly from the management cluster network**:

- Config rendered as a plain `AuthenticationConfiguration` ConfigMap, issuer CA inlined as
  a string — `v2/kas/auth.go:33-59`, `:155-178`; flag wiring at `v2/kas/config.go:204-208`
- KAS pod has **no konnectivity client sidecars**; the egress selector has only
  `controlplane: Direct`, `etcd: Direct`, `cluster: HTTPConnect`, and no selection covers
  authenticator egress — `assets/kube-apiserver/egress-selector-config.yaml:6-21`
- The only proxy applied to the KAS container is the **management cluster's own**, read from
  the CPO process env, populated from the management cluster's `configv1.Proxy` —
  `support/proxy/proxy.go:11-33`, `hypershift-operator/controllers/proxy/reconciler.go:88-97`

This is why login works today with zero workers.

*Caveat:* the kube-apiserver binary is not vendored, so whether the upstream OIDC
authenticator honours `HTTPS_PROXY` is not verifiable from this tree. What is verifiable:
HyperShift provides no OIDC-specific egress configuration and no konnectivity path for it.

---

## 2. The main question

> Currently you can log in to a HostedCluster in managed services before any workers are
> added to the cluster. Relying on konnectivity means that users will only be able to login
> to the API after at least one worker node has come up. — `csrwng`

Follow-up, from the inline comment: if we want authentication without worker nodes, how does
the management cluster reach the OIDC endpoint? (1) via the configured auth proxy, (2) via a
management cluster-wide proxy, (3) directly, (4) all of the above.

### Why zero workers breaks the tunnel

There are **two** `konnectivity-agent` workloads and they are not interchangeable:

| | CP-side agent | Data-plane agent |
| --- | --- | --- |
| Cluster / namespace | Management, HCP namespace | Guest, `kube-system` |
| Kind | Deployment (2 replicas) | DaemonSet |
| `--agent-identifiers` | `ipv4=<OAPI>&ipv4=<PackageServer>[&ipv4=<OAuthAPI>]` | `default-route=true` |
| Source | `v2/konnectivity_agent/deployment.go:53-70` | `hostedclusterconfigoperator/controllers/resources/konnectivity/reconcile.go:153` |

The konnectivity server runs `--proxy-strategies=destHost,defaultRoute`
(`assets/kube-apiserver/deployment.yaml:170`). Only the guest DaemonSet advertises
`default-route=true`, so a CONNECT to an external IdP falls through `destHost` to
`defaultRoute` and finds **no backend**. The DaemonSet tolerates everything and has no
nodeSelector (`reconcile.go:79-80`), but a DaemonSet with zero Node objects schedules zero
pods. Nothing runs a `default-route` agent on the management cluster.

HyperShift models this as a first-class state: `DataPlaneConnectionAvailable = Unknown`,
reason `NoWorkerNodesAvailable` (`hostedclusterconfigoperator/controllers/resources/resources.go:1695-1754`).
The dialer says it outright: konnectivity DNS failure "typically means konnectivity is down
(usually because there are no workers in the data plane)" (`support/konnectivityproxy/dialer.go:437-439`).

So the underlying mechanic is real. What changed is that the webhook PR already compensates
for it.

---

## 3. What the HyperShift webhook PR actually decides

Branch `external-oidc-as-webhook`, commits `2ea6ed801c` → `e64721fbac`, notably
`3c957d1421 feat(cpo): add new external-oidc-webhook component for ExternalOIDCAsWebhook`.

The component now exists at `control-plane-operator/controllers/hostedcontrolplane/v2/external_oidc_webhook/`
with assets under `v2/assets/external-oidc-webhook/`.

### 3a. It routes to the guest cluster — via Socks5, with the day-0 fallback enabled

`v2/external_oidc_webhook/component.go`:

```go
InjectKonnectivityContainer(component.KonnectivityContainerOptions{
    Mode: component.Socks5,
    Socks5Options: component.Socks5Options{
        ResolveFromGuestClusterDNS:      new(true),
        ResolveFromManagementClusterDNS: new(true),
    },
}).
```

Both flags are emitted by the Socks5 branch of the container builder
(`support/controlplane-component/konnectivity-container.go:168-195`, fallback flag at
`:185-186`).

### 3b. All webhook HTTP(S) egress goes through that sidecar

`v2/assets/external-oidc-webhook/deployment.yaml:30-36`:

```yaml
env:
- name: HTTP_PROXY
  value: socks5://127.0.0.1:8090
- name: HTTPS_PROXY
  value: socks5://127.0.0.1:8090
- name: NO_PROXY
  value: kube-apiserver
```

This is the important difference from `oauth-openshift`, where `HTTP_PROXY`/`HTTPS_PROXY`
point at the **HTTPS** sidecar (no fallback) and only `ALL_PROXY` reaches socks5. Here every
HTTP(S) request — discovery, JWKS, claim sources — traverses the socks5 sidecar that carries
`--resolve-from-management-cluster-dns=true`.

### 3c. Consequence: csrwng's day-0 concern is already largely mitigated

With no workers, the tunnel is unhealthy, so the dialer falls back to management-cluster DNS
and a direct dial (`support/konnectivityproxy/dialer.go:288-295`), reverting permanently once
the tunnel recovers. Day-0 login should therefore work — **through direct management-cluster
egress**.

Which is precisely what EP 2097 lines 1005-1006 forbid. The EP's prohibition now contradicts
shipped behavior in the component it describes.

### 3d. Three new gaps this exposes in EP 2097

1. **Mode mismatch.** The shipped component uses **Socks5**, which has *no*
   `--http-proxy`/`--https-proxy`/`--no-proxy` flags — those exist only in
   `konnectivity-https-proxy` (`konnectivity-socks5-proxy/main.go:40-51` vs
   `konnectivity-https-proxy/cmd.go:64-66`). EP 2097 lines 841-842 assume the HTTPS proxy.
   The EP's own hedge at line 714 turns out to be the right one: this must become an
   **HTTPS/Dual extension of the existing socks5 injection**.
2. **The cluster-wide proxy is not wired at all.** `grep Proxy` over the new component and
   `v2/kas/token_webhook.go` returns nothing. Proxy flags are emitted only in HTTPS mode and
   only when `hcp.Spec.Configuration.Proxy != nil`. So EP 2097's stated fallback — CPO uses
   the hosted cluster's cluster-wide proxy when the component field is absent — is
   unimplemented today and *cannot* be implemented in Socks5 mode.
3. **`NO_PROXY` baseline is minimal.** Just `kube-apiserver`, versus the EP's
   `NO_PROXY`/internal-endpoint rules.

### 3e. The generator is already shared, which simplifies the EP

`v2/external_oidc_webhook/auth.go` calls
`externaloidc.NewAuthenticationConfigurationGenerator` from
`vendor/github.com/openshift/library-go/pkg/operator/externaloidc`. The extraction the
webhook EP called for has happened, so CAO and CPO now generate the same config from one
place — EP 2097's parallel "CAO sets `trustedCAFilePath`" / "CPO sets `trustedCAFilePath`"
split collapses into a single library-go change.

Note that `trustedCAFilePath` does **not** exist yet: the generator inlines CA *content* as a
string (`generate.go:235-245`), exactly as KAS did. The EP's field is genuinely new.

---

## 4. Historical context

### 4a. Integrated auth already routes to the guest — and needed day-0 patches

The shipped status quo, patched three times for this exact problem:

| Mechanism | Added for | Reference |
| --- | --- | --- |
| `--resolve-from-management-cluster-dns` — fall back to mgmt DNS + direct dial until the tunnel is available, then stop permanently | "to allow quicker initialization of identity providers **while worker nodes have not joined**" | `support/controlplane-component/konnectivity-container.go:67-73`; hypershift#2261 |
| `ConnectDirectlyToCloudAPIs` — bypass konnectivity for cloud API suffixes | "to be able to create DNS records **before worker nodes are present** in the cluster" | `konnectivity-container.go:62-65`; hypershift#1601 |
| CPO in-process dialer for IdP config conversion sets `ResolveFromManagementClusterDNS: true` | IdP conversion at config-generation time | `v2/oauth/idp_convert.go:515-523` |

Runtime logic: `dialer.go:288-295` — bypass only while konnectivity is unhealthy, never
bypass port 53 (so the tunnel can be re-probed), flip back permanently once healthy. Health
tracking at `dialer.go:437-513`. The cloud-API bypass list includes `.microsoftonline.com`
and `.azure.com` (`dialer.go:524-550`), so Entra ID issuers get direct management egress
incidentally.

Sidecar configuration per component:

| Component | Mode | Guest DNS | Mgmt-DNS fallback |
| --- | --- | --- | --- |
| `oauth-openshift` (`v2/oauth/component.go:71-81`) | Dual | yes | yes (socks5 only; HTTP(S) traffic uses the HTTPS sidecar and misses it) |
| `openshift-oauth-apiserver` (`v2/oauth_apiserver/component.go:51-56`) | Socks5 | yes | **no** |
| `external-oidc-webhook` (new) | Socks5 | yes | **yes, and all HTTP(S) traffic uses it** |
| `kube-apiserver` | none | — | — |

*(The HTTP(S)-vs-`ALL_PROXY` split for oauth-openshift is inference from standard Go proxy
env semantics, not read off HyperShift code.)*

### 4b. Neither prerequisite EP considered egress

`external-oidc-additional-identity-information-sources.md` and
`direct-external-oidc-provider.md`: **zero** occurrences of konnectivity, egress, tunnel,
worker node, or data plane. The webhook EP's HyperShift section (lines 578-597) specifies the
new component's deployment, Service, config generation and secret projection, and stops.

Line 593 — CPO will "drop configuration via the structured authentication file (i.e. flag
`--authentication-config`)" — is the change that moves IdP-facing egress out of the KAS pod.
The implementation then chose konnectivity + fallback without the EP ever discussing it.

### 4c. Where EP 2097's konnectivity assumption came from

`proxy-support-for-integrated-auth-stack.md:313-319` notes HyperShift auth components
"already have proxy-aware transport plumbing for OIDC discovery … which includes konnectivity
sidecars for outbound proxy connectivity". EP 2097 followed that precedent; the routing has
been in the EP since its first commit (`b1a42fde`, 2026-09-09). The implementation
independently arrived at the same choice.

---

## 5. Ownership

The egress decision belongs to the webhook work, and **has now been made there** — in code,
not in an EP. EP 2097 inherits it rather than introducing it.

What remains EP 2097's to answer is narrower: applying a component proxy requires the HTTPS
sidecar, while the day-0 fallback the component now relies on is exposed only on the socks5
one. Reconciling the two is a small, concrete change ([§7](#7-how-it-can-be-solved)) that
nonetheless spans code EP 2097 does not otherwise touch.

---

## 6. Impact on EP 2097: what must change

1. **Lines 1005-1006** — "Do not fall back to direct management-cluster egress when the
   guest-network path fails" contradicts the shipped component, which does exactly that via
   `ResolveFromManagementClusterDNS`. Must be rewritten.
2. **Lines 841-842** — specify `konnectivity-https-proxy`; the component ships Socks5. The
   EP should state the switch to HTTPS mode and the accompanying fallback flag (see
   [§7](#7-how-it-can-be-solved)), and say who owns that change.
3. **Line 714 vs 839-840** — resolved in favour of 839-840: the sidecar now exists and the
   container env already points at it. Line 714's "new" is stale.
4. **Line 688** — guest-network reachability as a hard requirement needs qualifying against
   the day-0 window, during which egress is *not* from the guest network. If the §7
   recommendation is adopted, the proxy must be reachable from **both** networks.
5. **Cluster-wide proxy fallback** — the EP asserts CPO falls back to the hosted cluster-wide
   proxy; nothing implements this for the webhook. Either specify it as new work or drop the
   claim.
6. **Generated config ownership** — fold the CAO/CPO `trustedCAFilePath` split into the
   shared library-go generator.
7. **Test plan gap (independent finding)** — TLS interception is the stated justification for
   mounting the proxy CA on the webhook container, but every test case uses a non-intercepting
   proxy, the configuration in which that mount is unused. Nothing would catch CPO dropping it.

---

## 7. How it can be solved

### Recommendation: switch the webhook to the HTTPS sidecar and expose the fallback flag

The two capabilities the webhook needs are split across the two sidecar modes:

| | socks5 | https |
| --- | --- | --- |
| Chain to an external (component) proxy | **no** | **yes** |
| `--resolve-from-management-cluster-dns` (day-0 fallback) | **yes** | **no** |

Proxy chaining is the hard capability and only https has it; the fallback is the easy one
and is already implemented in the dialer both modes share. So move the fallback to https,
not the chaining to socks5.

**Three edits:**

1. `konnectivity-https-proxy/cmd.go` — expose the flag. The field already exists on the
   shared `konnectivityproxy.Options` (`dialer.go:77-83`); the options literal at
   `cmd.go:45-48` sets only `ResolveBeforeDial` and `ResolveFromGuestClusterDNS`. One line:
   `cmd.Flags().BoolVar(&opts.ResolveFromManagementClusterDNS, "resolve-from-management-cluster-dns", false, ...)`.
   No other change — `opts` is handed to `NewKonnectivityDialer`, which already implements
   the behavior.
2. `support/controlplane-component/konnectivity-container.go` — add
   `ResolveFromManagementClusterDNS *bool` to `HTTPSOptions` (`:39-53`, which lacks it;
   `Socks5Options` has it at `:67-73`) and emit it in the HTTPS branch (`:145-167`),
   mirroring `:185-186`.
3. `v2/external_oidc_webhook/component.go` — `Mode: component.Socks5` → `component.HTTPS`,
   with the new option set. The deployment env changes from `socks5://127.0.0.1:8090` to
   `http://127.0.0.1:<serving-port>`.

**HTTPS, not Dual.** Everything the webhook sends outbound is HTTP(S) — discovery, JWKS,
claim sources. Dual exists for components that also need raw TCP (LDAP). Using Dual here
would reproduce the `oauth-openshift` split, where `HTTP_PROXY`/`HTTPS_PROXY` point at the
https sidecar and the fallback — emitted only on the socks5 side — covers nothing that
matters.

### Correction: the fallback does not bypass the component proxy

An earlier draft of this document claimed that because `dialDirectWithoutProxy`
(`dialer.go:407-417`) ignores proxy settings, the day-0 window would bypass the component
proxy and fail in a restricted network. **That is wrong for https mode.** The layering:

```text
webhook container
  └─ HTTP_PROXY → konnectivity-https-proxy (goproxy)
       ├─ Layer 1: route via the component proxy?   ← decided here, ABOVE the dialer
       │            if yes, the address to connect to becomes proxy.example.com:3128
       └─ Layer 2: konnectivity dialer connects to that address
                    └─ tunnel unhealthy + fallback → dial it directly from management
```

The component proxy is selected by goproxy (`konnectivity-https-proxy/cmd.go:144-162`),
above the dialer. By the time the dialer runs, the address it is asked for is already the
*proxy's*. The fallback therefore bypasses konnectivity, not the proxy: it dials the
component proxy directly from the management network and the proxy hop survives.

`dialDirectWithoutProxy` is misleadingly named in this context — "without proxy" means
ignoring the *management cluster's* `HTTPS_PROXY` env var, not ignoring the component proxy.
Its sibling `dialDirectWithProxy` (`:419-435`) is the one that reads `os.Getenv("HTTPS_PROXY")`.

### What this leaves

Day-0 login keeps working **and** stays on the administrator's proxy. The cost is one added
requirement: the component proxy must also be DNS-resolvable and reachable from the
**management-cluster network**, because that is the path used until the first worker joins.
EP 2097 line 688 currently requires only guest-network reachability.

Remaining alternatives, should dual reachability be unacceptable:

| Option | Assessment |
| --- | --- |
| Accept degraded day-0 | Document that with a component proxy configured, login requires one worker. Cheap and honest, but may be unacceptable to managed services — csrwng's call. |
| Management cluster-wide proxy during the window | Matches today's KAS behavior (`support/proxy/proxy.go:11-33`), but it is the service provider's proxy, not the tenant's — cuts against the EP's tenant-isolation risk (lines 1018-1024). |

### Rejected: adding proxy chaining to the socks5 sidecar

Mechanically possible — `httpProxyDialer` (`support/konnectivityproxy/proxy_dialer.go:15-17`)
is an HTTP-CONNECT dialer with `Proxy-Authorization` support, already parameterized by
`forwardDial`, so it could take the konnectivity dialer instead of `proxy.Direct.Dial`.
Rejected for three reasons:

1. **The hostname is gone by dial time.** `armon/go-socks5` resolves FQDNs via the
   configured `Resolver` (`request.go:124-130`), and `Address()` returns `IP:port` whenever
   an IP is set (`:59-64`), which is what reaches `dial()` (`:177`). The chained dialer would
   emit `CONNECT 10.0.0.5:443`, breaking hostname-based proxy ACLs and SNI routing — and
   requiring the destination to resolve via guest DNS first, which in a restricted network is
   exactly the job being delegated to the proxy. The https proxy avoids this with
   `ResolveBeforeDial`, whose comment states the hostname must be resolved before being sent
   to the user's proxy.
2. **No scheme to match on.** `httpproxy.Config.ProxyFunc()` selects `httpProxy` vs
   `httpsProxy` by URL scheme; SOCKS5 supplies host:port only, so the scheme must be guessed.
3. **CONNECT is not a general TCP tunnel.** Many proxies restrict CONNECT to 443, which
   undercuts socks5's reason for existing (arbitrary TCP, e.g. LDAP).

---

## 8. Open items

- Is day-0 login before workers an explicit managed-services (ROSA/ARO HCP) requirement, or
  observed behavior that happens to work? Determines whether degraded day-0 is an acceptable
  fallback position if dual reachability is rejected.
- Is requiring the component proxy to be reachable from the management-cluster network
  acceptable, given the §7 recommendation depends on it?
- Who owns the Socks5 → HTTPS switch and the new flag — EP 2097, or the webhook
  implementation? The change spans `konnectivity-https-proxy`, the container builder, and
  the component, none of which EP 2097 otherwise touches.
- Does switching the webhook to HTTPS mode regress anything that relied on socks5 carrying
  non-HTTP traffic? Expected to be nothing, but worth confirming against the component's
  outbound dependencies.
- Does the upstream KAS OIDC authenticator honour `HTTPS_PROXY`? Not verifiable from the
  HyperShift tree (binary not vendored). Relevant only to the pre-webhook path.
