<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Sandboxing, Tool Policy, and Network Model

Condensed from `docs/vendor/openclaw-llms-full.txt`.

## Three Distinct Controls

(openclaw-llms-full.txt:62813-62953)

1. **Sandbox** (`agents.defaults.sandbox.*`, `agents.list[].sandbox.*`) — decides **where tools run** (Docker vs host vs SSH vs OpenShell).
2. **Tool policy** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) — decides **which tools are available/allowed**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) — exec-only escape hatch to run outside the sandbox (target: `gateway` by default, or `node` when configured).

### Inspector

```bash
openclaw sandbox explain [--session <key>] [--agent <id>] [--json]
```

Prints effective sandbox mode/scope/workspace access, whether the session is currently sandboxed, tool allow/deny origin, and elevated gates with fix-it keys.

## Sandboxing (`agents.defaults.sandbox.*`)

(openclaw-llms-full.txt:62954-63438)

### Modes (`sandbox.mode`)

- `"off"` — everything runs on the host.
- `"non-main"` — only **non-main** sessions are sandboxed (group/channel sessions count as non-main).
- `"all"` — everything is sandboxed.

Non-main is determined by `session.mainKey` (default `"main"`), not agent id.

### Scope (`sandbox.scope`)

- `"agent"` (default) — one container per agent.
- `"session"` — one container per session.
- `"shared"` — one container shared by all sandboxed sessions. Ignores per-agent binds.

### Backend (`sandbox.backend`)

| Backend | Where it runs | Workspace model | Network control | Browser sandbox | Best for |
|---------|---------------|-----------------|-----------------|-----------------|----------|
| `"docker"` (default) | Local container | Bind-mount or copy | `docker.network` (default: none) | Supported | Local dev, full isolation |
| `"ssh"` | Remote SSH host | Remote-canonical (seed once) | Remote host's | Not supported | Offload to remote |
| `"openshell"` | OpenShell-managed | `mirror` or `remote` | OpenShell | Not supported yet | Managed remote sandboxes |

### Docker Backend

- Uses Docker daemon socket (`/var/run/docker.sock`); isolation via Docker namespaces.
- **DooD (Docker-out-of-Docker) constraint:** when the Gateway itself runs as a container, `openclaw.json` `workspace` MUST use the Host's absolute path (e.g. `/home/user/.openclaw/workspaces`), not the Gateway's internal container path. Gateway deployment must include an identical volume map (`-v /home/user/.openclaw:/home/user/.openclaw`) or heartbeats will fail with `EACCES`.

### Bind Mounts (`docker.binds`)

- Pierces sandbox filesystem — whatever you mount is visible with the chosen mode (`:ro` or `:rw`). Default is `:rw` — prefer `:ro` for source/secrets.
- `scope: "shared"` ignores per-agent binds.
- OpenClaw validates bind sources twice: on normalized source path, then again after resolving through deepest existing ancestor. Symlink-parent escapes don't bypass blocked-path or allowed-root checks.
- Non-existent leaf paths are checked safely.
- Binding `/var/run/docker.sock` hands host control to the sandbox — only intentional.
- Workspace access (`workspaceAccess: "ro"|"rw"`) is independent of bind modes.

### SSH Backend

- Per-scope remote root under `sandbox.ssh.workspaceRoot`.
- First use (or after recreate) seeds the remote workspace from the local workspace **once**.
- Afterward, file tools and exec run directly against the remote workspace over SSH.
- OpenClaw does NOT sync remote changes back to local automatically.
- Auth material: `identityFile`/`certificateFile`/`knownHostsFile` (local files) OR `identityData`/`certificateData`/`knownHostsData` (inline strings or `SecretRef`). If both set, `*Data` wins per session. Secrets resolve through runtime snapshot, written to temp files at `0600`, deleted on session end.
- **Browser sandboxing is not supported on SSH.** `sandbox.docker.*` settings do not apply.

### OpenShell Backend

