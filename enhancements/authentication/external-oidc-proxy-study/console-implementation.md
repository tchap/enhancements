# Console backend implementation plan: component-scoped auth proxy for External OIDC

Study note for
[`proxy-support-for-external-oidc-auth-stack`](../proxy-support-for-external-oidc-auth-stack.md).
Covers the Console **operand** (`openshift/console`), the one library-go change it
depends on, and the corrections the enhancement proposal itself needs.

## Context

The enhancement lets an administrator route **only** the authentication stack's
outbound traffic through a proxy, without enabling the cluster-wide egress proxy.
For Console this means its server-side OIDC client — discovery, JWKS,
authorization-code exchange, token refresh — must use a dedicated proxy and a
dedicated trust bundle, while every other Console HTTP client (k8s proxy, Thanos,
Alertmanager, plugin proxies, Helm, …) keeps its current routing and trust.

Two of the three halves are already done and committed locally:

- **openshift/api** — `Console.Spec.AuthProxy` (`ConsoleAuthProxyConfig`) and the
  `AuthenticationComponentProxyExternalOIDC` gate.
- **console-operator** (`db89d1d79`) — resolves `spec.authProxy`, syncs the
  `trustedCA` ConfigMap from `openshift-config` into `openshift-console` as
  `v4-0-config-system-auth-proxy-ca`, mounts it at
  `/var/auth-proxy-ca/ca-bundle.crt`, and renders an `auth.proxy` block into
  `console-config.yaml`.

**The gap is the Console backend.** It ignores `auth.proxy` entirely
(`yaml.Unmarshal` at `pkg/serverconfig/config.go:145` is non-strict, so the key is
silently dropped) and its OIDC client only honours process-wide `HTTP(S)_PROXY`.

The operator deliberately does **not** roll the Console Deployment when the proxy CA
ConfigMap's *content* changes — that ConfigMap is not in `withConsoleAnnotations`'
content hash. **Hot-reloading the CA from disk is therefore a hard requirement**, not
an optimisation.

### What the operator hands us

```yaml
auth:
  authType: oidc
  oidcIssuer: https://idp.example.com
  oauthEndpointCAFile: /var/auth-server-ca/ca-bundle.crt   # existing issuer CA
  proxy:                                                    # new, OIDC-only
    httpProxy: http://proxy.example.com:3128
    httpsProxy: http://proxy.example.com:3128
    noProxy: [".cluster.local", ".cluster.local.", ".svc", ".svc.", "127.0.0.1", "idp.internal.example.com", "localhost", "localhost."]
    trustedCAFilePath: /var/auth-proxy-ca/ca-bundle.crt
```

The operator already appends the bypass defaults plus `KUBERNETES_SERVICE_HOST` and
sorts/dedups them (`console-operator pkg/proxyconfig/config.go` `Finalize()`), and
only emits the block when `authType: oidc`. **Console does no resolution or
defaulting of its own** — it consumes the block verbatim.

Required semantics:

- `auth.proxy` **absent** → today's behaviour exactly (env-var proxy, existing trust).
- `auth.proxy` **present** → used in full, with **no per-field inheritance from the
  environment**. An omitted `httpsProxy` means direct HTTPS even though `HTTPS_PROXY`
  is still set on the pod by the operator's cluster-wide-proxy env rendering.
- Scope is the OIDC clients only; Console's other clients and its process environment
  are untouched.

## Decisions

| Decision | Choice |
| --- | --- |
| Config key | `trustedCAFilePath` (matches the operator and the webhook); **fix the EP**, which says `trustedCAFile` |
| CLI flags | None — config-file only, like the existing plugin `proxy:` section |
| Issuer CA | Migrate to the same dynamic-CA mechanism; delete `newHTTPClient` and both `sync.Map` caches |
| Transport | `library-go/pkg/network/httptransport`, after adding a TLS-profile option upstream |

## Design

### Why a TLS-profile option is needed upstream

`httptransport.NewRoundTripper` builds a bare `&tls.Config{RootCAs: pool}`. Console's
current OIDC client wraps it in `oscrypto.SecureTLSConfig` (TLS 1.2 minimum plus
`DefaultCiphers()`: ECDHE+AEAD only). Adopting httptransport as-is would widen
Console's ClientHello to Go's defaults — CBC-SHA1 and static-RSA key exchange for
TLS 1.2 — on the login path, and would make this the only outbound TLS client in the
Console repo not applying `SecureTLSConfig`. The returned dynamic transport rebuilds
its own `tls.Config` on every CA swap, so post-hoc mutation is impossible: the option
must live inside the package. oauth-server will want it too.

