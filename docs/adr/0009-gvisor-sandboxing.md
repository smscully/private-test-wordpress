# ADR 0009: seccomp everywhere; gVisor for CI runners (and WordPress if performance allows); Kata not used

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** seccomp on both sites; gVisor on the home cluster only

## Context
Containers on a node share its Linux kernel, so a kernel exploit from inside one container could compromise the whole node. The design already limits what containers can do (Pod Security `baseline`, non-root, read-only root filesystem, dropped capabilities, CiliumNetworkPolicies, Falco), but didn't require a **seccomp** profile and had no **sandboxed runtime**.

Two workloads run code that isn't fully trusted:
- the **ARC self-hosted CI runners**, which run code from pull requests
- **WordPress**, internet-facing PHP whose plugins are its most common attack route

Constraints: home nodes have 8 GB of RAM; Talos offers official system extensions for **gVisor** and **Kata Containers**; EKS offers neither as a managed option.

## Decision
- **Require seccomp `RuntimeDefault`** for every workload on both sites (Kyverno), with narrow exceptions only for third-party components that need a different profile.
- **gVisor on the home cluster:** add Talos's gVisor system extension to both nodes and a `gvisor` RuntimeClass.
  - **ARC runners:** run under gVisor, **enforced** by a Kyverno rule in their namespace.
  - **WordPress:** **trial** under gVisor with a before-and-after load test; keep it only if response times stay acceptable (record the numbers here).
  - Falco's **gVisor event source** turned on, so detection still covers sandboxed pods.
- **EKS:** no gVisor (custom AMIs aren't worth it for a demo cluster, and the runners only run at home); seccomp `RuntimeDefault` still applies.
- **Kata Containers:** not used.

## Alternatives considered
- **Kata Containers:** a lightweight VM per pod, the strongest isolation. Each pod's VM needs its own memory, which 8 GB nodes can't spare, and on AWS it needs hardware virtualization inside the instance (historically bare-metal instances only). **AWS Fargate** also uses microVMs, but doesn't support DaemonSets (Cilium, Falco, Alloy) or GPUs.
- **gVisor everywhere:** slows system-call-heavy workloads and isn't compatible with node agents that need host access; little benefit for trusted project code.
- **gVisor on EKS via a custom AMI or node user data:** possible, but adds AMI maintenance for a cluster that only exists during demos.
- **seccomp only:** a strong, free baseline, but doesn't isolate pull-request code from the host kernel.

## Consequences
- Every container runs with the default seccomp filter at no cost.
- Code from pull requests runs in a user-space kernel sandbox, separated from box 2's host kernel.
- gVisor adds some CPU overhead and a little memory per sandboxed pod, and some programs that rely on unusual system calls may not work under it (test before enforcing).
- Falco needs its gVisor event source configured to keep visibility inside sandboxed pods.
- The two sites differ: gVisor is home-only, documented here.

## When to revisit
- WordPress under gVisor proves too slow: run it with seccomp only, and record the load-test results.
- More memory on the home nodes, or a need for VM-level isolation: reconsider Kata.
- AWS offering a managed sandboxed runtime for EKS, or running untrusted code on EKS: reconsider gVisor there.

## Open items
- Home plan step 6.5: load-test WordPress with and without gVisor; record response times and the decision.
- Home plan step 6.5: confirm the ARC runner jobs (evaluation, staging tests) all work under gVisor.
- Home plan step 10.1: confirm Falco's gVisor event source detects a shell in a sandboxed pod.
