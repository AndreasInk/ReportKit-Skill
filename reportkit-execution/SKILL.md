---
name: reportkit-execution
description: Execute a ReportKit send from inside an existing automation. Use when an agent, CI job, scheduled task, Codex/Cursor/Claude run, or workflow has already evaluated state and now needs to send or skip a ReportKit Live Activity update, Home/Lock widget refresh, grouped push notification, or Control Widget state update safely.
---

# ReportKit Execution

Use this skill at runtime, inside an automation. The setup and policy decisions should already exist.

Do not redesign the automation here. Do not ask broad onboarding questions unless required data is missing. Decide whether to skip, create a safe payload, and publish through the local `reportkit_send` MCP tool or the hosted CLI fallback.

## Runtime Rules

1. Read the automation's existing instructions and current state.
2. If the skip condition is true, do not send.
3. If sending locally and `reportkit_send` is available, call that MCP tool with the JSON payload.
4. If sending from hosted/cloud/CI without MCP, use a JSON payload file and `reportkit send --file payload.json`.
5. Keep `deepLink`, `apnsEnv`, and `idempotencyKey` inside JSON.
6. Do not print, echo, or inline `REPORTKIT_AGENT_TOKEN`.
7. Do not run `REPORTKIT_AGENT_TOKEN=... reportkit ...`.
8. Do not put passwords or Supabase keys into commands.
9. Do not install, auth, or create tokens unless the automation explicitly says setup is allowed.

In local AI apps, the macOS MCP helper should already have a scoped token created by the ReportKit macOS app. In hosted agents, `REPORTKIT_AGENT_TOKEN` should already be available through the secret environment. Local trusted machines may use the cached CLI session only when the MCP helper is unavailable or the automation explicitly chose the CLI path.

## Send Method

Prefer local MCP when available:

```text
reportkit_send
```

Pass the same JSON payload shape shown below as the tool arguments. The helper is stdio-only, defaults `apnsEnv` to `production`, generates deterministic idempotency keys when omitted, and cannot read files, run shell commands, listen on localhost, schedule jobs, or read the macOS app's normal Supabase refresh session.

For hosted/cloud/CI runtimes without MCP, write the payload to a file and run:

```bash
reportkit send --file payload.json
```

Use `reportkit alarm` only for an alarm-only push that should not update a Live Activity:

```bash
reportkit alarm --title "Review crash spike" --in-seconds 60
```

## Payload Contract

Use this shape:

```json
{
  "event": "update",
  "activityId": "release-watch",
  "apnsEnv": "production",
  "idempotencyKey": "release-watch-2026-05-15T18-00Z",
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
  },
  "surfaces": ["live_activity", "widget", "notification"],
  "widget": {
    "widgetId": "release-watch",
    "snapshot": {
      "title": "Release Verification",
      "summary": "One TestFlight note remains.",
      "status": "warning"
    },
    "reload": true
  },
  "notification": {
    "title": "Release Verification",
    "body": "Build passed and one TestFlight note remains.",
    "threadId": "release-watch",
    "summary": "Release",
    "interruptionLevel": "active",
    "control": {
      "isOn": true,
      "statusText": "Warning",
      "triageURL": "https://example.com/run"
    }
  }
}
```

Rules:

- `generatedAt` is a Unix timestamp in seconds.
- Use a stable `activityId` for repeated updates to the same Live Activity.
- Use a stable `idempotencyKey` for the same run or event.
- Use `event: "start"` for a new activity, `update` for current state, and `end` only when ending is intentional.
- Set `template` explicitly even though the CLI has an `ops` fallback.
- Use `status: "good"`, `"warning"`, or `"critical"`.
- Use `apnsEnv: "production"` unless the automation is explicitly testing a sandbox/TestFlight path.
- Omit `surfaces` only for the legacy Live Activity-only default.

## Surface Rules

Use file mode for multi-surface sends:

