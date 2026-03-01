# External Edge

**Status: RFC (Request for Comments) Phase**

`external-edge` is a proposed Kubernetes operator designed to manage Edge Security (WAF, Rate Limiting) and Edge Routing (Custom Hostnames, Origin SNI) across multiple cloud providers directly from your cluster.

## The Problem
While excellent tools like `external-dns` handle the provisioning of raw DNS records, they explicitly avoid managing Layer 7 edge proxy configurations. This leaves a massive gap in the Kubernetes ecosystem. If you want to dynamically provision a Cloudflare Custom Hostname with an Origin SNI override for a new multi-tenant ingress, or attach an AWS WAF WebACL to a Gateway, you are forced to step outside of Kubernetes and use CI/CD pipelines, Terraform, or bespoke scripts.

## The Solution
Inspired by the abstraction models of `external-dns` and `external-secrets`, `external-edge` aims to provide a unified, vendor-agnostic set of Custom Resource Definitions (CRDs) for configuring the edge. 

It acts as the missing actuation layer for the edge, sitting perfectly alongside your DNS controllers and the modern Gateway API.

## Get Involved
We are currently gathering feedback from the community, platform engineers, and SIG-Network members on the core architecture and CRD design. 

Please read the initial proposal and leave your thoughts in the issues or discussions:
👉 **[RFC 0001: Core Architecture and Vision](docs/rfcs/0001-core-architecture.md)**