(openclaw-llms-full.txt:61295-61594)

Enabled via `plugins.entries.openshell`:

```json5
{
  agents: { defaults: { sandbox: { mode: "all", backend: "openshell", scope: "session", workspaceAccess: "rw" } } },
  plugins: { entries: { openshell: { enabled: true, config: { from: "openclaw", mode: "remote" } } } }
}
```

OpenShell reuses the SSH backend's transport + filesystem bridge and adds OpenShell-specific lifecycle (`sandbox create/get/delete`, `sandbox ssh-config`) + optional `mirror` workspace mode.

#### Workspace modes

| | `mirror` | `remote` |
|-|-|-|
| Canonical workspace | Local host | Remote OpenShell |
| Sync | Bidirectional each exec | One-time seed |
| Per-turn overhead | Higher (upload + download) | Lower |
| Local edits visible | Yes, on next exec | No, until `openclaw sandbox recreate` |
| Best for | Dev workflows | Long-running agents, CI |

#### Plugin config keys (`plugins.entries.openshell.config.*`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mode` | `"mirror"\|"remote"` | `"mirror"` | Workspace sync mode |
| `command` | string | `"openshell"` | Path to `openshell` CLI |
| `from` | string | `"openclaw"` | Sandbox source for first-time create |
| `gateway` | string | — | OpenShell gateway name (`--gateway`) |
| `gatewayEndpoint` | string | — | OpenShell gateway endpoint URL |
| `policy` | string | — | OpenShell policy ID for sandbox creation |
| `providers` | string[] | `[]` | Provider names attached at create |
| `gpu` | boolean | `false` | Request GPU resources |
| `autoProviders` | boolean | `true` | Pass `--auto-providers` during create |
| `remoteWorkspaceDir` | string | `"/sandbox"` | Primary writable workspace inside sandbox |
| `remoteAgentWorkspaceDir` | string | `"/agent"` | Agent workspace mount path |
| `timeoutSeconds` | number | `120` | Timeout for `openshell` CLI ops |

Lifecycle: `openclaw sandbox list`, `openclaw sandbox explain`, `openclaw sandbox recreate [--all]`. **Recreate is critical for `remote` mode** — it deletes the canonical remote workspace so the next use seeds fresh from local.

Current OpenShell limitations: sandbox browser not supported yet, `sandbox.docker.binds` not supported, Docker-specific runtime knobs under `sandbox.docker.*` only apply to Docker backend.

### Browser Sandbox

- Auto-starts when browser tool needs it (`sandbox.browser.autoStart`, `autoStartTimeoutMs`).
- Dedicated Docker network `openclaw-sandbox-browser` (override via `sandbox.browser.network`).
- `sandbox.browser.cdpSourceRange` CIDR allowlist restricts container-edge CDP ingress (e.g. `172.21.0.1/32`).
- noVNC observer access password-protected; short-lived token URL opens noVNC with password in URL fragment (not query/header logs).
- `sandbox.browser.allowHostControl` + `allowedControlUrls`/`allowedControlHosts`/`allowedControlPorts` gate `target: "custom"`.

## What is Sandboxed vs Not

**Sandboxed:** `exec`, `read`, `write`, `edit`, `apply_patch`, `process`, optional browser.

**Not sandboxed:** the Gateway process itself, and any tool explicitly allowed outside (`tools.elevated`).

## Tool Policy (`tools.*`)

### Layers (in decreasing specificity)

- Tool profile: `tools.profile`, `agents.list[].tools.profile` (base allowlist).
- Provider tool profile: `tools.byProvider[provider].profile`, `agents.list[].tools.byProvider[provider].profile`.
- Global/per-agent: `tools.allow`, `tools.deny`, `agents.list[].tools.allow/deny`.
- Provider policy: `tools.byProvider[provider].allow/deny`, `agents.list[].tools.byProvider[provider].allow/deny`.
- Sandbox tool policy (only applies when sandboxed): `tools.sandbox.tools.allow/deny`, `agents.list[].tools.sandbox.tools.*`.