### New package `pkg/auth/httpclient`

```text
pkg/auth/httpclient/proxy.go     ProxyConfig, Validate(), ProxyFunc(), String() (redacted)
pkg/auth/httpclient/clients.go   Clients{OIDC, K8s *http.Client}, New(...), (*Clients).Run(ctx)
```

```go
// ProxyConfig mirrors console-config.yaml's auth.proxy, consumed verbatim.
type ProxyConfig struct {
	HTTPProxy, HTTPSProxy string
	NoProxy               []string
	TrustedCAFilePath     string
}

// New builds both clients and registers every CA listener. It must return
// before Run is called.
func New(issuerCAFile, k8sCAFile string, proxy *ProxyConfig) (*Clients, error)

// Run starts one goroutine per dynamic CA provider and returns immediately.
func (c *Clients) Run(ctx context.Context)
```

Construction:

- **OIDC client** — `httptransport.NewRoundTripper` with
  `WithTLSProfile(oscrypto.DefaultTLSVersion(), oscrypto.DefaultCiphers())`, a
  `WithCAContentProvider` per non-empty CA file (issuer CA, proxy CA), and — **only
  when `proxy != nil`** — `WithProxyFunc(cfg.ProxyFunc())`. Pool is system roots plus
  issuer CA plus proxy CA.
- **K8s client** — `k8sCAFile` only, static (`WithCAFile`), environment proxy, 5s
  timeout. Never the component proxy, never the proxy CA. Used solely for
  `/.well-known/oauth-authorization-server` in integrated-OAuth mode.

Reuse:

- `dynamiccertificates.NewDynamicCAContentFromFile` (`k8s.io/apiserver`) for the CA
  providers — fsnotify-backed, keeps last-known-good on invalid updates, exposes
  `RunOnce(ctx)` for deterministic tests. Precedent:
  `oauth-server/pkg/oauthserver/oauth_apiserver.go` `configureTransport`.
- `golang.org/x/net/http/httpproxy.Config{...}.ProxyFunc()` for matching — **struct
  literal, never `FromEnvironment()`**, which is exactly the enhancement's "no
  per-field inheritance". The adapter is one line:
  `func(r *http.Request) (*url.URL, error) { return inner(r.URL) }`.
  Verified against the enhancement's `NO_PROXY` table: CIDRs, `host:port`, `*`,
  leading-dot suffixes, bare-host-matches-subdomains, and the trailing-dot forms all
  behave as described.

### Correctness points to encode

1. **Integrated OAuth must never be proxied.** `openShiftAuth` embeds `*oidcConfig`
   (`pkg/auth/oauth2/auth_openshift.go:33`), so the issuer client also serves the
   integrated-OAuth HEAD probe (`:157`), refresh (`:258-261`) and the shared code
   exchange (`pkg/auth/oauth2/auth.go:402`). Today the only guard is the operator
   emitting the block solely for `authType: oidc` — in another repo, behind feature
   gates, across a version-skew boundary. Add an operand-side guard: a pure
   `effectiveProxy(authType, *serverconfig.AuthProxy) *httpclient.ProxyConfig` in
   `authoptions.go` returning `nil` for `openshift`/`disabled`, plus a
   `klog.Warning`-and-drop in `ApplyConfig`. Warn, don't error — console-config is
   parsed non-strictly by design and a hard error turns an operator bug into an
   unbootable Console.
2. **Always install an explicit proxy func when `auth.proxy` is present**, even if
   both URL fields are empty. The pod still carries `HTTP_PROXY`/`NO_PROXY` from the
   operator's cluster-wide-proxy env rendering; omitting the func silently inherits
   them.
3. **Listener-registration ordering is load-bearing and a data race.**
   `DynamicFileCAContent.AddListener` is an unsynchronized slice append that the
   worker goroutine ranges over. `New` must build *both* clients before any `Run`.
4. **A lifetime context already exists** — `cmd/bridge/main.go:620` creates a
   SIGTERM-cancelled `ctx` before `ApplyTo` at `:746`. Thread it through `ApplyTo` →
   `getAuthenticator` and replace `context.Background()` at `authoptions.go:341`.
   Incidental fix: the `asynccache` reload loops currently never stop on SIGTERM.
