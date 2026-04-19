<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Secrets Management and Security

Condensed from `docs/vendor/openclaw-llms-full.txt`. Focus: NemoClaw's security posture and secret handling responsibilities.

## Secrets Management

(openclaw-llms-full.txt:63439-63977)

OpenClaw supports **additive SecretRefs** so supported credentials do not need to be stored as plaintext. Plaintext still works; SecretRefs are opt-in per credential.

### Runtime Model

Secrets resolved into an **in-memory runtime snapshot**:

- **Eager resolution** during activation — not lazy on request paths.
- **Fail-fast startup** when an effectively active SecretRef cannot resolve.
- **Atomic swap on reload** — full success, or keep the last-known-good snapshot.
- Policy violations (e.g. OAuth-mode auth profiles combined with SecretRef input) fail activation before runtime swap.
- Runtime requests read from the **active in-memory snapshot only**.
- Outbound delivery paths also read from the active snapshot — they do NOT re-resolve SecretRefs on each send.

This keeps secret-provider outages off hot request paths.

### Active-Surface Filtering

SecretRefs validated only on effectively active surfaces:

- **Enabled surfaces:** unresolved refs block startup/reload.
- **Inactive surfaces:** unresolved refs do NOT block. Emit non-fatal diagnostics with code `SECRETS_REF_IGNORED_INACTIVE_SURFACE`.

Examples of inactive surfaces:

- Disabled channel/account entries.
- Top-level channel credentials that no enabled account inherits.
- Disabled tool/feature surfaces.
- Web search provider-specific keys not selected by `tools.web.search.provider`.
- Sandbox SSH auth material when effective backend is not `ssh`.
- `gateway.remote.token` / `gateway.remote.password` SecretRefs active only if `gateway.mode=remote`, `gateway.remote.url` configured, `gateway.tailscale.mode` is `serve`/`funnel`, or token auth can win locally with no env/auth token configured.
- `gateway.auth.token` SecretRef inactive when `OPENCLAW_GATEWAY_TOKEN` is set (env wins).

Gateway auth surface diagnostics log `SECRETS_GATEWAY_AUTH_SURFACE` with `active`/`inactive` + reason.

### SecretRef Contract

One object shape everywhere:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

#### `source: "env"`

```json5
{ source: "env", provider: "default", id: "OPENAI_API_KEY" }
```

Validation:

- `provider` must match `^[a-z][a-z0-9_-]{0,63}$`
- `id` must match `^[A-Z][A-Z0-9_]{0,127}$`

#### `source: "file"`

```json5
{ source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
```

Validation:

- `provider` matches `^[a-z][a-z0-9_-]{0,63}$`
- `id` must be an absolute JSON pointer (`/...`)
- RFC6901 escaping: `~` → `~0`, `/` → `~1`

#### `source: "exec"`

```json5
{ source: "exec", provider: "vault", id: "providers/openai/apiKey" }
```

Validation:

- `provider` matches `^[a-z][a-z0-9_-]{0,63}$`
- `id` matches `^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`
- `id` must NOT contain `.` or `..` as slash-delimited path segments (`a/../b` rejected).

