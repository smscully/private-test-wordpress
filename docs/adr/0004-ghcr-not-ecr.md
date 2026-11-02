# ADR 0004: GitHub Container Registry (GHCR) instead of Amazon ECR

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
The project builds two images, each with a Helm chart published as an OCI artifact: the WordPress image (`wp-image`) and the post-summarizer (`llm-summarizer`). Both sites must pull them:
- **russetleaf.dev** runs 24/7 on a home Talos cluster, **outside AWS**.
- **russetleaf.com** runs on EKS only during demos.

Images are built and signed (cosign keyless) in GitHub Actions, and Kyverno only admits images signed by those workflows and pulled from allowed registries. The repos are private while being built and become public later (home plan step 11.2).

## Decision
Use **GHCR** for all project images and charts: `ghcr.io/<github-user>/wp-image` and `ghcr.io/<github-user>/llm-summarizer`. Both sites pull from GHCR. On AWS, an **ECR pull-through cache** in front of GHCR is optional.

Reasons, in order of importance:
1. **The home cluster runs outside AWS.** Private ECR needs AWS credentials, and ECR login tokens **expire every 12 hours**, so the home cluster would need a job or credential helper refreshing its pull secret, which can quietly break pulls. Public GHCR packages need no credentials.
2. **Simpler CI and signing.** GitHub Actions pushes to GHCR with its built-in token, and cosign keyless signing uses GitHub's identity directly. ECR would need an AWS role per repo for pushing.
3. **One registry for both sites:** identical image references everywhere and a single Kyverno registry rule.
4. **Visible to employers:** packages appear on each repo's GitHub page, next to the code that built them.
5. **Cost (minor):** public GHCR packages are free; ECR would cost about $0.10/GB-month of storage plus data-transfer charges for pulls from home: a few dollars a month at most.

## Alternatives considered
- **Amazon ECR only:** the usual choice for production workloads on AWS, with IAM-controlled access, Amazon Inspector scanning, VPC endpoints (EKS pulls never leave AWS) and lifecycle policies. Rejected here mainly because of the 12-hour token refresh for the home cluster.
- **Both registries (build once, push to both):** keeps each site on its "native" registry, but doubles the image references, signing identities and Kyverno rules to maintain.
- **Docker Hub:** pull rate limits for anonymous and free accounts, and no advantage over GHCR for this project.

## Consequences
- **Package visibility is separate from repo visibility.** GHCR packages start **private** and stay private when their repo is made public, until changed in each package's settings. While private, both clusters need an image pull secret and Argo CD needs OCI credentials (a GitHub token with `read:packages` only, in Secrets Manager, synced by External Secrets).
- **Free-plan limits for private packages.** Until the packages are public, GitHub's free allowance for private packages applies, roughly **500 MB of storage** and limited monthly data transfer (check GitHub's current figures). A WordPress image with several versions can exceed that quickly. Mitigations:
  - make the **packages** public early (they contain no secrets, even while the repos are private), and/or
  - delete old **untagged** versions automatically in CI (a retention step after each release), keeping tagged releases.
- **Dependency on GitHub's availability** for pulls. On AWS, the optional ECR pull-through cache also covers short GitHub outages.
- No AWS-native image scanning (Amazon Inspector); Trivy in CI and Trivy Operator in the clusters cover vulnerability scanning instead.

## When to revisit
- A requirement that images stay **private inside AWS** (e.g. a compliance rule), or that EKS pulls never leave the VPC.
- The home cluster moving into AWS (removing the token-refresh problem).
- GitHub pull limits, outages or pricing changes affecting deployments.

## Open items
- Decide in home plan step 5.1 whether to make the packages public at first push, or keep them private with a retention rule until step 11.2; record the choice here.
- AWS plan step 0.2: decide whether to add the ECR pull-through cache.
