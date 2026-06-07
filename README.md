# ReportKit Codex Skills

Public Codex skills for setting up and running ReportKit automations.

This repo contains two skills:

- `$reportkit-setup`: use when a person asks an agent to set up an automation that should send ReportKit updates.
- `$reportkit-execution`: use inside the automation when the agent has evaluated state and needs to send or skip the ReportKit update.

The split matters: setup is conversational and designs the report contract; execution is narrow, secret-safe, and sends with the local `reportkit_send` MCP helper or the hosted CLI fallback `reportkit send --file payload.json`.

For local Codex, Claude, Cursor, and other MCP-compatible AI apps, ReportKit setup now prefers the macOS app:

1. Sign into the ReportKit iPhone and macOS apps with the same account.
2. In the macOS app, enable MCP and copy the config.
3. Let the runtime agent call `reportkit_send` with the ReportKit JSON payload.

Hosted/cloud agents that cannot use the local macOS MCP helper should use a revocable `REPORTKIT_AGENT_TOKEN` secret and the CLI fallback.

ReportKit sends can target multiple iPhone surfaces:

- `live_activity`: Dynamic Island / Lock Screen Live Activity updates.
- `widget`: Home Screen / Lock Screen WidgetKit refresh snapshots.
- `notification`: grouped APNs alert notifications with `notification.threadId`.
- Control Widget state updates through `notification.control`.

## Live Activity Examples

Use this skill when you want an agent to send focused, action-oriented Live Activity updates instead of another chat notification.

| Ops Calm | Release Readiness | Mixpanel Funnel |
| --- | --- | --- |
| ![Ops Calm](assets/live-activity-previews/live-activity-ops-calm.png) | ![Release Readiness](assets/live-activity-previews/live-activity-release-readiness.png) | ![Mixpanel Funnel](assets/live-activity-previews/live-activity-mixpanel-funnel.png) |

| App Store Analytics | Supabase Errors | GCloud Incident |
| --- | --- | --- |
| ![App Store Analytics](assets/live-activity-previews/live-activity-app-store-analytics.png) | ![Supabase Errors](assets/live-activity-previews/live-activity-supabase-errors.png) | ![GCloud Incident](assets/live-activity-previews/live-activity-gcloud-incident.png) |

| Codex Agent Progress | Builder Launch Console | Builder Compact Console |
| --- | --- | --- |
| ![Codex Agent Progress](assets/live-activity-previews/live-activity-codex-agent-progress.png) | ![Builder Launch Console](assets/live-activity-previews/live-activity-builder-launch-console.png) | ![Builder Compact Console](assets/live-activity-previews/live-activity-builder-compact-console.png) |

## Install

Ask your agent to install this skill repo, for example:

```text
Install https://github.com/AndreasInk/ReportKit-Skill.git so I can set up ReportKit automations.
Use $reportkit-setup when we are designing or installing an automation.
Use $reportkit-execution inside the automation when it is time to send or skip.
```

Then start with `$reportkit-setup` when asking Codex to configure a report, monitor, schedule, CI check, or agent workflow. For local AI apps, have the ReportKit macOS app installed and MCP enabled first.

Inside an automation prompt, include `$reportkit-execution` so the runtime agent uses the stricter send/skip rules.

## Repo Contents

- `reportkit-setup/SKILL.md`
- `reportkit-execution/SKILL.md`
- `assets/live-activity-previews/`

## More Info

Read the docs here: 
https://andreas.craft.me/qtX8oWJYSSxbJ2
