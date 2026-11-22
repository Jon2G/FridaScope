# Plan: FridaScope MVP

> Hand-off plan for Cursor (coding agent). Goal: a working Inspeckage-like
> Android API tracer on Frida, usable standalone on the phone or connected to
> a PC, with a live API report and an MCP server.

## Context

- Device: Google Pixel 10 Pro / XL, Android 17, KernelSU Next (rooted).
- Inspiration: Inspeckage (Xposed-based dynamic analysis + web report) — but
  Frida-based, so it also sees native calls, and with MCP exposure.
- Decision already taken (2026-11-21): **Frida, not LSPosed/Xposed**, because
  Frida hooks Java + JNI + native, iterates fast (JS scripts, no rebuild),
  and has a programmatic API that maps naturally to MCP tools.
- First test target: the Starbucks app (Frida trace plan + starter script
  already exist: `frida-starbucks-trace.js`).

## Hard requirements (from Jonathan)

1. **The tool must work perfectly in both modes:**
   - **Standalone:** run the companion app on the phone alone — no PC.
     The app starts/uses `frida-server` on localhost (root) and traces.
   - **PC-connected:** connect over USB/ADB; the same engine and report,
     driven from the computer.
   - Both modes share one engine and one report format. No feature that
     only works in one mode unless explicitly marked.
2. **App selector (UI, crucial):** list installed apps with icon + package
   name, text search, user/system filter, recent targets first. One tap
   picks the target and shows whether it is running.
3. **Live API report (UI, crucial):** API calls stream in real time as the
   target runs — timestamp, API name, category, arguments (truncated),
   per-API count. Controls: pause/resume, filter by category and text,
   clear, export session as JSON. The report must stay readable at high
   event rates (cap rendering, aggregate repeats).
4. **MCP exposure** — see `.meta/plans/fridascope-mcp.md`.

## Architecture

```
┌──────────────────────┐      ┌────────────────────┐
│ Target app process   │      │ FridaScope app     │
│  frida-server /      │◀────▶│  Engine (Kotlin)   │
│  frida-gadget        │      │  App selector UI   │
└──────────────────────┘      │  Live report UI    │
                              └──────┬─────────────┘
                                     │ localhost / ADB
                              ┌──────▼─────────────┐
                              │ MCP server (Python)│
                              │  tools + reports   │
                              └────────────────────┘
```

- **Engine:** Frida (Java + native hooks). Hook profiles per category:
  network (`connect`, `send`, `HttpURLConnection`, OkHttp), file
  (`open`, `openat`), crypto (`Cipher`, `MessageDigest`, OpenSSL),
  IPC (`Binder`, intents), location, and Activity lifecycle.
- **Event stream:** Frida script `send()`s JSON events; the engine
  aggregates (dedupe identical calls with counts) and persists the session.
- **Standalone mode:** companion app (Kotlin, .NET MAUI is Jonathan's usual
  stack — decide in implementation; Kotlin is preferred for direct Frida
  embedding) manages `frida-server` lifecycle via root and renders the UI.
- **PC mode:** same engine driven over ADB; MCP server on the PC talks to
  `frida-server` over USB.

## App icon — ideas (defined 2026-11-21)

The icon must read at small sizes, work as an Android adaptive icon
(foreground + background layers, plus a monochrome variant), and fit a dev
tool: dark background, one neon accent. Avoid generic purple gradients.

Palette: background `#0B0F14` (near-black), accent neon green `#00E676`
and/or cyan `#00E5FF`, white for marks.

1. **La mirilla (recommended)** — rounded square, dark background, a neon
   green scope/reticle ring with a small `</>` in the centre. Ties directly
   to the name (Scope) and to "watching what the app does". Scales best.
2. **Radar** — sonar sweep arc with dots (API calls lighting up). Great for
   the "live report" idea; slightly busier at small sizes.
3. **Ojo código** — an eye outline formed by code brackets `< >`. Memorable
   but harder to read small.
4. **Android + lupa** — Android robot-head silhouette with a magnifier over
   it. Clearest "Android + inspect" message; depends on the Android
   trademark guidelines for the robot.
5. **Crosshair** — target crosshair over a phone screen showing call lines.
   Clean and technical; close to option 1, less distinctive.
6. **FS monogram** — bold "FS" letters inside a scope ring. Minimal,
   scales perfectly, weakest storytelling.

Recommendation: build **1 (La mirilla)** as the primary icon, keep **6 (FS
monogram)** as the fallback if the reticle gets muddy at 48dp. Generate as
vector (Figma) or with local ComfyUI; export foreground/background/monochrome
layers.

## Steps

1. **Scaffold** — repo layout, engine module, empty app shell with the two
   screens (selector, live report).
2. **Engine core** — connect to `frida-server`, spawn/attach, load a hook
   profile, stream events as JSON.
3. **App selector UI** — installed-apps list, search, user/system filter,
   recents, running state.
4. **Live report UI** — streaming list, category filter, pause/resume,
   counts, export JSON.
5. **Standalone mode** — root management of `frida-server` lifecycle on
   device; verify tracing works with no PC attached.
6. **PC mode** — ADB path; verify the same session works from the computer.
7. **MCP server** — per `.meta/plans/fridascope-mcp.md`; verify an agent
   can run a full trace through MCP.
8. **First real trace** — Starbucks app end-to-end (launch → login →
   browse → order flow), findings doc.
9. **Icon** — implement the chosen icon concept as adaptive icon + monochrome.

## Done When

- [ ] App selector lists installed apps, search + filters work, target picks
      in one tap.
- [ ] Live report streams API calls with category, args, and counts;
      pause/filter/export work.
- [ ] Standalone mode traces with no PC attached; PC mode traces over ADB.
- [ ] MCP server runs a full trace and returns the report.
- [ ] Starbucks baseline trace captured; findings doc written.
- [ ] App icon implemented (adaptive + monochrome).
- [ ] No secrets/tokens in logs or reports.

## Risks / notes

- Target apps may detect root or Frida and refuse to run — report the exact
  behaviour instead of working around it silently.
- Repackaging with frida-gadget breaks the app signature; prefer
  `frida-server` attach/spawn on the rooted device.
- High event rates can flood the UI — aggregate repeats and cap rendering.
- Frida version must match the server build on the device.
