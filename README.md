# ReportKit Codex Skill

Public Codex skill for using ReportKit from agents and automation.

This repository contains only skill guidance:

- `reportkit/SKILL.md`
- `reportkit/agents/openai.yaml`

It intentionally does not include the ReportKit CLI source, iOS app source, Supabase migrations, Edge Functions, SQL, deployment config, user sessions, or agent tokens.

## Install

Copy or symlink the `reportkit` folder into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R reportkit ~/.codex/skills/reportkit
```

Then invoke it as `$reportkit` when asking Codex to draft ReportKit payloads, commands, or setup steps.
