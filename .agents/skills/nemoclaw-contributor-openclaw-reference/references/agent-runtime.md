<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Agent Runtime, Loop, and Workspace

Condensed from `docs/vendor/openclaw-llms-full.txt`. Source: https://docs.openclaw.ai/concepts/.

## Agent Runtime

OpenClaw runs a single embedded agent runtime built on the "pi-agent-core" (models, tools, prompt pipeline). Session management, discovery, tool wiring, and channel delivery are OpenClaw-owned layers on top. (openclaw-llms-full.txt:13058-13181)

Key facts:

- **Workspace** = agent's only `cwd`, set via `agents.defaults.workspace`. Default `~/.openclaw/workspace`; `OPENCLAW_PROFILE=<name>` switches to `~/.openclaw/workspace-<name>`.
- **Sessions** stored as JSONL at `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`.
- **Model refs** are parsed by splitting on the **first** `/` — use `provider/model`. OpenRouter-style IDs require explicit provider prefix (e.g. `openrouter/moonshotai/kimi-k2`).
- **Minimal config** needs `agents.defaults.workspace` and (strongly) `channels.<channel>.allowFrom`.

**Blueprint implication:** the sandbox must expose a workspace path that matches `agents.defaults.workspace`. Per-session sandbox workspaces live under `agents.defaults.sandbox.workspaceRoot`.

## Bootstrap Files (injected into context on session start)

Inside the workspace, OpenClaw expects:

| File | Purpose |
|------|---------|
| `AGENTS.md` | Operating instructions + "memory" |
| `SOUL.md` | Persona, boundaries, tone |
| `TOOLS.md` | User tool notes (does NOT control tool availability) |
| `BOOTSTRAP.md` | One-time first-run ritual (delete after) |
| `IDENTITY.md` | Agent name/vibe/emoji |
| `USER.md` | User profile + preferred address |
| `HEARTBEAT.md` | Optional short heartbeat checklist |
| `BOOT.md` | Optional startup checklist run on gateway restart |
| `memory/YYYY-MM-DD.md` | Daily memory log |
| `MEMORY.md` | Optional curated long-term memory |
| `skills/` | Workspace-specific skills (highest precedence) |
| `canvas/` | Canvas UI files |

Behavior:

- Blank files are skipped. Large files are truncated with a marker.
- Missing files get a single "missing file" marker injected.
- `BOOTSTRAP.md` is only created for brand-new workspaces.
- Disable all injection with `agent.skipBootstrap: true`.
- Truncation limits: `agents.defaults.bootstrapMaxChars` (default 12000), `agents.defaults.bootstrapTotalMaxChars` (default 60000).

**Blueprint implication:** when NemoClaw seeds a fresh sandbox workspace, preserve this file set exactly. Sandbox seed copies only accept regular in-workspace files — symlinks/hardlinks resolving outside the source workspace are ignored. (openclaw-llms-full.txt:13351-13595)

## What is NOT in the Workspace

These live under `~/.openclaw/` and must not be committed:

- `~/.openclaw/openclaw.json` (config)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (OAuth + API keys)
- `~/.openclaw/credentials/` (channel/provider state)
- `~/.openclaw/agents/<agentId>/sessions/` (transcripts + metadata)
- `~/.openclaw/skills/` (managed skills)

## Skills Precedence (OpenClaw)

Highest first:

1. `<workspace>/skills`
2. `<workspace>/.agents/skills`
3. `~/.agents/skills`
4. `~/.openclaw/skills` (managed)
5. Bundled
6. `skills.load.extraDirs`

## Agent Loop

An agentic loop = intake → context assembly → model inference → tool execution → streaming replies → persistence. One serialized run per session. (openclaw-llms-full.txt:13182-13350)

### Entry Points

- Gateway RPC: `agent` and `agent.wait`.
- CLI: `agent` command.

### Flow

1. `agent` RPC validates params, resolves session (sessionKey/sessionId), persists metadata, returns `{ runId, acceptedAt }` immediately.
2. `agentCommand` resolves model + thinking/verbose/trace defaults, loads skills snapshot, calls `runEmbeddedPiAgent`, emits lifecycle end/error fallback.
3. `runEmbeddedPiAgent` serializes via per-session + global queues, resolves auth profile, builds pi session, subscribes to pi events, enforces timeout, returns payloads + usage.
4. `subscribeEmbeddedPiSession` bridges pi events → OpenClaw streams: `tool`, `assistant`, `lifecycle (start|end|error)`.
5. `agent.wait` waits for lifecycle end/error, returns `{ status: ok|error|timeout, startedAt, endedAt, error? }`.

