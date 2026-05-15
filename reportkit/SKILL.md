---
name: reportkit
description: Help install, configure, and use the ReportKit CLI and iPhone app to send iOS Live Activity updates from local scripts, CI, Codex, Cursor, Claude, or other agents. Use when drafting `reportkit send` commands, JSON payload files, cloud-agent token setup, widget builder payloads, alarm pushes, or troubleshooting why a ReportKit Live Activity did not appear.
---

# ReportKit

## Scope

Help users send concise, action-oriented project signals to ReportKit Live Activities. Keep guidance practical, paste-ready, and centered on the shipped CLI contract.

Do not add scheduler or cron logic to ReportKit itself. Scheduling belongs to the user's workflow runner, CI system, agent, cron, launchd, or automation platform.

## Setup Flow

Use this sequence for the hosted beta:

1. Install the CLI with `npm install -g @andreasink/reportkit`.
2. Sign in locally with `reportkit auth --email <email>`.
3. Run `reportkit status`.
4. Install and open the ReportKit iPhone app.
5. Sign in to the app with the same email/password account.
6. Confirm notifications are allowed so Live Activities can appear.
7. Send a filled test payload with an explicit template.

The hosted beta CLI includes the public Supabase URL and publishable key. Ask for `REPORTKIT_SUPABASE_URL` and `REPORTKIT_SUPABASE_ANON_KEY` only if the user is self-hosting or targeting another ReportKit-compatible backend.

Never ask users to put passwords in command arguments. Use the interactive prompt by default, or `--password-stdin` for automation:

```bash
printf '%s\n' 'your-password' | reportkit auth --email you@example.com --password-stdin
```

## Cloud Agents

For hosted agents, do not copy local session files or passwords. Create a ReportKit agent token on a trusted local machine:

```bash
reportkit agent-token create --name "Codex Cloud" --json
```

Store the returned `token` value as `REPORTKIT_AGENT_TOKEN` in the provider's secret environment.

Agent tokens can send Live Activity updates for the owning user. They cannot upload iOS tokens, create/list/revoke other agent tokens, refresh Supabase sessions, or act as a Supabase password. If a token appears in logs, prompts, commits, or shared transcripts, tell the user to revoke it:

```bash
reportkit agent-token revoke --id TOKEN_ID
```

For Codex-style environments, make `REPORTKIT_AGENT_TOKEN` available before the command runs. Prefer commands that start with `reportkit`; avoid inline `REPORTKIT_AGENT_TOKEN=... reportkit ...` examples.

## Send Commands

Use rich, explicit payloads rather than generic hello-world notifications. Prefer `--file` for deep links, Craft URLs, builder layouts, or anything with shell-sensitive characters such as `?`, `&`, or `=`.

Simple manual send:

```bash
reportkit send \
  --event update \
  --activity-id revenue-watch \
  --title "Revenue Watch" \
  --summary "MRR is steady, trials dipped 8%, and crash-free sessions are normal." \
  --status warning \
  --template growth \
  --source-name "Stripe" \
  --source-icon "chart.line.uptrend.xyaxis" \
  --primary-label "Trial -> paid" \
  --primary-value "-8%" \
  --primary-delta "vs yesterday" \
  --secondary-label "MRR" \
  --secondary-value "Stable" \
  --footer "Review onboarding cohort before changing pricing." \
  --action "Open dashboard"
```

File-mode payload:

```json
{
  "event": "update",
  "activityId": "codex-agent",
  "apnsEnv": "production",
  "idempotencyKey": "codex-agent-2026-05-14T18-00Z",
  "payload": {
    "generatedAt": 1774000000,
    "title": "Release Verification",
    "summary": "Build passed, UI smoke test is running, and one TestFlight note remains.",
    "status": "warning",
    "template": "agent",
    "sourceName": "Codex",
    "sourceIcon": "terminal.fill",
    "primaryLabel": "Progress",
    "primaryValue": "68%",
    "primaryDelta": "active",
    "secondaryLabel": "Steps",
    "secondaryValue": "17/25",
    "footer": "Waiting on simulator smoke test.",
    "action": "Open run",
    "deepLink": "https://example.com/run"
  }
}
```

Run it with:

```bash
reportkit send --file payload.json
```

## Templates

Choose one explicit template:

- `ops`: backend health, failures, queues, deployments, migrations, uptime.
- `growth`: revenue, trials, conversion, App Store status, product metrics.
- `agent`: Codex, Cursor, Claude, CI, or background-agent progress.
- `builder`: constrained JSON-to-SwiftUI Live Activity layouts.

For `agent`, include `progressPercent`, `completedSteps`, and `totalSteps` when the workflow has meaningful progress.

For `builder`, use file mode and include `builderSpec` plus `data` inside the payload. The widget does not fetch layouts at render time. Keep the DSL finite: `vstack`, `hstack`, `zstack`, `grid`, `text`, `metric`, `chart`, `progress`, `statusDot`, `imageSymbol`, `button`, `spacer`, and `divider`. Do not generate arbitrary SwiftUI, scripts, HTML, Markdown rendering, or remote image fetches.

Minimal builder payload:

```json
{
  "event": "start",
  "activityId": "builder-demo",
  "payload": {
    "generatedAt": 1774000000,
    "title": "Revenue Console",
    "summary": "MRR and conversion need attention.",
    "status": "warning",
    "template": "builder",
    "builderSpec": {
      "version": 1,
      "surfaces": {
        "lockScreen": {
          "type": "vstack",
          "children": [
            { "type": "text", "text": { "path": "headline" }, "role": "title" },
            { "type": "metric", "label": "MRR", "value": { "path": "mrr" }, "delta": { "path": "mrrDelta" } },
            { "type": "chart", "values": { "path": "trend" }, "label": "Last 7 checks" }
          ]
        },
        "compact": {
          "type": "hstack",
          "children": [
            { "type": "text", "text": { "path": "mrr" }, "role": "value" },
            { "type": "statusDot" }
          ]
        },
        "minimal": { "type": "statusDot" }
      }
    },
    "data": {
      "headline": "Revenue dipped after paywall change",
      "mrr": "$18.2k",
      "mrrDelta": "-8%",
      "trend": [18.9, 18.6, 18.4, 18.1, 18.2]
    }
  }
}
```

## Widgets And Alarms

Use widget commands when saved widget config is involved:

```bash
reportkit widget list
reportkit widget list --json
reportkit widget update --id WIDGET_ID --accent-hex '#3366CC' --glass-opacity 0.62
reportkit widget update --id WIDGET_ID --template builder --builder-spec-file builder-spec.json
```

Do not ask users for `endpoint_url`; ReportKit derives endpoint wiring internally from the configured backend.

Use alarms only when the user needs a one-off push without changing a Live Activity:

```bash
reportkit alarm --title "Review crash spike" --in-seconds 60
```

## Troubleshooting

If nothing appears on the phone, check in this order:

1. The iPhone app is installed, opened, and signed into the same account as the CLI.
2. Notification permission is enabled.
3. `reportkit status` shows a usable local session for local-session sends.
4. Cloud agents have `REPORTKIT_AGENT_TOKEN` set and the token has not been revoked.
5. The send used an explicit template and a filled payload, not only a title and summary.
6. Deep links or Craft URLs were sent through `reportkit send --file payload.json`.
7. Free-tier quota errors return HTTP 402 with `live_activity_daily_limit_reached`; present that as an upgrade/paywall moment, not as an auth bug.

## Response Style

Return short, actionable output. Ask only the minimum needed question, then provide a command or payload file shape the user can run.
