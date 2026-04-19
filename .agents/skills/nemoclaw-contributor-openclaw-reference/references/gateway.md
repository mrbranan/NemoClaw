<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Gateway

Gateway architecture, WebSocket protocol, lock, heartbeat, discovery, bridge protocol. Condensed from `docs/vendor/openclaw-llms-full.txt`.

## Architecture

**One long-lived Gateway daemon per host** owns all messaging surfaces (WhatsApp via Baileys, Telegram via grammY, Slack, Discord, Signal, iMessage, WebChat). Clients (CLI / macOS app / web UI / automations) and nodes (macOS/iOS/Android/headless) connect over **WebSocket** at the configured bind host (default `127.0.0.1:18789`). (openclaw-llms-full.txt:13596-13752)

Invariants:

- Exactly one Gateway controls a single Baileys session per host.
- Handshake is mandatory; any non-JSON or non-`connect` first frame is a hard close.
- Events are **not** replayed — clients must refresh on gaps.
- Canvas host + A2UI host served by the Gateway HTTP server under `/__openclaw__/canvas/` and `/__openclaw__/a2ui/` on the same port (default 18789).

Typed protocol:

- TypeBox schemas define the protocol; JSON Schema is generated from those; Swift models are generated from the JSON Schema.

## WebSocket Protocol (v3)

(openclaw-llms-full.txt:61736-62396)

### Transport

- WebSocket, text frames with JSON payloads.
- First frame MUST be a `connect` request.

### Handshake

1. **Gateway → Client** pre-connect challenge event: `{type:"event", event:"connect.challenge", payload:{nonce, ts}}`.
2. **Client → Gateway** `connect` request with `minProtocol`, `maxProtocol`, `client` info, `role`, `scopes`, `caps`, `commands`, `permissions`, `auth`, `device` (signs the challenge nonce).
3. **Gateway → Client** `hello-ok` response: `{protocol, server:{version, connId}, features:{methods, events}, snapshot, policy:{maxPayload, maxBufferedBytes, tickIntervalMs}, auth:{role, scopes, deviceToken?, deviceTokens?}}`. All of `server`, `features`, `snapshot`, `policy` are required by `src/gateway/protocol/schema/frames.ts`.

### Framing

- Request: `{type:"req", id, method, params}`
- Response: `{type:"res", id, ok, payload|error}`
- Event: `{type:"event", event, payload, seq?, stateVersion?}`
- Side-effecting methods (`send`, `agent`) require **idempotency keys**; server keeps a short-lived dedupe cache.

### Roles + Scopes

- `operator` = control-plane client (CLI/UI/automation). Scopes: `operator.read`, `operator.write`, `operator.admin`, `operator.approvals`, `operator.talk.secrets`, etc.
- `node` = capability host (camera/screen/canvas/system.run). Nodes declare caps like `["camera","canvas","screen","location","voice"]` and commands like `["camera.snap","canvas.navigate","screen.record","location.get"]`.

Bootstrap scope rule: operator entries only satisfy operator requests; non-operator roles need scopes under their own role prefix.

### Auth Modes (`gateway.auth.mode`)

- Shared-secret (default): `connect.params.auth.token` or `connect.params.auth.password`.
- Identity-bearing: `gateway.auth.allowTailscale: true` or non-loopback `gateway.auth.mode: "trusted-proxy"` — satisfies auth from request headers, bypasses shared-secret.
- `"none"`: disables shared-secret entirely. Private ingress only; never expose publicly.

### Pairing + Local Trust

- Every WS client (operator + node) includes a **device identity** on connect.
- New device IDs require pairing approval; Gateway then issues a **device token** for reconnect.
- Direct local loopback connects can be auto-approved for same-host UX.
- OpenClaw has a narrow backend/container-local self-connect path for trusted shared-secret helpers.
- Tailnet and LAN connects (including same-host tailnet binds) still require explicit pairing approval.
- All connects must sign the `connect.challenge` nonce. Signature payload `v3` binds `platform` + `deviceFamily`; pinned on reconnect; metadata changes require re-pair.
- `gateway.auth.*` applies to **all** connections, local or remote.

## Events Emitted

`agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`, `tick`, `shutdown`, plus streaming sub-events for agent runs (`assistant`, `tool`, `lifecycle`, `compaction`).

