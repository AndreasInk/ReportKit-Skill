---
name: reportkit-setup
description: Set up a user-facing automation that sends ReportKit iPhone Live Activity updates, Home/Lock widget refreshes, grouped push notifications, or Control Widget state updates. Use when the user asks an agent to configure a report, monitor, schedule, CI job, Codex/Cursor/Claude automation, or other workflow that should publish concise ReportKit status updates.
---

# ReportKit Setup

Use this skill during setup time, when a human is deciding what an automation should monitor and how it should notify them through ReportKit.

Do not treat this as the execution skill. The setup skill designs the automation, verifies prerequisites, creates or references credentials, and writes the automation instructions. The runtime automation should use `$reportkit-execution` when it actually sends.

## Setup Goal

Create an automation that knows:

- what signal it watches
- when it should send
- when it should stay silent
- how to map outcomes to `good`, `warning`, and `critical`
- which ReportKit surfaces to send: `live_activity`, `widget`, `notification`
- which Live Activity template to use when `live_activity` is included
- how notifications should group, using `notification.threadId`
- whether notifications should also update the Control Widget state
- where the action or deep link should open
- how the runtime agent will publish: local macOS MCP helper or hosted `REPORTKIT_AGENT_TOKEN`

ReportKit is not the scheduler. Scheduling belongs to Codex, Claude, Cursor, CI, cron, launchd, or the user's workflow runner.

## Prerequisites

For the hosted beta:

1. Install and open the ReportKit iPhone app.
2. Install and open the ReportKit macOS app.
3. Sign into the iPhone and macOS apps with the same email/password account.
4. In the macOS app, click **Enable MCP** to create the scoped ReportKit MCP token.
5. Click **Copy Config** and paste the MCP config into Codex, Claude, Cursor, or another local MCP-compatible client.
6. Confirm iPhone notifications are allowed.

The local MCP helper command is:

```text
/Applications/ReportKit.app/Contents/Library/Helpers/ReportKitMCP
```

The copied MCP config should look like:

```json
{
  "mcpServers": {
    "reportkit": {
      "command": "/Applications/ReportKit.app/Contents/Library/Helpers/ReportKitMCP",
      "args": []
    }
  }
}
```

The hosted beta apps and helper already include the public Supabase URL and publishable key. Ask for `REPORTKIT_SUPABASE_URL` and `REPORTKIT_SUPABASE_ANON_KEY` only when the user is self-hosting or targeting another ReportKit-compatible backend.

The MCP helper is stdio-only. It can publish ReportKit updates with the scoped token from the macOS app, but it cannot read repos or documents, run shell commands, listen on localhost, schedule jobs, or read the macOS app's normal Supabase refresh session.

Only use the npm CLI path when the user's runtime cannot use local MCP, or when a hosted/cloud agent needs an environment secret.

## Cloud Agent Token

For hosted or background agents that cannot use the local macOS MCP helper, create a publish token on a trusted local machine. If using the npm CLI for this fallback path, sign in interactively first with `reportkit auth --email <email>`, then create the token:

```bash
reportkit agent-token create --name "Codex Automation" --json
```

Store the returned `token` value as `REPORTKIT_AGENT_TOKEN` in the automation provider's secret environment.

Rules:

- Prefer the macOS MCP helper for local Codex, Claude, Cursor, and other local AI apps.
- Do not copy local session files into cloud agents.
- Do not paste `REPORTKIT_AGENT_TOKEN` into prompts, code, logs, README files, or shell history.
- Do not use inline `REPORTKIT_AGENT_TOKEN=... reportkit ...` examples.
- Hosted CLI runtime commands should start with `reportkit` so automation rules can match them.
- If a token appears in logs, prompts, commits, or shared transcripts, revoke it with `reportkit agent-token revoke --id TOKEN_ID`.

## Planning Questions

Ask only what is needed, then produce concrete setup instructions.

1. What should this automation monitor?
2. What data source, command, API, or MCP tool decides the current state?
3. What trigger starts the automation: manual run, workflow event, schedule, or agent completion?
4. Which timezone should scheduled triggers use?
5. What maps to `good`, `warning`, and `critical`?
6. When should the automation do nothing?
7. Are there quiet hours, blackout windows, or people/projects to exclude?
8. Should each report use a separate `activityId`, or should multiple checks group into one Live Activity?
9. Which surfaces should this send to: Live Activity, Home/Lock widget, grouped notification, or Control Widget?
10. If notifications are enabled, what stable `threadId` should group them in Notification Center?
11. If the Control Widget is enabled, which states turn it on and which clear it?
12. What should the action text say?
13. What URL, app deep link, Craft doc, PR, dashboard, or run log should open?

