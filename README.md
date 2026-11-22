# FridaScope

On-device Android API tracer powered by Frida: pick an app, watch the APIs it
uses live, get a report. Inspired by Inspeckage, rebuilt on Frida — with
native-call tracing and an MCP server so AI agents can drive it.

## What it does

- **App selector** — list installed apps, search, filter user/system, pick a
  target in one tap.
- **Live API report** — API calls stream in as the target app runs: timestamp,
  API name, category (network, file, crypto, IPC, location…), arguments, and
  counts. Pause, filter, and export the session as JSON.
- **Two modes, no compromises**
  - **Standalone on-device:** the companion app runs on the rooted phone,
    talks to `frida-server` on localhost, and traces without any PC.
  - **PC-connected:** connect over USB/ADB and drive the same tracing from a
    computer — same engine, same report.
- **MCP server** — expose tracing as MCP tools (`list_apps`, `start_trace`,
  `get_report`, …) so coding agents like Cursor can run traces and read
  reports directly.

## How it works

Frida does the instrumentation (Java methods, JNI, and native functions —
something Xposed-based tools cannot see). A small engine collects the events,
the app (or the PC client) renders the live report, and the MCP server wraps
the same engine for agents.

```
Target app ──Frida──▶ Engine (event stream) ──▶ App UI (live report)
                                          └──▶ MCP server ──▶ AI agent
```

## Status

Early planning. See `.meta/plans/` for the MVP plan and the MCP server plan.

## Plans

- [MVP plan](.meta/plans/fridascope-mvp.md) — engine, app UI, both modes,
  icon ideas
- [MCP server plan](.meta/plans/fridascope-mcp.md) — tools, report schema,
  transport
