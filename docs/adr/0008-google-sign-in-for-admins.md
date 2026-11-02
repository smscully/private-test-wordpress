# ADR 0008: Google sign-in (OIDC) for WordPress admins, plus an edge login layer

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
Only admins and editors log in to WordPress; site visitors don't. By default WordPress uses its own username/password logins at `/wp-login.php`, which is a constant target for password-guessing and credential-stuffing attacks. The project already avoids human-handled passwords elsewhere (IAM auth for the database on AWS, generated passwords at home) and already uses Cloudflare Access for admin tools on the home site. Constraints: no cost, both sites working the same way, no plugin installs from the admin screen (ADR 0002), and the post-summarizer must keep logging in to the REST API with an application password.

## Decision
- **WordPress login with Google via OpenID Connect**, using a generic OIDC client plugin built into the image and configured from environment variables and secrets. One Google OAuth client per site (plus one for local testing).
- **Allow-list** of email addresses (or one Google Workspace domain).
- **Password logins turned off** for people; **application passwords kept** for the REST API (the summarizer).
- **An edge login layer** in front of `/wp-admin/*` and `/wp-login.php`:
  - russetleaf.dev: **Cloudflare Access** with Google as its login provider
  - russetleaf.com: the **ALB's built-in OIDC authentication** with Google
- **Break-glass** admin recovery with `wp-cli` inside the cluster, never by re-enabling password logins.

## Alternatives considered
- **Keep WordPress passwords (with a 2FA plugin):** still exposes a password login page to the internet, and passwords must be managed.
- **Edge layer only (Cloudflare Access / ALB OIDC) without WordPress OIDC:** strong protection, but WordPress would still need its own passwords behind it.
- **WordPress OIDC only, no edge layer:** removes passwords, but the login page and admin area stay reachable by anyone.
- **Amazon Cognito** (federated with Google): adds a user pool and configuration with no benefit for a handful of admins, and doesn't cover the home site.
- **Freemium "social login" plugins:** often lock features behind paid tiers and add more code than needed.

## Consequences
- No human WordPress passwords on either site; the admin area isn't reachable without a Google login at the edge.
- Admins sign in twice (edge, then WordPress), both with Google and usually a single click the second time.
- New dependency on Google's availability for admin access (break-glass via `wp-cli` covers outages).
- Extra configuration: OAuth clients per site (created by hand in the Google Cloud console and documented), CiliumNetworkPolicy egress to Google, uncached admin paths in CloudFront, a mock OIDC provider in ci.
- The summarizer is unaffected, as long as application passwords stay enabled.

## When to revisit
- Visitors need accounts (e.g. members or comments with login): consider a customer-facing identity service.
- An organisation moves to a different identity provider (e.g. Microsoft Entra ID or Okta): the OIDC plugin and edge layers support other providers with configuration changes.
- More than 50 users would need Cloudflare Access (beyond its free tier).

## Open items
- Home plan step 5.8: choose and verify the OIDC plugin (maintenance, environment-variable configuration, application passwords unaffected); record the choice here.
- AWS plan step 3.1: confirm the ALB OIDC rule works behind CloudFront (cookies forwarded, admin paths uncached).