### Queueing + Concurrency

- Runs serialized per session key (session lane) and optionally through a global lane — prevents tool/session races.
- Messaging channels pick queue modes (`collect`/`steer`/`followup`). See [Command Queue](https://docs.openclaw.ai/concepts/queue).

### Session + Workspace Prep

- Workspace resolved and created; sandboxed runs may redirect to a sandbox workspace root.
- Skills loaded (or reused from snapshot) → injected into env + prompt.
- Bootstrap/context files resolved → injected into system prompt.
- Session write lock acquired; `SessionManager` opened.

### Hook Points (Plugin Hooks Inside Loop)

- `before_model_resolve` (pre-session, no messages): override provider/model deterministically.
- `before_prompt_build` (post session load, with messages): inject `prependContext`, `systemPrompt`, `prependSystemContext`, `appendSystemContext`.
- `before_agent_start` — legacy compat; prefer the explicit hooks above.
- `before_agent_reply` — claim the turn, return synthetic reply, or silence turn.
- `agent_end` — inspect final messages + run metadata.
- `before_compaction` / `after_compaction`.
- `before_tool_call` / `after_tool_call` — intercept params/results.
- `before_install` — block skill/plugin installs.
- `tool_result_persist` — sync transform of tool results before transcript write.
- `message_received` / `message_sending` / `message_sent`.
- `session_start` / `session_end`.
- `gateway_start` / `gateway_stop`.

Hook decision rules:

- `before_tool_call`: `{ block: true }` is terminal; `{ block: false }` is a no-op (does NOT clear a prior block).
- `before_install`: same semantics.
- `message_sending`: `{ cancel: true }` is terminal; `{ cancel: false }` is a no-op.

### Internal (Gateway) Hooks

- `agent:bootstrap` — runs while building bootstrap files before system prompt finalizes. Used to add/remove bootstrap context files.
- Command hooks: `/new`, `/reset`, `/stop`, and other command events.

### Streaming + Reply Shaping

- Assistant deltas streamed from pi-agent-core → `assistant` events.
- Block streaming emits partial replies on `text_end` or `message_end` (default off).
- Silent tokens `NO_REPLY` / `no_reply` are filtered from outgoing payloads.
- Messaging tool duplicates removed from final payload list.
- If no renderable payloads remain and a tool errored → fallback error reply emitted.

### Compaction + Retries

- Auto-compaction emits `compaction` stream events and can trigger retry.
- On retry, in-memory buffers and tool summaries reset to avoid duplicate output.

### Timeouts

- `agent.wait` default: 30s.
- Agent runtime: `agents.defaults.timeoutSeconds` default 172800s (48h).
- LLM idle: `agents.defaults.llm.idleTimeoutSeconds` — set explicitly for slow local models. `0` disables. Unset → falls back to `timeoutSeconds` or 120s. Cron runs with no explicit timeout disable the idle watchdog.

### Where Runs Can End Early

- Agent timeout abort
- AbortSignal (cancel)
- Gateway disconnect or RPC timeout
- `agent.wait` timeout (wait-only; does not stop the agent)

## Steering While Streaming

- `steer` mode: inbound messages injected after the current assistant turn's tool calls complete, before next LLM call. No longer skips remaining tool calls.
- `followup` / `collect` modes: inbound held until turn ends, then new turn with queued payloads.
- Block streaming off by default (`blockStreamingDefault: "off"`); tuned via `blockStreamingBreak` (`text_end` vs `message_end`), `blockStreamingChunk` (800–1200 chars), `blockStreamingCoalesce` (idle merging).
- Non-Telegram channels require explicit `*.blockStreaming: true` to enable block replies.

## Blueprint-Relevant Takeaways

- NemoClaw blueprint must preserve `agents.defaults.workspace`, `agents.defaults.sandbox.workspaceRoot`, and session directory paths inside the sandbox.
- Plugin hook registration (`before_model_resolve`, `before_tool_call`, etc.) is the sanctioned way to intercept agent behavior from inside the `nemoclaw` plugin — do not patch pi-agent-core directly.
- Bootstrap file contents flow through `systemPrompt` space, so content that affects sandbox policy should live in `prependSystemContext` or `appendSystemContext` (not `prependContext`, which is per-turn dynamic).
- Sessions persist outside the workspace; any migration helpers must copy `~/.openclaw/agents/<agentId>/sessions/` separately.