5. **Collapse `clientFunc func() *http.Client` to a single `client *http.Client`.**
   go-oidc pins the client pointer into `remoteKeySet.ctx` at provider construction;
   with a dynamic transport the pointer is stable and the pool swap is visible to
   already-captured contexts immediately, rather than waiting up to five minutes for
   the `asynccache` provider rebuild. Strictly better, and it removes two unbounded
   `sync.Map` leaks (`pkg/auth/oauth2/auth.go:41-48`) that also leak idle connections.
6. **Fail fast on a missing or invalid proxy CA at startup.**
   `NewDynamicCAContentFromFile` errors if the file is absent. The operator's volume
   is non-optional and it blocks the sync with `AwaitingConfigMapSync`, so the window
   is tiny and self-heals in one rollout. The error message must name the path *and*
   the config key. Only create the provider when `TrustedCAFilePath != ""` — proxy
   URLs without a CA is a legal configuration.
7. **Validate proxy URLs explicitly.** `httpproxy`'s `init()` silently swallows
   unparseable URLs into "no proxy", producing confusing direct-connect failures —
   precisely the lockout-debugging scenario the enhancement's Support Procedures warn
   about. `url.Parse` plus a non-empty host check; don't be stricter about scheme than
   `parseProxy`, which accepts bare `host:port`.
8. **Redact credentials.** `httpProxy: http://user:pass@proxy:3128` is legal. Log the
   effective proxy at startup (support needs it) via a `String()` that strips userinfo.
9. **Timeouts.** Keep `5 * time.Second` when `auth.proxy` is absent (zero change for
   everyone not using the feature); use 30s when present — the client timeout covers
   the proxy `CONNECT` plus, under TLS interception, the proxy's own upstream
   handshake. Note also that `issuerCA == ""` currently yields a bare
   `http.DefaultClient` with **no** timeout (`auth.go:152-154`); unifying gives that
   path a timeout for the first time. That is a bug fix — today a hung IdP wedges the
   `asynccache` goroutine forever — but call it out in the PR body.

## Work items

### 0. library-go: add a TLS-profile option *(upstream; blocks step 1)*

In `library-go/pkg/network/httptransport/transport.go`, add
`WithTLSProfile(minVersion uint16, cipherSuites []uint16) Option` storing onto
`Config` and applied inside `newHTTPTransport`, so it survives every dynamic rebuild.
Follow the existing option conventions (at most once, error on repeat). Add a test
asserting the profile survives a `dynamicTransport` rebuild.

**MVS warning:** pin Console to the merge commit, not necessarily master tip.
`3f31f20002f6` (the original httptransport commit) requires an *older* `openshift/api`
than Console's pin — zero transitive churn. Current master tip requires newer
`openshift/api` and `client-go`, which would drag in a large unrelated vendor diff
since Console vendors much of `openshift/api`. Check this before choosing the
pseudo-version.

### 1. `deps:` vendor bump *(separate commit, per Console's CLAUDE.md)*

Bump `github.com/openshift/library-go` to the commit from step 0, then
`go mod tidy && go mod vendor`. Newly vendored packages:
`library-go/pkg/network/httptransport`,
`k8s.io/apiserver/pkg/server/dynamiccertificates`, and
`golang.org/x/net/http/httpproxy` — roughly 2,100 lines. Every transitive import of
`dynamiccertificates` is already vendored, and `fsnotify` is already an indirect
dependency. `k8s.io/apiserver` is promoted from indirect to direct.

`test-backend.sh:33-40` re-runs `go mod tidy && go mod vendor` and fails on any diff,
so produce this commit with exactly those commands on the same toolchain.

### 2. `pkg/serverconfig`: add `auth.proxy`

In `pkg/serverconfig/types.go`, append
``Proxy *AuthProxy `yaml:"proxy,omitempty"` `` to `Auth` (after
`InactivityTimeoutSeconds`, line 107) and add the `AuthProxy` struct. **Copy
`console-operator/pkg/console/subresource/consoleserver/types.go:102-109`
byte-for-byte**, doc comment included; both repos use `gopkg.in/yaml.v2`. No
`SetFlagsFromConfig` change — the `auth` section bypasses the flagset and is applied
through `AuthOptions.ApplyConfig`.

