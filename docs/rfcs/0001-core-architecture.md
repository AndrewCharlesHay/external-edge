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
* Provide a unified CRD interface for configuring Edge Routing (Custom Hostnames, SNI) and Edge Security (WAF).
* Support multiple providers, starting with Cloudflare (SaaS/WAF) and AWS (CloudFront/WAF).
* Implement the Kubernetes Gateway API `PolicyAttachment` pattern, allowing users to attach edge WAFs directly to `Gateway` or `HTTPRoute` resources.

## 4. Non-Goals
* **DNS Management:** We will not manage A, CNAME, or TXT records. That is the explicit domain of `external-dns`.
* **Cluster Ingress:** We will not act as an Ingress Controller or Gateway implementation. We configure the *external* edge proxy that sits in front of the cluster.

## 5. Proposed CRD Design

To prevent scope conflicts, the operator separates security and routing into distinct resources.

### 5.1 EdgeRoute
Manages the TLS termination and proxy behavior at the edge. This replaces complex flat-string annotations with declarative infrastructure.

```yaml
apiVersion: edge.k8s.io/v1alpha1
kind: EdgeRoute
metadata:
  name: tenant-a
  namespace: default
spec:
  provider: cloudflare
  customHostname: tenant-a.saas-platform.com
  originSni: tenant-a.internal.cluster.local
  tls:
    method: http
    minimumVersion: "1.2"
```

### 5.2 EdgeWAFPolicy
Manages the security posture of the exposed endpoints. This decouples the WAF ruleset from the routing logic, allowing a single strict policy to be applied across multiple tenants.

```yaml
apiVersion: edge.k8s.io/v1alpha1
kind: EdgeWAFPolicy
metadata:
  name: strict-api-firewall
  namespace: default
spec:
  provider: cloudflare
  zoneRef: "saas-platform.com"
  rules:
    - action: block
      expression: "(ip.src.country eq \"XX\")"
      description: "Block traffic from embargoed countries"
    - action: execute_ruleset
      rulesetRef: "cloudflare-managed-api-shield"
```

## 6. Gateway API Integration (PolicyAttachment)
To align with the future of Kubernetes networking, `EdgeWAFPolicy` will implement the `PolicyAttachment` pattern defined by the Gateway API. This allows platform teams to attach edge security directly to a `Gateway` or `HTTPRoute` resource, ensuring infrastructure-as-code stays tightly coupled to the routing definitions.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: secure-api-route
  annotations:
    edge.k8s.io/waf-policy: "strict-api-firewall"
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

## 7. Controller Architecture
The operator will be built using `kubebuilder` and `controller-runtime`.
* **Reconciler Loops:** Dedicated, independent reconcilers for `EdgeRoute` and `EdgeWAFPolicy` to ensure routing changes aren't blocked by complex WAF rule evaluations.
* **Provider Interface:** A pluggable Go interface (inspired by the `external-dns` provider pattern) to allow the community to easily contribute implementations for AWS WAF/CloudFront, Fastly, Azure Front Door, and GCP Cloud Armor.
* **State Management:** The controller will rely on Kubernetes finalizers to ensure edge configurations are cleanly deleted from the provider when the cluster resource is removed, preventing orphaned rulesets.

## 8. Open Questions / Request for Comments
We are actively seeking feedback from the community on the following design decisions:
1. **Status Reporting:** How should the operator report provider-side provisioning delays (e.g., Cloudflare DCV validation pending) back to the CRD status conditions without thrashing the Kubernetes API?
2. **Authentication:** Should the operator handle provider credentials via standard `Secret` references, or rely entirely on workload identity (e.g., AWS IAM Roles for Service Accounts) where applicable?
3. **Cross-Namespace Boundaries:** Should an `EdgeWAFPolicy` defined by platform admins in an `admin` namespace be attachable to tenant `HTTPRoute` resources in other namespaces? What are the RBAC implications?
