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
- how the runtime agent will access `REPORTKIT_AGENT_TOKEN`

ReportKit is not the scheduler. Scheduling belongs to Codex, Claude, Cursor, CI, cron, launchd, or the user's workflow runner.

## Prerequisites

For the hosted beta:

1. Install the CLI with `npm install -g @andreasink/reportkit`.
2. Install and open the ReportKit iPhone app.
3. Sign into the iPhone app and CLI with the same email/password account.
4. Run `reportkit auth --email <email>` locally.
5. Run `reportkit status`.
6. Confirm iPhone notifications are allowed.

The hosted beta CLI already includes the public Supabase URL and publishable key. Ask for `REPORTKIT_SUPABASE_URL` and `REPORTKIT_SUPABASE_ANON_KEY` only when the user is self-hosting or targeting another ReportKit-compatible backend.

Never ask users to put passwords in command arguments. Use the interactive password prompt by default. For non-interactive auth only:

```bash
printf '%s\n' 'your-password' | reportkit auth --email you@example.com --password-stdin
```

## Cloud Agent Token

For hosted or background agents, create a publish token on a trusted local machine:

```bash
reportkit agent-token create --name "Codex Automation" --json
```

Store the returned `token` value as `REPORTKIT_AGENT_TOKEN` in the automation provider's secret environment.

Rules:

- Do not copy local session files into cloud agents.
- Do not paste `REPORTKIT_AGENT_TOKEN` into prompts, code, logs, README files, or shell history.
- Do not use inline `REPORTKIT_AGENT_TOKEN=... reportkit ...` examples.
- Runtime commands should start with `reportkit` so automation rules can match them.
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

Omit `surfaces` only for the legacy Live Activity-only default. For new automations, use `reportkit send --file payload.json` with explicit surfaces:

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
- write a JSON payload file
- run `reportkit send --file payload.json`
- keep deep links and `idempotencyKey` inside JSON
- rely on `REPORTKIT_AGENT_TOKEN` from the secret environment when running in cloud

Good runtime instruction shape:

```text
When this automation finishes evaluating the current state, use $reportkit-execution.
If the skip condition is true, do not send.
If sending, create a ReportKit JSON payload with activityId "release-watch",
template "agent", status from the rules above, and a stable idempotencyKey for this run.
Run reportkit send --file payload.json. Do not print REPORTKIT_AGENT_TOKEN.
```

Multi-surface runtime instruction shape:

```text
When this automation finishes evaluating the current state, use $reportkit-execution.
If the skip condition is true, do not send.
If sending, create a ReportKit JSON payload with surfaces ["live_activity", "widget", "notification"], activityId "release-watch", template "agent", status from the rules above, widget.widgetId "release-watch", notification.threadId "release-watch", and notification.control.isOn set true for warning/critical and false for resolved.
Run reportkit send --file payload.json. Do not print REPORTKIT_AGENT_TOKEN.
```

## Quotas And Failure UX

Tell the user about limits in product terms:

- `reportkit widget list` prints the server-side plan and quota, such as `Plan: Free (1/1 widgets)`.
- Free accounts get two Live Activity sends per UTC day.
- Pro entitlement bypasses the daily Live Activity send limit.
- Daily send limit responses use HTTP 402 with `live_activity_daily_limit_reached`; present that as an upgrade/paywall moment, not an auth failure.

## Response Style

Return a short setup plan plus the exact command, payload template, or automation prompt the user can paste.

Do not include backend SQL, Supabase Edge Function deployment details, or repo-local file paths in public setup guidance.
