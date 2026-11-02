# AWS EKS Build Plan (russetleaf.com)

This is the step-by-step plan for writing and testing all the code for the **russetleaf.com AWS site** described in [`aws-wp.md`](aws-wp.md). It's the companion to [`homelab-plan.md`](homelab-plan.md), and it **starts after the home lab is built**: it reuses the home lab's repos, Terraform modules, base manifests, images, charts and CI, and adds only what's specific to AWS.

Each step ends with a **test** that must pass before moving on. Unlike the home lab, most of this stack costs money while it runs (about **$1/hour**), so the plan builds it in layers and tears each layer down between sessions until the demo workflows (Phase 6) automate it.

## How to use this plan with Claude

- **Start each session** with something like: *"Read `aws-wp.md`, `homelab-plan.md` and `aws-eks-plan.md`. We're working on AWS step 2.3. Here's where I left off: ..."*.
- **One step at a time.** Build it, run the step's test, fix, commit, and **tear down** anything expensive before ending the session.
- **Check for leftovers after every session.** Every resource is tagged (see step 0.2); search by tag (AWS Resource Explorer or the Tag Editor) to confirm nothing from `envs/aws` is still running. Check the AWS Budget the next day.
- **Change the AWS overlay only.** Put AWS-specific settings in `overlays/aws`. If the base needs changing, send the change through the home lab's environments first (local → ci → staging → russetleaf.dev), so russetleaf.dev never breaks. Application versions reach `overlays/aws` through the same `promote-to-prod` pull requests as russetleaf.dev.
- **Keep the plan current.** Tick the [progress checklist](#progress-checklist), record actual costs, and add to the [decision log](#decision-log).

## Workstation

Use the same Ubuntu 24.04 setup as the home lab ([Local workstation setup](homelab-plan.md#local-workstation-setup-ubuntu-2404)). Additions for AWS:
- An AWS SSO profile for the demo account (or the same account), and `aws eks update-kubeconfig --name <cluster> --alias eks-com --profile <profile>` after each `demo-up`, so the kubeconfig context is clearly named.
- Nothing else to install: `terraform`, `kubectl`, `helm`, `argocd` and the AWS CLI from the home lab setup cover EKS. Pin the `kubectl` version in `wp-infra`'s `.mise.toml` within one minor version of the EKS version.

## Prerequisites from the home lab

These home lab steps must be done first, because this plan builds on them:

| Home lab step | What this plan reuses |
|---|---|
| 1.1 Four repos | All four repos, branch protection, Renovate, secret scanning |
| 2.1 Terraform bootstrap | State bucket, GitHub OIDC provider, CI role pattern |
| 2.3 AWS + Cloudflare resources | The shared modules (`s3-media`, `s3-backups`, `sqs-queue`, `secrets`, `bedrock-access`) |
| 2.4 `wp-infra` CI | Terraform checks and plan comments |
| 3.1 GitOps structure | The `base/` + `overlays/` layout, app of apps, sync waves, manifest CI |
| 4.x Platform services | Base manifests for External Secrets, monitoring, Kyverno, Argo Rollouts, KEDA, Falco, Trivy Operator, OpenCost |
| 4.5 MariaDB backups | The russetleaf.dev backups used to seed russetleaf.com |
| 5.1–5.3 WordPress | The image, the chart and image promotion |
| 5.4–5.7 Environments | ci test clusters, ARC runners, on-demand staging and the promotion pipeline (which also bumps `overlays/aws`) |
| 6.1–6.2 Policy and rollouts | Kyverno policies, the Rollout definition |
| 7.x Summarizer | The summarizer image and chart, gateway config, LLM dashboards and alerts |
| 9.x MLOps pipeline | The evaluation set and gate (they stay on the home lab's runners) |

## Phase 0: Preparation

### Step 0.1: Long-lead items and guardrails
**Do:**
- **Request a GPU quota increase** for G-family instances (both on-demand and spot) in your region. New accounts usually start at 0 and approval can take days, so do this first.
- **russetleaf.com DNS:** if the domain is registered outside Route 53, plan to move its name servers to Route 53 in step 1.2.
- **Budget guardrails:** add a second AWS Budget for the demo (e.g. alert at $20 and $40 per month), and a **Cost Anomaly Detection** monitor. Send both to an **SNS topic** connected to Slack `#rl-costs` through **Amazon Q Developer in chat applications** (formerly AWS Chatbot), as well as by email. (Created by hand here; Terraform's `alerting` module takes over in step 1.7.)

**Test:** the quota request is submitted; a test budget notification arrives by email and in `#rl-costs`.

### Step 0.2: Decisions to make before writing code
Record each answer in the [decision log](#decision-log).

| Decision | Recommendation | Why it matters |
|---|---|---|
| **Region** | **us-east-1** for everything. | CloudFront certificates must be in us-east-1 anyway; one region keeps Terraform simple. G6 spot availability is good there. |
| **Pod AWS access** | **EKS Pod Identity** (the newer, simpler option) instead of IRSA, unless a component only supports IRSA. | Decides how every service account gets AWS permissions. Both avoid stored keys. |
| **Image registry** | Keep **GHCR** for both sites (one registry, one Kyverno policy). Optional: an ECR pull-through cache for GHCR, so EKS pulls stay inside AWS. If the GHCR packages are still **private** (their visibility is separate from the repos'), EKS also needs the image pull secret and Argo CD GHCR credentials described in home plan step 5.1. | One registry for both sites. |
| **NAT Gateways** | **One** NAT Gateway for demos (saves about $65/month-equivalent while running), documented as "three in production". | Cost while the demo runs vs showing full AZ redundancy. |
| **Networking** | **Decided: Cilium in chaining mode** with the Amazon VPC CNI (the VPC CNI assigns pod IPs; Cilium enforces **CiliumNetworkPolicies** and provides Hubble), matching the home lab's Cilium. | Same policies, dashboards and alerts on both clusters; staying on the VPC CNI for IP addressing keeps EKS close to its defaults. |
| **Node OS** | **Amazon Linux 2023** EKS AMIs (with the NVIDIA variant for GPU nodes). | Falco's modern eBPF driver, the GPU drivers and Karpenter all support it. |
| **Monitoring on AWS** | The same **kube-prometheus-stack** as the home lab, plus **CloudWatch** for things Prometheus can't see (ALB, RDS, ElastiCache metrics, via the CloudWatch exporter or Grafana's CloudWatch data source). Skip Container Insights for demos. | The Argo Rollouts analysis needs request metrics, which come from the ALB (CloudWatch) on AWS, not from ingress-nginx. |
| **Self-hosted model size** | Start with the **same 3B model** as the home lab, so evaluation results are comparable; try a larger one on the GPU later. | The GPU (24 GB on a g6.xlarge) can run 7–8B models easily. |
| **Tagging** | Every resource gets `Project=russetleaf`, `Site=com`, `Stack=edge|demo`, `ManagedBy=terraform`. Set as provider `default_tags`. | Makes leftover checks and cost reports possible. |

## Phase 1: Infrastructure (`wp-infra`)

### Step 1.1: CI roles for the AWS environments
**Do:** extend `wp-infra/bootstrap/` with plan and apply roles for `envs/aws-edge` and `envs/aws`, limited to what each environment manages (the demo-stack apply role can't touch the edge stack's resources, and vice versa, where IAM conditions allow).

**Test:** a pull request touching `envs/aws-edge` gets a plan comment using the plan role; the apply role can be assumed only from `main`.

### Step 1.2: Always-on edge (`envs/aws-edge`)
**Do:**
- Route 53 hosted zone for russetleaf.com (point the registrar's name servers at it if needed).
- ACM certificates: `russetleaf.com` + `www` in us-east-1 (CloudFront), and `origin.russetleaf.com` in the ALB's region (the same region if you chose us-east-1).
- S3 offline-page bucket with the offline page ("The AWS demo is offline. Visit russetleaf.dev."), the russetleaf.com **media bucket**, and a **model mirror bucket** (the self-hosted model's file, so demo startup doesn't depend on Hugging Face). Origin Access Control on all of them.
- CloudFront distribution: default behavior → `origin.russetleaf.com` (all methods, HTTPS only, the `X-Origin-Verify` secret header, Host not forwarded); `/wp-content/uploads/*` → media bucket; `/offline/*` → offline bucket; custom error responses 502/503/504 → `/offline/index.html` with status 503; 10-second error caching; origin timeout 2 s, 1 attempt.
- The secret header value in Secrets Manager.
- SES domain identity for russetleaf.com, with DKIM records in Route 53.
- Outputs for `envs/aws` (zone ID, certificate ARNs, secret ARN, bucket names).

**Test:** `curl -I https://russetleaf.com` returns **503** with the offline page; the page links to russetleaf.dev; the SES identity shows "verified". Add the russetleaf.com **external uptime check** (expects 200 or the 503 offline page) and confirm it's green and posts to `#rl-com-alerts` and `#rl-critical` when it fails (test by pointing it at a wrong URL). Leave this stack running from now on (about $1/month).

### Step 1.3: Network (`envs/aws`)
**Do:** VPC across three AZs: public and private subnets, Internet Gateway, the NAT Gateway(s) decided in step 0.2, an S3 gateway endpoint (free; keeps S3 traffic off the NAT), subnet tags for the Load Balancer Controller and Karpenter discovery.

**Test:** `terraform apply` then `destroy` cleanly; subnets carry the right tags. (Cost while up: mainly the NAT Gateway.)

### Step 1.4: EKS cluster
**Do:**
- EKS cluster with a small **managed node group** (2 nodes, e.g. t3.medium) for Karpenter, CoreDNS and other system pods.
- Add-ons: VPC CNI (with its own network policy enforcement **off**, since Cilium enforces policies), CoreDNS, kube-proxy, **EBS CSI driver**, **Pod Identity agent**.
- **Cilium in chaining mode**, installed by Terraform's Helm provider right after the cluster and managed node group exist: `cni.chainingMode: aws-cni`, `cni.exclusive: false`, `enableIPv4Masquerade: false`, `routingMode: native`; Hubble and Hubble Relay on; **WireGuard transparent encryption** on, **if** the Cilium version you install supports it in chaining mode with the VPC CNI (check Cilium's documentation; if it doesn't, leave it off and record that in the open items of `docs/adr/0001-no-service-mesh.md`, relying on the TLS already on every sensitive connection); ServiceMonitors **off** until monitoring exists (step 2.4); `lifecycle { ignore_changes }` so Argo CD manages it afterwards from `platform/cilium/overlays/aws`. Restart any pods that started before Cilium (e.g. CoreDNS) so Cilium manages them.
- A Kubernetes version within **standard support**. EKS charges about six times more per hour for clusters on versions in extended support.
- Access entries: your admin role, and the CI apply role.
- Control plane logging: audit logs only, short retention (they're the useful ones and cost little).

**Test:** `kubectl get nodes` shows the managed nodes; `cilium status` is green and `cilium connectivity test` passes; if WireGuard is on, `cilium encrypt status` lists the other nodes as peers; a gp3 PersistentVolumeClaim binds; tear down.

### Step 1.5: Karpenter infrastructure
**Do:** Karpenter's IAM (controller role via Pod Identity, node role and instance profile), the **interruption SQS queue** with EventBridge rules (spot interruption, rebalance, instance state change), and the Karpenter Helm release installed by Terraform (NodePools come from GitOps in step 2.3).

**Test:** the Karpenter controller is running and reports no errors.

### Step 1.6: Database and cache
**Do:**
- **RDS for MariaDB**, the same major version as the home lab (e.g. 11.4), Multi-AZ, private subnets, **`manage_master_user_password = true`** (RDS generates, stores and rotates the master password in Secrets Manager), security group allowing only the EKS nodes/pods, parameter group with `require_secure_transport = ON`, automated backups (7 days), encryption, `deletion_protection = false` (demo), `skip_final_snapshot = false` with a timestamped final snapshot name.
- **RDS Proxy** in front of it (engine family `MYSQL`, which covers MariaDB): `require_tls = true`; an auth entry for the `wordpress` user with **`iam_auth = REQUIRED`**, backed by a Secrets Manager secret holding a Terraform-generated password; a proxy IAM role that can read only that secret; security groups allowing EKS pods → proxy → RDS only. Optionally turn on Secrets Manager rotation for the `wordpress` secret.
- **Spike early (do this first in this step, and record the result in the decision log):** bring up a minimal RDS for MariaDB (single-AZ, smallest instance) with an RDS Proxy (`iam_auth = REQUIRED`), then from a test pod running the WordPress image, use a short PHP script to log in to the proxy with an IAM token over TLS. Costs a few cents for an hour. This decides between:
  - **Pass →** finish the `iam` mode of the `db.php` drop-in in `wp-image` (released through ci → staging → prod like any change), and continue with IAM auth as designed.
  - **Fail →** use the **fallback** from `aws-wp.md` ([Risk: confirm IAM tokens work from PHP first](aws-wp.md#risk-confirm-iam-tokens-work-from-php-first-with-a-fallback)): set the proxy's auth to `iam_auth = DISABLED`; create the `wordpress` secret with **Secrets Manager automatic rotation** using the **alternating-users** strategy (`wordpress_a` / `wordpress_b`, e.g. every 30 days); sync it into the cluster with External Secrets; install **Reloader** (or use Rollouts restarts) so WordPress restarts on a new password; set `database.auth: password` in `overlays/aws`; drop `rds-db:connect` from the WordPress and seed-Job roles and grant them read access to the secret through External Secrets instead. Update steps 1.7, 3.1 and 3.2 accordingly.
- **Restore logic:** look up the most recent `russetleaf-com-final-*` snapshot; if one exists, restore from it, otherwise create an empty database.
- **ElastiCache Valkey** replication group (primary + replica, Multi-AZ), TLS in transit, AUTH token in Secrets Manager.
- Snapshot housekeeping: keep the last 3 final snapshots (a small Lambda or a scheduled GitHub Action deletes older ones).

**Test:** the spike's result is recorded in the decision log. **IAM path:** from a test pod, generate a token (`aws rds generate-db-auth-token --hostname <proxy-endpoint> --port 3306 --username wordpress`) and log in to the **proxy** over TLS with it (the MariaDB client needs its cleartext plugin enabled to send the token); confirm a plain password and a non-TLS connection are refused. **Fallback path:** log in to the proxy with the synced password, trigger a rotation by hand (`aws secretsmanager rotate-secret`), and confirm the old password keeps working until the next rotation and WordPress picks up the new one after its restart. Either way: confirm the master password exists only in Secrets Manager; connect to Valkey with TLS and the token; destroy creates a final snapshot; the next apply restores from it (create a test table to prove it).

### Step 1.7: russetleaf.com app resources and permissions
**Do:**
- Using the shared modules: summary SQS queue + dead-letter queue, Secrets Manager entries for russetleaf.com, Bedrock access (gateway role + budget alert).
- **WAF web ACL** for the ALB (AWS managed rule groups: core rule set, known bad inputs, WordPress, IP reputation; rate limiting on `/wp-login.php`).
- **Pod Identity associations** (one IAM role per service account, least privilege): WordPress (media bucket, send to queue, SES send, **`rds-db:connect` on the proxy's `wordpress` user**), summarizer (read from queue), LiteLLM (Bedrock invoke on chosen models), External Secrets (russetleaf.com secrets), KEDA (queue attributes), ExternalDNS (the russetleaf.com zone), Load Balancer Controller, llama.cpp init container (model mirror bucket, read only), seed Job (read russetleaf.dev backup and media buckets, write russetleaf.com media bucket, `rds-db:connect` through the proxy), `db-setup` Job (read the RDS-managed master secret and the `wordpress` secret), Falcosidekick/OpenCost if they need AWS APIs.
- SES SMTP credentials (or an IAM role for an SES API plugin) in Secrets Manager, and the **Slack bot token** entry for russetleaf.com (same bot, separate entry so the sites stay independent).
- **`alerting` module:** SNS topics; CloudWatch alarms for the ALB (5xx rate, target response time), RDS (CPU, free storage, connections, replica lag) and ElastiCache (memory, evictions), created and destroyed with the demo stack; Amazon Q Developer channel configurations for `#rl-com-alerts`, `#rl-critical` and `#rl-costs` (the Slack workspace is authorised once in the AWS console, then managed in Terraform).

**Test:** with `aws iam simulate-principal-policy`, each role allows exactly what it should and nothing else; forcing a CloudWatch alarm into ALARM state (`aws cloudwatch set-alarm-state`) posts to `#rl-com-alerts`.

### Step 1.8: Argo CD on EKS
**Do:** Terraform installs Argo CD with Helm and applies the root Application pointing at `wp-gitops/bootstrap/aws`. Argo CD's UI is reached by `kubectl port-forward` only (no public endpoint for demos). **Argo CD Notifications** post to `#rl-deploys`, prefixed `[com]`.

**Test:** Argo CD lists the AWS root app (it will show missing components until Phase 2), and its first sync appears in `#rl-deploys`.

### Step 1.9: `wp-infra` CI for AWS
**Do:** extend CI to plan `envs/aws-edge` and `envs/aws` on pull requests, run **Checkov** on the source and plans of both AWS environments (with results in GitHub code scanning, as for the home environment), and run **`terraform test`** (with mocked providers where possible) for the AWS-specific modules (`alerting`, the CloudFront edge, RDS restore logic). Applies are done by the demo workflows (Phase 6) or locally, never automatically on merge, so merging never starts a paid stack.

**Test:** a pull request shows plans for all three environments, and `terraform test` passes (and fails on a deliberately broken module input).

## Phase 2: AWS platform (`wp-gitops`)

### Step 2.1: AWS root app and sync waves
**Do:** `bootstrap/aws/` root app with sync waves: Karpenter NodePools and storage first; then External Secrets, Load Balancer Controller, ExternalDNS, Kyverno; then monitoring, KEDA, Argo Rollouts, gateway, model server; then WordPress and the summarizer. Exclude home-only components (cloudflared, ingress-nginx, local-path, MariaDB, Valkey, ARC).

**Test:** Argo CD syncs the root app; the order of creation matches the waves.

### Step 2.2: AWS-only platform components
**Do:**
- `platform/ingress/overlays/aws`: AWS Load Balancer Controller (Pod Identity).
- `platform/external-dns/`: ExternalDNS for the russetleaf.com zone, `--policy=sync`, a domain filter limited to `origin.russetleaf.com`, TXT ownership records.
- `platform/storage/overlays/aws`: gp3 StorageClass as default.

**Test:** a test Ingress creates an ALB and an `origin` DNS record; deleting it removes both.

### Step 2.3: Karpenter NodePools
**Do:** `platform/karpenter/`: `EC2NodeClass` (AL2023, subnets and security groups by tag, encrypted gp3 root volume); **general NodePool** (spot + on-demand, several instance families, consolidation on, disruption budget); **GPU NodePool** (g6 family, spot preferred, label `workload=llm`, taint `dedicated=llm:NoSchedule`, consolidation off or limited so the model isn't moved needlessly). Both NodePools add the **startup taint** `node.cilium.io/agent-not-ready=true:NoExecute`, so no pod is scheduled on a new node until Cilium is running there.
- **NVIDIA device plugin** (DaemonSet) so pods can request `nvidia.com/gpu`. (The AL2023 NVIDIA AMI includes the drivers.)

**Test:** scaling a test Deployment up adds nodes, scaling down removes them; a pod requesting a GPU gets a g6 node from the GPU pool; new nodes only accept pods once Cilium is ready (the startup taint is removed); simulate a spot interruption (e.g. AWS FIS or by hand) and watch Karpenter replace the node.

### Step 2.4: Shared platform services on AWS
**Do:** add `overlays/aws` for each shared component, changing only what AWS needs:
- External Secrets: Pod Identity instead of the bootstrap key.
- Monitoring: same stack; add CloudWatch metrics (ALB, RDS, RDS Proxy, ElastiCache) per the step 0.2 decision, in place of the home lab's MariaDB and Valkey exporters. The WordPress chart's apache_exporter, custom metrics and alerts work unchanged; point the blackbox_exporter probes at WordPress's in-cluster Service. Grafana is reached by `kubectl port-forward` only (there's no Cloudflare Access on this site). Alertmanager overlay: external labels `site=com`, `cluster=eks`; receivers for `#rl-com-alerts` and `#rl-critical`, plus the shared `#rl-security`, `#rl-mlops` and `#rl-costs` routes; the `Watchdog` heartbeat goes to its **own** heartbeat check, which is paused when the demo stack is down.
- **CiliumNetworkPolicies:** the same cluster-wide baseline and per-component policies from each component's `base/` (home lab step 6.3), with AWS-specific destinations in `overlays/aws`: WordPress → the **RDS Proxy** and **ElastiCache** endpoints **by DNS name** instead of in-cluster MariaDB and Valkey; inbound to WordPress from the ALB (target-type `ip`, so traffic arrives from the VPC; allow the VPC's CIDR or the ALB's subnets); no cloudflared policy.
- **Cilium and Hubble metrics:** the same `platform/cilium/` monitoring values as home (ServiceMonitors, Hubble metric set, dashboards, alert rules), now that the Prometheus Operator exists; Hubble UI by `kubectl port-forward`.
- Kyverno: the same policy set as home; add AWS's add-on registries (the EKS add-on ECR registries and `public.ecr.aws`) to the **Restrict Image Registries** allow-list, and narrow **`PolicyException`s** for AWS-managed `kube-system` components that need privileged or host access or don't meet the label, probe, non-root or read-only rules (the VPC CNI's `aws-node`, `kube-proxy`, the EBS CSI node agent, the Pod Identity agent), plus Cilium, Falco and the NVIDIA device plugin. The project's own workloads get no exceptions.
- KEDA: Pod Identity for the SQS scaler.
- Argo Rollouts: an `AnalysisTemplate` variant using **CloudWatch** (ALB 5xx rate and target response time) instead of ingress-nginx metrics.
- Falco: modern eBPF on AL2023, same custom rules; Falcosidekick to `#rl-security`.
- Kyverno's Policy Reporter and Argo Rollouts notifications: same Slack setup as home, prefixed `[com]`.
- Trivy Operator, OpenCost (with AWS on-demand/spot pricing).

**Test:** each component is healthy; the home lab's Kyverno rejection tests (unsigned image, unlisted registry, `:latest`, missing labels or memory limit, privileged, root, writable root filesystem, no probes, NodePort/LoadBalancer Service, duplicate Ingress host) all still fail on EKS; OpenCost shows EC2 costs for the nodes.

### Step 2.5: kube-bench and Kubescape scan
**Do:**
- `platform/kube-bench/`: a Job using the **EKS CIS benchmark** profile (node and control-plane configuration), with results saved to `docs/` in `wp-gitops` (e.g. `docs/kube-bench-eks-<date>.txt`) and a short summary of any failures and why they're accepted or fixed.
- A **Kubescape CLI** scan of the EKS cluster from the workstation (NSA-CISA and MITRE ATT&CK frameworks, plus RBAC analysis; nothing installed in the cluster), saved in `docs/security/` with a summary, and compared with the home cluster's scores. Fill in the EKS open item in `docs/adr/0003-kubescape-cli-not-operator.md`.

**Test:** the Job completes and both reports are committed with summaries; any differences between the home and EKS Kubescape scores are explained (e.g. AWS-managed components).

## Phase 3: WordPress on russetleaf.com

### Step 3.1: WordPress AWS overlay
**Do:** `apps/wordpress/overlays/aws`:
- `WP_HOME`/`WP_SITEURL` = `https://russetleaf.com`; the HTTPS-behind-proxy setting.
- 3 replicas, HPA (3–10), PDB, **topology spread** across AZs.
- **`database.auth: iam`** with the **RDS Proxy endpoint**, region and the RDS CA bundle (no database password Secret at all); Valkey endpoint (TLS);
- `apps/db-setup/`: an idempotent **PreSync Job** that reads the master and `wordpress` secrets and runs `CREATE USER IF NOT EXISTS` / `GRANT` for the `wordpress` user (real work only on the first demo; the user survives in the final snapshot); russetleaf.com media bucket with the **CloudFront domain** as the media URL in the offload plugin, SQS queue URL, SES for email.
- Ingress for the ALB: `origin.russetleaf.com`, the ACM certificate, the WAF ACL, a listener rule requiring the `X-Origin-Verify` header (requests without it get 403), and a security group allowing only the CloudFront origin-facing prefix list.
- The Rollout using the CloudWatch analysis template.

**Test:** after sync, russetleaf.com switches from the offline page to the live site on its own; WordPress's custom metrics and apache_exporter are scraped, and the metrics port is **not** reachable through CloudFront or the ALB; the WordPress pods have **no database password** in their environment or Secrets; a forced Multi-AZ failover (`aws rds reboot-db-instance --force-failover`) causes only a few seconds of errors; requesting the ALB directly (without CloudFront) returns 403; media loads from CloudFront; a test SQL-injection request is blocked by WAF; the password-reset email arrives (SES sandbox: send to a verified address, or request production access).

### Step 3.2: Content
**Do:**
- `apps/seed-from-dev/`: the one-off seed Job (manual sync): download the latest russetleaf.dev backup, import into RDS **through RDS Proxy with an IAM token**, `wp search-replace 'russetleaf.dev' 'russetleaf.com' --all-tables`, `aws s3 sync` the media.
- Run it once for the first demo.

**Test:** russetleaf.com shows the same posts and images as russetleaf.dev; destroy and re-create the demo stack, and the content is still there (restored from the final snapshot).

## Phase 4: AI post summaries on EKS

### Step 4.1: Summarizer and gateway
**Do:**
- `overlays/aws` for `post-summarizer` (russetleaf.com queue, Pod Identity, gateway URL, WordPress's in-cluster Service URL) and `llm-gateway` (Pod Identity for Bedrock, same routing config as home).
- A **russetleaf.com application password** for the summarizer's WordPress user, created with `wp-cli` and stored in russetleaf.com's Secrets Manager entries. (After a seed from russetleaf.dev, the user exists but the russetleaf.dev password shouldn't be reused; revoke it on russetleaf.com and create a new one.)

**Test:** publishing a post on russetleaf.com produces a summary; the summarizer scales back to zero; LLM dashboards show russetleaf.com traffic.

### Step 4.2: Self-hosted model on the GPU pool
**Do:** `platform/llm-serving/overlays/aws`: the llama.cpp **CUDA** server image (a third-party image: add a Kyverno exception for its exact digest, or rebuild and sign it in your own CI), `nvidia.com/gpu: 1` limit, tolerations and node selector for the GPU pool, the init container reading the model from the **model mirror bucket**; add it to the gateway's routing.

**Test:** the pod lands on a spot g6 node; tokens per second are recorded (compare with box 2 in the decision log); gateway fallback to the local model works.

### Step 4.3: Evaluation on AWS (once)
**Do:** run the `llm-summarizer` evaluation script against the EKS gateway (via `kubectl port-forward`), and save the report next to the home lab's report in `llm-summarizer/evals/reports/`.

**Test:** the report compares the same models on CPU (home) and GPU (AWS), including latency and cost per summary.

## Phase 5: Security and cost review

### Step 5.1: Security review
**Do:** confirm and document: the ALB rejects non-CloudFront traffic; RDS, RDS Proxy and Valkey aren't reachable from outside the VPC; no application holds a database password (IAM tokens through the proxy), and the master password is RDS-managed and rotating; no pod has an AWS key (Pod Identity everywhere); pod traffic between nodes is encrypted (WireGuard), or the ADR explains why not; CiliumNetworkPolicies enforce (from a WordPress pod, `curl https://example.com` is blocked; `hubble observe --verdict DROPPED` shows only expected drops); Kyverno enforces; Falco alerts fire on EKS (repeat the home lab's `kubectl exec` test); Trivy Operator reports; kube-bench and Kubescape reports committed for this demo.

**Test:** a written checklist in `docs/` with each item checked.

### Step 5.2: Cost review
**Do:** after a full demo, compare **AWS Cost Explorer** (filtered by the `Project` and `Stack` tags) with the estimates in `aws-wp.md`, and with OpenCost.

**Test:** actual cost per demo hour is recorded in the decision log, and `aws-wp.md` is updated if the estimates were off.

### Step 5.3 (optional): Istio ambient experiment on EKS
**Do:** during a demo session, install **Istio ambient mode** alongside Cilium on the EKS cluster (Istio's instructions for running with Cilium), add the WordPress and summarizer namespaces to the mesh, and try mTLS plus one layer-7 authorization policy. Measure the extra memory and latency, note any conflicts with Cilium's policies, then remove it before `demo-down`. Add the findings to `docs/adr/0001-no-service-mesh.md`. Nothing in the site depends on it.

**Test:** the ADR includes the EKS findings (resource use, latency, setup issues, what it added), and Istio is fully removed afterwards.

## Phase 6: Demo operations

### Step 6.1: Demo workflows
**Do:** GitHub Actions in `wp-infra`, started by hand (`workflow_dispatch`):
- **`demo-up`:** `terraform apply` in `envs/aws`; wait for Argo CD to sync; wait until russetleaf.com returns 200; resume the russetleaf.com heartbeat check; post the start time (e.g. as a tag on the VPC) and "russetleaf.com demo started" to `#rl-com-alerts`.
- **`demo-down`:** create an Alertmanager **silence** for `site=com` and pause the russetleaf.com heartbeat check; suspend Argo CD auto-sync; delete the Applications in reverse order; wait until the ALB, the `origin` record and the Karpenter nodes are gone (with timeouts); `terraform destroy` in `envs/aws`; confirm russetleaf.com returns the 503 offline page and no tagged demo resources remain; post "russetleaf.com demo stopped" (with the run time) to `#rl-com-alerts`.
- **Safety net:** a scheduled workflow (hourly) that alerts `#rl-costs` and `#rl-critical` (and optionally runs `demo-down`) if the demo stack has been up longer than a set limit (e.g. 6 hours).

**Test:** run `demo-up` and `demo-down` end to end twice without manual steps, and confirm the teardown produces **no** alert noise in Slack (only the "stopped" message); force the safety net by lowering the limit.

### Step 6.2: Full rehearsal and demo script
**Do:** write `docs/demo-script.md`: what to show and in what order (offline page → `demo-up` → live site → publish a post and watch the summary → Grafana LLM dashboard → Argo Rollouts canary → Kyverno rejecting an unsigned image → Falco alert → OpenCost → `demo-down` → offline page). Rehearse, then record a walkthrough.

**Test:** a complete demo runs in under an hour of stack time, and the recording is linked from the hub README.

### Step 6.3: Documentation
**Do:** add the russetleaf.com architecture diagram and demo recording to the hub README; update `aws-wp.md` wherever the build differed from the design.

**Test:** the hub README explains both sites, with links to the recording, reports and dashboards.

## Ongoing operations (after the build)

| Task | How often | How |
|---|---|---|
| Run a demo (or at least `demo-up` / `demo-down`) | Monthly | Catches broken versions, expired assumptions and drift before an interview does |
| Keep the EKS version in standard support | Before each demo, if a new version is out | Bump the version in `envs/aws`; the stack is rebuilt each demo, so there's no in-place upgrade |
| Clean up old final snapshots | Automatic (step 1.6) | Check that only the last 3 remain |
| Check the always-on edge for drift | Weekly | Scheduled `terraform plan` for `envs/aws-edge` in `wp-infra` CI |
| Check for leftovers | After each demo | Search for resources tagged `Stack=demo`; the Budget and Cost Anomaly alerts in `#rl-costs` are the backstop |
| Review russetleaf.com alert noise | After each demo | Anything that fired during the demo should have been real; tune or remove alerts that weren't |
| Rotate the CloudFront-to-ALB secret header | Every few months | Change the value in `envs/aws-edge`; the next demo picks it up |
| Kubescape posture scan | Once per demo | Run the Kubescape CLI against EKS during the demo; save the report in `docs/security/` |
| Review costs | After each demo | Cost Explorer filtered by `Site=com`, compared with the demo hours recorded |

## Progress checklist

| Step | Done | Date | Stack cost this session | Notes |
|---|---|---|---|---|
| 0.1 GPU quota, DNS, guardrails | ☐ | | | |
| 0.2 Decisions | ☐ | | | |
| 1.1 CI roles | ☐ | | | |
| 1.2 Always-on edge | ☐ | | | |
| 1.3 Network | ☐ | | | |
| 1.4 EKS cluster | ☐ | | | |
| 1.5 Karpenter infrastructure | ☐ | | | |
| 1.6 RDS for MariaDB + Valkey | ☐ | | | |
| 1.7 App resources + permissions | ☐ | | | |
| 1.8 Argo CD on EKS | ☐ | | | |
| 1.9 CI for AWS environments | ☐ | | | |
| 2.1 AWS root app | ☐ | | | |
| 2.2 Load Balancer Controller, ExternalDNS, storage | ☐ | | | |
| 2.3 Karpenter NodePools + NVIDIA plugin | ☐ | | | |
| 2.4 Shared platform overlays | ☐ | | | |
| 2.5 kube-bench + Kubescape scan | ☐ | | | |
| 3.1 WordPress AWS overlay | ☐ | | | |
| 3.2 Seed + snapshot cycle | ☐ | | | |
| 4.1 Summarizer + gateway | ☐ | | | |
| 4.2 GPU model server | ☐ | | | |
| 4.3 Evaluation on AWS | ☐ | | | |
| 5.1 Security review | ☐ | | | |
| 5.2 Cost review | ☐ | | | |
| 5.3 Istio ambient experiment (optional) | ☐ | | | |
| 6.1 Demo workflows | ☐ | | | |
| 6.2 Rehearsal + recording | ☐ | | | |
| 6.3 Documentation | ☐ | | | |

## Decision log

| Date | Decision | Reason |
|---|---|---|
| | | |
