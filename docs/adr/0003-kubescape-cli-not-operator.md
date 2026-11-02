# ADR 0003: Kubescape as an on-demand CLI scan, not an in-cluster operator

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
Kubescape (a CNCF project) assesses a Kubernetes cluster's security posture. It can scan manifests and Helm charts in CI, run inside the cluster as an operator (with vulnerability scanning, a per-node agent and its own storage), score clusters against recognised frameworks (NSA-CISA Kubernetes hardening guidance, MITRE ATT&CK for Kubernetes, CIS), and analyse RBAC for risky permissions.

Most of this is already covered by tools in the design:

| Kubescape feature | Already covered by |
|---|---|
| Scanning manifests and Helm charts in CI | Checkov (rendered manifests, charts) and the Kyverno CLI |
| Admission-time policy | Kyverno |
| Vulnerability scanning of running images | Trivy Operator |
| CIS benchmark, node and control-plane level | kube-bench (EKS); Talos hardening notes plus Trivy Operator (home) |
| Runtime threat detection | Falco |
| Network policy suggestions | Hand-written CiliumNetworkPolicies, debugged with Hubble |
| **Framework scores (NSA-CISA, MITRE ATT&CK) and RBAC risk analysis** | **Nothing (the gap)** |

Box 1 of the home lab has 8 GB of RAM and already uses about 6–7 GB.

## Decision
- **Don't install the Kubescape operator** in either cluster.
- **Run the Kubescape CLI on demand** against each live cluster, from the workstation or the in-cluster CI runners, with nothing installed in the cluster:
  - frameworks: **NSA-CISA** and **MITRE ATT&CK**, plus the **RBAC** analysis
  - **monthly** on the home cluster, and **once per AWS demo** on EKS
  - reports saved in `wp-gitops/docs/` (e.g. `docs/security/kubescape-<site>-<date>.json` with a short Markdown summary), next to the kube-bench and Trivy reports; accepted findings explained in the summary

## Alternatives considered
- **Kubescape operator in the cluster:** continuous scanning and in-cluster reports, but several components using roughly 0.5–1 GB of memory, most of it duplicating Trivy Operator and Falco. Doesn't fit box 1.
- **Kubescape manifest scanning in CI:** largely duplicates Checkov and the Kyverno CLI; adds framework labels but little new coverage.
- **Not using Kubescape at all:** leaves no framework-based posture score or RBAC risk review.

## Consequences
- Fills the gap (framework scores, RBAC risks) at no memory or money cost.
- Scans are periodic, not continuous: a posture regression between scans is found at the next monthly scan (Kyverno, Checkov and Trivy Operator still catch most issues continuously).
- The CLI reads the cluster through the Kubernetes API only, so it **doesn't replace kube-bench**: kube-bench checks the node and control-plane configuration (kubelet flags, file permissions, API server settings) by reading the nodes directly; Kubescape checks workloads and permissions through the API. They complement each other.

## When to revisit
- More memory on the home nodes, making the operator affordable for continuous posture monitoring.
- A requirement for continuous compliance reporting (e.g. an audit).
- Trivy Operator or Falco being removed, leaving the operator's other features uncovered.

## Open items
- First home-cluster scan (home plan step 10.2): record the baseline scores and any accepted findings.
- First EKS scan (AWS plan step 2.5).
