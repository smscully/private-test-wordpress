# Home Lab Build Plan (russetleaf.dev)

This is the step-by-step plan for writing and testing all the code for the **russetleaf.dev home lab** described in [`aws-wp.md`](aws-wp.md). The AWS EKS site (russetleaf.com) is **out of scope** for now; see [Deferred to the AWS phase](#deferred-to-the-aws-phase).

Each step is sized for one or a few Claude sessions, and ends with a **test** that must pass before moving on. Steps are ordered so that everything a component depends on already exists and works.

## How to use this plan with Claude

- **Start each session** with something like: *"Read `aws-wp.md` and `homelab-plan.md`. We're working on step 7. Here's where I left off: ..."*. Claude has no memory of earlier sessions, so these two files are the shared context.
- **One step at a time.** Build it, run the step's test, fix, commit. Don't start the next step until the test passes.
- **Keep the plan current.** Tick the [progress checklist](#progress-checklist) and add to the [decision log](#decision-log) whenever something is decided or differs from `aws-wp.md`. Ask Claude to update this file at the end of each session.
- **Watch box 1's memory.** After every step that adds something to the cluster, run `kubectl top nodes` and note the result in the checklist. Box 1 has 8 GB and the full design uses about 6–7 GB of it.
- **Build the home overlays only, but keep the base clean.** Put anything environment-specific (domain, hostnames, endpoints, credentials) in `overlays/home` (prod), `overlays/home-staging` or `overlays/ci`, never in `base`, so `overlays/aws` can be added later without rework.
- **Use the environments.** Try changes **locally** first (Docker Compose or a local kind/Talos cluster); once step 5.4 exists, every pull request is tested in a **ci** kind cluster; once step 5.6 exists, new versions go through **staging** before production. See [Environments and promotion](aws-wp.md#environments-and-promotion).
- **Never commit secrets.** Secrets go in AWS Secrets Manager and reach the cluster through External Secrets. Terraform state (which can contain secrets) lives in an encrypted S3 bucket. kubeconfig and talosconfig files stay outside Git.

## Starting point (as of 2026-10-31)

- One Talos node (box 1, `192.168.1.6`, Talos v1.14.4, Kubernetes v1.37.1), control plane, no taints.
- Argo CD installed in the `argocd` namespace.
- This repo (`private-test-wordpress`) holds an Argo CD Application that deploys the Bitnami WordPress chart into `wordpress-test`. Its pods are stuck in `Pending` because the cluster has no storage provisioner. This test deployment will be replaced.
- russetleaf.dev is a Cloudflare zone.
- Box 2 (Intel i3, 8 GB) is available to become the LLM worker node.

## Local workstation setup (Ubuntu 24.04)

Everything is built and tested from your Ubuntu 24.04 machine, which is also the **local** environment (see [Environments and promotion](aws-wp.md#environments-and-promotion)). Do this before step 0.1's test.

**Hardware:** 16 GB of RAM or more is recommended; a local kind or Talos-in-Docker cluster plus LocalStack and WordPress use about 4–6 GB. At least 40 GB of free disk for container images.

### 1. Base packages

```bash
sudo apt update
sudo apt install -y git curl wget unzip jq make ca-certificates gnupg lsb-release \
  python3 python3-venv shellcheck direnv
```

### 2. Docker Engine (from Docker's own repository, not the snap)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"     # then log out and back in
```

**kind needs higher file-watch limits** on Ubuntu (otherwise pods fail with "too many open files"):

```bash
echo -e "fs.inotify.max_user_watches=524288\nfs.inotify.max_user_instances=512" \
  | sudo tee /etc/sysctl.d/99-kind.conf
sudo sysctl --system
```

### 3. CLI tools, pinned per repo with mise

Use **mise** to install the command-line tools, with the versions pinned in a `.mise.toml` file in each repo. The same file is used by CI (the `jdx/mise-action` GitHub Action), so your machine and CI always run the same versions, and Renovate can keep them updated.

```bash
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

Tools to pin (check each name with `mise registry | grep <tool>`; pin `kubectl` and `talosctl` to your cluster's versions):

| Tool | Used for |
|---|---|
| `terraform` (1.10+), `tflint` | Infrastructure (`wp-infra`) |
| `talosctl` | Talos clusters (home and local Talos-in-Docker) |
| `kubectl`, `helm`, `kustomize` | Kubernetes and charts |
| `kind` | Local and ci test clusters |
| `argocd` | Argo CD CLI (staging workflow, debugging) |
| `k9s`, `stern`, `kubectx` | Day-to-day cluster work (optional but very useful) |
| `kubeconform`, `chart-testing` (`ct`) | Manifest and chart validation |
| `cilium`, `hubble` | Cilium status and connectivity tests; Hubble flow queries |
| `kyverno` | Policy tests |
| `cosign`, `syft`, `trivy` | Image signing, SBOMs, scanning |
| `kubescape` | On-demand cluster posture scans (NSA-CISA, MITRE ATT&CK, RBAC) |
| `gh` | GitHub CLI |
| `awscli` (v2) | AWS |
| `uv` | Python tooling (the summarizer, Checkov, pre-commit, LocalStack CLI) |
| `promtool` (from Prometheus) | Testing alert rules |

Python-based tools, installed with uv:

```bash
uv tool install pre-commit
uv tool install checkov
uv tool install localstack      # optional: LocalStack CLI (LocalStack itself runs in Docker)
```

PHP and Composer aren't needed locally: the WordPress image installs plugins inside its Docker build.

### 4. Configuration

- **Git:** set your name and email; authenticate with `gh auth login` (SSH); turn on **commit signing** with your SSH key (`git config --global gpg.format ssh` and `user.signingkey`), and upload the key to GitHub as a signing key.
- **AWS:** set up **IAM Identity Center (SSO)** and a profile (e.g. `russetleaf-admin`) with `aws configure sso`; sign in with `aws sso login --profile russetleaf-admin`. Keep **no long-lived AWS access keys** on the workstation.
- **Cloudflare and Slack tokens:** don't put them in `~/.bashrc`. Use per-repo **direnv** files (`.envrc`, listed in `.gitignore`) that load them from a password manager CLI or from AWS Secrets Manager when you enter the repo folder.
- **Cluster access files:** keep kubeconfig contexts named clearly (`talos-home`, `kind-ci`, `talos-local`) and talosconfig at `~/.talos/config`, both with `chmod 600`. Back them up in your password manager. (If they're lost, they can be regenerated from the Talos machine secrets in Terraform state.)
- **Pre-commit hooks:** run `pre-commit install` in each repo after cloning (secret scan, Terraform formatting, YAML checks).
- **Editor (optional):** VS Code with the HashiCorp Terraform, YAML (Red Hat), Kubernetes and Docker extensions.

### 5. Check the setup

Add a `scripts/check-tools.sh` to `wp-gitops` that prints every tool's version and fails if one is missing. Then:

```bash
docker run --rm hello-world
kind create cluster --name smoke && kubectl get nodes && kind delete cluster --name smoke
talosctl cluster create --name local-smoke          # Talos in Docker
talosctl cluster destroy --name local-smoke
aws sts get-caller-identity --profile russetleaf-admin
gh auth status
```

**Test:** all of the above succeed, and `scripts/check-tools.sh` reports every tool.

## Phase 0: Prerequisites and decisions

### Step 0.1: Accounts, tools and budget alert
**Do:**
- Accounts: AWS (with MFA on the root user and an admin IAM user or SSO for daily work), GitHub, Cloudflare (russetleaf.dev zone).
- Set up the workstation: see [Local workstation setup](#local-workstation-setup-ubuntu-2404).
- Create an **AWS Budget** with an email alert at about $25/month (by hand in the console is fine; Terraform takes over in step 2.1).
- Pick an AWS region for the home lab's AWS resources (e.g. `us-east-1`) and use it everywhere.

- **Slack:** create the project workspace and the channels from `aws-wp.md` ([Alerting and notifications](aws-wp.md#alerting-and-notifications-slack)): `#rl-critical`, `#rl-dev-alerts`, `#rl-com-alerts`, `#rl-deploys`, `#rl-security`, `#rl-mlops`, `#rl-costs`, `#rl-staging`. Create the custom Slack app ("russetleaf-bot") with a bot token allowed to post to those channels, and install the **GitHub** app for Slack. Turn on phone notifications for `#rl-critical` only.
- **Outside monitoring accounts** (free plans): a heartbeat service (e.g. healthchecks.io) and an uptime checker (e.g. UptimeRobot or Better Stack), both posting to Slack.

**Test:** the workstation checks pass; a test message from the bot token (`curl` to Slack's `chat.postMessage`) appears in `#rl-dev-alerts`.

### Step 0.2: Decisions to make before writing code
Record each answer in the [decision log](#decision-log).

| Decision | Recommendation | Why it matters |
|---|---|---|
| **WordPress Helm chart** | Write a **small chart of your own** (Deployment/Rollout, Service, Ingress, HPA, PDB, CronJob for WP-Cron), kept in `wp-image/chart/` and released together with the image, for an image built on the **official `wordpress` image**. (`aws-wp.md` already assumes this.) | The Bitnami chart only works with Bitnami-format images (their startup scripts and environment variables). Since Bitnami cut back its free images in 2025, the alternatives are building Bitnami's image yourself from their open-source `bitnami/containers` repo, or your own chart. Your own chart is simpler to reason about, removes the Bitnami dependency, and is a portfolio piece. |
| **Container registry** | **GHCR** (GitHub Container Registry): free for public images, and cosign keyless signing works with GitHub Actions out of the box. Note that **GHCR packages have their own visibility, separate from the repo's**, and start private. | Kyverno policies and image references depend on it. |
| **Repos public or private** | Private while building; make public once each repo has been scanned for secrets (`gitleaks`). | Self-hosted runners on public repos need extra care (step 9.2). |
| **Summarizer language** | **Python** (boto3 for SQS, the `openai` client for the gateway, OpenTelemetry SDK). | Fastest path; the most common language for MLOps work. |
| **Existing Talos node** | **Rebuild it with Terraform** in step 2.2, rather than adopting the hand-built config. | Nothing on the cluster is worth keeping, and a Terraform-built cluster is reproducible. Adopting it means importing its existing machine secrets, which is fiddly. |
| **Bot for version-bump pull requests** | A **GitHub App** installed only on `wp-gitops`. | Narrower and longer-lived than a personal access token. |
| **Networking (CNI)** | **Decided: Cilium**, as the full CNI replacing Talos's default Flannel **and kube-proxy**, with Hubble. EKS also runs Cilium (chaining mode), so both clusters use **CiliumNetworkPolicies**. See [Networking: Cilium and Hubble](aws-wp.md#networking-cilium-and-hubble). | Changing the CNI later means rebuilding the cluster. Flannel doesn't enforce network policies at all, so without Cilium any pod could reach any other pod (e.g. the database). |
| **Database credentials** | Home: passwords **generated by MariaDB Operator** into Kubernetes Secrets (never typed, never in Git or Secrets Manager), rotated on a schedule, TLS required. AWS (later): IAM auth through RDS Proxy. See [Database access and credentials](aws-wp.md#database-access-and-credentials). | Decides how WordPress gets its database login on each site; the chart must support both. |
| **Staging** | **On-demand, scale-to-zero** staging on the home cluster (created for each promotion, deleted afterwards). Promotion with plain **GitHub Actions** workflows; consider **Kargo** later only if memory allows. | Box 1 has no memory to spare for an always-on staging copy. |
| **AWS stand-ins for ci tests** | **LocalStack** (S3, SQS, Secrets Manager), if its current free-tier terms suit; otherwise **MinIO** (S3) + **ElasticMQ** (SQS) + plain Kubernetes Secrets. | ci tests must run without an AWS account. |
| **Resource tagging** | Every AWS resource gets `Project=russetleaf`, `Site=dev`, `Env=prod|staging`, `ManagedBy=terraform`, set as the AWS provider's `default_tags`. | Lets you find leftovers and see costs per site in Cost Explorer. The AWS plan uses the same scheme with `Site=com`. |

## Phase 1: Repositories

### Step 1.1: Create the four repos
**Do:**
- Rename this repo to **`wp-gitops`**. Create **`wp-infra`**, **`wp-image`** and **`llm-summarizer`**.
- In each: a README stub, `.gitignore` (Terraform state, `.terraform/`, kubeconfig, talosconfig, `.env`), a `LICENSE` if public, branch protection on `main` (pull requests and passing checks required).
- In `wp-gitops`: a `docs/adr/` folder for **architecture decision records**, with a short template (context, decision, alternatives, consequences, when to revisit). Record the big step 0.2 decisions there as ADRs, and link them from the decision log. The template (`0000-template.md`) and three ADRs already exist as drafts in this repo's `docs/adr/`: `0001-no-service-mesh.md` (finish in step 6.4), `0002-s3-media-not-efs.md` (finish in step 5.1) and `0003-kubescape-cli-not-operator.md` (finish in step 10.2); move them with the other docs.
- Add a pre-commit or CI secret scan (`gitleaks`) to every repo.
- Add **Checkov** to every repo's CI from the start, scanning the **GitHub Actions workflows** (unpinned actions, over-broad `permissions`, script injection), with a `.checkov.yaml` per repo. Later steps add each repo's other targets (Terraform, manifests, Dockerfiles, charts). Skips must carry a reason. See [Infrastructure and configuration scanning](aws-wp.md#infrastructure-and-configuration-scanning-checkov).
- Add **Renovate** to every repo (Helm charts, Terraform providers, base images, GitHub Actions).
- In `wp-gitops`: move `aws-wp.md` and this file to `docs/`, and write the hub README (what the project is, links to all four repos).

**Test:** each repo has protected `main`, the secret scan and Checkov run on a test pull request (a workflow with `permissions: write-all` fails Checkov), and Renovate opens its onboarding pull request.

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
- **Cilium:** in the machine config, set `cluster.network.cni.name: none` and `cluster.proxy.disabled: true` (Cilium replaces kube-proxy). Install Cilium with Terraform's Helm provider right after bootstrap (nothing else can run without a CNI), using Talos's documented Cilium values: `kubeProxyReplacement: true`, `k8sServiceHost: localhost`, `k8sServicePort: 7445` (KubePrism), and the cgroup and capability settings Talos requires. Turn on Hubble and Hubble Relay, and **WireGuard transparent encryption** (`encryption.enabled: true`, `encryption.type: wireguard`). Keep Cilium's ServiceMonitors **off** here (the Prometheus Operator doesn't exist yet; step 4.4 turns them on). Use `lifecycle { ignore_changes }` so Terraform installs Cilium once and Argo CD manages it afterwards (step 3.1).
- Apply configs, bootstrap etcd, output kubeconfig and talosconfig (save them outside Git).
- Install **metrics-server** here or in step 4.1, so `kubectl top` works.

**Test:** `kubectl get nodes` shows both nodes `Ready`; `kubectl describe node <box2>` shows the label and taint; `talosctl health` passes; `cilium status` is all green and `cilium connectivity test` passes; `cilium encrypt status` shows WireGuard on with box 1 and box 2 as peers; there's no kube-proxy DaemonSet. Rebuild once from scratch to prove it's reproducible.

### Step 2.3: AWS and Cloudflare resources for the home lab
**Do** (`wp-infra/modules/` + `envs/home/`):
- Modules: `s3-media`, `s3-backups`, `sqs-queue` (with dead-letter queue), `secrets`, `bedrock-access`.
- russetleaf.dev resources: media bucket, database-backup bucket (lifecycle: expire after 30 days), **model mirror bucket** (for the self-hosted model's file, used in step 8.1), summary queue + dead-letter queue, Secrets Manager entries (empty placeholders; values set by hand or generated), Bedrock budget alert.
- **Staging resources** (a second, smaller set of the same modules, tagged `Env=staging`): staging media bucket, summary queue + dead-letter queue, Secrets Manager entries, and a scoped IAM key that can **read** the production backup bucket but write only to staging resources. Used from step 5.6.
- **Email:** an SES domain identity for russetleaf.dev, with its DKIM and SPF records created in Cloudflare, and SES SMTP credentials stored in Secrets Manager. (New SES accounts are in the "sandbox" and can only send to verified addresses; verify your own address for testing, or request production access.)
- Scoped IAM users and keys for the home cluster, one per job: External Secrets (read the russetleaf.dev secrets only), WordPress (media bucket + send to queue), MariaDB backups (backup bucket), summarizer (read from queue), LiteLLM (`bedrock:InvokeModel` on the chosen models only). Store each key in Secrets Manager.
- Cloudflare (Cloudflare provider): the tunnel, its token (stored in Secrets Manager), DNS records for `russetleaf.dev` and `www`, basic WAF/security settings, **Cloudflare Access** policies for admin UIs (Argo CD, Grafana) so they're never public, **Cloudflare notifications** (tunnel health, security events) to a Slack webhook for `#rl-dev-alerts`, and a `staging.russetleaf.dev` hostname behind Cloudflare Access (used from step 5.6).
- The Slack bot token in Secrets Manager (set by hand; Terraform creates the empty entry).
- Bedrock: confirm access to Claude Haiku and Amazon Nova Lite in the chosen region (Anthropic models may need a one-time use-case form in the console).

**Test:** with each scoped key, the allowed action works and a disallowed one fails (e.g. the WordPress key can't read the backup bucket); `aws bedrock-runtime converse` with the LiteLLM key returns a response; the tunnel shows as created in Cloudflare.

### Step 2.4: `wp-infra` CI
**Do:** GitHub Actions: `terraform fmt -check`, `validate`, `tflint`, **Checkov on the Terraform source and on the plan** (`terraform show -json` output), and `plan` on pull requests (plan posted as a comment). Applies to `envs/home` stay **local** for now (the Talos provider needs your home network); they can move to in-cluster runners after step 9.2.

**Test:** a pull request with a harmless change shows a passing check run and a plan comment; a deliberately insecure change (e.g. a public bucket) fails Checkov.

## Phase 3: GitOps foundation (`wp-gitops`)

### Step 3.1: Structure and Argo CD bootstrap
**Do:**
- Create the layout from `aws-wp.md`: `bootstrap/home/`, `apps/`, `platform/`, each component with `base/` and `overlays/home/`.
- Install Argo CD with Helm on the new cluster, then add Argo CD itself to `platform/argocd/` so it manages its own upgrades. Add **`platform/cilium/`** with the **same values** Terraform used, so Argo CD adopts Cilium without changing it.
- `bootstrap/home/`: the root "app of apps" that creates one Argo CD Application per component, with **sync waves** (Cilium and the `prometheus-operator-crds` chart first, then storage and secrets, then ingress, monitoring, operators, then apps).
- Expose the Argo CD UI only through Cloudflare Access (after step 4.3), or use `kubectl port-forward` until then.
- **Argo CD Notifications** to `#rl-deploys` (sync succeeded, sync failed, app degraded), using the Slack bot token. Until External Secrets exists (step 4.2), create the token Secret by hand; switch it to an `ExternalSecret` afterwards.
- CI: render every component's home overlay (Helm/Kustomize), validate with `kubeconform`, and run **Checkov on the rendered manifests** (what Argo CD would actually apply).

**Test:** apply the root app; Argo CD shows it `Synced`/`Healthy` with a trivial test component; a sync posts to `#rl-deploys`; a pull request that breaks a manifest fails CI.

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
- Put the Argo CD, Grafana and **Hubble UI** hostnames behind Cloudflare Access.

**Test:** a hello-world app at `https://russetleaf.dev` loads from outside your network (e.g. on mobile data); your router has no open ports; the Argo CD hostname asks for a Cloudflare Access login.

### Step 4.4: Monitoring
**Do:** `platform/monitoring/`: kube-prometheus-stack, trimmed for 8 GB: 7-day retention, 30–60 s scrape interval, resource limits on every part. Grafana behind Cloudflare Access. Then the alerting setup from `aws-wp.md`:
- Prometheus external labels `site=dev`, `cluster=talos-home`; `serviceMonitorSelectorNilUsesHelmValues: false` (and the PodMonitor and rule equivalents) so Prometheus finds ServiceMonitors in every namespace.
- **Cilium and Hubble metrics:** in `platform/cilium/` values, turn on the agent, operator and Hubble metrics and their **ServiceMonitors**, the Hubble metric set (`dns`, `drop`, `tcp`, `flow`, `port-distribution`, `httpV2` with a small label set), and the Cilium and Hubble **Grafana dashboards** (written to the `monitoring` namespace for Grafana's sidecar). Add the Cilium alert rules (`CiliumAgentNotReady`, `HubblePolicyDropsSpike`, `CiliumEndpointRegenerationFailing`, `HubbleDNSErrors`) with runbooks. Exact values are in `aws-wp.md` ([Hubble and Cilium metrics in Prometheus](aws-wp.md#hubble-and-cilium-metrics-in-prometheus)).
- **Alertmanager → Slack** with the bot token from External Secrets: criticals to `#rl-critical` and `#rl-dev-alerts`; warnings to `#rl-dev-alerts`; `team=security|mlops|cost` alerts to their channels; `env=staging` alerts to `#rl-staging` only; grouping and repeat intervals; **inhibition rules** (site down suppresses its downstream alerts).
- **Dead man's switch:** route the always-firing `Watchdog` alert to the heartbeat service's URL; the heartbeat service alerts `#rl-critical` if pings stop.
- **blackbox_exporter** (in `platform/monitoring/`) with probes for key URLs inside the cluster; start with a test target, then add `/`, `/wp-login.php` and `/wp-json/` once WordPress exists (step 5.2).
- **External uptime check** for `https://russetleaf.dev` (and, once the AWS edge exists, russetleaf.com expecting 200 or the 503 offline page).
- **Alerts as code:** a starter set of `PrometheusRule`s (site down, pod crash-looping, node memory high, disk filling, certificate/tunnel down), each with `severity`, `summary`, `description` and `runbook_url`; a runbook for each in `docs/runbooks/`; `promtool test rules` in `wp-gitops` CI.

**Test:** Grafana shows node and pod dashboards for both nodes, plus the Cilium and Hubble dashboards with live data; Prometheus' targets page lists the Cilium agent, operator and Hubble endpoints as up; a test critical alert appears in both `#rl-critical` and `#rl-dev-alerts` with a working runbook link; stopping Alertmanager for longer than the heartbeat period makes the heartbeat service post to `#rl-critical`; a broken alert rule fails `promtool` in CI.

### Step 4.5: Database and cache
**Do:**
- `platform/mariadb/`: MariaDB Operator; a `MariaDB` instance pinned to the **same major version RDS offers** (e.g. 11.4 LTS), a `Database`, `User` and `Grant` for WordPress, with the password **generated by the operator** into a Kubernetes Secret (no External Secrets for the database). Same for the backup user and root.
- **TLS** turned on in MariaDB (operator- or cert-manager-issued certificates) and **required** for the `wordpress` user.
- The daily **`Backup`** to the backup bucket (logical, gzip, 14-day retention), and an alert (`severity=critical`, with a runbook) if the last successful backup is more than 26 hours old.
- `platform/valkey/`: Valkey (single instance, no persistence needed for an object cache), with a **redis_exporter** sidecar and ServiceMonitor.
- **Database and cache metrics:** turn on MariaDB Operator's built-in metrics (mysqld-exporter sidecar and ServiceMonitor), and add alerts with runbooks (database down, connections near the limit, slow queries rising, Valkey memory near its limit).

**Test:** connect with a MariaDB client from a test pod using the generated Secret, and confirm a connection **without TLS is refused**; no database password appears in Git or Secrets Manager; trigger a backup by hand and see the file in S3; **restore it into a scratch MariaDB instance** and compare; `valkey-cli ping` returns `PONG`; MariaDB and Valkey metrics appear in Prometheus and on Grafana dashboards.

## Phase 5: WordPress

### Step 5.1: The WordPress image (`wp-image`)
**Do:**
- `Dockerfile` based on the official `wordpress` image (pinned version), with plugins and themes pinned in `plugins.json` and installed at build time: an S3 media offload plugin (see the comparison below), Redis Object Cache, an SMTP plugin (configured for Amazon SES). WordPress code is copied into `/var/www/html` **at build time**, so the image's startup script has nothing to copy at runtime.
- **Choose the S3 offload plugin** (see [Media storage: S3, not EFS](aws-wp.md#media-storage-s3-not-efs-by-decision)). Try both in the Compose setup against LocalStack (or MinIO) and record the choice in the decision log:
  - **S3-Uploads** (Human Made, open source): uploads go **directly to S3**, so no writable uploads directory is needed. Preferred if it works with a custom S3 endpoint (for the ci stand-ins) and with CloudFront as the media URL.
  - **WP Offload Media Lite**: writes locally first, then copies to S3; needs a writable uploads `emptyDir`.
  - Check for both: upload, image resizing (thumbnails), deleting media, and the media URL pointing at the right place (S3 at home, CloudFront on AWS).
- **Finish `docs/adr/0002-s3-media-not-efs.md`** (already drafted): fill in its open item with the plugin chosen and why, and whether the uploads `emptyDir` is needed.
- **Built to pass the Kyverno policies from step 6.1:** Apache runs as **`www-data`** listening on **port 8080** (non-root can't use port 80), and the container works with a **read-only root filesystem** given writable `emptyDir` volumes for `/tmp`, Apache's run and lock directories, PHP sessions, and the uploads directory only if the chosen plugin needs it. Test both in the Compose smoke test.
- `wp-config` settings read from environment variables (database, Valkey, S3, queue URL, `WP_HOME`/`WP_SITEURL`, `DISALLOW_FILE_MODS`, `DISABLE_WP_CRON`), with **TLS to the database** turned on.
- A **`db.php` drop-in** whose behaviour is chosen by the chart setting **`database.auth`**. Build only the **`password`** mode now (reads the password from a Secret, connects over TLS); it's what the home lab uses, and it's the AWS fallback too. Leave a clear extension point for an **`iam`** mode, which is written **only after** the AWS plan's early test (step 1.6) shows that PHP can log in to RDS Proxy with an IAM token. That way no work is wasted if the test fails. Add the **RDS CA certificate bundle** to the image now (harmless at home, needed on AWS).
- The **metrics must-use plugin** (`mu-plugins/metrics/`), as described in `aws-wp.md` ([WordPress monitoring](aws-wp.md#wordpress-monitoring)): `wordpress_posts_without_summary`, `wordpress_cron_overdue_events`, `wordpress_object_cache_hit_ratio`, `wordpress_info`, and simple counts, served on a **separate internal port** (a second Apache listener, e.g. 9117), cached for about a minute. Also turn on Apache's `mod_status`, reachable from localhost only (for the apache_exporter sidecar). Unit-test the output format.
- The **summary-queue must-use plugin**: on publish or update, send `{"post_id": ...}` to SQS (AWS SDK for PHP, installed with Composer at build time).
- `tests/`: a smoke test (Docker Compose with MariaDB: the container starts, plugins are active, health check passes).
- `chart/`: the WordPress Helm chart (see step 5.2 for what it deploys), versioned with the image.
- CI: **Checkov on the Dockerfile and the chart**, build, SBOM (Syft), scan (Trivy, failing on critical vulnerabilities), sign (cosign keyless), push the image and the chart (OCI) to GHCR with an immutable version.

- **Pulling private packages:** GHCR packages start **private** (even in a public repo), so until they're made public (step 11.2), the cluster needs credentials. Create a GitHub token with **`read:packages` only**, store it in Secrets Manager, and sync it with External Secrets into an **image pull secret** for the WordPress namespace (and, from step 7.4, the summarizer's), and into Argo CD's **OCI repository credentials** so it can pull the charts.

**Test:** the smoke test passes locally and in CI; `cosign verify` succeeds for the pushed image; the cluster can pull the private image and Argo CD can pull the private chart; publishing a post in the local Compose setup sends a message to the real SQS queue.

### Step 5.2: WordPress chart and deployment (`wp-gitops/apps/wordpress/`)
**Do:**
- The `wp-image` chart, which sets the pod security context (non-root user, read-only root filesystem, no privilege escalation, dropped capabilities), the `emptyDir` volumes above, **readiness and liveness probes**, the required labels (`app.kubernetes.io/name`, `app.kubernetes.io/part-of`, `env`) and **CPU/memory requests with a memory limit (no CPU limit)**, deployed from `wp-gitops` with the home overlay (values only): russetleaf.dev hostname, MariaDB and Valkey hosts, `database.auth: password` pointing at the **MariaDB Operator-generated Secret**, media bucket and queue settings, other credentials from External Secrets, resources, `persistence` off, WP-Cron as a Kubernetes CronJob.
- Remove the old test Application and the `wordpress-test` namespace.
- First-run setup (admin user, plugin settings) done by a one-off `wp-cli` Job or wp-config constants, so a rebuild needs no clicking.

- **Monitoring in the chart:** an **apache_exporter** sidecar; a `ServiceMonitor` for the exporter and the metrics port; `PrometheusRule` alerts with runbooks (`WordPressCronBacklog`, `WordPressObjectCacheHitRatioLow`, `WordPressApacheWorkersSaturated`, `WordPressSyntheticProbeFailing`; `WordPressPostsMissingSummaries` is added in step 7.4); a CiliumNetworkPolicy rule letting only Prometheus reach the metrics port. Add the WordPress URLs to the blackbox_exporter probes.

**Test:** `https://russetleaf.dev` serves WordPress; an uploaded image lands in the S3 bucket and displays; the Redis Object Cache plugin shows "connected"; killing the WordPress pod loses nothing; WP-Cron jobs run from the CronJob; a password-reset email arrives through SES; Prometheus scrapes the apache_exporter and the custom metrics; the metrics port is **not** reachable through `https://russetleaf.dev` or from a non-Prometheus pod; the blackbox probes are green.

### Step 5.3: Image promotion
**Do:** create the GitHub App; the `wp-image` release workflow opens a pull request on `wp-gitops` that changes only the chart and image version in `apps/wordpress/overlays/home`, linking the release notes. (Step 5.7 later points this at staging first.)

**Test:** merge a trivial change in `wp-image` → a signed image is pushed → a bump pull request appears in `wp-gitops` → merging it rolls out the new image, and the rollout appears in `#rl-deploys`. Subscribe `#rl-deploys` to the four repos' releases and merged pull requests with the GitHub Slack app.

### Step 5.4: ci test clusters (kind + AWS stand-ins)
**Do:**
- `wp-image/ci/`: a kind cluster config and `ci` chart values (LocalStack, or MinIO + ElasticMQ, per step 0.2; a MariaDB and Valkey installed into the kind cluster; test credentials).
- Integration tests (e.g. a shell or Python test script run after `helm install`): the site installs and loads; an upload lands in the stand-in S3 bucket; publishing a post sends a message to the stand-in SQS queue.
- A pull request workflow on **GitHub-hosted** runners: create the kind cluster (`helm/kind-action`), load the freshly built image, install the chart with `ct install` or `helm install`, run the tests, collect logs on failure.
- `compose.yaml` for the **local** environment: WordPress + MariaDB + Valkey + LocalStack, so the same checks can be run by hand.
- In `wp-gitops` CI: install changed **platform** components into a kind cluster to check they start (where they can run without the real cluster's dependencies).

**Test:** a pull request in `wp-image` runs the kind tests and passes; a deliberately broken change (e.g. a wrong S3 setting) fails them; `docker compose up` gives a working local site.

### Step 5.5: Self-hosted runners (ARC)
**Do:** `platform/actions-runner-controller/`: ARC with **ephemeral** runners, tolerating box 2's taint so they run off box 1. Start with a runner scale set for **`wp-gitops`** (used by the staging workflow); `llm-summarizer` gets its own in step 9.2. Before any repo goes public: require approval for workflows from fork pull requests, and limit which workflows can use the self-hosted label.

**Test:** a test workflow in `wp-gitops` runs on a self-hosted runner pod, which is deleted afterwards; a fork pull request waits for approval.

### Step 5.6: On-demand staging
**Do:**
- `overlays/home-staging` for WordPress (1 replica, `staging.russetleaf.dev`, no Valkey), a small staging **MariaDB** (MariaDB Operator, with its own generated passwords), and the staging AWS resources and secrets from step 2.3. Label everything `env=staging`.
- Staging Argo CD Applications in their own **Argo CD project**, with **auto-sync off**, so nothing runs until a test starts.
- `apps/staging-data/`: a Job that downloads the latest russetleaf.dev production backup from S3, imports it into the staging MariaDB, and **scrubs personal data** (user emails replaced with `user<ID>@example.invalid`, commenter names/emails/IPs cleared, application passwords and sessions deleted), then runs `wp search-replace 'russetleaf.dev' 'staging.russetleaf.dev'`.
- `apps/smoke-tests/`: a Job that checks the home page and sample posts load, an upload goes to the staging bucket, and a published post gets a summary (once Phase 7 exists; skip that check until then).
- An Argo CD account with RBAC limited to the staging project (sync and delete only); its token in Secrets Manager.
- The **`staging-test`** workflow (on the ARC runners): sync staging → wait for healthy → run `staging-data` → run `smoke-tests` → soak ~30 minutes checking Prometheus for errors and restarts → record results → **delete** the staging Applications with cascade (volumes included) → post results to `#rl-staging`.

**Test:** run `staging-test` by hand: staging comes up at `staging.russetleaf.dev` (behind Cloudflare Access), shows production posts with **no real email addresses** in the database, passes the smoke tests, and is fully removed afterwards (`kubectl get all,pvc -n <staging namespaces>` is empty). Record box 1's peak memory during the run in the checklist.

### Step 5.7: Promotion pipeline (staging → prod)
**Do:**
- Change the `wp-image` release bot (step 5.3) to bump **`overlays/home-staging`** instead of prod, with **auto-merge** once `wp-gitops` CI passes.
- Trigger `staging-test` on pushes to `main` that change a staging version.
- On a pass, a **`promote-to-prod`** job opens a pull request bumping the same version in `overlays/home` (and `overlays/aws` once it exists), with the staging results attached. On a failure, post to `#rl-staging` and `#rl-deploys` and promote nothing.

**Test:** release a trivial `wp-image` change → it auto-merges to staging → `staging-test` passes → a promotion pull request appears with results → merging it rolls out to russetleaf.dev. Release a deliberately broken version → staging fails → no promotion pull request, and `#rl-staging` shows why.

## Phase 6: Policy and progressive delivery

### Step 6.1: Kyverno
**Do:** `platform/kyverno/`: Kyverno plus the policy set from `aws-wp.md` ([Supply chain and policy](aws-wp.md#supply-chain-and-policy)), all in **audit mode** first:
- **Signed images** (`verifyImages`): images signed by the `wp-image` and `llm-summarizer` release workflows (cosign keyless, GitHub OIDC issuer, subject matching each repo's workflow), with **`mutateDigest`** so tags are pinned to digests at admission
- **Restrict Image Registries:** `ghcr.io/<you>/*` plus an explicit list of the upstream registries the platform charts use (include init and ephemeral containers)
- **Disallow Latest Tag**
- **Require Labels** (`app.kubernetes.io/name`, `app.kubernetes.io/part-of`, `env`), plus a **mutate** rule that adds `env` from the namespace
- **Require Requests and Limits:** CPU and memory requests and a memory limit; **no CPU limit required**
- **Pod Security Standards `baseline`** (includes Disallow Privileged Containers, host paths and host namespaces)
- **Require Run As Non-Root**
- **Require Read-Only Root Filesystem**
- **Require Pod Probes** (readiness and liveness on long-running workloads)
- **Disallow NodePort**, plus blocking `LoadBalancer` Services
- **Unique Ingress Host and Path**
- **`PolicyException`s**, each naming the exact namespace and workload, only where needed: Cilium, Falco, local-path-provisioner (privileged/host access); upstream images that can't run read-only or as non-root (e.g. some operators or llama.cpp's image, if so). The project's own workloads get **no** exceptions.
- Policy tests with `kyverno test` (a passing and a failing example per policy), run in `wp-gitops` CI.

- **Policy Reporter** (Kyverno's reporting add-on) sending new policy violations to `#rl-security`.

Review the policy reports, fix violations, then switch to **enforce**.

**Test:** in enforce mode, each of these is rejected: an unsigned image, an image from an unlisted registry, a `:latest` image, a pod without the required labels (where the mutate rule can't fill them), a container without a memory limit, a privileged pod, a root container, a writable root filesystem, a Deployment without probes, a NodePort Service, and an Ingress reusing an existing host and path. Running pods show image digests rather than tags. The existing workloads keep running; Policy Reporter shows no unexplained violations; a pull request that violates a policy fails CI.

### Step 6.2: Argo Rollouts
**Do:** `platform/argo-rollouts/`; convert WordPress to a `Rollout` with canary steps (e.g. 20% → 50% → 100%) and an `AnalysisTemplate` that queries Prometheus (ingress-nginx error rate and latency for the WordPress service). Argo Rollouts notifications post canary steps, aborts and rollbacks to `#rl-deploys`.

**Test:** a good image rolls out step by step; a deliberately broken image (e.g. one that returns 500s) is **rolled back automatically**, and the rollback is announced in `#rl-deploys`.

### Step 6.3: CiliumNetworkPolicies
**Do:** use **CiliumNetworkPolicies**, not plain NetworkPolicies (see [Policies](aws-wp.md#policies-ciliumnetworkpolicies-not-plain-networkpolicies) in `aws-wp.md`):
- A **`CiliumClusterwideNetworkPolicy`** baseline: default-deny for application namespaces, DNS to CoreDNS allowed with Cilium's **DNS proxy** on (needed for DNS-name rules), Prometheus scraping allowed.
- Per-component **CiliumNetworkPolicies** in each component's `base/` (shared with the AWS site), with site-specific destinations in the overlays. Outbound traffic to AWS is allowed **by DNS name** (`toFQDNs`): WordPress → S3, SQS, SES and STS endpoints; summarizer → SQS, the gateway and WordPress's REST API (a **layer-7** rule: `GET`/`POST` on `/wp-json/*` only); gateway → llama.cpp and `bedrock-runtime`; llama.cpp's init container → the model mirror bucket only; MariaDB → S3 for backups; cloudflared → Cloudflare's tunnel endpoints.
- A **Kyverno generate rule** that gives every new application namespace the baseline allowances, so none is ever left open.
- CI: validate the policies against the Cilium CRD schemas, and exercise them in the ci kind cluster (install Cilium there too).
- Roll out one namespace at a time, watching `hubble observe --verdict DROPPED` for anything legitimate being blocked.

**Test:** from a test pod in another namespace, MariaDB is unreachable; from a WordPress pod, `curl https://example.com` is **blocked** while uploads to S3 still work; a non-REST-API request from the summarizer to WordPress is blocked at layer 7; a burst of blocked requests triggers `HubblePolicyDropsSpike` in `#rl-security`; the site, the summarizer, backups and monitoring all still work.

### Step 6.4: Service mesh decision (ADR) and optional Istio experiment
**Do:**
- Finish `docs/adr/0001-no-service-mesh.md` (already drafted; review it and resolve its open items as they're done): what a mesh like Istio would add, how Cilium (WireGuard, layer-7 policies, Hubble), Argo Rollouts and LiteLLM already cover it, the memory and complexity cost on 8 GB nodes, and **when you'd revisit** (many more services, a requirement to verify service identity on every connection, multi-cluster traffic). See [Service mesh (Istio): not used, by decision](aws-wp.md#service-mesh-istio-not-used-by-decision).
- **Optional experiment (local only):** in a kind cluster on your workstation running Cilium, install **Istio ambient mode** (following Istio's instructions for running alongside Cilium), deploy the WordPress chart with test values, and enable mTLS and one layer-7 authorization policy. Record memory use, setup issues and what it added in the ADR. Delete the cluster afterwards; nothing in either site depends on it.

**Test:** the ADR is reviewed and linked from the hub README and the decision log; if the experiment was run, its findings are in the ADR.

## Phase 7: AI post summaries

### Step 7.1: KEDA
**Do:** `platform/keda/`: KEDA, with AWS credentials for the SQS scaler from External Secrets.

**Test:** a test `ScaledObject` scales a dummy Deployment from 0 to 1 when a message is sent to a test queue, and back to 0 when the queue is empty.

### Step 7.2: LiteLLM gateway
**Do:** `platform/llm-gateway/`: LiteLLM with Claude Haiku (default) and Amazon Nova Lite on Bedrock, using the scoped Bedrock key; per-app budgets and rate limits (a separate key and small budget for staging); metrics exposed to Prometheus; an internal API key for callers.

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
- `ci/`: kind end-to-end tests on GitHub-hosted runners, with the AWS stand-ins (SQS) and a **mock LLM** (LiteLLM with mock responses): a queued post gets a summary written back to a test WordPress; a failing message reaches the dead-letter queue.
- **Checkov on the summarizer's Dockerfile and chart** in pull request CI.
- Release workflow: build, SBOM, scan, sign, push the image and the chart (OCI) to GHCR, open a chart-bump pull request on `wp-gitops` for **`overlays/home-staging`** (promoted to prod by the step 5.7 pipeline).
- `wp-gitops/apps/post-summarizer/overlays/home` and `overlays/home-staging`: queue URL, gateway URL (staging uses its own LiteLLM key and budget), WordPress URL, credentials from External Secrets.
- Turn on the summarizer check in the staging smoke tests (step 5.6).

Then turn on the **`WordPressPostsMissingSummaries`** alert (from the custom metrics plugin), routed to `#rl-mlops`.

**Test (end to end):** the kind tests pass in CI; a release goes through staging and is promoted; then publish a post on russetleaf.dev → a summarizer pod starts → the summary appears on the post within a few minutes → the pod scales back to zero. A message that always fails ends up in the dead-letter queue.

### Step 7.5: LLM monitoring
**Do:** OpenTelemetry metrics in the summarizer (tokens, latency, errors, fallbacks, estimated cost, all labelled by model and prompt version); the Grafana LLM dashboard and `PrometheusRule` alerts shipped **in the summarizer chart** (error spike, dead-letter queue not empty, daily Bedrock spend over a limit, all labelled `team=mlops` so they go to `#rl-mlops`, with runbooks); SQS queue depth in Grafana.

**Test:** the dashboard shows real numbers after a few posts; temporarily lowering the spend limit makes the alert appear in `#rl-mlops`.

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

### Step 9.2: Runners for `llm-summarizer`
**Do:** add a second ARC runner scale set (step 5.5) registered to **`llm-summarizer`**, on box 2 next to the model, with the same protections (approval for fork pull requests, self-hosted label limited to the evaluation workflows).

**Test:** a test workflow runs on a self-hosted runner pod, which is deleted afterwards; a fork pull request waits for approval.

### Step 9.3: Evaluation gate and releases
**Do:**
- Pull request workflow (on ARC runners): run the evaluation, post per-model scores as a pull request comment compared with `main`, fail below the threshold.
- Release workflow: run the gate again, then publish (step 7.4, into staging first), attach the report to the GitHub release, update `MODEL_CARD.md`.
- Post gate results (pass/fail, per-model scores, link to the report) to `#rl-mlops`.
- Optionally convert the summarizer to an Argo Rollouts canary.

**Test:** a pull request with a deliberately bad prompt fails the gate with a clear comment; a good one passes; a release tag publishes only after the gate passes.

## Phase 10: Runtime security, compliance and cost

### Step 10.1: Falco
**Do:** `platform/falco/`: Falco with the **modern eBPF** driver (Talos doesn't allow kernel modules), Falcosidekick to Alertmanager, Grafana and Slack (`#rl-security`, warning priority and above), and custom rules: shell in WordPress, summarizer or gateway pods; writes under `/var/www/html`; unexpected outbound connections from box 2.

**Test:** `kubectl exec` into a WordPress pod triggers an alert in Grafana and `#rl-security`; creating a file under `/var/www/html` triggers another.

### Step 10.2: Trivy Operator, Kubescape scan and Talos hardening notes
**Do:** `platform/trivy-operator/` with modest scan concurrency (it's memory hungry on 8 GB); write `docs/talos-hardening.md` explaining how Talos meets the CIS benchmark (no shell or SSH, read-only system, API-only management, default pod security). Run the first **Kubescape CLI** scan of the home cluster from the workstation (NSA-CISA and MITRE ATT&CK frameworks, plus RBAC analysis; nothing installed in the cluster), save the report and a short summary of accepted findings in `docs/security/`, and fill in the baseline in `docs/adr/0003-kubescape-cli-not-operator.md`.

**Test:** vulnerability and compliance reports exist for the running workloads; the hardening doc is reviewed; the Kubescape report is committed, and any high-risk RBAC findings (e.g. a service account that can read all Secrets cluster-wide) are fixed or explained.

### Step 10.3: OpenCost
**Do:** `platform/opencost/` using the existing Prometheus, with custom prices for the home hardware (electricity per node) and the Bedrock spend shown alongside.

**Test:** OpenCost shows cost per namespace, with box 2 attributed to the LLM namespace.


### Step 10.4 (optional): Request tracing
**Do:** add OpenTelemetry's PHP auto-instrumentation (PHP extension and SDK) to the WordPress image, switched off by default with an environment variable; install **Grafana Tempo** (single-binary, small retention) only while investigating; link traces from Grafana. Record the memory cost in the decision log.

**Test:** with tracing on, a slow page's trace shows its database queries and outgoing calls in Grafana; with it off, there's no measurable overhead; Tempo is removed afterwards if memory is tight.

## Phase 11: Optional and finishing

### Step 11.1 (optional): Keyless AWS access for pods
**Do:** publish the Talos cluster's service-account OIDC discovery files (public S3 bucket), register it as an IAM OIDC provider in `wp-infra`, switch pods from scoped keys to IAM roles one at a time, then delete the keys.

**Test:** each pod works with no AWS key in its Secrets; the old keys are deleted from IAM.

### Step 11.2: Documentation and polish
**Do:** hub README with the russetleaf.dev architecture diagram and links to all repos; screenshots of Grafana (site, LLM, cost), Argo CD and an evaluation report; update `aws-wp.md` wherever the build differed from the design; check every repo with `gitleaks`, then make them public; **also make each GHCR package public** in its package settings (a repo going public doesn't change its packages), then remove the image pull secrets and Argo CD's GHCR credentials and confirm everything still pulls; and switch on **SARIF upload** of Checkov's results to GitHub code scanning in each (free for public repos), so findings appear on pull requests and in the Security tab.

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
| Check staging still works | Monthly, if nothing has been promoted | Run `staging-test` by hand, so a broken staging setup is found before it blocks a real release |
| Check the staging scrub | After any plugin that stores personal data is added | Confirm the `staging-data` Job also clears that plugin's tables |
| Kubescape posture scan | Monthly | Run the Kubescape CLI against the home cluster; save the report in `docs/security/`; compare scores with the previous run and investigate any drop |
| Rotate MariaDB passwords | Quarterly | Regenerate the operator-managed Secret (or use the operator's rotation feature), then restart WordPress through its Rollout; check the backup user still works |
| Rotate scoped IAM keys | Every 90 days, until step 11.1 removes them | Create a new key in Terraform, update Secrets Manager, let External Secrets sync it, delete the old key |
| Review alerts, costs and Bedrock spend | Monthly | Grafana, OpenCost, the AWS Budget and Cost Explorer (filter by `Site=dev`) |
| Review alert noise | Monthly | Look at what fired in the Slack channels: remove or tune alerts nobody acted on, and update runbooks that didn't help |
| Test the outside monitoring | Quarterly | Pause the heartbeat or block the uptime check and confirm `#rl-critical` is notified |

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
| 4.4 Monitoring + Slack alerting | ☐ | | | |
| 4.5 MariaDB + backups + Valkey | ☐ | | | |
| 5.1 WordPress image | ☐ | | | |
| 5.2 WordPress deployment | ☐ | | | |
| 5.3 Image promotion | ☐ | | | |
| 5.4 ci test clusters | ☐ | | | |
| 5.5 Self-hosted runners (ARC) | ☐ | | | |
| 5.6 On-demand staging | ☐ | | | |
| 5.7 Promotion pipeline | ☐ | | | |
| 6.1 Kyverno | ☐ | | | |
| 6.2 Argo Rollouts | ☐ | | | |
| 6.3 CiliumNetworkPolicies | ☐ | | | |
| 6.4 Service mesh ADR (+ optional Istio experiment) | ☐ | | | |
| 7.1 KEDA | ☐ | | | |
| 7.2 LiteLLM gateway | ☐ | | | |
| 7.3 Summarizer code | ☐ | | | |
| 7.4 Summarizer chart + deploy | ☐ | | | |
| 7.5 LLM monitoring | ☐ | | | |
| 8.1 llama.cpp on box 2 | ☐ | | | |
| 8.2 Gateway routing to local model | ☐ | | | |
| 9.1 Evaluation set | ☐ | | | |
| 9.2 Runners for `llm-summarizer` | ☐ | | | |
| 9.3 Evaluation gate + releases | ☐ | | | |
| 10.1 Falco | ☐ | | | |
| 10.2 Trivy Operator, Kubescape scan, Talos notes | ☐ | | | |
| 10.3 OpenCost | ☐ | | | |
| 10.4 Request tracing (optional) | ☐ | | | |
| 11.1 Keyless AWS access (optional) | ☐ | | | |
| 11.2 Docs + go public | ☐ | | | |

## Decision log

| Date | Decision | Reason |
|---|---|---|
| | | |
