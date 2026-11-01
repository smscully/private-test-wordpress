# ADR 0001: No service mesh (Istio not used)

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
Service meshes such as Istio are common in larger production Kubernetes environments. They add mutual TLS (mTLS) between services, layer-7 authorization, traffic shifting, retries and timeouts, and per-service request metrics.

This project has about four services that talk to each other: WordPress, the post-summarizer, the LiteLLM gateway and the llama.cpp model server. The home lab runs on two nodes with 8 GB of RAM each, and box 1 already uses about 6–7 GB. Both clusters run Cilium, which provides network policies, Hubble flow visibility and optional transparent encryption.

## Decision
Don't run a service mesh. Cover a mesh's benefits with tools already in the design:

| What a mesh would add | How this design covers it |
|---|---|
| mTLS between services | Cilium **WireGuard** transparent encryption for pod traffic between nodes, plus TLS on every sensitive connection (WordPress → database, everything → AWS APIs, CloudFront → ALB) |
| Layer-7 authorization | **CiliumNetworkPolicies** with layer-7 rules (e.g. the summarizer may only call WordPress's REST API) |
| Traffic shifting for canaries | **Argo Rollouts** through ingress-nginx and the ALB for code; the **LiteLLM gateway's** canary split for models |
| Retries, timeouts, circuit breaking | The **SQS** queue with retries and a dead-letter queue; LiteLLM's retries and fallbacks |
| Per-service request metrics | **Hubble** HTTP metrics, ingress and ALB metrics, and the apps' own **OpenTelemetry** metrics |

## Alternatives considered
- **Istio, sidecar mode:** a proxy in every pod (roughly 50–100 MB each) plus `istiod` (~300–500 MB). Doesn't fit box 1's memory.
- **Istio, ambient mode:** no per-pod proxies; a small proxy per node plus optional layer-7 "waypoint" proxies. Much lighter, but still a few hundred MB, a second policy system overlapping Cilium's, specific settings to run alongside Cilium, and more to upgrade and debug.
- **Linkerd:** lighter than Istio, but the same overlap with Cilium and the same small benefit at this scale.
- **Cilium's own mutual authentication:** not used, as it adds little over WireGuard plus layer-7 policies for four services.

## Consequences
- Simpler operations: one networking and policy system (Cilium) on both clusters.
- Memory stays available for the actual workloads on box 1.
- Not covered: **per-connection service identity checks** (every call proving which service it comes from). Network policies restrict *who can connect* by labels, and encryption protects traffic in transit, but neither proves service identity cryptographically on each request.

## When to revisit
- Many more services calling each other (roughly ten or more), where per-service retries, timeouts and metrics become hard to manage without a mesh.
- A requirement (e.g. compliance) to verify service identity on every connection.
- Traffic between several clusters that needs consistent identity and encryption.
- More memory on the home nodes (e.g. 16 GB), making Istio ambient mode affordable.

## Open items
- **EKS WireGuard support:** confirm whether Cilium's WireGuard encryption is supported in chaining mode with the VPC CNI for the Cilium version installed (AWS plan step 1.4). If not, record it here; application-level TLS then covers encryption in transit on EKS.
- **Optional Istio ambient experiments:** if run, add the findings (memory used, latency added, setup issues with Cilium, what it added) from the local kind experiment (home plan step 6.4) and the EKS experiment (AWS plan step 5.3).