### Provider Config

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // or "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
    },
    defaults: { env: "default", file: "filemain", exec: "vault" },
    resolution: {
      maxProviderConcurrency: 4,
      maxRefsPerProvider: 512,
      maxBatchBytes: 262144,
    },
  },
}
```

#### File Provider

- `mode: "json"` — JSON object payload, `id` as pointer.
- `mode: "singleValue"` — ref id `"value"`, returns file contents.
- Path must pass ownership/permission checks.
- Windows fail-closed: ACL verification unavailable → resolution fails. `allowInsecurePath: true` bypasses (trusted paths only).

#### Exec Provider

- Runs configured absolute binary path, **no shell**.
- Default: `command` must be a regular file (not a symlink).
- `allowSymlinkCommand: true` allows symlink command paths (e.g. Homebrew shims).
- Pair with `trustedDirs` (e.g. `["/opt/homebrew"]`).
- Supports timeout, no-output timeout, output byte limits, env allowlist.

Request (stdin): `{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }`

Response (stdout): `{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<key>" } }`

Per-id errors: `{ "protocolVersion": 1, "values": {}, "errors": { "providers/openai/apiKey": { "message": "not found" } } }`

### Exec Integrations Shipped

- **1Password CLI** (`op read op://...`)
- **HashiCorp Vault CLI** (`vault kv get -field=KEY secret/...`)
- **sops** (`sops -d --extract ... file.enc.json`)

All use `allowSymlinkCommand: true` + `trustedDirs: ["/opt/homebrew"]` for Homebrew symlinked binaries.

### MCP Server Environment Variables

`plugins.entries.acpx.config.mcpServers` env vars support SecretInput (keeps API keys/tokens out of plaintext config).

### Secrets Apply Plan Contract

(openclaw-llms-full.txt:63978-64090)

- `openclaw secrets apply --validate-only <plan>` — no writes.
- `openclaw secrets apply <plan>` — apply for real.
- Exec-containing plans require explicit opt-in in both modes.

## Security Model

(openclaw-llms-full.txt:64092-64885)

### Trust Model (CRITICAL)

**Personal assistant trust model** — one trusted operator boundary per gateway (single-user/personal-assistant model).

OpenClaw is **NOT** a hostile multi-tenant security boundary for multiple adversarial users sharing one agent/gateway.

For mixed-trust or adversarial-user operation, split trust boundaries — separate gateway + credentials, ideally separate OS users/hosts.

### Deployment and Host Trust

- Anyone who can modify `~/.openclaw` (including `openclaw.json`) = trusted operator.
- Running one Gateway for mutually untrusted operators is **not a recommended setup**.
- Recommended default: one user per machine/host, one gateway, one-or-more agents.
- Inside one Gateway instance, authenticated operator access is a **trusted control-plane role**, not a per-user tenant role.
- **`sessionKey` is a routing selector, not an authorization token.**

### Shared Slack Workspace: Real Risk

If "everyone in Slack can message the bot":

- Any allowed sender can induce tool calls (`exec`, browser, network/file tools) within the agent's policy.
- Prompt/content injection from one sender can cause actions affecting shared state, devices, or outputs.
- Shared agent with sensitive credentials/files — any allowed sender can drive exfiltration via tool usage.

Use separate agents/gateways with minimal tools for team workflows; keep personal-data agents private.

### Company-Shared Agent: Acceptable Pattern

Acceptable when everyone is in the same trust boundary (one company team) and the agent is strictly business-scoped:

- Dedicated machine/VM/container.
- Dedicated OS user + dedicated browser/profile/accounts.
- Do NOT sign into personal Apple/Google or personal password-manager/browser profiles.

### Trust Boundary Matrix

| Boundary / control | What it means | Common misread |
|---|---|---|
| `gateway.auth` (token/password/trusted-proxy/device auth) | Authenticates callers to gateway APIs | "Needs per-message signatures to be secure" |
| `sessionKey` | Routing key for context/session selection | "Session key is a user auth boundary" |
| Prompt/content guardrails | Reduce model abuse risk | "Prompt injection alone proves auth bypass" |
| `canvas.eval` / browser evaluate | Intentional operator capability when enabled | "Any JS eval is a vuln in this trust model" |
| Local TUI `!` shell | Explicit operator-triggered local exec | "Local shell convenience command is remote injection" |
| Node pairing + node commands | Operator-level remote exec on paired devices | "Remote device control is untrusted user access by default" |

### Not Vulnerabilities By Design

Commonly reported, usually closed no-action:

- Prompt-injection-only chains without policy/auth/sandbox bypass.
- Claims assuming hostile multi-tenant on one shared host/config.
- Normal operator read-path access classified as IDOR in shared-gateway setup.
- Localhost-only deployment findings (e.g. HSTS on loopback-only gateway).
- "Missing per-user authorization" findings treating `sessionKey` as an auth token.

### Hardened Baseline (60 Seconds)

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: { dmScope: "per-channel-peer" },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

### Quick Audit

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
openclaw security audit --json
```

`--fix` is narrow: flips open group policies to allowlists, restores `logging.redactSensitive: "tools"`, tightens state/config/include-file permissions, uses Windows ACL resets on Windows. Flags: Gateway auth exposure, browser control exposure, elevated allowlists, FS permissions, permissive exec approvals, open-channel tool exposure.

### Shared Inbox Quick Rule

If more than one person can DM the bot:

- Set `session.dmScope: "per-channel-peer"` (or `"per-account-channel-peer"` for multi-account).
- Keep `dmPolicy: "pairing"` or strict allowlists.
- Never combine shared DMs with broad tool access.
- Hardens cooperative/shared inboxes, NOT designed as hostile co-tenant isolation when users share host/config write access.

### Context Visibility Model

Two concepts:

- **Trigger authorization:** who can trigger the agent (`dmPolicy`, `groupPolicy`, allowlists, mention gates).
- **Context visibility:** what supplemental context is injected into model input (reply body, quoted text, thread history, forwarded metadata).

## Blueprint-Relevant Takeaways

- NemoClaw's blueprint MUST encode SecretRefs for all credentials rather than plaintext. Use `source: "env"` by default for the sandbox (with the env var injected by NemoClaw), `source: "file"` for host-local JSON secrets, `source: "exec"` when an external secret manager is available.
- **Active-surface filtering means NemoClaw doesn't need to resolve secrets for channels the blueprint doesn't enable.** Keep disabled channel entries in the blueprint with their SecretRefs intact — startup won't fail.
- Gateway auth MUST use `mode: "token"` + a strong random token in NemoClaw blueprint defaults. `mode: "none"` is only for fully isolated ingress (inside the sandbox loopback).
- `OPENCLAW_GATEWAY_TOKEN` env var wins over the SecretRef on `gateway.auth.token`; NemoClaw's sandbox bootstrap can inject this to avoid persisting the token in blueprint-managed config.
- NemoClaw's security posture is **personal-assistant trust model** — one sandbox = one trust boundary. Multi-tenant NemoClaw deployments must provision separate sandboxes, not shared agents.
- The hardened baseline above is a good NemoClaw default: `gateway.bind: "loopback"` + denied `group:runtime`/`group:fs`/`group:automation` + `exec.security: "deny"` + `elevated.enabled: false`.
- File provider mode `singleValue` + `id: "value"` is the simplest pattern for NemoClaw to wire a secret file path (e.g. a NIM API key in `~/.openclaw/secrets/nim.key`).
- When NemoClaw writes `openclaw.json`, it should never inline plaintext secrets — use SecretRefs. The strict validation layer will accept the SecretRef shape; it never persists the resolved value.
- `openclaw security audit --fix` is the right hook for NemoClaw to invoke post-onboarding and pre-start to enforce a safe default posture.
- For Windows deployments, document `allowInsecurePath: true` as a last resort; ACL verification is the default and should stay on.