```json
{
  "surfaces": ["live_activity", "widget", "notification"]
}
```

- `live_activity` updates the Dynamic Island / Lock Screen Live Activity.
- `widget` refreshes Home Screen and Lock Screen WidgetKit surfaces with a silent APNs push. Include `widget.widgetId`, a small non-secret `widget.snapshot`, and `reload: true`.
- `notification` sends a visible APNs alert. Set `notification.threadId` to group related notifications in Notification Center.

Grouped notifications use `notification.threadId`; do not rely on `activityId` for Notification Center grouping.

Control Widget state can be updated by a notification payload:

```json
{
  "notification": {
    "threadId": "release-watch",
    "control": {
      "isOn": false,
      "statusText": "Clear",
      "triageURL": "https://example.com/run"
    }
  }
}
```

Use `control.isOn: true` for active warning/critical states and `false` for clear/resolved states. Treat notification push state as authoritative for the Control Widget.

## Template Rules

Choose one:

- `ops`: backend health, logs, failures, migrations, queues, uptime.
- `growth`: revenue, trials, conversion, App Store status, product metrics.
- `agent`: Codex, Cursor, Claude, CI, or background-agent progress.
- `builder`: constrained JSON-to-SwiftUI Live Activity layouts.

For `agent`, include progress fields when meaningful:

```json
{
  "payload": {
    "template": "agent",
    "primaryValue": "68%",
    "progressPercent": 68,
    "completedSteps": 17,
    "totalSteps": 25
  }
}
```

For `builder`, include `builderSpec` and `data` inside the payload. The widget does not fetch layouts at render time. The DSL is finite: `vstack`, `hstack`, `zstack`, `grid`, `text`, `metric`, `chart`, `progress`, `statusDot`, `imageSymbol`, `button`, `spacer`, and `divider`.

Do not generate arbitrary SwiftUI, scripts, HTML, Markdown rendering, remote images, or runtime widget network fetches.

## Deep Links

Always use file mode for deep links with shell-sensitive characters.

Craft URLs such as `craftdocs://open?spaceId=...&blockId=...` must stay in JSON data. Do not pass them as raw shell arguments.

## Skip Discipline

If the automation's current state is not actionable, prefer no send. Examples:

- status remains normal and the setup says only notify on changes
- source command failed in a way that makes the data untrustworthy
- quiet hours or blackout window applies
- the same idempotent event was already sent
- the report lacks the minimum fields needed for a useful Live Activity
- a multi-surface send is requested but the payload lacks the required `widget` or `notification` object

If skipping, return a concise reason and do not call `reportkit_send` or `reportkit`.

## Error Handling

Do not hide delivery failures.

- Missing `reportkit_send`: report that the local MCP helper is not configured or unavailable, then use the CLI fallback only if the automation explicitly allows it.
- Missing `reportkit`: on the hosted CLI path, report that the CLI is not installed in this runtime.
- Missing auth/token: report that the runtime lacks a local MCP token, local CLI session, or `REPORTKIT_AGENT_TOKEN`, depending on the selected send method.
- `live_activity_daily_limit_reached` or HTTP 402: report the daily send limit/paywall state.
- No registered targets: report that the iPhone app likely has not signed in/uploaded tokens.
- Blank or partial Live Activity risk: check that `template` and filled payload fields are present.
- Widget refresh rejected: check that `surfaces` includes `widget` and the payload includes `widget.widgetId` plus `widget.snapshot`.
- Notification rejected: check that `surfaces` includes `notification` and the payload includes `notification.title`, `notification.body`, and optional `notification.threadId`.
- Control Widget did not flip: check that the send used the current MCP helper or CLI, generated a fresh `idempotencyKey`, and included `notification.control.isOn`.

## Response Style

At runtime, be brief:

- say skipped with reason, or
- say sent with activity id, status, template, and payload file path, or
- say failed with the actionable missing prerequisite.

Do not include secrets in the response.