## Gateway Lock

(openclaw-llms-full.txt:59331-59366)

- Lock is the **exclusive TCP bind** on the WS listener port, not a lockfile.
- `EADDRINUSE` → `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- OS releases the listener on any process exit (including SIGKILL) — no cleanup step needed.
- Additional gateways must use isolated profiles and unique ports (see `openclaw gateway --port <port>`).

## Heartbeat

(openclaw-llms-full.txt:59427-59774)

- Periodic **main-session** agent turn. Does NOT create background task records.
- Default interval: `30m`; `1h` when Anthropic OAuth/token auth (incl. Claude CLI reuse) detected.
- Config keys under `agents.defaults.heartbeat.*` (or per-agent `agents.list[].heartbeat.*`):
  - `every` (duration, `0m` to disable)
  - `target` (`"none"` default | `"last"` to route to last contact)
  - `directPolicy` (`"allow"` default | `"block"` for DM targets)
  - `lightContext: true` — only inject `HEARTBEAT.md`
  - `isolatedSession: true` — fresh session each run
  - `activeHours: { start, end }` — quiet-hours in configured timezone
  - `includeReasoning: true` — deliver reasoning as separate message
  - `prompt` — override default body
- **Response contract:** reply `HEARTBEAT_OK` for "nothing to report"; token must appear at **start or end** (middle is ignored). Stripped, dropped if remaining content ≤ `ackMaxChars` (default 300).
- Disabling with `0m` also omits `HEARTBEAT.md` from bootstrap context on normal runs.

## Health Checks

(openclaw-llms-full.txt:59367-59426)

- `openclaw status` — local summary.
- `openclaw status --deep` — asks gateway for a live `health` probe with `probe:true`.
- `openclaw health [--verbose|--json|--timeout <ms>|--debug]` — WS-only snapshot, no direct channel sockets from CLI. Reports `ok`, `ts`, `durationMs`, per-channel status, agent availability, session-store summary.
- Key config: `gateway.channelHealthCheckMinutes` (default 5, `0` disables), `gateway.channelStaleEventThresholdMinutes` (default 30), `gateway.channelMaxRestartsPerHour` (default 10).
- Per-channel override: `channels.<provider>.healthMonitor.enabled`; multi-account override: `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`.

## Bridge Protocol (Legacy)

(openclaw-llms-full.txt:53074-53160) — older node transport retained for backwards compatibility. Favor the WebSocket protocol above for new integrations.

## Discovery and Transports

(openclaw-llms-full.txt:58649-58787) — mDNS/Bonjour + tailnet advertisement so clients find the Gateway without manual URL entry.

## Remote Access

- Preferred: **Tailscale or VPN**.
- Alternative: SSH tunnel (`ssh -N -L 18789:127.0.0.1:18789 user@host`) — same handshake + auth token apply.
- TLS + optional pinning can be enabled for WS in remote setups.

## Operations

- Start: `openclaw gateway` (foreground, logs to stdout).
- Health: `health` over WS (also in `hello-ok`).
- Supervision: launchd/systemd for auto-restart.

## Gateway Runbook Highlights

(openclaw-llms-full.txt:59878-60233) — startup, shutdown, log tailing (`/tmp/openclaw/openclaw-*.log`), channel relink (`openclaw channels logout && openclaw channels login --verbose`), port-conflict recovery (`openclaw gateway --port <port>` with `--force`).

## Blueprint-Relevant Takeaways

- The NemoClaw blueprint must expose the Gateway WS port (default 18789) to authorized operators only — SSRF validation in `nemoclaw/src/blueprint/ssrf.ts` exists because this port is the control plane.
- Gateway auth mode is a policy choice: loopback + shared-secret is the default; do not set `mode: "none"` on sandbox egress-accessible binds.
- Pairing and device tokens are required for non-loopback connects; the NemoClaw policy presets should not bypass these.
- Canvas/A2UI served on the same port — if NemoClaw restricts egress, canvas HTTP routes must be allowed through the same listener.
- Health monitor restarts happen automatically; NemoClaw sandbox restart logic must not fight this (watch `gateway.channelMaxRestartsPerHour`).
- Heartbeat is a main-session turn and runs inside the sandbox — any background-task isolation NemoClaw provides must allow this main-session path.
