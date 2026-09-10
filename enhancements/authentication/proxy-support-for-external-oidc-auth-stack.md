---
title: proxy-support-for-external-oidc-auth-stack
authors:
  - "@tchap"
  - "@wouldgo"
reviewers:
  - "@liouk" # The author of the original External OIDC EP, to review the whole EP.
approvers:
  - "@benluddy"
api-approvers:
  - "None"
creation-date: 2026-09-10
last-updated: 2026-09-10
status: provisional
tracking-link:
  - "https://redhat.atlassian.net/browse/OCPSTRAT-3721"
see-also:
  - "/enhancements/authentication/direct-external-oidc-provider.md"
  - "/enhancements/authentication/external-oidc-additional-identity-information-sources.md"
  - "/enhancements/authentication/proxy-support-for-integrated-auth-stack.md"
replaces:
superseded-by:
---

# Proxy Support for External OIDC Auth Stack

## Summary

This enhancement extends proxy support to External OIDC authentication mode, which uses oauth-apiserver as a webhook authenticator for kube-apiserver. It reuses the `Authentication.spec.proxy` API field introduced in the integrated auth proxy enhancement, enabling oauth-apiserver to reach external OIDC providers through a component-scoped proxy.

This addresses the same disconnected environment challenges as the integrated auth proxy enhancement but for the External OIDC authentication architecture.
The feature requires the `AuthenticationComponentProxyExternalOIDC` feature gate (which requires `AuthenticationComponentProxy` as a prerequisite) and complements the integrated auth proxy enhancement to provide proxy support across all authentication modes.

## Motivation

External OIDC authentication delegates token validation from kube-apiserver to oauth-apiserver as a webhook authenticator. On every API request with a bearer token, kube-apiserver calls oauth-apiserver, which validates the token against an external OIDC provider by fetching JWKS keys and calling UserInfo endpoints.

