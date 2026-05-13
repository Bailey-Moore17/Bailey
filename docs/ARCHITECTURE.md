# Konex OS — Architecture

A deeper look at how the system is wired together.

## Design principles

1. **Self-hosted by default.** Konex OS runs on a machine I own. No third-party SaaS sits between me and my agents.
2. **One pane of glass.** Every active session — chat, terminal, agent task — lives in the same UI.
3. **Real terminals, not pretend ones.** Sessions are backed by actual PTYs, so they behave like a shell, not a chat input that returns text.
4. **Model-agnostic.** The bridge can route to any provider with an API. Today: OpenAI and Anthropic. Tomorrow: local models via the same shape.
5. **Reachable everywhere.** Mobile is a first-class client, not an afterthought. Cloudflare Tunnel handles the network layer so I don't have to think about NAT.

## Three layers

### 1. Client (browser / PWA)

A React + TypeScript single-page app that ships as an installable Progressive Web App. The most interesting piece is `TerminalCanvas` — a custom component that renders a live PTY stream and forwards keystrokes back over WebSocket. The service worker caches the shell so the UI loads instantly even on flaky mobile data.

Two clients talk to the backend:

- `src/lib/closedclaw.ts` — agent runtime control (start sessions, switch models, read/write workspace files)
- `src/lib/terminals.ts` — terminal session control (open PTY, send input, subscribe to output)

### 2. Bridge (Node.js)

Three Node processes, kept small on purpose so each has one job:

- `bridge/server.mjs` — the main API. REST endpoints for agent runtime control, plus a WebSocket channel for streaming agent output.
- `bridge/web-server.mjs` — production static server for the built PWA. Separate from the API so I can scale or replace it independently.
- `bridge/terminal.mjs` — PTY lifecycle. Spawns shells, multiplexes IO over WebSocket, cleans up on disconnect.

The bridge is intentionally thin. Most of the "logic" lives in the agent runtime below.

### 3. Agent runtime (`closedclaw/`)

An isolated agent runtime derived from OpenClaw, sandboxed into its own directory so it can run alongside other installs without colliding on config or state.

- `closedclaw/config/openclaw.json` — declarative runtime config (model defaults, memory search, hooks)
- `closedclaw/state/` — persistent agent state. Identity files, memory index, session journals.
- `closedclaw/workspace/` — the active project the agent is working in

> **Why `closedclaw`?** The folder name predates the public "Konex OS" branding and stayed because every API path and config reference would have to be rewritten in lockstep. The user never sees it.

## Request flow — a typical terminal session

```
1. User opens Konex OS on phone
2. PWA shell loads from service-worker cache (~50ms)
3. UI authenticates against the bridge through the Cloudflare Tunnel
4. User taps "new session" → POST /sessions to bridge/server.mjs
5. Bridge boots a closedclaw runtime instance, registers it
6. UI opens a WebSocket to bridge/terminal.mjs for that session
7. terminal.mjs spawns a PTY, pipes stdout/stderr over the socket
8. User types → keystrokes go over WS → PTY input
9. Agent output streams back the same way
10. On disconnect, terminal.mjs keeps the PTY alive for N seconds
    so a flaky mobile connection doesn't kill the work
```

## Remote access

The remote story is deliberately simple:

```
[Mobile / remote browser]
        │
        ▼
[Cloudflare Edge] ── HTTPS, auth, DDoS shield ──┐
                                                 │
        Cloudflare Tunnel (outbound from host)   │
                                                 ▼
[bridge/web-server.mjs on home machine] ─── bridge/server.mjs
                                                 │
                                                 ▼
                                            closedclaw runtime
```

The home machine never opens an inbound port. The tunnel dials out to Cloudflare, and Cloudflare reverse-proxies authenticated requests back through.

`scripts/Setup-KonexOSRemote.ps1` does the one-time setup. `scripts/Start-KonexOSRemoteStack.ps1` is what I run day-to-day to bring everything up.

## What I'd change if I rebuilt it

- Swap the custom REST + WS shape on the bridge for a single typed RPC layer (probably tRPC or just custom over WS) so the client and server stop drifting.
- Push more of the session state down into SQLite instead of holding it in process memory.
- First-class multi-agent view in the UI. Right now sessions are independent; the next interesting thing is having them aware of each other.
