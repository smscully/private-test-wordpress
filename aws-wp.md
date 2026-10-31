# Production WordPress on AWS (EKS + RDS + S3 Offload)

EKS, the WordPress Helm chart, S3 media offload and RDS together make a standard production design for WordPress on AWS. The design alone doesn't make the site production-ready: that comes from how each piece is configured, as described below. The design also adds **AI-generated post summaries** using Amazon Bedrock.

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
EKS: WordPress pods (stateless, 2–3+ replicas across AZs) ◄── REST API ── Post-summarizer CronJob
  │                │                                                         │
  ▼                ▼                                                         ▼
RDS MySQL       ElastiCache Valkey                                    Amazon Bedrock
(Multi-AZ)      (object cache)                                      (Claude Haiku / Nova)
```

## What production-ready adds

### High availability
- RDS (or Aurora) with **Multi-AZ** turned on.
- At least **2–3 WordPress replicas** spread across availability zones, plus a **HorizontalPodAutoscaler** and **PodDisruptionBudgets**.
- EKS worker nodes in **multiple AZs**.

### Traffic and edge
- **Application Load Balancer** through the AWS Load Balancer Controller, with **TLS certificates from ACM**.
- **CloudFront** in front of both the site and the S3 media.
- **AWS WAF** on CloudFront or the ALB.

### Performance
- **Object cache** (ElastiCache Redis or Valkey) with a Redis object-cache plugin. This cuts database load sharply.
- **PHP OPcache** (on by default in the Bitnami image).

### Security
- S3 and Bedrock access through **IRSA or EKS Pod Identity**, not stored AWS access keys. The summarizer's role allows only `bedrock:InvokeModel` on the one chosen model.
- Database password in **AWS Secrets Manager**, synced into the cluster by **External Secrets Operator**.
- RDS in **private subnets**, with **TLS required** on connections.
- `DISALLOW_FILE_MODS` set to `true` (no plugin or theme installs from the admin screen).
- **Image scanning** (e.g. ECR scanning).

### Stateless operation
- **Custom WordPress image** with plugins and themes built in, built by CI, pushed to ECR, and deployed by Argo CD.
- **Media uploads offloaded to S3** (e.g. the WP Offload Media plugin), so WordPress pods need no volume (`persistence.enabled: false`).
- **WP-Cron turned off** in WordPress (`DISABLE_WP_CRON`) and run as a **Kubernetes CronJob** instead.
- **Outgoing email through Amazon SES** with an SMTP plugin.

### Backup and recovery
- Automated **RDS backups** with point-in-time recovery.
- **S3 versioning**, plus cross-region replication if you need disaster recovery.
- A **restore you have actually tested**.

### Operations
- **Metrics, logs and alerts**: CloudWatch Container Insights, or Prometheus and Grafana.
- **Infrastructure as code** (Terraform or CDK) for the VPC, EKS, RDS and S3.
- **GitOps** (Argo CD) for what runs in the cluster.
- **AWS Budgets alerts**, including one on Bedrock spending, in case a bug calls the model in a loop.

### AI post summaries (Amazon Bedrock)
- A **post-summarizer** service runs as a **Kubernetes CronJob** (e.g. every 15 minutes). It asks the WordPress REST API for posts published or edited since its last run.
- For each post, it sends the text to a small, low-cost Bedrock model (e.g. **Claude Haiku** or **Amazon Nova Lite**) and asks for a short summary.
- It writes the summary back through the REST API, as the post excerpt or a custom field, and the theme shows it on the post and in listings.
- It authenticates to WordPress with an **application password** stored in **Secrets Manager** and synced by External Secrets.
- It reaches Bedrock through **IRSA or Pod Identity**, with **Bedrock model access enabled** for that model in the region.
- A CronJob runs the same way on both clusters. For an event-driven version on AWS, a publish webhook can send to **SQS**, with the summarizer (or a Lambda) reading from the queue.

### Bitnami caveat
Since Bitnami cut back its free images in 2025, many teams use the Bitnami chart with their **own image** (built from the official `wordpress` image or from Bitnami's source). Alternatively, they write a small chart of their own. A custom image fits this design anyway.

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
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/wordpress-s3   # IRSA for S3 access

wordpressExtraConfigContent: |
  define('DISALLOW_FILE_MODS', true);
  define('DISABLE_WP_CRON', true);
  define('WP_REDIS_HOST', '<valkey-endpoint>');
```

