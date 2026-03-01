# RFC 0001: Core Architecture and Vision for external-edge

**Author:** Andrew Hay

**Status:** Draft

## 1. Summary
`external-edge` is a multi-cloud Kubernetes operator that manages Layer 7 edge security policies (WAF rulesets, IP blocking) and edge proxy routing configurations (Custom Hostnames, TLS termination, Origin SNI overrides) natively via Custom Resource Definitions (CRDs). 

## 2. Motivation
The Kubernetes ecosystem lacks a native, unified abstraction for managing edge network configurations. 

When a platform team provisions a new tenant dynamically (e.g., using Envoy Gateway Merged Gateways), they typically need three things:
1. A DNS record (`tenant.example.com` -> Edge IP)
2. An Edge Proxy configuration (TLS certificate provisioning, Origin SNI override to route traffic to the correct Envoy backend)
3. An Edge Security policy (WAF ruleset, Rate Limiting)

Currently, `external-dns` handles step 1 perfectly. However, attempts to manage steps 2 and 3 within `external-dns` are routinely rejected as out-of-scope (e.g., Issue [#5537](https://github.com/kubernetes-sigs/external-dns/issues/5537), PR [#6211](https://github.com/kubernetes-sigs/external-dns/pull/6211)), forcing teams to build fragmented, vendor-specific workarounds. `external-edge` exists to cleanly separate these concerns, giving Edge Proxy and Edge Security configurations a first-class home in Kubernetes.

## 3. Goals
* **Initial Implementation:** Launch with first-class support for Cloudflare (SaaS Custom Hostnames and WAF Rulesets), expanding later to AWS (CloudFront/WAF).
* Provide a unified CRD interface for configuring Edge Routing (Custom Hostnames, SNI) and Edge Security (WAF).
* Implement the Kubernetes Gateway API `PolicyAttachment` pattern, allowing users to attach edge WAFs directly to `Gateway` or `HTTPRoute` resources.

## 4. Non-Goals
* **DNS Management:** We will not manage A, CNAME, or TXT records. That is the explicit domain of `external-dns`.
* **Cluster Ingress:** We will not act as an Ingress Controller or Gateway implementation.
* **Annotation Overloading:** We will strictly watch our own CRDs and standard Gateway API objects. We will *not* scrape or consume `external-dns` annotations off standard `Ingress` objects. Separation of concerns is paramount.

## 5. Proposed CRD Design

To prevent scope conflicts, the operator separates security and routing into distinct resources.

### 5.1 EdgeRoute
Manages the TLS termination and proxy behavior at the edge. 

```yaml
apiVersion: edge.k8s.io/v1alpha1
kind: EdgeRoute
metadata:
  name: tenant-a
  namespace: tenant-a-ns
spec:
  provider: cloudflare
  customHostname: tenant-a.saas-platform.com
  originSni: tenant-a.internal.cluster.local
  tls:
    method: http
    minimumVersion: "1.2"
```

### 5.2 EdgeWAFPolicy / ClusterEdgeWAFPolicy
Manages the security posture of the exposed endpoints. To support platform teams, we will implement both namespaced (`EdgeWAFPolicy`) and cluster-scoped (`ClusterEdgeWAFPolicy`) variants so global security rules do not need to be duplicated across tenant namespaces.

```yaml
apiVersion: edge.k8s.io/v1alpha1
kind: ClusterEdgeWAFPolicy
metadata:
  name: global-strict-api-firewall
spec:
  provider: cloudflare
  rules:
    - action: block
      expression: "(ip.src.country eq \"XX\")"
      description: "Block traffic from embargoed countries"
```

## 6. Gateway API Integration & Cross-Namespace Routing
`EdgeWAFPolicy` will implement the `PolicyAttachment` pattern defined by the Gateway API. 

To allow a tenant `HTTPRoute` to attach a WAF policy managed by a platform admin in a different namespace (or a cluster-scoped policy), `external-edge` will fully support the Gateway API `ReferenceGrant` specification.



```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: secure-api-route
  namespace: tenant-a-ns
  annotations:
    edge.k8s.io/waf-policy: "global-strict-api-firewall"
    edge.k8s.io/waf-policy-namespace: "cluster"
spec:
  parentRefs:
  - name: my-envoy-gateway
  hostnames:
  - "api.saas-platform.com"
  rules:
  - backendRefs:
    - name: api-svc
      port: 8080
```

## 7. Controller Architecture & Best Practices
The operator will be built using `kubebuilder` and `controller-runtime`, adhering to the following patterns:
* **Status Reporting & Async Provisioning:** To handle pending provider states (like Cloudflare DCV certificate validation), the operator will rely on `status.conditions` coupled with exponential backoff (`ctrl.Result{RequeueAfter: ...}`). This prevents API thrashing while accurately reflecting state.
* **Authentication Hierarchy:** The primary authentication path will be Workload Identity (e.g., EKS Pod Identity, AWS IRSA) where supported by the provider SDK. For token-based APIs like Cloudflare, the operator will cleanly fall back to explicit `Secret` references defined in the provider configuration.
* **Provider Interface:** A pluggable Go interface to allow the community to easily contribute implementations for additional edge networks.