Add a fixture `pkg/serverconfig/test/auth-proxy-config.yaml` holding the operator's
exact golden output.

### 3. `pkg/auth/httpclient`: the new package

As designed above.

### 4. `pkg/auth/oauth2`: inject clients instead of constructing them

- Delete `newHTTPClient` (`auth.go:151-198`) and both `sync.Map`s (`:41-48`); drop the
  now-unused `os`, `crypto/x509`, `crypto/tls`, `oscrypto` and `sync` imports.
- `Config` (`:108-143`): replace `IssuerCA` and `K8sCA` with
  `OIDCClient *http.Client` and `K8sClient *http.Client`.
- `completedConfig` (`:145-149`) and `OAuth2Authenticator` (`:50`): `clientFunc` →
  `client`. `Complete()` (`:447-485`) defaults both to `http.DefaultClient` when nil,
  so the existing `auth_test.go` cases keep working untouched.
- `NewOAuth2Authenticator` (`:202-264`): the `AuthSourceOpenShift` branch uses
  `c.K8sClient` instead of `newHTTPClient(c.K8sCA, true)` at `:233`.
- Update the five `getClient()`/`clientFunc()` call sites: `auth.go:402`,
  `auth_oidc.go:56` and `:110`, `auth_openshift.go:157` and `:259`; and
  `oidcConfig.getClient` (`auth_oidc.go:37`) → `client`.

### 5. `cmd/bridge`: wire it up

- `cmd/bridge/config/auth/authoptions.go`: `AuthOptions` gains
  `Proxy *serverconfig.AuthProxy` (no flags); `ApplyConfig` copies it; `Complete`
  validates the URLs and populates `completedOptions.Proxy *httpclient.ProxyConfig`
  via the new `effectiveProxy` helper; `ApplyTo` and `getAuthenticator` gain a leading
  `ctx`; `getAuthenticator` builds `httpclient.New(...)`, calls `clients.Run(ctx)`
  **before** `oauth2.NewOAuth2Authenticator` (which blocks up to five minutes on
  discovery), sets the two client fields, and logs the redacted effective proxy.
- `cmd/bridge/main.go:746` — pass `ctx`.

### 6. Update the enhancement proposal

**Status: applied.** All eight items below are now reflected in
[`proxy-support-for-external-oidc-auth-stack.md`](../proxy-support-for-external-oidc-auth-stack.md);
the list is retained as the rationale for those edits.

Mismatches found between the enhancement text and the committed operator behaviour:

1. **`auth.proxy.trustedCAFile` → `trustedCAFilePath`** (EP ~L539 and ~L926). The
   webhook section already says `trustedCAFilePath`; the Console section is the sole
   outlier.
2. The rendered `noProxy` example is shown in insertion order; the operator sorts and
   dedups.
3. The EP never states the emission precondition the operator implements:
   `ResolveProxy` returns `nil` unless `httpProxy` **or** `httpsProxy` is non-empty,
   so a `spec.authProxy` with only `trustedCA`/`noProxy` silently produces no block
   and no CA mount. State the rule — the API's CEL validation already requires at
   least one URL, so say so.
4. "Proxy Resolution" says Console Operator reads `proxy.config.openshift.io/cluster`;
   "Console Login and Token Refresh" says Console retains an environment-based
   fallback. Both are true at different layers — the operator has always rendered the
   cluster-wide proxy as **Deployment environment variables**. Reword, and add that
   those variables remain set even when `auth.proxy` is present, which is why Console
   must always install an explicit proxy func.
5. "the caller's Kubernetes service IP when available through
   `KUBERNETES_SERVICE_HOST`" — the *operator* reads its own environment and bakes the
   value in; the operand does not resolve it at runtime. Reword, and note the value
   must be emitted unbracketed for IPv6 (`httpproxy` turns `[fd02::1]` into a bogus
   domain matcher rather than an IP matcher).
6. "Console applies `auth.proxy` only to its OIDC discovery, JWKS, code-exchange, and
   refresh clients" — in the backend these are a single client that *also* serves
   integrated OAuth. Add a sentence requiring the operand to guard on
   `authType == oidc`, not just the operator.
7. "Invalid updates are reported while retaining the last successfully loaded trust
   bundle" — for Console, "reported" is only a pod log line. Console has no status
   condition of its own, and Console Operator can observe source/sync failures but not
   an operand-side load failure. Say so explicitly, or require a metric or event.