## Template Choice

Use one explicit template:

- `ops`: backend health, logs, failures, migrations, queues, uptime.
- `growth`: revenue, trials, conversion, App Store status, product metrics.
- `agent`: Codex, Cursor, Claude, CI, or background-agent progress.
- `builder`: constrained JSON-to-SwiftUI Live Activity layouts.

For visual setup tests, use a filled payload with `template`, `sourceName`, `sourceIcon`, primary/secondary metrics, `footer`, `action`, and `deepLink`.

`accentHex` or `accent_hex` may control the Live Activity accent color and lock-screen mesh when present.

For `builder`, use file mode and include `builderSpec` plus `data` directly in the payload. The widget does not fetch layouts at render time.

## Surfaces

Omit `surfaces` only for the legacy Live Activity-only default. For new automations, send an explicit JSON payload through local `reportkit_send` or hosted `reportkit send --file payload.json`:

- `live_activity`: Dynamic Island / Lock Screen Live Activity update.
- `widget`: Home Screen / Lock Screen WidgetKit refresh through a silent push. Include `widget.widgetId`, a small non-secret `widget.snapshot`, and `reload: true`.
- `notification`: visible APNs alert. Include `notification.threadId` so related alerts group in Notification Center.

Control Widget state is carried on the notification payload:

```json
{
  "notification": {
    "title": "Release Watch",
    "body": "Issue resolved.",
    "threadId": "release-watch",
    "control": {
      "isOn": false,
      "statusText": "Clear",
      "triageURL": "https://example.com/run"
    }
  }
}
```

Use `control.isOn: true` for active warning/critical states and `false` for clear/resolved states. Treat the push payload as the Control Widget source of truth; do not require manual Control Center taps to correct automation state.

## Handoff To Runtime

End setup by writing the exact runtime instruction that the automation should follow. Include:

- use `$reportkit-execution`
- check the automation's skip conditions before sending
- for local MCP, call `reportkit_send` with the JSON payload
- for hosted/cloud CLI runtimes, write a JSON payload file and run `reportkit send --file payload.json`
- keep deep links and `idempotencyKey` inside JSON
- rely on the macOS helper's scoped token locally or `REPORTKIT_AGENT_TOKEN` from the secret environment when running in cloud

Good runtime instruction shape:

```text
When this automation finishes evaluating the current state, use $reportkit-execution.
If the skip condition is true, do not send.
If sending, create a ReportKit JSON payload with activityId "release-watch",
template "agent", status from the rules above, and a stable idempotencyKey for this run.
In local MCP clients, call `reportkit_send` with that JSON payload.
In hosted CLI runtimes, run `reportkit send --file payload.json`. Do not print `REPORTKIT_AGENT_TOKEN`.
```

Multi-surface runtime instruction shape:

```text
When this automation finishes evaluating the current state, use $reportkit-execution.
If the skip condition is true, do not send.
If sending, create a ReportKit JSON payload with surfaces ["live_activity", "widget", "notification"], activityId "release-watch", template "agent", status from the rules above, widget.widgetId "release-watch", notification.threadId "release-watch", and notification.control.isOn set true for warning/critical and false for resolved.
In local MCP clients, call `reportkit_send` with that JSON payload.
In hosted CLI runtimes, run `reportkit send --file payload.json`. Do not print `REPORTKIT_AGENT_TOKEN`.
```

## Quotas And Failure UX

Tell the user about limits in product terms:

- The app surfaces plan and quota state; on the CLI fallback path, `reportkit widget list` prints the server-side plan and quota, such as `Plan: Free (1/1 widgets)`.
- Free accounts get two Live Activity sends per UTC day.
- Pro entitlement bypasses the daily Live Activity send limit.
- Daily send limit responses use HTTP 402 with `live_activity_daily_limit_reached`; present that as an upgrade/paywall moment, not an auth failure.

## Response Style

Return a short setup plan plus the exact command, payload template, or automation prompt the user can paste.

Do not include backend SQL, Supabase Edge Function deployment details, or repo-local file paths in public setup guidance.
