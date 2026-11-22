# Plan: FridaScope MCP server

> Hand-off plan for Cursor (coding agent). Goal: expose the FridaScope
> tracing engine as MCP tools so AI agents (Cursor, Muse, …) can run traces
> and read reports without touching the phone UI.

## Context

- Engine: Frida (see `.meta/plans/fridascope-mvp.md`). The MCP server is a
  thin orchestration layer over the engine — it does not reimplement hooking.
- Transport: Python with FastMCP (or the official MCP Python SDK). stdio
  transport for local agents; optional HTTP/SSE for remote use.
- Runs either on the dev PC (talking to `frida-server` over ADB/USB) or on
  the phone via Termux (talking to `frida-server` on localhost).

## Tools

| Tool | Purpose |
| ---- | ------- |
| `list_devices` | Connected devices (ADB / localhost frida-server). |
| `list_apps` | Installed apps on the device (package, label, system/user). |
| `list_api_profiles` | Hook profiles: network, file, crypto, IPC, location, activities. |
| `start_trace` | Spawn/attach to a package with a profile; returns `session_id`. |
| `stop_trace` | Stop a session; returns the report summary. |
| `get_live_events` | Poll/stream events for a session (since a cursor). |
| `get_report` | Full session report as JSON (and optional Markdown). |
| `list_sessions` | Past sessions with status and event counts. |

## Report schema (JSON)

```json
{
  "session_id": "uuid",
  "package": "com.starbucks.mobileapp",
  "started_at": "2026-11-21T22:00:00-06:00",
  "events": [
    {"t": 0.123, "category": "network", "api": "connect",
     "args": {"host": "api.starbucks.com"}, "count": 1}
  ],
  "summary": {"by_category": {"network": 42, "file": 17},
              "hosts": ["api.starbucks.com"]}
}
```

Rules: no secrets, tokens, or auth headers in the report — redact
`Authorization`, cookies, and token-like values at the engine level.

## Steps

1. **Scaffold** — FastMCP server, config (device selection, frida-server
   address), stdio transport.
2. **Device + app tools** — `list_devices`, `list_apps` via ADB/frida.
3. **Trace lifecycle** — `start_trace` / `stop_trace` driving the engine;
   sessions persisted to disk.
4. **Events + report** — `get_live_events` (cursor-based polling),
   `get_report` (JSON + Markdown).
5. **Profiles** — `list_api_profiles`; profiles map to Frida hook scripts.
6. **Verification** — an agent runs a full Starbucks trace through MCP
   only (no phone UI, no manual Frida CLI) and gets a correct report.

## Done When

- [ ] All tools above work against a real device.
- [ ] A full trace (start → events → report) runs through MCP alone.
- [ ] Reports contain no secrets/tokens.
- [ ] Server runs both on the PC (ADB) and on-device (Termux, localhost).
