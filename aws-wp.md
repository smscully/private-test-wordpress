# Production WordPress on AWS (EKS + RDS + S3 Offload)

EKS, the WordPress Helm chart, S3 media offload and RDS together make a standard production design for WordPress on AWS. The design alone doesn't make the site production-ready: that comes from how each piece is configured, as described below. The design also adds **AI-generated post summaries**, run as an event-driven, monitored and tested LLM service. Summaries go through an **LLM gateway** that routes between **Amazon Bedrock** models and a **self-hosted model** on dedicated LLM nodes.

## Architecture overview

```
Users
  │
  ▼
CloudFront (CDN) + AWS WAF
  │                      │
  ▼                      ▼
ALB (TLS via ACM)      S3 bucket (media uploads)
  │
  ▼
EKS: WordPress pods (stateless, 2–3+ replicas across AZs, Argo Rollouts canaries)
  │                │                  │  publish/update event          ▲
  ▼                ▼                  ▼                                │ REST API (save summary)
RDS MySQL       ElastiCache Valkey   Amazon SQS ──► Post-summarizer pods (KEDA: 0–N)
(Multi-AZ)      (object cache)                            │
                                                          ▼
                                              LiteLLM gateway (routing, fallback,
                                              canary splits, rate limits, budgets)
                                                 │                        │
                                                 ▼                        ▼
                                          Amazon Bedrock          Self-hosted model (llama.cpp)
                                       (Claude Haiku, Nova Lite)  on dedicated LLM nodes (Karpenter GPU pool)

            Prometheus + Grafana: tokens, latency, cost, errors per model and prompt version
```

## What production-ready adds

### High availability
- RDS (or Aurora) with **Multi-AZ** turned on.
- At least **2–3 WordPress replicas** spread across availability zones, plus a **HorizontalPodAutoscaler** and **PodDisruptionBudgets**.
- EKS worker nodes in **multiple AZs**, provisioned by **Karpenter** (see Delivery and operations).

### Traffic and edge
- **Application Load Balancer** through the AWS Load Balancer Controller, with **TLS certificates from ACM**.
- **CloudFront** in front of both the site and the S3 media.
- **AWS WAF** on CloudFront or the ALB.

### Performance
- **Object cache** (ElastiCache Redis or Valkey) with a Redis object-cache plugin. This cuts database load sharply.
- **PHP OPcache** (on by default in the Bitnami image).

### Security
- S3, SQS and Bedrock access through **IRSA or EKS Pod Identity**, not stored AWS access keys. Each role gets only what it needs:
  - WordPress may write to its S3 bucket and send to the queue.
  - The summarizer may read from the queue; it reaches models only through the gateway.
  - The LLM gateway may call `bedrock:InvokeModel` on the chosen Bedrock models only.
- Database password in **AWS Secrets Manager**, synced into the cluster by **External Secrets Operator**.
- RDS in **private subnets**, with **TLS required** on connections.
- `DISALLOW_FILE_MODS` set to `true` (no plugin or theme installs from the admin screen).

