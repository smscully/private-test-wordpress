# Home Lab Build Plan (russetleaf.dev)

This is the step-by-step plan for writing and testing all the code for the **russetleaf.dev home lab** described in [`aws-wp.md`](aws-wp.md). The AWS EKS site (russetleaf.com) is **out of scope** for now; see [Deferred to the AWS phase](#deferred-to-the-aws-phase).

Each step is sized for one or a few Claude sessions, and ends with a **test** that must pass before moving on. Steps are ordered so that everything a component depends on already exists and works.

## How to use this plan with Claude

- **Start each session** with something like: *"Read `aws-wp.md` and `homelab-plan.md`. We're working on step 7. Here's where I left off: ..."*. Claude has no memory of earlier sessions, so these two files are the shared context.
- **One step at a time.** Build it, run the step's test, fix, commit. Don't start the next step until the test passes.
- **Keep the plan current.** Tick the [progress checklist](#progress-checklist) and add to the [decision log](#decision-log) whenever something is decided or differs from `aws-wp.md`. Ask Claude to update this file at the end of each session.
- **Watch box 1's memory.** After every step that adds something to the cluster, run `kubectl top nodes` and note the result in the checklist. Box 1 has 8 GB and the full design uses about 6–7 GB of it.
- **Build the home overlay only, but keep the base clean.** Put anything environment-specific (domain, hostnames, endpoints, credentials) in `overlays/home`, never in `base`, so `overlays/aws` can be added later without rework.
- **Never commit secrets.** Secrets go in AWS Secrets Manager and reach the cluster through External Secrets. Terraform state (which can contain secrets) lives in an encrypted S3 bucket. kubeconfig and talosconfig files stay outside Git.

## Starting point (as of 2026-10-31)

- One Talos node (box 1, `192.168.1.6`, Talos v1.14.4, Kubernetes v1.37.1), control plane, no taints.
- Argo CD installed in the `argocd` namespace.
- This repo (`private-test-wordpress`) holds an Argo CD Application that deploys the Bitnami WordPress chart into `wordpress-test`. Its pods are stuck in `Pending` because the cluster has no storage provisioner. This test deployment will be replaced.
- russetleaf.dev is a Cloudflare zone.
- Box 2 (Intel i3, 8 GB) is available to become the LLM worker node.

## Phase 0: Prerequisites and decisions

### Step 0.1: Accounts, tools and budget alert
**Do:**
- Accounts: AWS (with MFA on the root user and an admin IAM user or SSO for daily work), GitHub, Cloudflare (russetleaf.dev zone).
- Install locally: `terraform` (1.10+), `talosctl` (matching your Talos version), `kubectl`, `helm`, `kustomize`, `aws` CLI v2, `gh`, `cosign`, `kyverno` CLI, `kubeconform`, `docker`.
- Create an **AWS Budget** with an email alert at about $25/month (by hand in the console is fine; Terraform takes over in step 2.1).
- Pick an AWS region for the home lab's AWS resources (e.g. `us-east-1`) and use it everywhere.

**Test:** `aws sts get-caller-identity` works; every tool prints its version.

### Step 0.2: Decisions to make before writing code
Record each answer in the [decision log](#decision-log).

| Decision | Recommendation | Why it matters |
|---|---|---|
| **WordPress Helm chart** | Write a **small chart of your own** (Deployment/Rollout, Service, Ingress, HPA, PDB, CronJob for WP-Cron), kept in `wp-image/chart/` and released together with the image, for an image built on the **official `wordpress` image**. (`aws-wp.md` already assumes this.) | The Bitnami chart only works with Bitnami-format images (their startup scripts and environment variables). Since Bitnami cut back its free images in 2025, the alternatives are building Bitnami's image yourself from their open-source `bitnami/containers` repo, or your own chart. Your own chart is simpler to reason about, removes the Bitnami dependency, and is a portfolio piece. |
| **Container registry** | **GHCR** (GitHub Container Registry): free for public images, and cosign keyless signing works with GitHub Actions out of the box. | Kyverno policies and image references depend on it. |
| **Repos public or private** | Private while building; make public once each repo has been scanned for secrets (`gitleaks`). | Self-hosted runners on public repos need extra care (step 9.2). |
| **Summarizer language** | **Python** (boto3 for SQS, the `openai` client for the gateway, OpenTelemetry SDK). | Fastest path; the most common language for MLOps work. |
| **Existing Talos node** | **Rebuild it with Terraform** in step 2.2, rather than adopting the hand-built config. | Nothing on the cluster is worth keeping, and a Terraform-built cluster is reproducible. Adopting it means importing its existing machine secrets, which is fiddly. |
| **Bot for version-bump pull requests** | A **GitHub App** installed only on `wp-gitops`. | Narrower and longer-lived than a personal access token. |
| **Network policies (CNI)** | Decide **now**: either keep Talos's default **Flannel** (simpler, but it does **not** enforce Kubernetes NetworkPolicies), or install **Cilium** as the CNI (Talos config `cni: none`, then Cilium), which enforces them. Cilium is recommended if you want step 6.3. | Changing the CNI later means rebuilding the cluster. Without enforced NetworkPolicies, any pod can reach any other pod (e.g. the database). |
| **Resource tagging** | Every AWS resource gets `Project=russetleaf`, `Site=dev`, `ManagedBy=terraform`, set as the AWS provider's `default_tags`. | Lets you find leftovers and see costs per site in Cost Explorer. The AWS plan uses the same scheme with `Site=com`. |

## Phase 1: Repositories

### Step 1.1: Create the four repos
**Do:**
- Rename this repo to **`wp-gitops`**. Create **`wp-infra`**, **`wp-image`** and **`llm-summarizer`**.
- In each: a README stub, `.gitignore` (Terraform state, `.terraform/`, kubeconfig, talosconfig, `.env`), a `LICENSE` if public, branch protection on `main` (pull requests and passing checks required).
- Add a pre-commit or CI secret scan (`gitleaks`) to every repo.
- Add **Renovate** to every repo (Helm charts, Terraform providers, base images, GitHub Actions).
- In `wp-gitops`: move `aws-wp.md` and this file to `docs/`, and write the hub README (what the project is, links to all four repos).

**Test:** each repo has protected `main`, the secret scan runs on a test pull request, and Renovate opens its onboarding pull request.

## Phase 2: Infrastructure (`wp-infra`)

### Step 2.1: Bootstrap (state bucket and CI roles)
**Do** (`wp-infra/bootstrap/`, applied locally):
- S3 bucket for Terraform state: versioning, encryption, public access blocked. Backends use `use_lockfile = true`.
- GitHub OIDC provider in IAM, plus roles for `wp-infra` CI: a read-only **plan** role (pull requests) and an **apply** role (`main` only).
- The AWS Budget alert (import the one from step 0.1 or recreate it).

**Test:** a scratch root module with the S3 backend runs `terraform init` and `plan`; a GitHub Actions job in `wp-infra` assumes the plan role and runs `aws sts get-caller-identity`.

### Step 2.2: Talos cluster (box 1 + box 2)
**Do** (`wp-infra/envs/home/`, applied locally; uses the `siderolabs/talos` provider):
- Generate machine secrets; machine configs for **box 1 (control plane, schedulable)** and **box 2 (worker)**.
- Box 2: node label `workload=llm`, taint `dedicated=llm:NoSchedule`.
- Both nodes: kubelet extra mount for `/var/local-path-provisioner` (needed in step 4.1).
- If you chose Cilium in step 0.2: set the CNI to `none` in the machine config and install Cilium (Helm, from Terraform or as the first Argo CD component).
- Apply configs, bootstrap etcd, output kubeconfig and talosconfig (save them outside Git).
- Install **metrics-server** here or in step 4.1, so `kubectl top` works.

**Test:** `kubectl get nodes` shows both nodes `Ready`; `kubectl describe node <box2>` shows the label and taint; `talosctl health` passes. Rebuild once from scratch to prove it's reproducible.

### Step 2.3: AWS and Cloudflare resources for the home lab
**Do** (`wp-infra/modules/` + `envs/home/`):
- Modules: `s3-media`, `s3-backups`, `sqs-queue` (with dead-letter queue), `secrets`, `bedrock-access`.
- russetleaf.dev resources: media bucket, database-backup bucket (lifecycle: expire after 30 days), **model mirror bucket** (for the self-hosted model's file, used in step 8.1), summary queue + dead-letter queue, Secrets Manager entries (empty placeholders; values set by hand or generated), Bedrock budget alert.
- **Email:** an SES domain identity for russetleaf.dev, with its DKIM and SPF records created in Cloudflare, and SES SMTP credentials stored in Secrets Manager. (New SES accounts are in the "sandbox" and can only send to verified addresses; verify your own address for testing, or request production access.)
- Scoped IAM users and keys for the home cluster, one per job: External Secrets (read the russetleaf.dev secrets only), WordPress (media bucket + send to queue), MariaDB backups (backup bucket), summarizer (read from queue), LiteLLM (`bedrock:InvokeModel` on the chosen models only). Store each key in Secrets Manager.
- Cloudflare (Cloudflare provider): the tunnel, its token (stored in Secrets Manager), DNS records for `russetleaf.dev` and `www`, basic WAF/security settings, and **Cloudflare Access** policies for admin UIs (Argo CD, Grafana) so they're never public.
- Bedrock: confirm access to Claude Haiku and Amazon Nova Lite in the chosen region (Anthropic models may need a one-time use-case form in the console).

**Test:** with each scoped key, the allowed action works and a disallowed one fails (e.g. the WordPress key can't read the backup bucket); `aws bedrock-runtime converse` with the LiteLLM key returns a response; the tunnel shows as created in Cloudflare.

### Step 2.4: `wp-infra` CI
**Do:** GitHub Actions: `terraform fmt -check`, `validate`, `tflint`, Checkov, and `plan` on pull requests (plan posted as a comment). Applies to `envs/home` stay **local** for now (the Talos provider needs your home network); they can move to in-cluster runners after step 9.2.

**Test:** a pull request with a harmless change shows a passing check run and a plan comment; a deliberately insecure change (e.g. a public bucket) fails Checkov.

## Phase 3: GitOps foundation (`wp-gitops`)

### Step 3.1: Structure and Argo CD bootstrap
**Do:**
- Create the layout from `aws-wp.md`: `bootstrap/home/`, `apps/`, `platform/`, each component with `base/` and `overlays/home/`.
- Install Argo CD with Helm on the new cluster, then add Argo CD itself to `platform/argocd/` so it manages its own upgrades.
- `bootstrap/home/`: the root "app of apps" that creates one Argo CD Application per component, with **sync waves** (storage and secrets first, then ingress, monitoring, operators, then apps).
- Expose the Argo CD UI only through Cloudflare Access (after step 4.3), or use `kubectl port-forward` until then.
- CI: render every component's home overlay (Helm/Kustomize) and validate with `kubeconform`.

**Test:** apply the root app; Argo CD shows it `Synced`/`Healthy` with a trivial test component; a pull request that breaks a manifest fails CI.

## Phase 4: Platform services (`wp-gitops/platform/`)

Check `kubectl top nodes` after each step.

### Step 4.1: Storage
**Do:** `platform/storage/`: local-path-provisioner as the default StorageClass, using `/var/local-path-provisioner`, its namespace labelled for privileged pod security (Talos enforces "baseline" by default).

**Test:** a test PersistentVolumeClaim binds and a pod writes to it; the data survives the pod being deleted.

### Step 4.2: Secrets
**Do:** `platform/external-secrets/`: External Secrets Operator plus a `ClusterSecretStore` for AWS Secrets Manager.
- **Bootstrap credential:** External Secrets needs one AWS key to start. Create it once by hand as a Kubernetes Secret from the Terraform output (document the exact command in the README, never commit it). Everything else comes from Secrets Manager.

**Test:** an `ExternalSecret` for a test value creates a Kubernetes Secret; changing the value in Secrets Manager updates it within the refresh interval.

### Step 4.3: Ingress and the Cloudflare Tunnel
**Do:**
- `platform/ingress/`: ingress-nginx (one replica, modest resources), with its **Prometheus metrics enabled** (the Argo Rollouts analysis in step 6.2 uses them).
- `platform/cloudflared/`: cloudflared Deployment (2 replicas) using the tunnel token from External Secrets, routing `russetleaf.dev` and the admin hostnames to ingress-nginx.
- Put the Argo CD and Grafana hostnames behind Cloudflare Access.

**Test:** a hello-world app at `https://russetleaf.dev` loads from outside your network (e.g. on mobile data); your router has no open ports; the Argo CD hostname asks for a Cloudflare Access login.

### Step 4.4: Monitoring
**Do:** `platform/monitoring/`: kube-prometheus-stack, trimmed for 8 GB: 7-day retention, 30–60 s scrape interval, resource limits on every part. Alertmanager sends to email (or Discord/Slack). Grafana behind Cloudflare Access.

**Test:** Grafana shows node and pod dashboards for both nodes; a test alert (e.g. a temporary rule that always fires) reaches you.

### Step 4.5: Database and cache
**Do:**
- `platform/mariadb/`: MariaDB Operator; a `MariaDB` instance pinned to the **same major version RDS offers** (e.g. 11.4 LTS), a `Database`, `User` and `Grant` for WordPress (passwords from External Secrets).
- The daily **`Backup`** to the backup bucket (logical, gzip, 14-day retention), and an alert if the last successful backup is more than 26 hours old.
- `platform/valkey/`: Valkey (single instance, no persistence needed for an object cache).

**Test:** connect with a MariaDB client from a test pod; trigger a backup by hand and see the file in S3; **restore it into a scratch MariaDB instance** and compare; `valkey-cli ping` returns `PONG`.

## Phase 5: WordPress

### Step 5.1: The WordPress image (`wp-image`)
**Do:**
- `Dockerfile` based on the official `wordpress` image (pinned version), with plugins and themes pinned in `plugins.json` and installed at build time: S3 media offload (e.g. WP Offload Media Lite), Redis Object Cache, an SMTP plugin (configured for Amazon SES).
- `wp-config` settings read from environment variables (database, Valkey, S3, queue URL, `WP_HOME`/`WP_SITEURL`, `DISALLOW_FILE_MODS`, `DISABLE_WP_CRON`).
- The **summary-queue must-use plugin**: on publish or update, send `{"post_id": ...}` to SQS (AWS SDK for PHP, installed with Composer at build time).
- `tests/`: a smoke test (Docker Compose with MariaDB: the container starts, plugins are active, health check passes).
- `chart/`: the WordPress Helm chart (see step 5.2 for what it deploys), versioned with the image.
- CI: build, SBOM (Syft), scan (Trivy, failing on critical vulnerabilities), sign (cosign keyless), push the image and the chart (OCI) to GHCR with an immutable version.

**Test:** the smoke test passes locally and in CI; `cosign verify` succeeds for the pushed image; publishing a post in the local Compose setup sends a message to the real SQS queue.

### Step 5.2: WordPress chart and deployment (`wp-gitops/apps/wordpress/`)
**Do:**
- The `wp-image` chart, deployed from `wp-gitops` with the home overlay (values only): russetleaf.dev hostname, MariaDB and Valkey hosts, media bucket and queue settings, credentials from External Secrets, resources, `persistence` off, WP-Cron as a Kubernetes CronJob.
- Remove the old test Application and the `wordpress-test` namespace.
- First-run setup (admin user, plugin settings) done by a one-off `wp-cli` Job or wp-config constants, so a rebuild needs no clicking.

**Test:** `https://russetleaf.dev` serves WordPress; an uploaded image lands in the S3 bucket and displays; the Redis Object Cache plugin shows "connected"; killing the WordPress pod loses nothing; WP-Cron jobs run from the CronJob; a password-reset email arrives through SES.

### Step 5.3: Image promotion
**Do:** create the GitHub App; the `wp-image` release workflow opens a pull request on `wp-gitops` that changes only the chart and image version in `apps/wordpress/`, linking the release notes.

**Test:** merge a trivial change in `wp-image` → a signed image is pushed → a bump pull request appears in `wp-gitops` → merging it rolls out the new image.

## Phase 6: Policy and progressive delivery

### Step 6.1: Kyverno
**Do:** `platform/kyverno/`: Kyverno plus policies, in **audit mode** first:
- only images signed by the `wp-image` and `llm-summarizer` release workflows (cosign keyless, GitHub OIDC issuer, subject matching each repo's workflow), pulled from GHCR (exclude third-party platform images, or add their publishers' signatures where available)
- no `:latest` tags
- CPU and memory requests and limits required
- no privileged pods outside namespaces that need them (storage, Falco)
- Policy tests with `kyverno test`, run in `wp-gitops` CI.

Review the policy reports, fix violations, then switch to **enforce**.

**Test:** in enforce mode, an unsigned image and a `:latest` image are both rejected; the existing workloads keep running; a pull request that violates a policy fails CI.

### Step 6.2: Argo Rollouts
**Do:** `platform/argo-rollouts/`; convert WordPress to a `Rollout` with canary steps (e.g. 20% → 50% → 100%) and an `AnalysisTemplate` that queries Prometheus (ingress-nginx error rate and latency for the WordPress service).

**Test:** a good image rolls out step by step; a deliberately broken image (e.g. one that returns 500s) is **rolled back automatically**.

### Step 6.3 (optional, needs Cilium): Network policies
**Do:** a default-deny NetworkPolicy in each app namespace, then allow only what's needed: ingress-nginx → WordPress; WordPress → MariaDB, Valkey and the internet (S3, SQS, SES); summarizer → WordPress, the gateway and the internet (SQS); gateway → llama.cpp and the internet (Bedrock); Prometheus → everything it scrapes. Keep the policies in each component's `base/` so the AWS site can use them too.

**Test:** from a test pod in another namespace, MariaDB is unreachable; the site, the summarizer and monitoring all still work.

## Phase 7: AI post summaries

### Step 7.1: KEDA
**Do:** `platform/keda/`: KEDA, with AWS credentials for the SQS scaler from External Secrets.

**Test:** a test `ScaledObject` scales a dummy Deployment from 0 to 1 when a message is sent to a test queue, and back to 0 when the queue is empty.

### Step 7.2: LiteLLM gateway
**Do:** `platform/llm-gateway/`: LiteLLM with Claude Haiku (default) and Amazon Nova Lite on Bedrock, using the scoped Bedrock key; per-app budgets and rate limits; metrics exposed to Prometheus; an internal API key for callers.

**Test:** from a test pod, an OpenAI-style chat request to the gateway returns answers from both models; requests over the rate limit are refused; the gateway's metrics appear in Prometheus.

### Step 7.3: Summarizer code (`llm-summarizer`)
**Do:**
- A dedicated WordPress user for the summarizer (role: editor or a custom role that can edit posts only), with an **application password** created by `wp-cli` and stored in Secrets Manager.
- `src/`: SQS consumer (long polling, delete only after success, retries then dead-letter queue), WordPress adapter (REST API with that application password, calling WordPress's in-cluster Service rather than going out through Cloudflare: read the post, write the summary to the excerpt or a custom field), OpenAI-compatible model client pointed at the gateway, prompt loading from `prompts/` with the version recorded alongside each summary.
- Unit tests (adapter and consumer with mocks).
- `demo/`: Docker Compose with sample posts and a small mock WordPress endpoint, calling Bedrock through a local LiteLLM, so anyone can run it without Kubernetes.

**Test:** unit tests pass; `docker compose up` in `demo/` produces summaries for the sample posts.

### Step 7.4: Summarizer chart, release and deployment
**Do:**
- `chart/`: Deployment (Rollout later), KEDA `ScaledObject` on the SQS queue (0–N), daily sweep CronJob, ServiceAccount, resources.
- Release workflow: build, SBOM, scan, sign, push the image and the chart (OCI) to GHCR, open a chart-bump pull request on `wp-gitops`.
- `wp-gitops/apps/post-summarizer/overlays/home`: queue URL, gateway URL, WordPress URL, credentials from External Secrets.

**Test (end to end):** publish a post on russetleaf.dev → a summarizer pod starts → the summary appears on the post within a few minutes → the pod scales back to zero. A message that always fails ends up in the dead-letter queue.

### Step 7.5: LLM monitoring
**Do:** OpenTelemetry metrics in the summarizer (tokens, latency, errors, fallbacks, estimated cost, all labelled by model and prompt version); the Grafana LLM dashboard and `PrometheusRule` alerts shipped **in the summarizer chart** (error spike, dead-letter queue not empty, daily Bedrock spend over a limit); SQS queue depth in Grafana.

**Test:** the dashboard shows real numbers after a few posts; temporarily lowering the spend limit makes the alert fire.

## Phase 8: Self-hosted model (box 2)

### Step 8.1: llama.cpp model server
**Do:** `platform/llm-serving/`: llama.cpp server (CPU build) with a toleration and node selector for box 2; an init container that downloads a 4-bit **3B model** (e.g. Qwen2.5-3B or Llama 3.2 3B) into a local-path volume (mirror the file in S3 so rebuilds don't depend on Hugging Face); metrics enabled; resource limits leaving ~1 GB free on the node.

**Test:** the pod runs **only** on box 2; an OpenAI-style request returns a summary; record the time per summary and memory use in the decision log.

### Step 8.2: Gateway routing to the local model
**Do:** add the local model to LiteLLM: fallback (Bedrock first, local if Bedrock fails), and a canary split (e.g. 10% of summary requests to the local model); llama.cpp metrics and model-down alert in Grafana.

**Test:** with Bedrock blocked (e.g. a wrong model ID in a test config), requests fall back to the local model; with the split on, about 10% of summaries show the local model's name in the metrics.

## Phase 9: MLOps pipeline

### Step 9.1: Evaluation set and scorers
**Do** (`llm-summarizer/evals/`): 20–30 sample posts with reference notes; rule-based checks (length, required keywords, no made-up links); an LLM-as-judge prompt for faithfulness; a script that runs the set against each model through a gateway URL and writes a report (JSON + Markdown).

**Test:** running the script locally against the gateway (via `kubectl port-forward`) produces a report comparing Haiku, Nova Lite and the local model.

### Step 9.2: Self-hosted runners (ARC)
**Do:** `platform/actions-runner-controller/`: ARC with **ephemeral** runners registered to `llm-summarizer`, tolerating box 2's taint so they run beside the model and off box 1. Before the repo goes public: require approval for workflows from fork pull requests, and limit which workflows can use the self-hosted label.

**Test:** a test workflow runs on a self-hosted runner pod, which is deleted afterwards; a fork pull request waits for approval.

### Step 9.3: Evaluation gate and releases
**Do:**
- Pull request workflow (on ARC runners): run the evaluation, post per-model scores as a pull request comment compared with `main`, fail below the threshold.
- Release workflow: run the gate again, then publish (step 7.4), attach the report to the GitHub release, update `MODEL_CARD.md`.
- Optionally convert the summarizer to an Argo Rollouts canary.

**Test:** a pull request with a deliberately bad prompt fails the gate with a clear comment; a good one passes; a release tag publishes only after the gate passes.

## Phase 10: Runtime security, compliance and cost

### Step 10.1: Falco
**Do:** `platform/falco/`: Falco with the **modern eBPF** driver (Talos doesn't allow kernel modules), Falcosidekick to Alertmanager/Grafana, and custom rules: shell in WordPress, summarizer or gateway pods; writes under `/var/www/html`; unexpected outbound connections from box 2.

**Test:** `kubectl exec` into a WordPress pod triggers an alert in Grafana; creating a file under `/var/www/html` triggers another.

### Step 10.2: Trivy Operator and Talos hardening notes
**Do:** `platform/trivy-operator/` with modest scan concurrency (it's memory hungry on 8 GB); write `docs/talos-hardening.md` explaining how Talos meets the CIS benchmark (no shell or SSH, read-only system, API-only management, default pod security).

**Test:** vulnerability and compliance reports exist for the running workloads; the hardening doc is reviewed.

### Step 10.3: OpenCost
**Do:** `platform/opencost/` using the existing Prometheus, with custom prices for the home hardware (electricity per node) and the Bedrock spend shown alongside.

**Test:** OpenCost shows cost per namespace, with box 2 attributed to the LLM namespace.

## Phase 11: Optional and finishing

### Step 11.1 (optional): Keyless AWS access for pods
**Do:** publish the Talos cluster's service-account OIDC discovery files (public S3 bucket), register it as an IAM OIDC provider in `wp-infra`, switch pods from scoped keys to IAM roles one at a time, then delete the keys.

**Test:** each pod works with no AWS key in its Secrets; the old keys are deleted from IAM.

### Step 11.2: Documentation and polish
**Do:** hub README with the russetleaf.dev architecture diagram and links to all repos; screenshots of Grafana (site, LLM, cost), Argo CD and an evaluation report; update `aws-wp.md` wherever the build differed from the design; check every repo with `gitleaks`, then make them public.

**Test:** someone unfamiliar with the project can follow the README to understand it in five minutes.

## Ongoing operations (after the build)

These keep russetleaf.dev healthy once it's running. Add each as a calendar reminder or a scheduled job.

| Task | How often | How |
|---|---|---|
| Test a database restore | Monthly | Restore the latest backup into a scratch MariaDB instance and check the post count (step 4.5) |
| Merge Renovate pull requests | Weekly | Review and merge; Argo Rollouts and the CI checks catch bad updates |
| Talos and Kubernetes upgrades | When new versions are out (Talos releases roughly every few months) | Bump versions in `wp-infra/envs/home` and apply; upgrade box 2 first, then box 1 |
| Back up cluster state (etcd) | Weekly | `talosctl etcd snapshot` to the backup bucket (a scheduled job on your machine or a CronJob using a Talos API key). GitOps can rebuild everything else, but this speeds up recovery. |
| Check for Terraform drift | Weekly | A scheduled `terraform plan` in `wp-infra` CI for the AWS and Cloudflare resources (alerts if anything changed outside Terraform) |
| Rotate scoped IAM keys | Every 90 days, until step 11.1 removes them | Create a new key in Terraform, update Secrets Manager, let External Secrets sync it, delete the old key |
| Review alerts, costs and Bedrock spend | Monthly | Grafana, OpenCost, the AWS Budget and Cost Explorer (filter by `Site=dev`) |

## Deferred to the AWS phase

Not part of this plan; built later for russetleaf.com (see [`aws-eks-plan.md`](aws-eks-plan.md)):
- `wp-infra/envs/aws-edge` (Route 53, CloudFront, offline page, russetleaf.com media bucket) and `envs/aws` (VPC, EKS, RDS for MariaDB, ElastiCache, WAF)
- `overlays/aws` for every component
- Karpenter, AWS Load Balancer Controller, ExternalDNS, EBS CSI, kube-bench
- llama.cpp CUDA build and the GPU NodePool
- The `seed-from-dev` Job
- IRSA / Pod Identity roles

The home lab already builds most of what these need: the shared Terraform modules, the base manifests, the images and charts, and the CI.

## Progress checklist

| Step | Done | Date | Box 1 memory after | Notes |
|---|---|---|---|---|
| 0.1 Accounts, tools, budget | ☐ | | | |
| 0.2 Decisions | ☐ | | | |
| 1.1 Four repos | ☐ | | | |
| 2.1 Terraform bootstrap | ☐ | | | |
| 2.2 Talos cluster | ☐ | | | |
| 2.3 AWS + Cloudflare resources | ☐ | | | |
| 2.4 `wp-infra` CI | ☐ | | | |
| 3.1 GitOps structure + Argo CD | ☐ | | | |
| 4.1 Storage | ☐ | | | |
| 4.2 Secrets | ☐ | | | |
| 4.3 Ingress + tunnel | ☐ | | | |
| 4.4 Monitoring | ☐ | | | |
| 4.5 MariaDB + backups + Valkey | ☐ | | | |
| 5.1 WordPress image | ☐ | | | |
| 5.2 WordPress deployment | ☐ | | | |
| 5.3 Image promotion | ☐ | | | |
| 6.1 Kyverno | ☐ | | | |
| 6.2 Argo Rollouts | ☐ | | | |
| 6.3 Network policies (optional) | ☐ | | | |
| 7.1 KEDA | ☐ | | | |
| 7.2 LiteLLM gateway | ☐ | | | |
| 7.3 Summarizer code | ☐ | | | |
| 7.4 Summarizer chart + deploy | ☐ | | | |
| 7.5 LLM monitoring | ☐ | | | |
| 8.1 llama.cpp on box 2 | ☐ | | | |
| 8.2 Gateway routing to local model | ☐ | | | |
| 9.1 Evaluation set | ☐ | | | |
| 9.2 ARC runners | ☐ | | | |
| 9.3 Evaluation gate + releases | ☐ | | | |
| 10.1 Falco | ☐ | | | |
| 10.2 Trivy Operator + Talos notes | ☐ | | | |
| 10.3 OpenCost | ☐ | | | |
| 11.1 Keyless AWS access (optional) | ☐ | | | |
| 11.2 Docs + go public | ☐ | | | |

## Decision log

| Date | Decision | Reason |
|---|---|---|
| | | |
