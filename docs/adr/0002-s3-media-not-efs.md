# ADR 0002: Media on S3, code in the image (not EFS)

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
WordPress needs somewhere to keep `wp-content`: uploaded media, plus (traditionally) plugins and themes. When WordPress runs on several servers or pods, they must all see the same files. Many WordPress-on-AWS guides, including AWS's own reference architecture for WordPress on EC2 Auto Scaling groups, put `wp-content` on **Amazon EFS**, a shared network filesystem.

This project is containerised and GitOps-managed. Images are signed and pinned by digest, Kyverno requires a read-only root filesystem, Falco alerts on writes to `/var/www/html`, and every change is promoted through ci and staging. The home lab has no EFS, and box 1 has little spare memory.

## Decision
- **WordPress code** (core, plugins, themes) is **built into the container image** in the `wp-image` repo. Plugin installs and updates from the admin screen are disabled (`DISALLOW_FILE_MODS`).
- **Media uploads go to S3** through an offload plugin, served through CloudFront on russetleaf.com and from S3 at home.
- **No shared filesystem** is used on either site.

This is the pattern used by large containerised WordPress platforms such as WordPress VIP and Human Made's Altis.

## Alternatives considered

| | EFS (shared `wp-content`) | S3 media + code in the image (chosen) |
|---|---|---|
| What's running | Can change at runtime (admin installs, plugin updates, edited files), drifting from Git | Exactly what's in Git: a signed, scanned image pinned by digest |
| Rollback | Rolling back the image doesn't roll back files changed on EFS | Roll back the image version |
| Security | A writable code directory is where attackers hide web shells, which persist across restarts | Read-only root filesystem; a compromised pod is replaced clean |
| Performance | PHP reads many small files per request over the network; needs careful OPcache tuning | Code on local disk; media served by the CDN from S3 |
| Cost | ~$0.30/GB-month, plus throughput charges at scale | ~$0.023/GB-month |
| Home lab | Needs an NFS server, adding load on 8 GB nodes | Same S3 approach at home |
| Environments | A shared filesystem per environment, kept in sync | Each environment points at its own bucket |

Also considered:
- **EFS for uploads only** (code in the image, media on EFS): avoids an offload plugin, but keeps EFS's cost, needs NFS at home, and loses CDN-native media delivery.
- **A persistent volume per pod:** doesn't work with several replicas, since each pod would see different files.

## Consequences
- **No plugin installs from the admin screen.** Every plugin change is a new image built by CI and promoted through staging. Intended here; a real drawback for sites where editors manage plugins themselves.
- **Plugins that write files** (some caching, form and backup plugins) must be checked before adoption, or given a writable scratch volume.
- **The offload plugin is a key dependency** and must work with a custom S3 endpoint, so ci tests can use the LocalStack or MinIO stand-ins.
- Rollbacks, security scanning and promotion all work on a single artifact: the image.

## When to revisit
- Non-technical editors need to install or update plugins themselves.
- The site comes to depend on plugins that must write to `wp-content` and have no S3-compatible alternative.
- A move away from containers to a traditional server-based setup.

## Open items
- **Offload plugin choice** (home plan step 5.1): compare **S3-Uploads** (Human Made; writes directly to S3, so no writable uploads folder) and **WP Offload Media Lite** (writes locally first, then copies to S3). Check uploads, thumbnails, deletion, the media URL (S3 at home, CloudFront on AWS) and custom S3 endpoint support. Record the choice and reasons here, and whether the uploads `emptyDir` is needed.
