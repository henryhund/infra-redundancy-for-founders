# infra-redundancy-for-founders

Purpose: a Claude skill (`infra-redundancy-for-founders/SKILL.md`) that helps founders and small teams prepare for and respond to hosting provider breaches and outages. Public repo; `README.md` explains the skill, install steps and the scope disclaimer.

Rules:
- The skill lives in `infra-redundancy-for-founders/SKILL.md`. Keep its frontmatter `name` matching the folder name.
- Keep the README's scope disclaimer (SRE and CISO-directed planning for production systems) intact, and keep the SKILL.md scope note consistent with it.
- When the skill's behavior changes, update the "What the skill does" section of the README in the same change.
- Never commit real secrets, tokens or customer data in examples.
- Release zips are build outputs, not sources; `*.zip` is gitignored.
