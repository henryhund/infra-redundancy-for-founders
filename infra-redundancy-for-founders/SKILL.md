---
name: infra-redundancy-for-founders
description: Help founders and small teams make their app survive a hosting provider breach or outage (leaked env vars, paused deploys) with simple, low-cost redundancy, and guide them through an active incident. Use when someone asks about infra redundancy, a backup host, Cloudflare failover, disaster recovery, secrets rotation, whether their app is ready for a host outage, or their host (Railway, Render, Fly, Vercel, Heroku) just had a security incident.
---

# Infra Redundancy for Founders

Goal: if the app's host leaks every secret or stops deploying, recovery should take about an hour, not a week, without building an enterprise multi-cloud setup.

Who this is for: founders and small teams, often building with AI coding tools, who may not have an infrastructure background. Explain each recommendation in plain language (what it is, why it matters, roughly how long it takes), avoid unexplained jargon, and do the setup work for them where possible instead of handing over instructions.

Scope: this skill is a starting point for early-stage and small projects. If the app is relied on by paying customers, handles payments, or stores sensitive or regulated data, say once (not repeatedly) that the company should also invest in disaster recovery and incident response planning led by experienced SRE staff and directed by a CISO or equivalent security leader. Then continue helping.

Core assumptions to hold throughout:
- Any secret stored on a platform can leak with that platform.
- Any platform's deploy pipeline can be frozen for a day or more.
- Prevention is mostly about limiting what a stolen key can do and being able to run somewhere else.

First decide which mode applies:
- **Incident mode**: the user's host has announced a breach or outage that affects them now. Go straight to "Incident response".
- **Readiness mode**: anything else. Run the audit, then help fix gaps in priority order.

## Readiness mode

### Step 1: Inventory secrets

If a codebase is available, find every secret the app uses:
- Search for environment variable reads (`process.env.`, `os.environ`, `os.getenv`, `ENV[`, `System.getenv`, `env::var`) and `.env.example`, `docker-compose`, CI workflow, and platform config files (`railway.json`, `render.yaml`, `fly.toml`, `vercel.json`).
- Never print secret values. Work with names only.

If there is no codebase, ask the user to list the services the app talks to.

Produce a table with one row per secret: name, service it belongs to, what it can do (read, write, spend money, admin), where it is stored today, how to rotate it, and risk (high = money, customer data, or infrastructure control; low = everything else). Recommend the user keep this inventory somewhere other than the host.

### Step 2: Audit against the checklist

Check each item, mark it done, missing, or unknown, and explain gaps in one line each. Items are in priority order.

1. **Blast radius of keys**
   - Hard spend limits and usage alerts on AI provider keys.
   - Restricted Stripe keys scoped to what the app uses.
   - Least-privilege IAM for cloud keys, never admin or root.
   - Separate production and development keys.
   - Where the service allows it, separate keys for the primary and backup hosts, so a leak from one host does not force rotating the other's.
   - No crypto wallet private keys or seed phrases in env vars. Treat this as urgent if found.
2. **Database independence**
   - Database hosted with a different provider than the app (Neon, Supabase, Crunchy Data, RDS, etc.).
   - Point-in-time recovery enabled and one restore actually tested.
   - TLS required. Access restricted to the hosts' static outbound IPs where the plan supports it.
3. **Portability**
   - App builds from a Dockerfile with no host-specific build steps.
   - Deploys run from CI (for example GitHub Actions), so the same commit can go to any host.
   - A backup deployment exists on a second host and points at the same database. Do not suggest cross-provider database replication; one independent database shared by both hosts is simpler and cheaper.
   - DNS is on Cloudflare (or similar) so failover is one record change. With Cloudflare's proxy on, the switch takes effect in seconds; for unproxied records, keep the TTL low (60 to 300 seconds) so clients pick up the change quickly.
   - The backup host has a current copy of every secret the app needs, so it can actually serve traffic on the day it is needed.
4. **Fewer static secrets**
   - OIDC or workload identity for AWS/GCP instead of long-lived access keys, where the host supports it.
   - Optional: a secrets manager (Doppler, Infisical, 1Password service accounts) for central revocation and an access log. Explain honestly that the bootstrap token stored on the host can still leak; the gains are fast revocation and knowing whether secrets were read. Do not recommend integrations that sync secrets back into the host's env vars, since that puts them back in the blast radius.
5. **Early warning**
   - Billing alerts on cloud and AI accounts.
   - Subscribed to the host's status page and security notices.
   - Uptime monitor on the main domain.
   - 2FA on all provider accounts, with recovery codes stored offline.

### Step 3: Fix gaps

Offer to do the work, starting with the highest-priority gap. Things you can typically produce:
- A Dockerfile if missing.
- A GitHub Actions workflow that deploys the same commit to the primary and backup hosts (backup can be manual-trigger only).
- A written Cloudflare failover procedure for the user's domain.
- A rotation runbook generated from the inventory, one section per high-risk secret.

Keep recommendations proportionate. A solo builder with a small app needs the first three checklist sections, not an enterprise program. Prefer managed Postgres like Neon or Supabase over RDS unless the user is already on AWS or has a compliance need, because RDS adds networking, IAM and egress cost work.

## Incident response

Work through these in order and keep the user moving. Rotating the dangerous keys matters more than perfect analysis.

1. **Confirm scope.** Read the provider's notice. Until the user knows they were unaffected, treat every secret on that host as exposed.
2. **Rotate high-risk keys at the source.** Revoke and reissue in the issuing service's own dashboard (AWS, Stripe, AI providers, database), not in the host. Order: anything that can move money or crypto, then cloud infrastructure, then customer data access, then the rest. Also secure the host account itself: change its password, check 2FA, revoke its API and deploy tokens, and review the GitHub app or OAuth access it has to the user's repositories. Warn the user before rotating session or JWT signing secrets that every signed-in user will be logged out, and before rotating webhook signing secrets that the sender (for example Stripe) must be updated at the same time.
3. **Check for misuse.** AWS CloudTrail, Stripe request logs, AI provider usage dashboards, database logs, and the secrets manager access log if one is used. Look for unfamiliar IPs and spikes.
4. **Get new values running.**
   - If the host can deploy: update variables, then redeploy or restart every service that uses them. Running containers keep old values until restart.
   - If the host's deploys are frozen: deploy the same commit with the new secrets to the backup host, verify it, then switch DNS. If no backup host exists, help stand one up now on Render, Fly, or similar from the Dockerfile.
   - Either way, update the new values on every host that holds them, including the backup, and confirm the old values no longer work.
5. **Rotate low-risk keys** from the rest of the inventory.
6. **Communicate.** If customer data or payments may be affected, help draft a customer notice and suggest the user check their legal notification obligations. Do not give legal conclusions.

When a secrets manager's logs show a bootstrap token was never used, revoking just that token can be reasonable, but only if the logs cover the full exposure window (which may start well before the provider's announcement), show individual reads, and the token was the only secret on the host. Still recommend rotating the high-value keys, since that costs an hour and being wrong costs much more.

## Quarterly drill

Recommend the user schedule this after hours, at a low-traffic time they choose, so the first failover is never during a real incident:
- Deploy to the backup host and switch DNS briefly, then switch back.
- Rotate one real key end to end and time it.
- Do a test database restore.
- Confirm the backup host's secrets are current.
- Update the inventory and runbook with anything that surprised them.
