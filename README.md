# Infra Redundancy for Founders

A Claude skill that helps founders and small teams keep their app running when their hosting provider has a security incident or outage. It audits your app, fixes the gaps with you, and walks you through response if an incident is happening right now.

Built after the October 2026 Railway security incident, where customers were told to rotate every secret while deployments were paused, so many teams could not ship the new credentials. The same lessons apply to Render, Fly, Vercel, Heroku and any other platform that stores your secrets and runs your deploys.

This repo accompanies my post on the incident, on [LinkedIn](https://www.linkedin.com/feed/update/urn:li:share:7519142574809350144) and [X](https://x.com/henryhund/status/2113378882753646934).

> [!IMPORTANT]
> **This skill is a starting point, not a substitute for professional security and reliability work.**
>
> It is designed for early-stage projects, side projects and small teams that have no dedicated infrastructure staff. If your application is relied on by customers, handles payments, or stores sensitive information (personal data, health or financial records, credentials, or anything subject to regulation or contractual obligations), your company should invest in disaster recovery planning and incident response preparation led by experienced Site Reliability Engineering (SRE) staff and directed by a Chief Information Security Officer (CISO) or equivalent security leader. That includes formal recovery objectives, tested runbooks, access reviews, logging and alerting, legal and customer notification procedures, and regular exercises.
>
> Nothing in this skill is legal, compliance or security certification advice. Review every change it proposes before applying it to production, and never paste secret values into a chat.
>
> **No warranty.** This skill is provided "as is", without warranty of any kind, under the [MIT License](LICENSE). You are responsible for how you use it and for any changes made to your systems.

## The idea

Assume your host will eventually leak your secrets or stop deploying, and set things up so that day costs you about an hour instead of a week. You do not need an enterprise multi-cloud architecture to get there. Five things cover most of it:

1. **Limit what any stolen key can do.** Spend caps on AI provider keys, restricted Stripe keys, least-privilege cloud roles, separate production and development keys.
2. **Keep a secrets inventory.** What each secret is, where it was issued, how to rotate it, and how risky it is. Store it somewhere other than your host.
3. **Host your database somewhere else.** A managed Postgres provider such as Neon, Supabase or Crunchy Data, with point-in-time recovery and a tested restore. These are easier than RDS for most small teams.
4. **Make the app portable.** A Dockerfile, deploys from GitHub Actions, and DNS on Cloudflare.
5. **Keep a cold backup on a second host,** pointed at the same database, and rehearse switching to it. If you run on Railway, keep the backup on Render, and vice versa.

## What the skill does

The skill has two modes. Claude picks the right one based on what you ask.

**Readiness mode** (the default)

1. **Inventories your secrets.** If you are working in a codebase, Claude finds every environment variable and platform config file the app uses. It works with secret names only and never prints values. It produces a table of each secret, what it can do, where it lives, how to rotate it, and its risk level.
2. **Audits your setup** against a prioritized checklist: blast radius of keys, database independence, portability, fewer static secrets (OIDC for AWS and Google Cloud, optional secrets managers such as Doppler or Infisical), and early warning (billing alerts, status page subscriptions, uptime monitoring, 2FA).
3. **Fixes the gaps with you,** highest priority first. It can write a Dockerfile, a GitHub Actions workflow that deploys to a primary and backup host, a Cloudflare failover procedure, and a rotation runbook built from your inventory.

**Incident mode** (when your host is having a breach or outage now)

1. Confirm scope, treating every secret on the host as exposed until you know otherwise.
2. Rotate high-risk keys at the source (money, cloud infrastructure, customer data first).
3. Check AWS, Stripe, AI provider and database logs for misuse.
4. Get the new secrets running: redeploy if the host allows it, otherwise deploy to the backup host and switch DNS. Update the backup host too, so it is not left holding revoked keys.
5. Rotate the remaining low-risk keys.
6. Help draft a customer notice if customer data or payments may be affected.

It also recommends a quarterly drill, run after hours at a time you choose rather than during a real incident: fail over to the backup host and back, rotate one real key end to end, test a database restore, and confirm the backup host's secrets are current.

## Install

**Claude apps (claude.ai and desktop)**

1. Download this repository and zip the inner `infra-redundancy-for-founders` folder, so the zip contains that folder with `SKILL.md` inside it:

   ```bash
   zip -r infra-redundancy-for-founders.zip infra-redundancy-for-founders
   ```
2. Upload the zip in the Skills section of Claude's settings.

**Claude Code**

Copy the skill folder into your personal skills directory:

```bash
git clone https://github.com/henryhund/infra-redundancy-for-founders.git
mkdir -p ~/.claude/skills
cp -R infra-redundancy-for-founders/infra-redundancy-for-founders ~/.claude/skills/
```

Or copy it into a project's `.claude/skills/` folder to use it only in that project.

**Codex and other agents**

The skill is a plain `SKILL.md` file with YAML frontmatter. Codex and other coding agents that support the same skills format can load it from their own skills directory.

## Use

Once installed, Claude uses the skill automatically when your request matches. Some prompts to try:

- "Is my app ready if Railway goes down or leaks my secrets?"
- "Audit this repo for infra redundancy and fix the biggest gaps."
- "Set up a cold backup of this app on Render with Cloudflare failover."
- "Build me a secrets inventory and rotation runbook for this project."
- "Our host just announced a security incident. Walk me through what to do."

Run it from inside your project's codebase in Claude Code for the best results, since it can then find your secrets and write the setup files directly.

## Repository layout

```
.
├── README.md
├── LICENSE
├── AGENTS.md            # guidance for coding agents editing this repo
└── infra-redundancy-for-founders/
    └── SKILL.md         # the skill itself
```

## Contributing

Issues and pull requests are welcome, especially provider-specific rotation steps and failover recipes for other hosts. Please never include real secrets, tokens or customer data in examples.

## License

MIT, provided as is with no warranty. See [LICENSE](LICENSE).