### Rules of thumb

- `deny` always wins.
- Non-empty `allow` → everything else blocked.
- Tool policy is the hard stop: `/exec` cannot override a denied `exec` tool.
- `/exec` only changes session defaults for authorized senders; does NOT grant tool access.
- Provider tool keys accept `provider` (e.g. `google-antigravity`) or `provider/model` (e.g. `openai/gpt-5.4`).

### Tool groups (`group:*`)

| Group | Expands to |
|-------|-----------|
| `group:runtime` | `exec`, `process`, `code_execution` (`bash` = alias for `exec`) |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:web` | `web_search`, `x_search`, `web_fetch` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:agents` | `agents_list` |
| `group:media` | `image`, `image_generate`, `video_generate`, `tts` |
| `group:openclaw` | all built-in OpenClaw tools (excludes provider plugins) |

## Elevated (`tools.elevated.*`)

- Exec-only escape hatch; does NOT grant extra tools.
- If sandboxed, `/elevated on` (or `exec` with `elevated: true`) runs outside the sandbox.
- `/elevated full` skips exec approvals for the session.
- If already running direct, elevated is effectively a no-op.
- Not skill-scoped; does NOT override tool allow/deny.
- Does NOT grant arbitrary cross-host overrides from `host=auto`; follows normal exec target rules.
- `/exec` is separate — only adjusts per-session exec defaults for authorized senders.

Gates:

- `tools.elevated.enabled`
- `tools.elevated.allowFrom.<provider>` (sender allowlist)

## Network Model

(openclaw-llms-full.txt:60652-60675)

- **One Gateway per host** recommended. Only process allowed to own the WhatsApp Web session.
- **Loopback first:** Gateway WS defaults to `ws://127.0.0.1:18789`. Wizard creates shared-secret auth by default.
- **Non-loopback access** requires a valid gateway auth path: shared-secret token/password OR non-loopback `trusted-proxy`. Tailnet/mobile works best through Tailscale Serve or another `wss://` endpoint, not raw tailnet `ws://`.
- Nodes connect over LAN/tailnet/SSH. Legacy TCP bridge removed.
- Canvas host + A2UI are on the same port as Gateway (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`). When Gateway binds beyond loopback with `gateway.auth` configured, these routes are protected. Node clients use node-scoped capability URLs tied to their active WS session.
- Remote use: SSH tunnel or tailnet VPN.

## Blueprint-Relevant Takeaways

- NemoClaw blueprint sets `backend: "openshell"` and expects `plugins.entries.openshell.config.from = "openclaw"` so the sandbox seeds from the OpenClaw-provided workspace.
- `nemoclaw-blueprint/policies/` network policies = egress allowlist for the sandboxed backend. `deny` wins; keep presets narrow.
- NemoClaw's SSRF validation (`nemoclaw/src/blueprint/ssrf.ts`) protects Gateway WS (port 18789) from in-sandbox reachability abuse — any policy preset must not open the Gateway WS port to sandbox clients.
- `/var/run/docker.sock` must NEVER be bind-mounted by a NemoClaw policy preset; doing so hands host control to the sandbox.
- Browser sandbox is Docker-only today — NemoClaw running on `backend: "openshell"` cannot use the sandboxed browser.
- OpenShell `remote` mode matches the NemoClaw security posture (canonical remote workspace, one-time seed). Use `mirror` only for dev loops where bidirectional sync is tolerable.
- Elevated exec is an escape hatch that bypasses sandboxing — NemoClaw should leave `tools.elevated.enabled` off by default and require explicit opt-in with a narrow `allowFrom` list.
- Tool policy precedence matters: NemoClaw's blueprint should set `tools.sandbox.tools.allow` to a minimal group like `["group:runtime","group:fs","group:sessions","group:memory"]` rather than rely on deny-only rules.
