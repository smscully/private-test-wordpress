# Production WordPress on AWS (EKS + RDS + S3 Offload)

EKS, the WordPress Helm chart, S3 media offload and RDS together make a standard production design for WordPress on AWS. The design alone doesn't make the site production-ready: that comes from how each piece is configured, as described below. The design also adds **AI-generated post summaries**, run as an event-driven, monitored and tested LLM service. Summaries go through an **LLM gateway** that routes between **Amazon Bedrock** models and a **self-hosted model** on dedicated LLM nodes.

The project runs **two separate WordPress sites** from the same code and manifests: **russetleaf.com** on the full AWS design (brought up only for demos) and **russetleaf.dev** on a budget home lab (running 24/7). See [The two sites](#the-two-sites).

Step-by-step build plans: [`homelab-plan.md`](homelab-plan.md) (russetleaf.dev, built first) and [`aws-eks-plan.md`](aws-eks-plan.md) (russetleaf.com, built second).

The project is split across **four Git repositories** (`wp-gitops`, `wp-infra`, `wp-image` and `llm-summarizer`); see [Repository structure](#repository-structure-four-repos) at the end.

## The two sites

The two sites are **fully independent**: each has its own URL, database, cache, media bucket, queues, secrets and content. Nothing is shared at runtime. What they share is how they're built: the same WordPress image (`wp-image`), the same summarizer chart (`llm-summarizer`), the same base manifests (`wp-gitops`) and the same Terraform modules (`wp-infra`). Each site's specifics (domain, endpoints, bucket names, queue URLs, secrets) live in its own overlay and Terraform environment.

| | **russetleaf.com** (AWS) | **russetleaf.dev** (home lab) |
|---|---|---|
| Purpose | Full production design, for demos | Always-on portfolio site |
| Running | Only during demos; an offline page points visitors to russetleaf.dev the rest of the time | 24/7 |
| DNS | Route 53 | Cloudflare |
| Path in | Route 53 → CloudFront (always on) → ALB + AWS WAF → EKS | Cloudflare (CDN, WAF, TLS) → Cloudflare Tunnel → cloudflared → ingress-nginx → Talos |
| TLS certificates | ACM | Cloudflare |
| Kubernetes | EKS, Karpenter general + GPU NodePools | Talos: box 1 (control plane) + box 2 (LLM worker) |
| Database | RDS for MariaDB (Multi-AZ), restored from a snapshot each demo | MariaDB in-cluster (MariaDB Operator), daily backups to S3 |
| Object cache | ElastiCache Valkey | Valkey in-cluster |
| Media | Its own S3 bucket (always on), served through CloudFront | Its own S3 bucket |
| Summary queue | Its own SQS queue + dead-letter queue | Its own SQS queue + dead-letter queue |
| Secrets | Its own Secrets Manager entries | Its own Secrets Manager entries |
| Email | Amazon SES, its own domain identity (DKIM records in Route 53) | Amazon SES, its own domain identity (DKIM records in Cloudflare) |
| Network policies | Enforced by the VPC CNI's network policy support | Enforced by Cilium (if chosen as the CNI; Talos's default Flannel doesn't enforce them) |
| AWS resource tags | `Project=russetleaf`, `Site=com` | `Project=russetleaf`, `Site=dev` |
| Self-hosted model | llama.cpp (CUDA build) on a Karpenter GPU node | llama.cpp (CPU build) on box 2 |
| Bedrock | Same models through the gateway, with its own budget alert | Same models through the gateway, with its own budget alert |
| Terraform environment (`wp-infra`) | `envs/aws-edge` (always on) + `envs/aws` (demo stack) | `envs/home` |
| GitOps overlay (`wp-gitops`) | `overlays/aws` | `overlays/home` |
| Monthly cost | ~$1 always on, plus ~$1/hour during demos | ~$7–$14 |

Running MariaDB and Valkey inside the home cluster, instead of RDS and ElastiCache, is the biggest saving in the home lab: those two managed services alone would cost about $130/month at the AWS design's sizes. Both sites use the **MariaDB** engine (in-cluster at home, RDS for MariaDB on AWS), so database backups move between them without conversion.

## Architecture overview

### russetleaf.com (AWS)

```
Visitors to russetleaf.com
  │
  ▼
Route 53 ──► CloudFront ─────────────── always on (envs/aws-edge) ───────────────
               │            │                              │
               │ default    │ /wp-content/uploads/*        │ if the origin is unreachable
               │            ▼                              ▼ (502/503/504)
               │       S3 media bucket              S3 offline page
               │                                    ("demo offline, visit russetleaf.dev")
               ▼
origin.russetleaf.com ── demo stack only (envs/aws) ──────────────────────────────
  │
  ▼
ALB + AWS WAF (TLS via ACM; accepts only CloudFront traffic)
  │
  ▼
EKS: WordPress pods (stateless, 2–3+ replicas across AZs, Argo Rollouts canaries)
  │                │                  │  publish/update event          ▲
  ▼                ▼                  ▼                                │ REST API (save summary)
RDS MariaDB     ElastiCache Valkey   Amazon SQS ──► Post-summarizer pods (KEDA: 0–N)
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

### russetleaf.dev (home lab)

```
Visitors to russetleaf.dev
  │
  ▼
Cloudflare (DNS, CDN, WAF, TLS)
  │
  ▼
Cloudflare Tunnel (outbound-only connection from home; no open ports on the router)
  │
  ▼
cloudflared pods ──► ingress-nginx
                        │
                        ▼
Talos box 1: WordPress pods (Argo Rollouts canaries)            media ──► S3 bucket (real AWS)
  │                │                │  publish/update event          ▲
  ▼                ▼                ▼                                │ REST API (save summary)
MariaDB         Valkey          Amazon SQS ──► Post-summarizer pods (KEDA: 0–N)
(in-cluster,    (in-cluster)    (real AWS)                │
 backups to S3)                                          ▼
                                              LiteLLM gateway
                                                 │                        │
                                                 ▼                        ▼
                                          Amazon Bedrock          Self-hosted model (llama.cpp, CPU)
                                          (real AWS)              on Talos box 2

            Prometheus + Grafana: same dashboards and alerts as on EKS
```

## What production-ready adds

### High availability
- **RDS for MariaDB** with **Multi-AZ** turned on. (Aurora doesn't offer MariaDB; if the site outgrew RDS, Aurora MySQL would be the next step, at the cost of switching engines.)
- At least **2–3 WordPress replicas** spread across availability zones, plus a **HorizontalPodAutoscaler** and **PodDisruptionBudgets**.
- EKS worker nodes in **multiple AZs**, provisioned by **Karpenter** (see Delivery and operations).

### Traffic and edge
- **Application Load Balancer** through the AWS Load Balancer Controller, with **TLS certificates from ACM**. Its DNS name (`origin.russetleaf.com`) is managed by **ExternalDNS**.
- **CloudFront** in front of both the site and the S3 media. In this project CloudFront is **always on** (it costs almost nothing idle), so it can show an offline page when the demo stack is down; see [russetleaf.com between demos](#russetleafcom-between-demos).
- **The ALB accepts only CloudFront traffic:** its security group allows only AWS's CloudFront origin-facing IP list, and a listener rule requires a secret header that CloudFront adds to every request.
- **AWS WAF** on the ALB. A 24/7 production site would usually put WAF on CloudFront instead; here it sits on the ALB so it exists (and costs money) only during demos.

### Performance
- **Object cache** (ElastiCache Redis or Valkey) with a Redis object-cache plugin. This cuts database load sharply.
- **PHP OPcache** (enabled in the custom WordPress image).

### Security
- S3, SQS, SES and Bedrock access through **EKS Pod Identity** (IRSA only for any component that doesn't support Pod Identity), not stored AWS access keys. Each role gets only what it needs:
  - WordPress may write to its S3 bucket and send to the queue.
  - The summarizer may read from the queue; it reaches models only through the gateway.
  - The LLM gateway may call `bedrock:InvokeModel` on the chosen Bedrock models only.
- Database password in **AWS Secrets Manager**, synced into the cluster by **External Secrets Operator**.
- **NetworkPolicies**: default-deny in each application namespace, then only the connections each workload needs (e.g. only WordPress may reach the database). On EKS, turn on network policy enforcement in the VPC CNI add-on.
- RDS in **private subnets**, with **TLS required** on connections.
- `DISALLOW_FILE_MODS` set to `true` (no plugin or theme installs from the admin screen).

### Supply chain and policy
- CI builds each image, generates an **SBOM** (a list of every package in the image) with **Syft**, scans it with **Trivy**, and **signs** it with **cosign** (keyless, using GitHub's OIDC identity).
- **Kyverno** policies in the cluster:
  - accept only images signed by the release workflows of `wp-image` and `llm-summarizer`, pulled from the project's registry
  - reject `:latest` tags
  - require CPU and memory requests and limits
  - block privileged pods outside the namespaces that need them
- The **Kyverno CLI** (`kyverno test` and `kyverno apply`) runs the same policies in `wp-gitops` CI against every manifest change, so violations fail the pull request before Argo CD ever deploys them. This covers the "policy checks before deploy" role often given to OPA, without a second policy engine.
- **Checkov** (in `wp-infra` CI) checks the Terraform for insecure settings before anything is applied.

### Runtime security and compliance
- **Falco** watches running containers (using its modern eBPF driver) and alerts on suspicious behaviour. Kyverno only checks resources when they're created; Falco covers what happens after. Custom rules for this design:
  - a **shell started** in a WordPress, summarizer or gateway pod, which normally never happens
  - **files written under `/var/www/html`**, since WordPress code is built into the image and `DISALLOW_FILE_MODS` is set
  - **unexpected outbound connections** from the LLM nodes, which should only download model files
- **Falcosidekick** sends Falco alerts to Alertmanager and Grafana. (On AWS, GuardDuty EKS Runtime Monitoring is the managed alternative; Falco runs the same on both clusters.)
- **kube-bench** audits the cluster against the **CIS Kubernetes Benchmark**, run as a Job with the EKS profile. Reports are saved to `docs/` in `wp-gitops`.
- **Trivy Operator** runs continuously in the cluster, producing CIS compliance reports, vulnerability reports for running images, and configuration audits.

### Stateless operation
- **Custom WordPress image** with plugins and themes built in, built by the `wp-image` repo's CI, pushed to the registry, and deployed by Argo CD.
- **Media uploads offloaded to S3** (e.g. the WP Offload Media plugin), so WordPress pods need no volume (`persistence.enabled: false`).
- **WP-Cron turned off** in WordPress (`DISABLE_WP_CRON`) and run as a **Kubernetes CronJob** instead.
- **Outgoing email through Amazon SES** with an SMTP plugin; each site has its own SES domain identity with DKIM.

### Backup and recovery
- Automated **RDS backups** with point-in-time recovery, plus a **final snapshot** each time the russetleaf.com demo stack is destroyed.
- Daily **logical backups** of the russetleaf.dev database to S3; see [russetleaf.dev database backups](#russetleafdev-database-backups).
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
- **Progressive delivery** with **Argo Rollouts**: new WordPress and summarizer versions go out as canaries (e.g. 10% → 50% → 100%). An analysis step checks Prometheus metrics (error rate, latency) at each stage and **rolls back automatically** if they get worse. The metrics come from **CloudWatch** on AWS (ALB 5xx rate and response time, through Argo Rollouts' CloudWatch provider) and from **ingress-nginx's Prometheus metrics** at home.
- **Metrics, logs and alerts**: Prometheus, Grafana and Alertmanager (or CloudWatch Container Insights), including the LLM metrics described below.
- **AWS Budgets alerts**, including one on Bedrock spending, in case a bug calls the model in a loop.
- **Tagging**: every AWS resource gets `Project`, `Site` and `ManagedBy` tags (plus `Stack=edge|demo` on russetleaf.com) through the AWS provider's `default_tags`, so leftovers can be found and costs split per site.

### WordPress image and Helm chart
The site uses a **custom image built on the official `wordpress` image** and a **small Helm chart of its own** (Deployment/Rollout, Service, Ingress, HPA, PDB, WP-Cron CronJob), both maintained in the `wp-image` repo and released together. The Bitnami WordPress chart isn't used: it only works with Bitnami-format images, and Bitnami cut back its free images in 2025. Owning the chart removes that dependency and keeps every setting visible.

## AI post summaries

### How it works (event-driven)
1. When a post is published or updated, a small **must-use plugin** in the custom WordPress image sends the post ID to an **SQS queue**.
2. **KEDA** watches the queue and scales the **post-summarizer** Deployment from **0 to N pods** based on how many messages are waiting. With nothing to do, no summarizer pods run.
3. The summarizer reads the post through the **WordPress REST API** and sends the text to the **LLM gateway**, which picks a model (see below). The summarizer writes the summary back as the post excerpt or a custom field.
4. Failed messages are retried and, after several attempts, moved to an **SQS dead-letter queue**, which has an alert on it.
5. A daily **CronJob sweep** re-queues any posts still missing a summary, as a backstop.

The summarizer authenticates to WordPress with an **application password** stored in **Secrets Manager** and synced by External Secrets. It reaches SQS through **EKS Pod Identity** (scoped IAM keys at home). **Bedrock model access** must be enabled for each chosen model in the region.

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
  - Model files are downloaded once by an init container from each site's **model mirror bucket** in S3 (so rebuilds don't depend on Hugging Face) into a volume on the node.
  - The model **stays loaded**, since the node is dedicated to it; KEDA scales only the summarizer.
- On AWS, the LLM nodes come from Karpenter's **GPU NodePool** (e.g. g6.xlarge on spot), using a CUDA build of llama.cpp. At home, it's a **dedicated CPU worker node** (see the home lab section). The same manifests deploy both, with per-environment overlays. On EKS, the **NVIDIA device plugin** (a DaemonSet) lets the model server request the GPU (`nvidia.com/gpu: 1`); the Amazon Linux 2023 NVIDIA AMI already includes the drivers.

### Prompt versioning and evaluation in CI
The summarizer is its own repo, `llm-summarizer`, built to stand alone as an MLOps project (see [Repository structure](#repository-structure-four-repos)).

- The summarizer prompts and model settings live in `llm-summarizer/prompts/`, with a **version number** that the summarizer records alongside each summary.
- A fixed **evaluation set** of 20–30 sample posts lives in `llm-summarizer/evals/`.
- The gateway's **routing settings** (which models run in production, fallbacks, canary weights) are deployment config, so they live in `wp-gitops`. A routing change there should link to the evaluation report that justifies it.
- On every `llm-summarizer` pull request that changes a prompt, the model list being evaluated, or the summarizer code, a GitHub Actions job:
  - generates summaries for the evaluation set **with each model behind the gateway**
  - scores them with **rule-based checks** (length, required keywords, no made-up links) and an **LLM-as-judge** call that grades how faithful each summary is to its post
  - posts the scores **per model** as a **PR comment**, comparing them with the main branch
  - **fails the check** if quality drops below a set threshold
- The job runs on **self-hosted GitHub Actions runners inside the cluster**, managed by **Actions Runner Controller (ARC)**, so it can reach the gateway and the self-hosted model. Runners are **ephemeral** (a fresh pod per job), and jobs from pull requests opened from forks require manual approval, since the repo is public.
- The same gate runs again on a **release tag**. Only if it passes does CI publish the image and Helm chart, attach the evaluation report to the GitHub release, update `MODEL_CARD.md`, and open a version-bump pull request on `wp-gitops`.
- New code rolls out through the Argo Rollouts canary; new models roll out through the gateway's canary split.

### LLM monitoring
The summarizer reports **OpenTelemetry metrics** to Prometheus, labelled by model and prompt version:
- input and output **tokens**
- **latency** per model call
- **errors**, throttling and **gateway fallbacks**
- estimated **cost per summary** and per day
- **queue depth** and age of the oldest message (from SQS)

Prometheus also collects the gateway's metrics (requests, spend and fallbacks per model) and **llama.cpp's built-in metrics** (tokens per second, requests waiting). A **Grafana dashboard** shows these alongside the WordPress metrics, the LLM node's CPU and memory, and the LLM node's cost from OpenCost. **Alertmanager** alerts on error spikes, messages in the dead-letter queue, the self-hosted model being down, and daily Bedrock spend over a set limit. The Argo Rollouts canary analysis uses the same metrics.

## Example Helm values (outline)

These are illustrative russetleaf.com (AWS overlay) values for the project's own WordPress chart; the exact keys are defined when the chart is written. The russetleaf.dev overlay differs only in the site-specific settings: its domain, the in-cluster MariaDB and Valkey hosts, ingress-nginx instead of the ALB, its own bucket, queue and email settings, scoped keys instead of Pod Identity, and the Prometheus-based canary analysis.

```yaml
image:
  repository: ghcr.io/<github-user>/wp-image
  tag: "1.0.0"                     # bumped by the wp-image release bot

site:
  url: https://russetleaf.com      # sets WP_HOME and WP_SITEURL

replicas: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
podDisruptionBudget:
  minAvailable: 2
topologySpread: zone               # spread pods across availability zones

database:
  host: <rds-endpoint>.rds.amazonaws.com
  port: 3306
  name: wordpress
  user: wordpress
  existingSecret: wordpress-db     # synced from Secrets Manager by External Secrets
  tls: true

cache:
  host: <valkey-endpoint>
  tls: true
  existingSecret: valkey-auth

media:
  bucket: russetleaf-com-media
  cdnDomain: russetleaf.com        # media served through CloudFront

summaryQueue:
  url: https://sqs.us-east-1.amazonaws.com/<account-id>/russetleaf-com-post-summaries

email:
  smtpSecret: ses-smtp             # Amazon SES SMTP credentials

serviceAccount:
  create: true                     # AWS access (S3, SQS, SES) via EKS Pod Identity; no annotation needed

resources:                         # required by the Kyverno policies
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    memory: 512Mi

wpCron:
  enabled: true                    # WP-Cron as a Kubernetes CronJob
  schedule: "*/5 * * * *"

ingress:
  className: alb
  host: origin.russetleaf.com      # CloudFront's origin; visitors use russetleaf.com
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/wafv2-acl-arn: <waf-web-acl-arn>
    external-dns.alpha.kubernetes.io/hostname: origin.russetleaf.com

rollout:
  canarySteps: [20, 50, 100]
  analysisTemplate: cloudwatch-alb # home overlay: prometheus-ingress-nginx

extraConfig: |
  define('DISALLOW_FILE_MODS', true);
  define('DISABLE_WP_CRON', true);
```

## Estimated monthly AWS costs

These are approximate prices for **us-east-1**, on demand unless noted, for a small-to-medium production site (roughly 1 TB/month of CDN traffic). Prices change over time, so confirm with the [AWS Pricing Calculator](https://calculator.aws/) before budgeting.

| Item | Assumed sizing | Est. monthly cost (USD) |
|---|---|---|
| EKS control plane | 1 cluster ($0.10/hr) | ~$73 |
| EKS worker nodes (EC2) | Karpenter general pool, ~3 × t3.medium equivalent across AZs (spot can lower this), plus a small managed node group for Karpenter | ~$91–$120 |
| EKS LLM nodes (GPU) | Karpenter GPU pool, 1 × g6.xlarge on spot (~$0.80/hr on demand, ~$590/month) | ~$200–$250 |
| EBS for worker nodes | 3 × 20 GB + 1 × 100 GB gp3 (LLM node and model files) | ~$13 |
| RDS for MariaDB | db.t4g.medium, Multi-AZ (same price as RDS for MySQL) | ~$95 |
| RDS storage | 50 GB gp3, Multi-AZ | ~$12 |
| ElastiCache Valkey | 2 × cache.t4g.small (primary + replica) | ~$37 |
| Application Load Balancer | 1 ALB + light LCU usage | ~$22 |
| NAT Gateways | 3 (one per AZ) + light data processing; demos use 1 to save cost | ~$100 |
| Public IPv4 addresses | ~6 (ALB + NAT), $0.005/hr each | ~$22 |
| CloudFront | ~1 TB out / 10M requests (free tier or flat-rate plan) | $0–$15 |
| AWS WAF | 1 web ACL + a few managed rules | ~$10 |
| S3 (media + model files) | ~60 GB + requests, versioning on | ~$2 |
| Secrets Manager | ~5 secrets | ~$2 |
| Container images | GHCR (free for public images); optional ECR pull-through cache | $0–$1 |
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

## russetleaf.com between demos

### Requirement
When the EKS demo stack isn't running, **russetleaf.com must still answer** with a simple static page, served by CloudFront, that says the AWS demo is offline and links visitors to **russetleaf.dev**. It must switch to the offline page **automatically** when the stack is destroyed, and back to the live site automatically when the stack comes up, with no manual DNS changes.

### Mechanism: an always-on CloudFront edge with error fallback (recommended)
Split the AWS Terraform into two root modules in `wp-infra`:
- **`envs/aws-edge` (always on, ~$1/month):** the Route 53 hosted zone for russetleaf.com, ACM certificates, the CloudFront distribution, the S3 bucket holding the offline page, the russetleaf.com **media bucket**, the **model mirror bucket**, and the secret header value CloudFront sends to the ALB.
- **`envs/aws` (demo stack):** everything else (VPC, EKS, Karpenter, RDS, ElastiCache, WAF, SQS, ...). The ALB itself is created by the AWS Load Balancer Controller from the WordPress Ingress, and **ExternalDNS** creates the `origin.russetleaf.com` record pointing at it.

The CloudFront distribution is set up like this:

| Setting | Value |
|---|---|
| Default behavior (`*`) | Origin `origin.russetleaf.com` (the ALB), HTTPS only, all HTTP methods |
| `/wp-content/uploads/*` | Origin: the russetleaf.com media bucket (Origin Access Control) |
| `/offline/*` | Origin: the offline-page bucket (Origin Access Control) |
| Custom error responses | **502, 503, 504 → `/offline/index.html`**, returned with status **503** |
| Error caching (minimum TTL) | **10 seconds**, so the live site returns quickly once the demo is up |
| Origin connection timeout / attempts | **2 seconds / 1 attempt**, so the offline page appears in a few seconds instead of ~30 |
| Origin custom header | `X-Origin-Verify: <secret>` (checked by the ALB listener rule) |

How the switch works:
1. **Demo stack down:** `origin.russetleaf.com` doesn't exist, so CloudFront can't reach the origin and gets a 502. The custom error response serves the offline page instead. Visitors see "The AWS demo of this site is offline. Visit russetleaf.dev to see the live site."
2. **Demo stack up:** Argo CD deploys WordPress, the Load Balancer Controller creates the ALB, and ExternalDNS creates `origin.russetleaf.com`. Within a minute or so (the DNS record's TTL plus the 10-second error cache), CloudFront reaches the origin and serves the live site.
3. **Demo stack torn down:** removing the Argo CD Applications deletes the Ingress, so the ALB is removed and ExternalDNS deletes the DNS record (run ExternalDNS with `--policy=sync` so it removes records it created). CloudFront falls back to the offline page again.

Returning **503** (rather than 200) tells search engines the site is temporarily unavailable, so they don't index the offline page as the site's content. If the live site is ever broken during a demo (the ALB returns 5xx), visitors see the same offline page rather than an error.

This also works for any HTTP method (logins and form posts get the offline page too), and needs no health checks or DNS changes outside Kubernetes.

**Alternative: Route 53 failover routing.** A primary record pointing at the demo stack with a Route 53 health check, and a secondary record pointing at the offline page. It works, but it's a worse fit here: the primary record would point at resources that are deleted after every demo, health checks add cost (about $0.50–$2/month each), and DNS caching slows the switch. The CloudFront approach needs neither.

### Keeping russetleaf.com content between demos
Each demo creates a new RDS instance, so content needs a plan:
- **Final snapshot on destroy (recommended):** set `skip_final_snapshot = false` on the RDS instance, and have the next `terraform apply` restore from the most recent snapshot. Posts, summaries and settings carry over. Snapshot storage costs a few cents a month for a site this size.
- **Media is never lost:** the russetleaf.com media bucket lives in `envs/aws-edge`, so uploads survive between demos.
- **Optional: seed from russetleaf.dev.** For the first demo, or to refresh russetleaf.com with the live site's content, run the **seed Job** (`wp-gitops/apps/seed-from-dev/`, an Argo CD Application set to manual sync, AWS only). It:
  1. downloads the latest daily backup of russetleaf.dev from its S3 backup bucket (see [russetleaf.dev database backups](#russetleafdev-database-backups))
  2. imports it into RDS for MariaDB (same engine, so no conversion is needed)
  3. runs `wp search-replace 'russetleaf.dev' 'russetleaf.com' --all-tables` to fix links and settings
  4. copies the media with `aws s3 sync` from the russetleaf.dev media bucket to the russetleaf.com media bucket

  Its IAM role (Pod Identity) may only read the russetleaf.dev backup and media buckets and write to the russetleaf.com media bucket. After it runs, russetleaf.com shows the same posts as the live site, and the next final snapshot keeps them.

### Always-on cost for russetleaf.com

| Item | Est. monthly cost (USD) |
|---|---|
| Route 53 hosted zone + queries | ~$0.50 |
| CloudFront (offline page traffic, within the free tier) | $0 |
| S3 (offline page + russetleaf.com media) | <$0.50 |
| ACM certificates | $0 |
| RDS final snapshot storage | <$0.25 |
| **Total** | **~$1 / month** |

## Budget home lab that mirrors the AWS design

The full AWS stack costs too much to run 24/7 as a portfolio project. Instead:

- Keep the **complete AWS design in Terraform on GitHub**, with CI, so employers can review it.
- Run **russetleaf.dev all the time on a two-node home Talos cluster**, published through a Cloudflare Tunnel.
- Use **real AWS services where they're cheap** (S3, SQS, Secrets Manager, IAM, Bedrock).
- Bring up **russetleaf.com's full AWS stack only for short demos**, then destroy it. Its always-on CloudFront edge shows an offline page pointing to russetleaf.dev the rest of the time.

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

### russetleaf.dev database backups
**MariaDB Operator** (not Talos) backs up the russetleaf.dev database. Talos has its own snapshot feature, but it only covers the cluster's internal state (etcd), not application data. The operator's `Backup` resource, deployed by Argo CD from `wp-gitops/platform/mariadb/`, runs a scheduled dump and uploads it to S3.

| Setting | Value |
|---|---|
| Backup type | **Logical** (`mariadb-dump`, plain SQL, gzip-compressed). It can be restored into the home MariaDB **and** into RDS for MariaDB. Physical backups (`mariadb-backup`) can't be restored into RDS. |
| Schedule | Daily at 03:00 |
| Retention | 14 days (the operator's `maxRetention`), plus an S3 lifecycle rule as a safety net |
| Bucket | `russetleaf-dev-db-backups` (example name), created in `wp-infra/envs/home` with the `s3-backups` module: versioning on, encryption on, public access blocked, objects and old versions expire after 30 days |
| Credentials | IAM key allowed only to write to and read from that bucket, kept in Secrets Manager and synced by External Secrets (or a keyless IAM role once the OIDC stretch goal is done) |
| Monitoring | The `Backup` runs as a Kubernetes CronJob, so kube-state-metrics reports its last success. Alertmanager alerts if the last successful backup is more than 26 hours old. |
| Restore test | Monthly: restore the latest backup into a scratch MariaDB instance in the cluster (the operator's `Restore` resource, or a new `MariaDB` that bootstraps from the backup), check the post count, then delete it |
| Cost | Under $1/month (daily dumps of a small site are a few MB each) |

Example `Backup` resource (outline; check field names against the operator version you install):

```yaml
apiVersion: k8s.mariadb.com/v1alpha1
kind: Backup
metadata:
  name: russetleaf-dev-daily
  namespace: wordpress
spec:
  mariaDbRef:
    name: mariadb
  schedule:
    cron: "0 3 * * *"
  maxRetention: 336h            # 14 days
  compression: gzip
  storage:
    s3:
      bucket: russetleaf-dev-db-backups
      prefix: mariadb
      endpoint: s3.us-east-1.amazonaws.com
      region: us-east-1
      accessKeyIdSecretKeyRef:
        name: mariadb-backup-s3     # synced from Secrets Manager by External Secrets
        key: access-key-id
      secretAccessKeySecretKeyRef:
        name: mariadb-backup-s3
        key: secret-access-key
      tls:
        enabled: true
```

**Same engine on both sites:** russetleaf.com uses **RDS for MariaDB**, so russetleaf.dev's backups restore into it directly, with no collation rewrites or MySQL compatibility fixes. Pin the home MariaDB to the **same major version** RDS offers (e.g. 11.4 LTS), so a dump from one always loads into the other.

**Media is backed up separately:** uploads live in russetleaf.dev's S3 media bucket, which has versioning on, so deleted or overwritten files can be recovered from older versions.

Two parts of the AWS design deliberately don't run at home:
- **Karpenter** needs a cloud provider to create nodes; the home nodes are fixed hardware.
- **kube-bench** looks for config files and processes where most Linux distributions keep them, and Talos doesn't, so many checks fail or come back "not applicable". Talos is hardened by default (no shell or SSH, read-only system, API-only management), so the home lab documents how Talos meets the CIS benchmark instead, and relies on Trivy Operator's compliance reports for continuous checks.

### Home component mapping and costs

The cost column is what russetleaf.dev costs to run 24/7, plus russetleaf.com's always-on edge.

| Component | russetleaf.dev (home, 24/7) | russetleaf.com (AWS, Terraform in GitHub) | Est. monthly cost (USD) |
|---|---|---|---|
| Kubernetes | Talos: box 1 (control plane) + box 2 (LLM worker) | EKS: general + GPU Karpenter NodePools | $0 |
| Node autoscaling | None (fixed hardware) | Karpenter | $0 |
| Ingress / edge | Cloudflare Tunnel + Cloudflare WAF (free plan) → ingress-nginx | CloudFront (always on) → ALB + AWS WAF | $0 |
| DNS | Cloudflare (russetleaf.dev) | Route 53 (russetleaf.com) + ExternalDNS | $0 |
| Offline page | n/a | S3 + CloudFront error fallback, pointing to russetleaf.dev | (in the edge row below) |
| Database | MariaDB in-cluster (MariaDB Operator) | RDS for MariaDB Multi-AZ, restored from snapshot | $0 |
| Object cache | Valkey in-cluster | ElastiCache Valkey | $0 |
| Volumes | local-path-provisioner or Longhorn | EBS | $0 |
| Media | **Real S3 bucket** (russetleaf.dev) + offload plugin | Its own S3 bucket (always on) + CloudFront | ~$1 |
| Secrets | **Real AWS Secrets Manager** + External Secrets Operator | Same | ~$2 |
| Container images | GHCR (free) | GHCR (optional ECR pull-through cache) | $0 |
| Monitoring | kube-prometheus-stack (Prometheus, Grafana, Alertmanager) + LLM metrics | Same, or Amazon Managed Prometheus | $0 |
| Database backups | Daily MariaDB Operator logical backups to **S3** (14-day retention) | RDS automated backups + final snapshot on destroy | <$1 |
| AI post summaries | **Real SQS** + KEDA-scaled summarizer | Same, in EKS | <$1 |
| LLM gateway | LiteLLM in-cluster, routing to **real Amazon Bedrock** and the self-hosted model | Same | ~$1–$2 (Bedrock tokens) |
| Self-hosted model | llama.cpp (CPU build) on box 2 | llama.cpp (CUDA build) on the Karpenter GPU NodePool | ~$2–$4 (electricity for box 2) |
| Prompt evaluation | GitHub Actions on ARC runners in the cluster | Same | (included above) |
| Progressive delivery | Argo Rollouts | Same | $0 |
| Supply chain and policy | cosign, Syft, Trivy, Kyverno CLI in CI; Kyverno in the cluster | Same | $0 |
| Network policies | Cilium (if chosen as the CNI) | VPC CNI network policy enforcement | $0 |
| Email | Amazon SES (russetleaf.dev identity) | Amazon SES (russetleaf.com identity) | <$1 |
| Runtime security | Falco + Falcosidekick | Same (or GuardDuty EKS Runtime Monitoring) | $0 |
| Compliance | Talos hardened defaults (documented) + Trivy Operator | kube-bench (EKS profile) + Trivy Operator | $0 |
| Cost visibility | OpenCost | OpenCost or Kubecost EKS bundle | $0 |
| russetleaf.com always-on edge | n/a | Route 53 zone, CloudFront, offline page, media bucket, RDS snapshot (`envs/aws-edge`) | ~$1 |
| **Total (24/7)** | | | **~$8–$15 / month** |

### Short AWS demos
The full AWS design costs roughly **$1.00/hour** while running, including about $0.25–$0.35/hour for the spot GPU node. Run `terraform apply` in `envs/aws`, record a walkthrough of russetleaf.com or take screenshots, then tear it down. The `demo-up` and `demo-down` GitHub Actions workflows in `wp-infra` do this in the right order, and a scheduled safety-net workflow alerts (or tears down) if the stack is left running too long; see [Building it from scratch](#building-it-from-scratch). The always-on edge (`envs/aws-edge`) is never destroyed, so russetleaf.com switches to its offline page automatically.

| Demo time per month | Est. cost |
|---|---|
| 5 hours | ~$5 |
| 10 hours | ~$10 |
| 20 hours | ~$20 |

Together with the home lab, the total stays within **$30–$50/month**. If the AWS account is new, its free-tier credits can cover some of this.

**Tip:** set an AWS Budget with an email alert (e.g. at $25) so a forgotten `terraform destroy` doesn't run up a bill. The GPU node is the most expensive item to leave running.

### What makes this a strong portfolio project
- **One set of GitOps manifests, two live sites.** Argo CD deploys the same Helm charts and values to russetleaf.dev and russetleaf.com, with small per-environment overrides (`overlays/home`, `overlays/aws`). Only the domain and backing services change: in-cluster MariaDB vs RDS, Valkey vs ElastiCache, Cloudflare vs CloudFront, a CPU worker vs a Karpenter GPU NodePool for the model.
- **An always-on edge for an on-demand stack:** russetleaf.com's CloudFront distribution falls back to an offline page automatically whenever the EKS stack is down, with ExternalDNS and the Load Balancer Controller handling the switch, and RDS snapshots carrying content between demos.
- **Real AWS integration from home:** S3 media offload, SQS, Secrets Manager, S3 backups, Bedrock and IAM are all managed by Terraform.
- **Generative AI run like production software:** Bedrock integration with least-privilege IAM, event-driven scaling to zero, a dead-letter queue, and cost controls, with the same code running on both clusters.
- **Model serving on Kubernetes:** a self-hosted model on dedicated nodes (labels, taints, tolerations, node selectors), behind an LLM gateway with routing, fallback, canary splits, rate limits and budgets, so no single model provider is a dependency.
- **A standalone MLOps repo (`llm-summarizer`):** versioned prompts, an evaluation set, a CI gate that compares every model and blocks changes that lower summary quality, releases gated on evaluation, a model card, a published Helm chart, and a local demo anyone can run. It's the repo to link on a CV for AI and MLOps roles.
- **A four-repo GitOps structure:** infrastructure, deployment config, the WordPress image and the summarizer are separated by purpose and permissions, with automated, reviewed promotion of signed images and charts into the GitOps repo.
- **LLM monitoring:** token, latency, error, fallback and cost metrics per model and prompt version, plus model-server throughput, in Prometheus and Grafana, with alerts.
- **Progressive delivery:** Argo Rollouts canaries with automatic rollback based on Prometheus metrics, plus gateway canaries for model changes.
- **Supply-chain security and policy as code:** SBOMs, vulnerability scans and signed images in CI, with the same Kyverno policies tested in CI and enforced in the cluster (signatures, tags, resource limits), plus Checkov on the Terraform.
- **Runtime security and compliance:** Falco rules written for this application (shells in pods, writes to WordPress code, unexpected egress from LLM nodes), alerts in Grafana, CIS benchmark reports from kube-bench on EKS, and continuous Trivy Operator compliance reports on both clusters.
- **Cost awareness (FinOps):** OpenCost shows cost per namespace and workload, including the GPU node and cost per summary, next to the Bedrock spend.
- **EKS node autoscaling with Karpenter:** general and GPU NodePools, spot with interruption handling, and consolidation, with a documented reason it doesn't apply to fixed home hardware.
- **Self-hosted CI on Kubernetes:** ephemeral GitHub Actions runners managed by ARC, with fork pull requests gated for a public repo.
- **Pod permissions without stored keys at home (stretch goal):** publish the Talos cluster's service-account OIDC discovery files (e.g. in a public S3 bucket) and register them as an IAM OIDC provider. Pods such as WordPress (S3, SQS), the summarizer (SQS) and the gateway (Bedrock) then assume IAM roles without stored keys, just like IRSA on EKS. Until then, the home cluster uses narrowly scoped IAM user keys kept in Secrets Manager.
- **Terraform for both sides, in one repo (`wp-infra`):** shared modules, plus separate root modules for russetleaf.com's always-on edge, its demo stack (including Karpenter and its NodePools), and the russetleaf.dev home lab (Talos provider `siderolabs/talos` for both nodes' machine configs, Cloudflare provider for the tunnel, DNS and WAF rules, and the AWS pieces used from home).
- **CI on GitHub in every repo:** Terraform checks and plans in `wp-infra`; manifest validation and Kyverno policy tests in `wp-gitops`; image builds with SBOMs, scans and signing in `wp-image` and `llm-summarizer`; and the prompt evaluation gate in `llm-summarizer`. Employers see all of it without the AWS stack running.
- **Documentation:** architecture diagrams for both environments, the cost breakdown, Grafana dashboard screenshots, sample evaluation reports comparing models, and demo recordings.

## Repository structure (four repos)

The project is split across four Git repositories, by **what each one controls** and **how often it changes**, not by environment. Both clusters (home Talos and AWS EKS) are driven from the same repos.

| Repo | Purpose | Publishes | Who reads or writes it | Changes when |
|---|---|---|---|---|
| **`wp-gitops`** | The **source of truth for Argo CD**: everything that runs on both clusters (WordPress, monitoring, platform services, gateway, model server, summarizer settings), with `overlays/home` and `overlays/aws`. Also the project's **hub** (README, this document, diagrams, reports). | Nothing; it's what Argo CD reads | Both Argo CD instances (read only); bots from `wp-image` and `llm-summarizer` open version-bump pull requests | Config or version changes (often) |
| **`wp-infra`** | All **Terraform**: shared modules, plus root modules for russetleaf.com (`envs/aws-edge`, always on; `envs/aws`, demo stack) and russetleaf.dev (`envs/home`) | Cloud and home infrastructure | Its own CI (the only place with cloud admin permissions), and you, locally | Infrastructure changes (rarely) |
| **`wp-image`** | The **custom WordPress image**: plugins, themes, the SQS must-use plugin | Signed image + SBOM | Its own CI | Plugin, theme or WordPress updates |
| **`llm-summarizer`** | The **standalone MLOps project**: summarizer code, prompts, evaluation set and gate, Helm chart, model card, local demo | Signed image + **Helm chart** (OCI), evaluation reports | Its own CI | Code, prompt or model-evaluation changes |

### Why four repos instead of one
- **Config is kept separate from code** (standard Argo CD practice). Argo CD watches only `wp-gitops`, so building an image never triggers a deploy by accident, and every deploy is a reviewed commit in one place.
- **Permissions stay limited by repo.** Only `wp-infra`'s CI can change cloud infrastructure. The image repos can push images and open pull requests, nothing more. Argo CD only reads.
- **Each repo changes at its own pace**, with its own history, CI and releases.
- **`llm-summarizer` stands on its own** as a CV piece for AI and MLOps roles: it can be understood, tested and deployed without the rest of the project.

### Why Terraform isn't split by environment
The home lab's Terraform creates AWS resources too (S3, SQS, Secrets Manager, Bedrock IAM), and the AWS stack needs the same ones. Two Terraform repos would mean duplicating those modules or making one repo depend on the other. One `wp-infra` repo with **shared modules** and a **separate root module and state per environment** avoids both. Destroying the AWS demo never touches the home lab's state.

### How the repos connect

```
wp-infra ─────────► cloud + home infrastructure (EKS, RDS, SQS, IAM, Talos, Cloudflare)
                                 ▲ used by everything below

wp-image ────────── build, scan, sign ──► image ──────────────┐
                                                              ├──► bot pull request ──► wp-gitops ──► Argo CD ──► Talos / EKS
llm-summarizer ─── eval gate ─► build, scan, sign ─► image + chart ┘
```

### `wp-gitops`: the source of truth for Argo CD

```
wp-gitops/
  README.md                 # project hub: architecture diagram, links to all four repos, quick start
  docs/
    aws-wp.md               # this document
    homelab-plan.md         # build plan for russetleaf.dev
    aws-eks-plan.md         # build plan for russetleaf.com
                            # + diagrams, cost breakdown, Grafana screenshots, CIS/kube-bench reports,
                            #   Talos hardening notes, demo recordings
  bootstrap/
    home/                   # root "app of apps" Application for the home cluster
    aws/                    # root "app of apps" Application for EKS
  apps/                     # the applications, each with base + overlays/home, overlays/aws
    wordpress/              # values for the wp-image Helm chart (version pinned), analysis templates
                            #   (home: Prometheus/ingress-nginx; aws: CloudWatch/ALB)
    post-summarizer/        # values for the llm-summarizer Helm chart (chart version pinned), per environment:
                            #   queue URL, gateway URL, scaling limits, IAM role
    seed-from-dev/          # one-off Job: load the latest russetleaf.dev backup + media into russetleaf.com
                            #   (aws only, manual sync)
  platform/                 # shared cluster services, each with base + overlays/home, overlays/aws
    monitoring/             # kube-prometheus-stack, Grafana dashboards, alert rules
    external-secrets/       # External Secrets Operator + AWS Secrets Manager store
    ingress/                # home: ingress-nginx; aws: AWS Load Balancer Controller
    external-dns/           # ExternalDNS, manages origin.russetleaf.com in Route 53 (aws only)
    cloudflared/            # Cloudflare Tunnel connector (home only)
    storage/                # home: local-path-provisioner or Longhorn; aws: EBS CSI StorageClass
    cilium/                 # Cilium CNI with NetworkPolicy enforcement (home only, if chosen; EKS uses the VPC CNI)
    nvidia-device-plugin/   # exposes GPUs to pods (aws only)
    mariadb/                # MariaDB Operator + database + scheduled Backup to S3 (home only; RDS on AWS)
    valkey/                 # Valkey object cache (home only; ElastiCache on AWS)
    keda/                   # KEDA operator (event-driven autoscaling)
    argo-rollouts/          # Argo Rollouts controller (canary releases)
    kyverno/                # Kyverno + policies (signed images, no :latest, resource limits,
                            #   no privileged pods) + policy tests
    falco/                  # Falco (modern eBPF) + custom rules + Falcosidekick
    trivy-operator/         # continuous CIS compliance, running-image vulnerability and config reports
    opencost/               # OpenCost (cost per namespace/workload/node)
    karpenter/              # Karpenter general and GPU NodePools, EC2NodeClass (aws only)
    kube-bench/             # CIS benchmark Job, EKS profile (aws only)
    llm-gateway/            # LiteLLM + routing config: model list, fallbacks, canary weights, budgets
    llm-serving/            # llama.cpp model server, tolerations/node selector;
                            #   home: CPU build on box 2, aws: CUDA build on the GPU NodePool
    actions-runner-controller/  # ARC + ephemeral self-hosted GitHub Actions runners
  .github/workflows/        # render every overlay (Helm/Kustomize), validate against Kubernetes schemas
                            #   (kubeconform), run Kyverno CLI policy tests
```

- **App of apps:** each cluster gets one root Argo CD Application (`bootstrap/home` or `bootstrap/aws`) that points at the overlays for that environment. Argo CD then creates an Application for every app and platform service.
- **Sync order:** use Argo CD **sync waves** so operators and CRDs come up before the things that need them: storage, External Secrets and Kyverno first, then monitoring, KEDA, Argo Rollouts and the gateway, then WordPress and the summarizer. Start Kyverno policies in **audit mode** and switch them to enforce once everything passes.
- **What's in `platform/` vs `apps/`:** `platform/` holds cluster services the applications depend on; `apps/` holds the applications themselves. Components marked "home only" exist only in the home overlays because AWS supplies a managed service in their place. Components marked "aws only" exist only in the AWS overlays because they don't apply to fixed home hardware or to Talos.
- **Which model runs in production** is decided here, in `platform/llm-gateway/`. **Whether a model is good enough** is decided by the evaluation in `llm-summarizer`. A routing change should link to the evaluation report that justifies it.

### `wp-infra`: Terraform

```
wp-infra/
  bootstrap/                # applied once, locally: S3 bucket for Terraform state, GitHub OIDC provider,
                            #   IAM roles for each repo's CI (plan, apply, image push)
  modules/                  # reusable modules shared by both environments
    s3-media/               # media bucket, versioning, IAM policy
    s3-backups/             # database backup bucket: versioning, encryption, lifecycle rules, IAM policy
    sqs-queue/              # queue + dead-letter queue + alarms
    secrets/                # Secrets Manager entries
    bedrock-access/         # model access, gateway IAM role, Bedrock budget alert
    ...                     # vpc, eks, karpenter, rds, elasticache, cloudfront, waf
  envs/
    aws-edge/               # russetleaf.com, ALWAYS ON (~$1/month): Route 53 hosted zone, ACM certificates
                            #   (us-east-1 for CloudFront; the ALB's region for origin.russetleaf.com), CloudFront
                            #   distribution with the offline-page fallback, offline-page bucket, media bucket,
                            #   model mirror bucket, SES domain identity,
                            #   CloudFront-to-ALB secret header
    aws/                    # russetleaf.com DEMO STACK: VPC, EKS (small managed node group for Karpenter),
                            #   Karpenter IAM + interruption queue, RDS (restored from the latest snapshot),
                            #   ElastiCache, WAF, SQS, Budgets, plus the shared modules; installs Argo CD and
                            #   points it at wp-gitops/bootstrap/aws; reads aws-edge outputs (zone, certificate,
                            #   secret header) from its remote state
    home/                   # russetleaf.dev: Talos machine configs (box 1 control plane; box 2 LLM worker with
                            #   label and taint), Cloudflare tunnel/DNS/WAF, plus the shared modules
                            #   (media, database-backup and model mirror buckets, SQS, Secrets Manager,
                            #   SES identity, Bedrock IAM)
  .github/workflows/        # fmt, validate, tflint, Checkov and plan on every pull request (plan posted as a
                            #   comment); apply on merge to main, with a manual approval step
```

- **State:** all three environments store state in the S3 bucket from `bootstrap/`, under separate keys, using Terraform's built-in S3 state locking (`use_lockfile = true`, Terraform 1.10+; no DynamoDB table needed).
- **CI permissions without stored keys:** GitHub Actions authenticates to AWS through **GitHub's OIDC provider**. Pull requests get a read-only *plan* role; merges to `main` get the *apply* role. The Cloudflare API token is a GitHub secret.
- **The home environment needs local network access:** the Talos provider talks to the boxes directly, which GitHub's hosted runners can't reach. Run the first `envs/home` apply from your own machine. After the cluster and ARC are up, later home applies can run on the in-cluster ARC runners.
- **Why the AWS side has two environments:** `envs/aws-edge` holds what russetleaf.com needs even when the demo is down (DNS, CloudFront, the offline page, media), and is never destroyed. `envs/aws` holds the expensive demo stack and is created and destroyed for each demo. Destroying one can never touch the other's state.
- **Tearing down the AWS demo:** delete the Argo CD Applications first (or scale the root app to nothing) so the AWS Load Balancer Controller removes the ALB, ExternalDNS removes `origin.russetleaf.com`, and Karpenter removes its nodes. Otherwise `terraform destroy` can hang on resources Terraform didn't create. `terraform destroy` then takes the final RDS snapshot that the next demo restores from.

### `wp-image`: the WordPress image

```
wp-image/
  Dockerfile                # based on the official wordpress image, plugins and themes added, OPcache on
  plugins.json              # pinned plugin and theme versions (offload media, Redis object cache, SMTP, ...)
  chart/                    # the WordPress Helm chart: Rollout, Service, Ingress, HPA, PDB, WP-Cron CronJob,
                            #   NetworkPolicy
  mu-plugins/
    summary-queue/          # sends post IDs to SQS when a post is published or updated
  tests/                    # smoke test: the container starts, plugins are active, health check passes
  .github/workflows/        # build, SBOM (Syft), scan (Trivy), sign (cosign), push image + chart to GHCR,
                            #   open a version-bump pull request on wp-gitops
```

### `llm-summarizer`: the standalone MLOps project

```
llm-summarizer/
  src/                      # SQS consumer, content source/sink adapters, OpenAI-compatible model client,
                            #   OpenTelemetry metrics
    adapters/wordpress/     # WordPress REST API adapter (the first of possibly several)
  prompts/                  # versioned prompts + model settings (the version is recorded with each summary)
  evals/
    dataset/                # 20–30 sample posts + reference notes
    scorers/                # rule-based checks + LLM-as-judge prompt
    reports/                # saved results per release, compared across models
  chart/                    # Helm chart: Rollout, KEDA ScaledObject, sweep CronJob, ServiceAccount,
                            #   PrometheusRule alerts, Grafana dashboard ConfigMap
  docker/                   # Dockerfile
  demo/                     # docker compose: sample posts + local model or Bedrock, no Kubernetes needed
  MODEL_CARD.md             # models evaluated, scores, cost per summary, known limitations
  .github/workflows/        # unit tests; evaluation gate (on ARC runners); on release: gate again, build image
                            #   and chart, SBOM, scan, sign, publish, open a chart-bump pull request on wp-gitops
```

Design choices that make it stand on its own:
- **Clear interfaces:** jobs come in from a queue (`{"post_id": ...}`), content is read and written through an **adapter** (WordPress is the first), and models are reached through any **OpenAI-compatible endpoint**. LiteLLM (and Bedrock through it) and llama.cpp both qualify, so the summarizer has no hard dependency on any one provider.
- **Its own Helm chart**, published to an OCI registry with the same version as the image. `wp-gitops` only supplies values (queue URL, gateway URL, scaling limits, IAM role). The chart also ships the summarizer's alerts and Grafana dashboard, so whoever deploys it gets monitoring too.
- **Releases gated on quality:** a release tag runs the evaluation gate first; only a passing release is published, with its evaluation report attached to the GitHub release.
- **Model card:** reviewers see which models were compared (Claude Haiku, Nova Lite and the self-hosted model), the scores, the cost per summary and known limitations, without running anything.
- **Local demo:** `docker compose up` runs it against sample posts, so anyone can try it in minutes.
- **Self-hosted runners:** the evaluation gate runs on ARC runners in the home cluster, registered to this repo, because it needs to reach the gateway and the self-hosted model. Runners are ephemeral, and jobs from fork pull requests need manual approval.

### Where each AI component lives

| AI component | Repo |
|---|---|
| Summarizer code, prompts, evaluation set and gate, model card, Helm chart, dashboard and alerts | `llm-summarizer` |
| Summarizer deployment settings per environment (queue URL, gateway URL, scaling, chart version) | `wp-gitops` |
| LiteLLM gateway and its routing config (models, fallbacks, canary weights, budgets) | `wp-gitops` |
| Self-hosted model server (llama.cpp Deployment, tolerations, home CPU / AWS CUDA overlays) | `wp-gitops` |
| Box 2's Talos label and taint; Karpenter GPU NodePool IAM | `wp-infra` |
| Bedrock model access, gateway IAM role, SQS queues, Bedrock budget alert | `wp-infra` |
| SQS must-use plugin that queues posts | `wp-image` |

The self-hosted model isn't a special case: its deployment is ordinary platform config in `wp-gitops`, on a dedicated node defined in `wp-infra`, and its quality is judged in `llm-summarizer` alongside the Bedrock models.

### Release and promotion flow
1. A change is merged in `wp-image` or `llm-summarizer` (for `llm-summarizer`, a release tag such as `v1.4.0`, after the evaluation gate passes).
2. CI builds the image (and chart), generates the SBOM, scans it, signs it with cosign, and pushes it to the registry.
3. CI opens a pull request on `wp-gitops` that changes only the version: the image tag in `apps/wordpress/`, or the chart version in `apps/post-summarizer/`. The pull request links to the release notes and, for the summarizer, the evaluation report.
4. `wp-gitops` CI renders the manifests and runs the Kyverno policy tests. You review and merge.
5. Argo CD syncs the home cluster. Argo Rollouts runs the canary and rolls back automatically if the Prometheus checks fail. The AWS overlay uses the same versions, so the next AWS demo deploys exactly what's running at home.

**Bot identity:** use a **GitHub App** (preferred) or a fine-grained personal access token that can only open pull requests on `wp-gitops`, stored as a secret in `wp-image` and `llm-summarizer`. Argo CD Image Updater is an alternative, but bot pull requests keep every deploy reviewable.

### Who can do what

| Identity | Access |
|---|---|
| `wp-infra` CI | AWS through GitHub OIDC: read-only *plan* role on pull requests, *apply* role on `main`; Cloudflare API token |
| `wp-image` CI | Push to the image registry; cosign keyless signing (GitHub OIDC); pull requests on `wp-gitops` via the bot |
| `llm-summarizer` CI | Push images and charts; cosign keyless signing; pull requests on `wp-gitops` via the bot; evaluation runs on in-cluster ARC runners, which call Bedrock through the gateway |
| Argo CD (both clusters) | Read `wp-gitops` and pull charts and images (public repos and registries need no credentials) |
| Kyverno | Admits only images signed by the `wp-image` and `llm-summarizer` release workflows (cosign keyless: GitHub OIDC issuer `https://token.actions.githubusercontent.com`, subject matching each repo's release workflow on `main` or a tag) |

### Making a change that spans repos
Work **infrastructure first, then code, then config**, so nothing is deployed before what it depends on exists. Example: the summarizer needs a new SQS queue.
1. `wp-infra`: add the queue and its IAM permission; merge and apply.
2. `llm-summarizer`: add support for it in code; release.
3. `wp-gitops`: merge the bot's chart-bump pull request and add the new queue URL to the values for each environment.

### Building it from scratch
**russetleaf.dev home lab (once):**
1. `wp-infra/bootstrap`: apply locally to create the Terraform state bucket and the GitHub OIDC roles.
2. `wp-infra/envs/home`: apply locally to create the Talos cluster on both boxes, the Cloudflare tunnel and DNS for russetleaf.dev, and the AWS resources used from home. Save the kubeconfig and talosconfig it produces somewhere safe (not in Git).
3. Install Argo CD once with Helm, then apply `wp-gitops/bootstrap/home`. From then on, Argo CD manages everything else, including itself if you add Argo CD to `platform/`.
4. Add the secrets the cluster needs to AWS Secrets Manager (database and WordPress application passwords, Cloudflare tunnel token, scoped IAM keys for home until the OIDC stretch goal is done); External Secrets syncs them in.

**russetleaf.com always-on edge (once):**
1. If russetleaf.com is registered outside Route 53, point the domain's name servers at the Route 53 hosted zone after the first apply.
2. `terraform apply` in `wp-infra/envs/aws-edge`. This creates the hosted zone, certificates, CloudFront distribution, offline page and media bucket.
3. Check that russetleaf.com shows the offline page with a link to russetleaf.dev.

**russetleaf.com demo (each time):** run the `demo-up` and `demo-down` workflows in `wp-infra`, which perform these steps:
1. `terraform apply` in `wp-infra/envs/aws`. This creates the stack (restoring RDS from the latest snapshot) and installs Argo CD pointed at `wp-gitops/bootstrap/aws`.
2. Wait for Argo CD to sync. Once ExternalDNS creates `origin.russetleaf.com`, russetleaf.com switches from the offline page to the live site on its own. Demo.
3. Remove the Argo CD Applications (the ALB and `origin.russetleaf.com` are deleted, and russetleaf.com falls back to the offline page), then `terraform destroy` in `envs/aws`. Never destroy `envs/aws-edge`.

### Moving from the current repo
The current repo (`private-test-wordpress`) becomes `wp-gitops`:
- Rename the repo on GitHub (GitHub redirects the old URL, but update the `repoURL` in Argo CD anyway).
- Move `helm/wordpress/values.yaml` to `apps/wordpress/base/values.yaml`, and update the `$values` path in the WordPress Application.
- Move `application.yaml` into the app-of-apps structure (`apps/wordpress/` plus `bootstrap/home/`).
- Move this document and the two build plans to `docs/`.
- Before making the repo public, check its history for secrets (e.g. with `gitleaks`).

**Keeping versions current:** enable **Renovate** (or Dependabot) on all four repos, for Helm chart versions, Terraform providers, base images, plugins and GitHub Actions. Each update then arrives as a pull request that goes through the same CI.