### Supply chain and policy
- CI builds each image, generates an **SBOM** (a list of every package in the image) with **Syft**, scans it with **Trivy**, and **signs** it with **cosign** (keyless, using GitHub's OIDC identity).
- **Kyverno** policies in the cluster:
  - accept only images signed by this repo's CI and pulled from its registry
  - reject `:latest` tags
  - require CPU and memory requests and limits
  - block privileged pods outside the namespaces that need them
- The **Kyverno CLI** (`kyverno test` and `kyverno apply`) runs the same policies in CI against every manifest change, so violations fail the pull request before Argo CD ever deploys them. This covers the "policy checks before deploy" role often given to OPA, without a second policy engine.
- **Checkov** (in CI) checks the Terraform for insecure settings before anything is applied.

### Runtime security and compliance
- **Falco** watches running containers (using its modern eBPF driver) and alerts on suspicious behaviour. Kyverno only checks resources when they're created; Falco covers what happens after. Custom rules for this design:
  - a **shell started** in a WordPress, summarizer or gateway pod, which normally never happens
  - **files written under `/var/www/html`**, since WordPress code is built into the image and `DISALLOW_FILE_MODS` is set
  - **unexpected outbound connections** from the LLM nodes, which should only download model files
- **Falcosidekick** sends Falco alerts to Alertmanager and Grafana. (On AWS, GuardDuty EKS Runtime Monitoring is the managed alternative; Falco runs the same on both clusters.)
- **kube-bench** audits the cluster against the **CIS Kubernetes Benchmark**, run as a Job with the EKS profile. Reports are saved to `docs/`.
- **Trivy Operator** runs continuously in the cluster, producing CIS compliance reports, vulnerability reports for running images, and configuration audits.

### Stateless operation
- **Custom WordPress image** with plugins and themes built in, built by CI, pushed to ECR, and deployed by Argo CD.
- **Media uploads offloaded to S3** (e.g. the WP Offload Media plugin), so WordPress pods need no volume (`persistence.enabled: false`).
- **WP-Cron turned off** in WordPress (`DISABLE_WP_CRON`) and run as a **Kubernetes CronJob** instead.
- **Outgoing email through Amazon SES** with an SMTP plugin.

### Backup and recovery
- Automated **RDS backups** with point-in-time recovery.
- **S3 versioning**, plus cross-region replication if you need disaster recovery.
- A **restore you have actually tested**.

### Delivery and operations
- **Infrastructure as code** (Terraform or CDK) for the VPC, EKS, Karpenter, RDS, S3 and SQS.
- **GitOps** (Argo CD) for what runs in the cluster.
- **Node autoscaling** with **Karpenter** instead of fixed managed node groups. A small managed node group runs Karpenter itself; Karpenter then creates EC2 nodes directly from two **NodePools**:
  - a **general pool** (spot and on-demand, several instance types) for WordPress and platform services, with **consolidation** to remove idle or underused nodes
  - a **GPU pool** for the self-hosted model, with the `workload=llm` label and `dedicated=llm:NoSchedule` taint, on spot
  - **spot interruption handling** through an SQS queue fed by EventBridge, so pods move before AWS reclaims a spot node
- **Cost visibility** with **OpenCost** (or the free Kubecost EKS bundle, which reads AWS billing data): cost per namespace, workload and node, including the GPU node's share and cost per summary alongside the Bedrock spend.
- **Progressive delivery** with **Argo Rollouts**: new WordPress and summarizer versions go out as canaries (e.g. 10% → 50% → 100%). An analysis step checks Prometheus metrics (error rate, latency) at each stage and **rolls back automatically** if they get worse.
- **Metrics, logs and alerts**: Prometheus, Grafana and Alertmanager (or CloudWatch Container Insights), including the LLM metrics described below.
- **AWS Budgets alerts**, including one on Bedrock spending, in case a bug calls the model in a loop.

### Bitnami caveat
Since Bitnami cut back its free images in 2025, many teams use the Bitnami chart with their **own image** (built from the official `wordpress` image or from Bitnami's source). Alternatively, they write a small chart of their own. A custom image fits this design anyway.

## AI post summaries

### How it works (event-driven)
1. When a post is published or updated, a small **must-use plugin** in the custom WordPress image sends the post ID to an **SQS queue**.
2. **KEDA** watches the queue and scales the **post-summarizer** Deployment from **0 to N pods** based on how many messages are waiting. With nothing to do, no summarizer pods run.
3. The summarizer reads the post through the **WordPress REST API** and sends the text to the **LLM gateway**, which picks a model (see below). The summarizer writes the summary back as the post excerpt or a custom field.
4. Failed messages are retried and, after several attempts, moved to an **SQS dead-letter queue**, which has an alert on it.
5. A daily **CronJob sweep** re-queues any posts still missing a summary, as a backstop.

The summarizer authenticates to WordPress with an **application password** stored in **Secrets Manager** and synced by External Secrets. It reaches SQS through **IRSA or Pod Identity**. **Bedrock model access** must be enabled for each chosen model in the region.

### LLM gateway and model serving
- **LiteLLM** runs in the cluster as the single entry point for all model calls. It gives the summarizer one OpenAI-compatible API, whichever model answers.
- Models behind the gateway:
  - **Amazon Bedrock: Claude Haiku** (default)
  - **Amazon Bedrock: Amazon Nova Lite** (lower-cost alternative)
  - **Self-hosted model** served by **llama.cpp** (OpenAI-compatible server), e.g. a 3B model such as Qwen2.5-3B or Llama 3.2 3B, 4-bit quantized
- The gateway handles:
  - **routing and fallback** (e.g. Bedrock first, the self-hosted model if Bedrock fails or is throttled)
  - **canary splits** for model changes (e.g. send 10% of requests to a new model, then compare quality scores and cost before switching)
  - **rate limits and spending budgets** per app
- The self-hosted model runs on **dedicated LLM nodes** (a Karpenter GPU NodePool on AWS, box 2 at home):
  - The nodes carry the label `workload=llm` and the taint `dedicated=llm:NoSchedule`, so only the model server (plus cluster-wide agents such as node-exporter) runs there.
  - The model server Deployment has a matching **toleration** and **node selector**.
  - Model files are downloaded once from S3 or Hugging Face by an init container into a volume on the node.
  - The model **stays loaded**, since the node is dedicated to it; KEDA scales only the summarizer.
- On AWS, the LLM nodes come from Karpenter's **GPU NodePool** (e.g. g6.xlarge on spot), using a CUDA build of llama.cpp. At home, it's a **dedicated CPU worker node** (see the home lab section). The same manifests deploy both, with per-environment overlays.

### Prompt versioning and evaluation in CI
- The summarizer prompt, the model choices and the gateway routing settings live in Git (`llm/`), with a **version number** that the summarizer records alongside each summary.
- A fixed **evaluation set** of 20–30 sample posts lives in `llm/evals/`.
- On every pull request that changes the prompt, a model, the routing or the summarizer code, a GitHub Actions job:
  - generates summaries for the evaluation set **with each model behind the gateway**
  - scores them with **rule-based checks** (length, required keywords, no made-up links) and an **LLM-as-judge** call that grades how faithful each summary is to its post
  - posts the scores **per model** as a **PR comment**, comparing them with the main branch
  - **fails the check** if quality drops below a set threshold
- The job runs on **self-hosted GitHub Actions runners inside the cluster**, managed by **Actions Runner Controller (ARC)**, so it can reach the gateway and the self-hosted model. Runners are **ephemeral** (a fresh pod per job), and jobs from pull requests opened from forks require manual approval, since the repo is public.
- Only prompt, model and routing versions that pass are merged and rolled out, via the Argo Rollouts canary for code and the gateway's canary split for models.

### LLM monitoring
The summarizer reports **OpenTelemetry metrics** to Prometheus, labelled by model and prompt version:
- input and output **tokens**
- **latency** per model call
- **errors**, throttling and **gateway fallbacks**
- estimated **cost per summary** and per day
- **queue depth** and age of the oldest message (from SQS)

Prometheus also collects the gateway's metrics (requests, spend and fallbacks per model) and **llama.cpp's built-in metrics** (tokens per second, requests waiting). A **Grafana dashboard** shows these alongside the WordPress metrics, the LLM node's CPU and memory, and the LLM node's cost from OpenCost. **Alertmanager** alerts on error spikes, messages in the dead-letter queue, the self-hosted model being down, and daily Bedrock spend over a set limit. The Argo Rollouts canary analysis uses the same metrics.

## Example Helm values (outline)

```yaml
image:
  registry: <account-id>.dkr.ecr.<region>.amazonaws.com
  repository: wordpress-custom
  tag: "1.0.0"

replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
pdb:
  create: true

persistence:
  enabled: false          # media lives in S3; code lives in the image

mariadb:
  enabled: false
externalDatabase:
  host: <rds-endpoint>.rds.amazonaws.com
  port: 3306
  user: wordpress
  database: wordpress
  existingSecret: wordpress-db   # synced from Secrets Manager by External Secrets

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/wordpress   # IRSA for S3 + SQS access

resources:                 # required by the Kyverno policies
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    memory: 512Mi

wordpressExtraConfigContent: |
  define('DISALLOW_FILE_MODS', true);
  define('DISABLE_WP_CRON', true);
  define('WP_REDIS_HOST', '<valkey-endpoint>');
  define('SUMMARY_QUEUE_URL', 'https://sqs.<region>.amazonaws.com/<account-id>/post-summaries');
```

## Estimated monthly AWS costs

These are approximate prices for **us-east-1**, on demand unless noted, for a small-to-medium production site (roughly 1 TB/month of CDN traffic). Prices change over time, so confirm with the [AWS Pricing Calculator](https://calculator.aws/) before budgeting.

| Item | Assumed sizing | Est. monthly cost (USD) |
|---|---|---|
| EKS control plane | 1 cluster ($0.10/hr) | ~$73 |
| EKS worker nodes (EC2) | Karpenter general pool, ~3 × t3.medium equivalent across AZs (spot can lower this), plus a small managed node group for Karpenter | ~$91–$120 |
| EKS LLM nodes (GPU) | Karpenter GPU pool, 1 × g6.xlarge on spot (~$0.80/hr on demand, ~$590/month) | ~$200–$250 |
| EBS for worker nodes | 3 × 20 GB + 1 × 100 GB gp3 (LLM node and model files) | ~$13 |
| RDS MySQL | db.t4g.medium, Multi-AZ | ~$95 |
| RDS storage | 50 GB gp3, Multi-AZ | ~$12 |
| ElastiCache Valkey | 2 × cache.t4g.small (primary + replica) | ~$37 |
| Application Load Balancer | 1 ALB + light LCU usage | ~$22 |
| NAT Gateways | 3 (one per AZ) + light data processing | ~$100 |
| Public IPv4 addresses | ~6 (ALB + NAT), $0.005/hr each | ~$22 |
| CloudFront | ~1 TB out / 10M requests (free tier or flat-rate plan) | $0–$15 |
| AWS WAF | 1 web ACL + a few managed rules | ~$10 |
| S3 (media + model files) | ~60 GB + requests, versioning on | ~$2 |
| Secrets Manager | ~5 secrets | ~$2 |
| ECR | a few GB of images | ~$1 |
| CloudWatch (logs + Container Insights) | moderate log volume | ~$20 |
| Route 53 | 1 hosted zone + queries | ~$1 |
| ACM certificates | public certificates | $0 |
| SES | low email volume ($0.10 per 1,000) | <$1 |
| Amazon SQS + EventBridge | summary queue + dead-letter queue, KEDA polling, Karpenter interruption queue (within free tiers) | <$1 |
| Amazon Bedrock | ~10–20 new or edited posts/month plus CI evaluation runs, on Haiku and Nova (~$1 per 1M input tokens, ~$5 per 1M output for Haiku; Nova less) | ~$1–$2 |
| LiteLLM, ARC runners, KEDA, Argo Rollouts, Kyverno, Falco, Trivy Operator, OpenCost, Karpenter, Prometheus/Grafana | run on the existing worker nodes | $0 |
| kube-bench | Job run on demand | $0 |
| **Total** | | **~$700–$800 / month** |

Without the self-hosted model (Bedrock only behind the gateway), the total is **~$490–$540 / month**. GuardDuty EKS Runtime Monitoring, if used instead of Falco, is billed per vCPU monitored and would add roughly $10–$20/month at this size. New AWS accounts usually have a **GPU instance quota of 0**, so request a quota increase for G-family instances (both on-demand and spot) before the first deploy.

Summarizing an existing back catalogue is a one-time cost: about **$0.40 for 100 posts** (~2,700 tokens in, ~200 tokens out each) on Bedrock. Each CI evaluation run (25 posts per model, plus judge calls) costs a few cents.

## Budget home lab that mirrors the AWS design

The full AWS stack costs too much to run 24/7 as a portfolio project. Instead:

- Keep the **complete AWS design in Terraform on GitHub**, with CI, so employers can review it.
- Run the **same application all the time on a two-node home Talos cluster**, published through a Cloudflare Tunnel.
- Use **real AWS services where they're cheap** (S3, SQS, Secrets Manager, IAM, Bedrock).
- Bring up the **full AWS stack only for short demos**, then destroy it.

### Home hardware
| Node | Hardware | Role |
|---|---|---|
| Box 1 | Intel i3, 8 GB RAM | Talos **control plane**, also runs all workloads except the self-hosted model |
| Box 2 | Intel i3, 8 GB RAM | Talos **worker**, dedicated to the self-hosted model (label `workload=llm`, taint `dedicated=llm:NoSchedule`, set in its Talos machine config) |

Box 2 has about **6.5–7 GB** free for the model. A **3B model** (4-bit) is the best fit: about 2.5–3.5 GB of RAM, and roughly 1–2 minutes per summary on an i3 (depending on its generation and core count). That's fine for a queue handling 10–20 posts a month. A 7–8B model also fits (~5–5.5 GB) with better summaries, at about 3–6 minutes each and little memory to spare.

Box 1 is the tight one: Talos, Argo CD, monitoring, WordPress and the platform services (Kyverno, KEDA, Falco, OpenCost, LiteLLM and others) come to roughly 6–7 GB of its 8 GB. To keep headroom:
- Let the **ARC runners** tolerate box 2's taint, so evaluation jobs run next to the model (they mostly wait on it anyway).
- Keep **Prometheus retention** short (e.g. 7 days) and scrape every 30–60 seconds rather than 15.
- Set requests and limits on everything (Kyverno already requires them), so a busy pod can't starve the others.

The cluster still has a single control plane, so it isn't highly available. That's acceptable for a home lab; the AWS design covers high availability.

Two parts of the AWS design deliberately don't run at home:
- **Karpenter** needs a cloud provider to create nodes; the home nodes are fixed hardware.
- **kube-bench** looks for config files and processes where most Linux distributions keep them, and Talos doesn't, so many checks fail or come back "not applicable". Talos is hardened by default (no shell or SSH, read-only system, API-only management), so the home lab documents how Talos meets the CIS benchmark instead, and relies on Trivy Operator's compliance reports for continuous checks.

### Home component mapping and costs

| Component | Home (24/7) | AWS (Terraform in GitHub) | Est. monthly cost (USD) |
|---|---|---|---|
| Kubernetes | Talos: box 1 (control plane) + box 2 (LLM worker) | EKS: general + GPU Karpenter NodePools | $0 |
| Node autoscaling | None (fixed hardware) | Karpenter | $0 |
| Ingress / edge | Cloudflare Tunnel + Cloudflare WAF (free plan) | ALB + CloudFront + AWS WAF | $0 |
| DNS | Cloudflare | Route 53 (or Cloudflare) | $0 |
| Database | MariaDB in-cluster (MariaDB Operator) | RDS MySQL Multi-AZ | $0 |
| Object cache | Valkey in-cluster | ElastiCache Valkey | $0 |
| Volumes | local-path-provisioner or Longhorn | EBS | $0 |
| Media | **Real S3 bucket** + offload plugin | S3 + CloudFront | ~$1 |
| Secrets | **Real AWS Secrets Manager** + External Secrets Operator | Same | ~$2 |
| Container images | GHCR (free) or ECR | ECR | $0–$1 |
| Monitoring | kube-prometheus-stack (Prometheus, Grafana, Alertmanager) + LLM metrics | Same, or Amazon Managed Prometheus | $0 |
| Database backups | MariaDB Operator backups to **S3** | RDS automated snapshots | ~$1 |
| AI post summaries | **Real SQS** + KEDA-scaled summarizer | Same, in EKS | <$1 |
| LLM gateway | LiteLLM in-cluster, routing to **real Amazon Bedrock** and the self-hosted model | Same | ~$1–$2 (Bedrock tokens) |
| Self-hosted model | llama.cpp (CPU build) on box 2 | llama.cpp (CUDA build) on the Karpenter GPU NodePool | ~$2–$4 (electricity for box 2) |
| Prompt evaluation | GitHub Actions on ARC runners in the cluster | Same | (included above) |
| Progressive delivery | Argo Rollouts | Same | $0 |
| Supply chain and policy | cosign, Syft, Trivy, Kyverno CLI in CI; Kyverno in the cluster | Same | $0 |
| Runtime security | Falco + Falcosidekick | Same (or GuardDuty EKS Runtime Monitoring) | $0 |
| Compliance | Talos hardened defaults (documented) + Trivy Operator | kube-bench (EKS profile) + Trivy Operator | $0 |
| Cost visibility | OpenCost | OpenCost or Kubecost EKS bundle | $0 |
| **Total (24/7)** | | | **~$7–$14 / month** |

### Short AWS demos
The full AWS design costs roughly **$1.00/hour** while running, including about $0.25–$0.35/hour for the spot GPU node. Run `terraform apply`, record a walkthrough or take screenshots, then run `terraform destroy`.

| Demo time per month | Est. cost |
|---|---|
| 5 hours | ~$5 |
| 10 hours | ~$10 |
| 20 hours | ~$20 |

Together with the home lab, the total stays within **$30–$50/month**. If the AWS account is new, its free-tier credits can cover some of this.

**Tip:** set an AWS Budget with an email alert (e.g. at $25) so a forgotten `terraform destroy` doesn't run up a bill. The GPU node is the most expensive item to leave running.

### What makes this a strong portfolio project
- **One set of GitOps manifests, two environments.** Argo CD deploys the same Helm charts and values, with small per-environment overrides (`overlays/home`, `overlays/aws`). Only the backing services change: in-cluster MariaDB vs RDS, Valkey vs ElastiCache, a CPU worker vs a Karpenter GPU NodePool for the model.
- **Real AWS integration from home:** S3 media offload, SQS, Secrets Manager, S3 backups, Bedrock and IAM are all managed by Terraform.
- **Generative AI run like production software:** Bedrock integration with least-privilege IAM, event-driven scaling to zero, a dead-letter queue, and cost controls, with the same code running on both clusters.
- **Model serving on Kubernetes:** a self-hosted model on dedicated nodes (labels, taints, tolerations, node selectors), behind an LLM gateway with routing, fallback, canary splits, rate limits and budgets, so no single model provider is a dependency.
- **MLOps practice:** versioned prompts and routing, an evaluation set, and a CI gate that compares every model and blocks changes that lower summary quality, with scores posted on each pull request.
- **LLM monitoring:** token, latency, error, fallback and cost metrics per model and prompt version, plus model-server throughput, in Prometheus and Grafana, with alerts.
- **Progressive delivery:** Argo Rollouts canaries with automatic rollback based on Prometheus metrics, plus gateway canaries for model changes.
- **Supply-chain security and policy as code:** SBOMs, vulnerability scans and signed images in CI, with the same Kyverno policies tested in CI and enforced in the cluster (signatures, tags, resource limits), plus Checkov on the Terraform.
- **Runtime security and compliance:** Falco rules written for this application (shells in pods, writes to WordPress code, unexpected egress from LLM nodes), alerts in Grafana, CIS benchmark reports from kube-bench on EKS, and continuous Trivy Operator compliance reports on both clusters.
- **Cost awareness (FinOps):** OpenCost shows cost per namespace and workload, including the GPU node and cost per summary, next to the Bedrock spend.
- **EKS node autoscaling with Karpenter:** general and GPU NodePools, spot with interruption handling, and consolidation, with a documented reason it doesn't apply to fixed home hardware.
- **Self-hosted CI on Kubernetes:** ephemeral GitHub Actions runners managed by ARC, with fork pull requests gated for a public repo.
- **Pod permissions without stored keys at home (stretch goal):** publish the Talos cluster's service-account OIDC discovery files (e.g. in a public S3 bucket) and register them as an IAM OIDC provider. Pods such as WordPress (S3, SQS), the summarizer (SQS) and the gateway (Bedrock) then assume IAM roles without stored keys, just like IRSA on EKS. Until then, the home cluster uses narrowly scoped IAM user keys kept in Secrets Manager.
- **Terraform for both sides:** AWS provider for the cloud stack (including Karpenter and its NodePools); Cloudflare provider (tunnel, DNS, WAF rules), Talos provider (`siderolabs/talos`, both nodes' machine configs), and Helm and Kubernetes providers for the home lab.
- **CI on GitHub:** `terraform fmt`, `validate`, `plan`, tflint and Checkov on every pull request; image builds with SBOMs, scans and signing; and the prompt evaluation gate. Employers see all of it without the AWS stack running.
- **Documentation:** architecture diagrams for both environments, the cost breakdown, Grafana dashboard screenshots, sample evaluation reports comparing models, and demo recordings.

### Suggested repo layout
```
terraform/
  aws/                      # VPC, EKS (small managed node group for Karpenter), Karpenter IAM + interruption
                            #   queue, RDS, ElastiCache, S3, SQS, CloudFront, WAF, IAM (incl. Bedrock role), Budgets
  home/                     # Talos machine configs (box 1 control plane, box 2 LLM worker with label/taint),
                            #   Cloudflare tunnel/DNS/WAF, S3 + SQS + IAM (incl. Bedrock access), Secrets Manager
gitops/
  bootstrap/                # Argo CD root "app of apps" per environment (home, aws)
  apps/
    wordpress/              # Helm values, Argo Rollouts canary + analysis; base + overlays/home, overlays/aws
    post-summarizer/        # Rollout, KEDA ScaledObject (SQS), sweep CronJob, ServiceAccount; base + overlays
  platform/                 # shared cluster services, each with base + overlays/home, overlays/aws
    monitoring/             # kube-prometheus-stack, Grafana dashboards (incl. LLM), alert rules
    external-secrets/       # External Secrets Operator + AWS Secrets Manager store
    ingress/                # home: ingress-nginx; aws: AWS Load Balancer Controller
    cloudflared/            # Cloudflare Tunnel connector (home only)
    storage/                # home: local-path-provisioner or Longhorn; aws: EBS CSI StorageClass
    mariadb/                # MariaDB Operator + database (home only; RDS on AWS)
    valkey/                 # Valkey object cache (home only; ElastiCache on AWS)
    keda/                   # KEDA operator (event-driven autoscaling)
    argo-rollouts/          # Argo Rollouts controller (canary releases)
    kyverno/                # Kyverno + policies (signed images, no :latest, resource limits, no privileged pods)
                            #   + policy tests run by the Kyverno CLI in CI
    falco/                  # Falco (modern eBPF) + custom rules + Falcosidekick
    trivy-operator/         # continuous CIS compliance, running-image vulnerability and config reports
    opencost/               # OpenCost (cost per namespace/workload/node)
    karpenter/              # Karpenter + general and GPU NodePools, EC2NodeClass (aws only)
    kube-bench/             # CIS benchmark Job, EKS profile (aws only)
    llm-gateway/            # LiteLLM gateway (routes to Bedrock models + self-hosted model)
    llm-serving/            # llama.cpp model server, tolerations/node selector; home: CPU build, aws: CUDA build
    actions-runner-controller/  # ARC + ephemeral self-hosted GitHub Actions runners
llm/
  prompts/                  # versioned summarizer prompts + model settings
  routing/                  # LiteLLM model list, fallbacks, canary weights, budgets
  evals/                    # evaluation set (sample posts), scoring rules, judge prompt, eval script
docker/
  wordpress/                # custom image (plugins, themes, and the SQS must-use plugin built in)
  post-summarizer/          # summarizer image (SQS consumer; calls WordPress REST API + LLM gateway; OpenTelemetry metrics)
docs/                       # architecture diagrams, cost breakdown, dashboards, eval reports, CIS/kube-bench
                            #   reports, Talos hardening notes, demo recordings
.github/workflows/          # Terraform checks, Kyverno CLI policy tests, image build + SBOM + scan + sign,
                            #   prompt evaluation gate (on ARC runners)
```

Prometheus lives under `gitops/platform/monitoring/`. Everything under `platform/` is a cluster service that WordPress depends on but that isn't the application itself. Components marked "home only" exist only in the home overlays, because AWS supplies a managed service in their place. Components marked "aws only" exist only in the AWS overlays, because they don't apply to fixed home hardware or to Talos.