8. Add a sentence to the credential-leakage risk noting that
   `Console.spec.authProxy.httpProxy` is plain text in the CRD and in the generated
   ConfigMap, and that operands must redact it when logging.

## Tests

| File | Coverage |
| --- | --- |
| `pkg/auth/httpclient/proxy_test.go` | Matching table over the operator's exact `noProxy` output × request URLs: `.svc` versus `.svc.` (locks in why both forms are emitted), `.cluster.local`, short names, IP, CIDR, `host:port` mismatch, `*`, loopback short-circuit. Plus no-environment-inheritance (`t.Setenv` `HTTPS_PROXY`/`NO_PROXY`, assert they are ignored), `nil` config → environment honoured, URL validation, credential redaction. |
| `pkg/auth/httpclient/clients_test.go` | Dynamic proxy-CA reload driven by `RunOnce` rather than racing fsnotify — fail, rewrite CA, succeed; invalid update keeps last-known-good; missing CA file is fatal; empty `TrustedCAFilePath` skips the provider; issuer CA and proxy CA both trusted alongside system roots; end-to-end through a hijacking `CONNECT` proxy; K8s client ignores the proxy and the proxy CA; timeout selection. |
| `pkg/serverconfig/auth_proxy_test.go` | Cross-repo contract: unmarshal the operator's golden YAML fixture and `DeepEqual` the result. Include a `trustedCAFile`-spelled case asserting it is **ignored**, so a future rename fails loudly. Absent block → `Proxy == nil`. |
| `cmd/bridge/config/auth/authoptions_test.go` | `effectiveProxy` table — `(oidc, non-nil)` → non-nil; `(openshift, non-nil)` → **nil**; `(disabled, non-nil)` → nil. `ApplyConfig` copies the block; `Complete` rejects malformed URLs. |
| `pkg/auth/oauth2/auth_test.go` | Existing cases pass unchanged (nil clients → `http.DefaultClient`). Add: `Complete` uses the injected client; integrated-OAuth metadata discovery uses `K8sClient` while the issuer probe uses `OIDCClient` (counting round trippers). |

Repo patterns to follow: plain `testing` with table tests, `httptest`,
`stretchr/testify/require`, and the signing `mockOIDCProvider` already in
`pkg/auth/oauth2/auth_oidc_test.go`. Reference implementation for the reload test:
`oauth-server/pkg/oauthserver/dynamic_transport_test.go:29-90`.

## Verification

```bash
./build-backend.sh                      # compiles
./test-backend.sh                       # unit tests + the go mod tidy/vendor gate
git rebase --exec './build-backend.sh'  # confirm the series is bisect-clean
```

Manual off-cluster smoke test (no flags exist by design, so drive it through a config
file):

1. Run a forward proxy with a self-signed CA (for example `mitmproxy`) and an OIDC
   IdP (Keycloak, or the `mockOIDCProvider` from the tests served over TLS).
2. Write a `console-config.yaml` with `auth.authType: oidc`, `auth.oidcIssuer`, and
   the `auth.proxy` block pointing `trustedCAFilePath` at the proxy's CA.
3. `./bin/bridge --config=console-config.yaml …`; confirm the startup log shows the
   redacted effective proxy, and that the proxy's access log records discovery, JWKS,
   and the token-endpoint exchange on login.
4. **Hot reload:** rotate the proxy's serving certificate, overwrite the CA file in
   place, and confirm a fresh login succeeds **without restarting bridge** — this is
   the requirement the operator's no-rollout-on-CA-content behaviour depends on. Then
   write garbage into the file and confirm login still works on the last-known-good
   bundle.
5. **Negative:** set `auth.authType: openshift` with an `auth.proxy` block present and
   confirm the warning fires and no traffic reaches the proxy.

On-cluster coverage (requires `AuthenticationComponentProxyExternalOIDC`, currently
`DevPreviewNoUpgrade`) follows the enhancement's
[Console Login](../proxy-support-for-external-oidc-auth-stack.md#console-login) test
list: set `spec.authProxy` with a `trustedCA` and confirm login succeeds with the OIDC
requests observed on the proxy and the `console` cluster operator healthy; delete the
`trustedCA` ConfigMap and confirm Degraded; recreate it and confirm recovery; remove
the field and confirm the component proxy drops out of the path.