## Estimated monthly AWS costs

These are approximate prices for **us-east-1**, on demand, for a small-to-medium production site (roughly 1 TB/month of CDN traffic). Prices change over time, so confirm with the [AWS Pricing Calculator](https://calculator.aws/) before budgeting.

| Item | Assumed sizing | Est. monthly cost (USD) |
|---|---|---|
| EKS control plane | 1 cluster ($0.10/hr) | ~$73 |
| EKS worker nodes (EC2) | 3 × t3.medium, one per AZ | ~$91 |
| EBS for worker nodes | 3 × 20 GB gp3 | ~$5 |
| RDS MySQL | db.t4g.medium, Multi-AZ | ~$95 |
| RDS storage | 50 GB gp3, Multi-AZ | ~$12 |
| ElastiCache Valkey | 2 × cache.t4g.small (primary + replica) | ~$37 |
| Application Load Balancer | 1 ALB + light LCU usage | ~$22 |
| NAT Gateways | 3 (one per AZ) + light data processing | ~$100 |
| Public IPv4 addresses | ~6 (ALB + NAT), $0.005/hr each | ~$22 |
| CloudFront | ~1 TB out / 10M requests (free tier or flat-rate plan) | $0–$15 |
| AWS WAF | 1 web ACL + a few managed rules | ~$10 |
| S3 (media) | ~50 GB + requests, versioning on | ~$2 |
| Secrets Manager | ~3 secrets | ~$2 |
| ECR | a few GB of images | ~$1 |
| CloudWatch (logs + Container Insights) | moderate log volume | ~$20 |
| Route 53 | 1 hosted zone + queries | ~$1 |
| ACM certificates | public certificates | $0 |
| SES | low email volume ($0.10 per 1,000) | <$1 |
| Amazon Bedrock (post summaries) | ~10–20 new or edited posts/month on a Haiku-class model (~$1 per 1M input tokens, ~$5 per 1M output) | <$1 |
| **Total** | | **~$490–$510 / month** |

Summarizing an existing back catalogue is a one-time cost: about **$0.40 for 100 posts** (~2,700 tokens in, ~200 tokens out each). Amazon Nova models cost less than this.

## Budget home lab that mirrors the AWS design

The full AWS stack costs too much to run 24/7 as a portfolio project. Instead:

- Keep the **complete AWS design in Terraform on GitHub**, with CI, so employers can review it.
- Run the **same application all the time on a home Talos cluster**, published through a Cloudflare Tunnel.
- Use **real AWS services where they're cheap** (S3, Secrets Manager, IAM, Bedrock).
- Bring up the **full AWS stack only for short demos**, then destroy it.

### Home component mapping and costs

| Component | Home (24/7) | AWS (Terraform in GitHub) | Est. monthly cost (USD) |
|---|---|---|---|
| Kubernetes | Talos (home server) | EKS | $0 |
| Ingress / edge | Cloudflare Tunnel + Cloudflare WAF (free plan) | ALB + CloudFront + AWS WAF | $0 |
| DNS | Cloudflare | Route 53 (or Cloudflare) | $0 |
| Database | MariaDB in-cluster (MariaDB Operator) | RDS MySQL Multi-AZ | $0 |
| Object cache | Valkey in-cluster | ElastiCache Valkey | $0 |
| Volumes | local-path-provisioner or Longhorn | EBS | $0 |
| Media | **Real S3 bucket** + offload plugin | S3 + CloudFront | ~$1 |
| Secrets | **Real AWS Secrets Manager** + External Secrets Operator | Same | ~$2 |
| Container images | GHCR (free) or ECR | ECR | $0–$1 |
| Monitoring | kube-prometheus-stack (Prometheus + Grafana) | Same, or Amazon Managed Prometheus | $0 |
| Database backups | MariaDB Operator backups to **S3** | RDS automated snapshots | ~$1 |
| AI post summaries | Summarizer CronJob in the cluster, calling **real Amazon Bedrock** over the internet | Same CronJob in EKS | <$1 |
| **Total (24/7)** | | | **~$5–$10 / month** |

### Short AWS demos
The full AWS design costs roughly **$0.70/hour** while running. Run `terraform apply`, record a walkthrough or take screenshots, then run `terraform destroy`.

| Demo time per month | Est. cost |
|---|---|
| 5 hours | ~$4 |
| 10 hours | ~$7 |
| 20 hours | ~$14 |

Together with the home lab, the total stays well under **$30–$50/month**. If the AWS account is new, its free-tier credits can cover some of this.

**Tip:** set an AWS Budget with an email alert (e.g. at $25) so a forgotten `terraform destroy` doesn't run up a bill.

### What makes this a strong portfolio project
- **One set of GitOps manifests, two environments.** Argo CD deploys the same Helm chart and values, with small per-environment overrides (`overlays/home`, `overlays/aws`). Only the backing services change: in-cluster MariaDB vs RDS, Valkey vs ElastiCache.
- **Real AWS integration from home:** S3 media offload, Secrets Manager, S3 backups, Bedrock summaries and IAM are all managed by Terraform.
- **Generative AI in a real application:** the summarizer shows Bedrock integration, least-privilege IAM for model access, and cost controls, with the same code running on both clusters.
- **Pod permissions without stored keys at home (stretch goal):** publish the Talos cluster's service-account OIDC discovery files (e.g. in a public S3 bucket) and register them as an IAM OIDC provider. Pods such as WordPress (S3) and the summarizer (Bedrock) then assume IAM roles without stored keys, just like IRSA on EKS. Until then, the home cluster uses a narrowly scoped IAM user key kept in Secrets Manager.
- **Terraform for both sides:** AWS provider for the cloud stack; Cloudflare provider (tunnel, DNS, WAF rules), Talos provider (`siderolabs/talos`), and Helm and Kubernetes providers for the home lab.
- **CI on GitHub:** `terraform fmt`, `validate`, `plan`, tflint and checkov on every pull request, plus builds and scans of the WordPress and summarizer images. Employers see all of it without the AWS stack running.
- **Documentation:** architecture diagrams for both environments, the cost breakdown, and demo recordings.

### Suggested repo layout
```
terraform/
  aws/                      # VPC, EKS, RDS, ElastiCache, S3, CloudFront, WAF, IAM (incl. Bedrock role), Budgets
  home/                     # Cloudflare tunnel/DNS/WAF, S3 + IAM (incl. Bedrock access) for home, Secrets Manager
gitops/
  bootstrap/                # Argo CD root "app of apps" per environment (home, aws)
  apps/
    wordpress/              # base + overlays/home, overlays/aws
    post-summarizer/        # CronJob + ServiceAccount; base + overlays/home, overlays/aws
  platform/                 # shared cluster services, each with base + overlays/home, overlays/aws
    monitoring/             # kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
    external-secrets/       # External Secrets Operator + AWS Secrets Manager store
    ingress/                # home: ingress-nginx; aws: AWS Load Balancer Controller
    cloudflared/            # Cloudflare Tunnel connector (home only)
    storage/                # home: local-path-provisioner or Longhorn; aws: EBS CSI StorageClass
    mariadb/                # MariaDB Operator + database (home only; RDS on AWS)
    valkey/                 # Valkey object cache (home only; ElastiCache on AWS)
docker/
  wordpress/                # custom image (plugins and themes built in)
  post-summarizer/          # summarizer image (calls WordPress REST API + Bedrock)
docs/                       # architecture diagrams, cost breakdown, demo recordings
.github/workflows/          # Terraform checks, image builds and scans
```

Prometheus lives under `gitops/platform/monitoring/`. Everything under `platform/` is a cluster service that WordPress depends on but that isn't the application itself. Components marked "home only" exist only in the home overlays, because AWS supplies a managed service in their place.