This architecture faces the same disconnected environment challenges as integrated OAuth authentication (see the "Proxy Support for Integrated Auth Stack" [Motivation](./proxy-support-for-integrated-auth-stack.md#motivation) section), but with higher impact: External OIDC validates tokens on every API request (not just during login), making proxy configuration even more critical for cluster operation.

Currently, customers must use cluster-wide proxy configuration, which opens egress for all cluster components and creates operational overhead for ACL management and security auditing. A component-scoped proxy for oauth-apiserver provides the same benefits as the integrated auth proxy enhancement of a clearer security boundaries.

An outline of what this enhancement is going to improve can is shown in this diagram:
```mermaid
---
config:
  htmlLabels: true
  theme: dagre
  flowchart:
    curve: linear
---
flowchart TB
  User[End User]
  KAS[kube-apiserver]
  subgraph Involved component in the enhancement
    CAO[Cluster Auth Operator]
    OAAS[oauth-apiserver webhook]
  end
  Proxy[Proxy Server]
  IDP[External OIDC Provider]

  User  -- Bearer token                                                               --> KAS

  KAS   -- POST - TokenReview                                                         --> OAAS
  OAAS  -- OIDC interactions (JWKS, UserInfo)                                         --> Proxy
  Proxy -- Forward                                                                    --> IDP

  CAO   -- Validate issuer CA                                                         --> Proxy
  CAO   -- Deploy with \nHTTP_PROXY, \nHTTPS_PROXY, \nNO_PROXY \nenvironment variables --> OAAS
```

### User Stories

* As an OpenShift cluster administrator using External OIDC authentication, I want to configure proxy settings scoped to authentication components, so that oauth-apiserver can validate tokens against external OIDC providers without opening cluster-wide egress for all components.

### Goals

- Enable proxy configuration for oauth-apiserver webhook authenticator to reach external OIDC providers
- Reuse the existing `Authentication.spec.proxy` API field (no new API surface)
- Support operator-time OIDC issuer validation through proxy (validateCACert function)
- Support runtime token validation through proxy (JWKS fetch, UserInfo calls)
- Follow the same proxy resolution pattern as integrated auth (component-scoped > cluster-wide > none)

### Non-Goals

- Per-OIDC-provider proxy configuration (proxy applies uniformly to all configured OIDC providers)
- Modifications to cluster-wide proxy behavior or configuration
- Modifications to upstream k8s.io/apiserver OIDC code (vendored code already respects HTTP_PROXY environment variables)
- Hot-reload of proxy CA certificates (proxy CA changes trigger oauth-apiserver redeployment)

## Proposal

To achieve the given goals, this document propose to extend cluster-authentication-operator to apply `Authentication.spec.proxy` configuration to External OIDC components through two mechanisms:

1. **Operator validation** - Update the Cluster Authentication Operator (CAO)  controller to use a proxy-aware HTTP transport when validating OIDC issuer CA certificates during configuration generation;
2. **Runtime oauth-apiserver** - Inject HTTP_PROXY, HTTPS_PROXY, and NO_PROXY environment variables into the oauth-apiserver deployment so that upstream Kubernetes OIDC code uses the proxy for JWKS fetches and UserInfo calls.

### Workflow Description

**Cluster Administrator** is a human user responsible for configuring cluster authentication.

#### Configuration Workflow

1. Cluster administrator sets `spec.proxy` on `operator.openshift.io/v1 Authentication/cluster` resource (see the "Proxy Support for Integrated Auth Stack" [API Extensions](./proxy-support-for-integrated-auth-stack.md#api-extensions));
2. Cluster authentication operator picks up the change and triggers:
   - Re-validation of OIDC issuer via proxy;
   - Re-deployment of oauth-apiserver with proxy environment variables injected;
   - Proxy trustedCA ConfigMap sync from openshift-config to openshift-oauth-apiserver namespace;
5. oauth-apiserver pods restart with new proxy configuration

#### Runtime Authentication Workflow

1. End user authenticates with external OIDC provider (client-side, outside cluster scope)
2. End user receives OIDC token from provider
3. End user makes API request with `Authorization: Bearer <token>` header
4. kube-apiserver receives request and calls oauth-apiserver webhook authenticator
5. oauth-apiserver validates token through configured proxy:
   - Fetches JWKS keys from OIDC provider's .well-known/openid-configuration endpoint
   - Validates token signature against JWKS
   - Calls UserInfo endpoint for additional claims
   - Executes CEL claim mapping expressions
6. oauth-apiserver returns authentication response to kube-apiserver
7. kube-apiserver authorizes request and returns response to user

All OIDC provider HTTP calls automatically use proxy configured via HTTP_PROXY/HTTPS_PROXY/NO_PROXY environment variables.

#### Proxy Configuration Update Workflow

1. Cluster administrator updates `spec.proxy` on Authentication CR (e.g., changes httpProxy URL)
2. Operator detects change via informer
3. Operator re-validates OIDC issuer via new proxy configuration
4. Operator updates oauth-apiserver deployment with new environment variables
5. Deployment controller triggers rolling update
6. New oauth-apiserver pods start with updated proxy configuration
7. Old pods terminate after new pods are ready

### API Extensions

**No new API extensions** - This enhancement reuses the existing `Authentication.spec.proxy` field defined in the integrated auth proxy enhancement (ref [API Extensions](./proxy-support-for-integrated-auth-stack.md#api-extensions)).

Reference the integrated auth proxy enhancement for full API definition:
- `spec.proxy.httpProxy` - string, 1-2048 characters, http/https URL for HTTP connections
- `spec.proxy.httpsProxy` - string, 1-2048 characters, http/https URL for HTTPS connections
- `spec.proxy.noProxy` - string array, maximum 64 items, each 1-253 characters, comma-separated list of domains/IPs to exclude from proxy
- `spec.proxy.trustedCA.name` - ConfigMap reference in openshift-config namespace containing PEM-encoded proxy CA certificate bundle

CEL validation enforces at least one of httpProxy or httpsProxy must be set when spec.proxy is configured.

### Topology Considerations

#### Hypershift / Hosted Control Planes

TBD

#### Standalone Clusters

Yes, this is applicable to standalone clusters.

#### Single-node Deployments or MicroShift

> How does this proposal affect the resource consumption of a
single-node OpenShift deployment (SNO), CPU and memory?

This does not add any additional overhead besides network latency for the one extra proxy hop.

> How does this proposal affect MicroShift? For example, if the proposal
adds configuration options through API resources, should any of those
behaviors also be exposed to MicroShift admins through the
configuration file for MicroShift?

The auth stack is not present on MicroShift.

#### OpenShift Kubernetes Engine

Not affected.

### Implementation Details/Notes/Constraints

#### Proxy Resolution

Component-scoped proxy completely replaces the cluser-wide proxy following the same policy as described in the "Proxy Support for Integrated Auth Stack" [Proxy Resoultion](./proxy-support-for-integrated-auth-stack.md#proxy-resolution) section

#### Cluster Auth Operator

When CAO is creating/updating the `auth-config` ConfigMap triggerd by changes on the `Authentication` configuration or on the provider's CA bundle in the `openshift-config` namespace, the verification of the OIDC issuer CA certificate must be proxy aware.
This is achieved by adding a `ProxyResolver` (e.g. `github.com/openshift/cluster-authentication-operator/pkg/controllers/common.AuthProxyResolver`) field to the `github.com/openshift/cluster-authentication-operator/pkg/controllers/externaloidc/generation/oauthapiserver.AuthenticationConfigurationGenerator` struct and use the values resolved to adapt the CA validation method.

#### oauth-apiserver workload

oauth-apiserver way how get deployed by CAO must change in the scenario where the usage of and external oidc issuer is enabled by injecting the HTTP_PROXY, HTTPS_PROXY and NO_PROXY environment variables into container spec and mount a volume with the proxy CA certificate.

### Risks and Mitigations

**Cluster lockout from invalid proxy configuration.**
A misconfigured component proxy (wrong URL, missing CA) can prevent
the OAuth Server from reaching the external IdP, locking all users out of the cluster.
The proxy validation controller tests IdP connectivity on configuration change and reports
warnings for unreachable IdPs and `Degraded` for proxy-level failures (connection refused,
TLS handshake errors), but these conditions are informational — the configuration is applied
regardless. Recovery is possible via `kubeadmin` credentials or client certificate
authentication, which bypass OAuth entirely. The risk is the same class as any IdP
misconfiguration today.

**Proxy as an untrusted intermediary.**
A proxy positioned between auth components and the IdP can observe or tamper with
authorization codes, tokens, and user info. This is the same trust model as the
cluster-wide proxy — the administrator who configures the proxy is assumed to control it.
The `trustedCA` field pins the proxy's TLS certificate, and all IdP traffic uses HTTPS,
so the proxy cannot silently intercept without a trusted CA. The security model and
implications should be documented.

**Network dependency in the authentication path.**
Adding a proxy hop introduces a new availability dependency: if the proxy is down,
all authentication fails. This is the same failure class as a cluster-wide proxy outage
and is mitigated by setting a `Degraded` condition when the proxy is unreachable, giving
administrators visibility. Proxy high availability is the administrator's responsibility
and should be documented as a prerequisite.

**Proxy credential leakage.**
Proxy credentials embedded in the URL (e.g., `http://user:pass@proxy:3128`) are stored
in the operator spec and propagated as environment variables to OAuth Server pods. This
mirrors the cluster-wide proxy's approach. Support for a `proxyCredentials`
SecretNameReference for improved credential handling could be added as a follow-up.

**Debugging complexity from dual proxy sources.**
When both a cluster-wide proxy and a component-scoped proxy exist, diagnosing connectivity
issues requires understanding the precedence rules. The proxy validation controller reports conditions when the proxy itself is misconfigured
and emits events when IdP endpoints are unreachable through the proxy. The resolved proxy
values are visible as environment variables on the OAuth Server pod spec.
The precedence rules (component-scoped > cluster-wide > none) should be documented
clearly.

### Drawbacks

TBD

## Alternatives (Not Implemented)

### Alternatives inherited from integrated auth proxy enhancement

1. **Per-OIDC-provider proxy configuration**
   - Alternative: Add proxy fields per OIDC provider in issuer configuration
   - Rejected: Adds API complexity. Component-scoped proxy is sufficient for disconnected environment use cases where all external traffic routes through the same proxy infrastructure.

2. **Annotation-based proxy configuration**
   - Alternative: Configure proxy via annotations on Authentication resource
   - Rejected: Loses strongly-typed API validation. Increases risk of configuration errors. Annotations are for non-critical metadata, not core configuration.

3. **Discovery URL override with reverse proxy**
   - Alternative: Allow overriding OIDC discovery URLs to point to in-cluster reverse proxy
   - Rejected: Only solves OIDC-specific problem. Doesn't address general egress requirements for other external dependencies. Adds configuration complexity by requiring administrators to deploy and maintain reverse proxy infrastructure.

### Alternative unique to this enhancement

4. **Proxy support for integrated auth only, exclude External OIDC**
   - Alternative: Document that External OIDC requires cluster-wide proxy, only implement component-scoped proxy for integrated auth
   - Rejected: External OIDC is the target authentication architecture. Not supporting component-scoped proxy forces customers to use cluster-wide proxy (defeating the security benefits) or remain on legacy integrated OAuth mode. This would block External OIDC adoption in disconnected environments where component-scoped proxy is a security requirement.

## Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

## Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

## Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

## Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

## Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.
