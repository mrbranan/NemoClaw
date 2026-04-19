<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Skills, Hooks, and Automation (Standing Orders, Task Flow, Background Tasks)

Condensed from `docs/vendor/openclaw-llms-full.txt`.

## Skills

(openclaw-llms-full.txt:50059-50441)

OpenClaw uses **AgentSkills-compatible** skill folders. Each skill = directory with `SKILL.md` (YAML frontmatter + instructions). OpenClaw loads bundled + optional local overrides, filtered at load time based on env/config/binary presence.

### Load Locations (precedence highest → lowest)

1. `<workspace>/skills`
2. `<workspace>/.agents/skills`
3. `~/.agents/skills`
4. `~/.openclaw/skills` (managed/local)
5. Bundled skills
6. `skills.load.extraDirs`

### Per-Agent vs Shared Skills

- **Per-agent:** `<workspace>/skills` — that agent only.
- **Project agent:** `<workspace>/.agents/skills` — applies to that workspace before the normal `skills/` folder.
- **Personal agent:** `~/.agents/skills` — applies across workspaces on that machine.
- **Shared:** `~/.openclaw/skills` — visible to all agents on same machine.
- **Shared folders via `skills.load.extraDirs`** — lowest precedence, common skills pack.

### Agent Skill Allowlists

Skill **location** and **visibility** are separate:

- Location/precedence decides which copy wins.
- Agent allowlists (`agents.defaults.skills`, `agents.list[].skills`) decide which visible skills an agent can actually use.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" },                        // inherits defaults
      { id: "docs", skills: ["docs-search"] }, // replaces defaults (does NOT merge)
      { id: "locked-down", skills: [] },       // no skills
    ],
  },
}
```

### Plugins + Skills

Plugins ship skills by listing `skills` directories in `openclaw.plugin.json` (relative to plugin root). Plugin skills load when plugin is enabled. Merged into same low-precedence path as `skills.load.extraDirs` — same-named bundled/managed/agent/workspace skill overrides them.

### ClawHub

Public skills registry at `https://clawhub.ai`:

- `openclaw skills install <slug>` — installs into active workspace `skills/`.
- `openclaw skills update --all`
- `clawhub sync --all` (separate CLI for publish/sync)

### Security

- Treat third-party skills as **untrusted code**. Read before enabling.
- Prefer sandboxed runs for untrusted inputs and risky tools.
- Workspace + extra-dir skill discovery only accepts skill roots and `SKILL.md` files whose resolved realpath stays inside the configured root.
- Gateway-backed skill dependency installs run built-in **dangerous-code scanner** before executing installer metadata. `critical` findings block by default unless caller explicitly sets dangerous override; suspicious findings warn only.
- `openclaw skills install <slug>` is different — downloads ClawHub skill into workspace, does NOT use installer-metadata path.
- `skills.entries.*.env` and `skills.entries.*.apiKey` inject secrets into **host** process for that agent turn (not the sandbox). Keep secrets out of prompts and logs.

### Skill Format

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
---
```

- Follows AgentSkills spec.
- Embedded agent parser supports **single-line frontmatter keys only**.
- `metadata` must be a **single-line JSON object**.
- Use `{baseDir}` in instructions to reference skill folder path.

Optional frontmatter:

- `homepage` — URL for "Website" in macOS Skills UI (also `metadata.openclaw.homepage`).
- `user-invocable` — `true|false` (default `true`). Exposes as user slash command.
- `disable-model-invocation` — `true|false` (default `false`). Excludes from model prompt (still available via user invocation).
- `command-dispatch: tool` — slash command bypasses model, dispatches directly to tool.
- `command-tool: <name>` — tool to invoke when `command-dispatch: tool`.
- `command-arg-mode: raw` (default) — forwards raw args string to tool.

### Load-Time Gating (`metadata.openclaw`)

```markdown
metadata:
  {
    "openclaw": {
      "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
      "primaryEnv": "GEMINI_API_KEY"
    }
  }
```

Fields:

- `always: true` — always include, skip other gates.
- `emoji`, `homepage` — UI metadata.
- `os` — list of `darwin`|`linux`|`win32`. Eligible only on those OSes.
- `requires.bins` — list; each must exist on `PATH`.
- `requires.anyBins` — list; at least one must exist on `PATH`.
- `requires.env` — list; env var exists OR provided in config.
- `requires.config` — list of `openclaw.json` paths that must be truthy.
- `primaryEnv` — env var name associated with `skills.entries.<name>.apiKey`.

## Hooks

(openclaw-llms-full.txt:25949-26266)

Small scripts that run when something happens inside the Gateway. Two kinds:

- **Internal hooks** (this section) — run inside Gateway when agent events fire.
- **Webhooks** — external HTTP endpoints that let other systems trigger work in OpenClaw.

Hooks can be bundled inside plugins. `openclaw hooks list` shows both standalone and plugin-managed.

### CLI

```bash
openclaw hooks list
openclaw hooks enable session-memory
openclaw hooks check
openclaw hooks info session-memory
```

### Event Types

| Event | When it fires |
|-------|--------------|
| `command:new` | `/new` command issued |
| `command:reset` | `/reset` command issued |
| `command:stop` | `/stop` command issued |
| `command` | Any command event (general listener) |
| `session:compact:before` | Before compaction summarizes history |
| `session:compact:after` | After compaction completes |
| `session:patch` | When session properties are modified |
| `agent:bootstrap` | Before workspace bootstrap files are injected |
| `gateway:startup` | After channels start and hooks are loaded |
| `message:received` | Inbound message from any channel |
| `message:transcribed` | After audio transcription completes |
| `message:preprocessed` | After all media and link understanding completes |
| `message:sent` | Outbound message delivered |

### Hook Structure

```
my-hook/
├── HOOK.md          # Metadata + documentation
└── handler.ts       # Handler implementation
```

`HOOK.md`:

```markdown
---
name: my-hook
description: "Short description"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---
```

`metadata.openclaw` fields: `emoji`, `events`, `export` (named export, default `"default"`), `os`, `requires` (`bins`/`anyBins`/`env`/`config`), `always`, `install`.

### Handler Shape

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") return;
  event.messages.push("Hook executed!");
};
export default handler;
```

Each event: `type`, `action`, `sessionKey`, `timestamp`, `messages` (push to send to user), `context` (event-specific).

### Event Context (highlights)

- **`command:*`:** `sessionEntry`, `previousSessionEntry`, `commandSource`, `workspaceDir`, `cfg`.
- **`message:received`:** `from`, `content`, `channelId`, `metadata` (`senderId`, `senderName`, `guildId`).
- **`message:sent`:** `to`, `content`, `success`, `channelId`.
- **`message:transcribed`:** `transcript`, `from`, `channelId`, `mediaPath`.
- **`message:preprocessed`:** `bodyForAgent` (final enriched body), `from`, `channelId`.
- **`agent:bootstrap`:** `bootstrapFiles` (**mutable array**), `agentId`.
- **`session:patch`:** `sessionEntry`, `patch` (only changed fields), `cfg`. Only privileged clients can trigger.
- **`session:compact:before`:** `messageCount`, `tokenCount`.
- **`session:compact:after`:** `compactedCount`, `summaryLength`, `tokensBefore`, `tokensAfter`.

### Hook Discovery Precedence

1. Bundled hooks (with OpenClaw).
2. Plugin hooks (inside installed plugins).
3. Managed hooks: `~/.openclaw/hooks/` — user-installed, shared across workspaces. `hooks.internal.load.extraDirs` share this precedence.
4. Workspace hooks: `<workspace>/hooks/` — per-agent, disabled by default until explicitly enabled.

Workspace hooks can add new hook names but cannot override bundled/managed/plugin hooks with the same name.

### Hook Packs

npm packages exporting hooks via `openclaw.hooks` in `package.json`. Install:

```bash
openclaw plugins install <path-or-spec>
```

npm specs are **registry-only** (package name + optional exact version or dist-tag). Git/URL/file specs and semver ranges rejected.

## Automation & Tasks

(openclaw-llms-full.txt:26267-27037)

### Decision Matrix

| Use case | Recommended | Why |
|----------|-------------|-----|
| Send daily report at 9 AM sharp | Scheduled Tasks (Cron) | Exact timing, isolated execution |
| Remind me in 20 minutes | Scheduled Tasks (Cron) | One-shot with precise timing (`--at`) |
| Run weekly deep analysis | Scheduled Tasks (Cron) | Standalone task, can use different model |
| Check inbox every 30 min | Heartbeat | Batches with other checks, context-aware |
| Monitor calendar for upcoming events | Heartbeat | Natural fit for periodic awareness |
| Inspect status of subagent or ACP run | Background Tasks | Tasks ledger tracks all detached work |
| Audit what ran and when | Background Tasks | `openclaw tasks list`, `openclaw tasks audit` |
| Multi-step research then summarize | Task Flow | Durable orchestration with revision tracking |
| Run a script on session reset | Hooks | Event-driven |
| Execute code on every tool call | Hooks | Filterable by event type |
| Always check compliance before replying | Standing Orders | Injected into every session automatically |

### Cron vs Heartbeat

| | Scheduled Tasks (Cron) | Heartbeat |
|-|-|-|
| Timing | Exact (cron expressions, one-shot) | Approximate (default every 30 min) |
| Session context | Fresh (isolated) or shared | Full main-session context |
| Task records | Always created | Never created |
| Delivery | Channel, webhook, or silent | Inline in main session |
| Best for | Reports, reminders, background jobs | Inbox checks, calendar, notifications |

### Scheduled Tasks (Cron)

Gateway built-in scheduler for precise timing. Persists jobs, wakes agent at right time, delivers output to chat channel or webhook. Supports one-shot reminders, recurring expressions, inbound webhook triggers. All cron runs create task records.

Common commands (source lines 25527-25949):

- Add one-shot: `openclaw cron add --at "2026-05-01T09:00:00Z" --task "send report"`
- List jobs: `openclaw cron list`
- Edit: `openclaw cron edit <jobId>`
- Force run now: `openclaw cron run <jobId>`
- Run only if due: `openclaw cron run-due <jobId>`
- View run history: `openclaw cron history <jobId>`
- Delete: `openclaw cron delete <jobId>`

### Background Tasks

Ledger that tracks all detached work: ACP runs, subagent spawns, isolated cron executions, CLI operations. Tasks are **records, not schedulers**.

- `openclaw tasks list` (newest first; filter by runtime/status)
- `openclaw tasks info <id|runId|sessionKey>`
- `openclaw tasks cancel <id>` — kills child session
- `openclaw tasks audit` — health audit
- `openclaw tasks flow list|show|cancel` — Task Flow inspection

### Task Flow

Flow orchestration substrate above background tasks. Manages durable multi-step flows with managed and mirrored sync modes + revision tracking.

- `openclaw tasks flow list`
- `openclaw tasks flow show <flowId>`
- `openclaw tasks flow cancel <flowId>` — cancel running flow + its active tasks

### Standing Orders

(openclaw-llms-full.txt:26380-26631)

Permanent operating authority defined in workspace files. Recommended location: `AGENTS.md` (auto-injected every session). Larger configurations can use dedicated file referenced from `AGENTS.md`.

Each program specifies:

1. **Scope** — what the agent is authorized to do.
2. **Triggers** — when to execute (schedule, event, or condition).
3. **Approval gates** — what requires human sign-off before acting.
4. **Escalation rules** — when to stop and ask for help.

Example:

```markdown
## Program: Weekly Status Report

**Authority:** Compile data, generate report, deliver to stakeholders
**Trigger:** Every Friday at 4 PM (enforced via cron job)
**Approval gate:** None for standard reports. Flag anomalies for human review.
**Escalation:** If data source unavailable or metrics look unusual (>2σ from norm)

### Execution Steps
1. Pull metrics from configured sources
2. Compare to prior week and targets
3. Generate report in Reports/weekly/YYYY-MM-DD.md
4. Deliver summary via configured channel

### What NOT to Do
- Do not send reports to external parties
- Do not modify source data
```

Combine with cron for time-based enforcement.

### Heartbeat

Periodic main-session turn (default every 30 min — see `gateway.md`). Batches multiple checks (inbox, calendar, notifications) in one agent turn with full session context. Does NOT create task records. `HEARTBEAT.md` workspace file is a small checklist; `tasks:` block inside for due-only periodic checks.

## Blueprint-Relevant Takeaways

- NemoClaw ships its own skills under `.agents/skills/` at repo root + plugin root. These should follow the AgentSkills format with single-line `metadata` JSON so the embedded parser accepts them.
- For any NemoClaw skill that requires an external binary (e.g. `uv`, `openshell`), declare it in `metadata.openclaw.requires.bins` so skills are filtered out when unavailable.
- NemoClaw can ship plugin-level skills via `openclaw.plugin.json` → `skills`, but they load at lowest precedence. For primary UX, install/link into `~/.openclaw/skills` or the workspace folder instead.
- **Hooks are the sanctioned extension point** for NemoClaw to react to agent lifecycle events (`agent:bootstrap` for seed file injection, `message:sent` for audit logging, `session:patch` for privilege-escalation monitoring).
- Do NOT bundle NemoClaw-specific hooks in a random workspace `hooks/` directory — they need to be bundled with the NemoClaw plugin or installed into `~/.openclaw/hooks/` so they load correctly.
- For NemoClaw-managed scheduled work (policy refresh, telemetry flush), prefer **cron jobs** over hooks — they produce task ledger entries that can be audited.
- Standing orders are the correct place for NemoClaw-operator-authored agent rules. NemoClaw should NOT inject its own standing orders automatically — that's the operator's workspace.
- Skill dependency installs run through OpenClaw's dangerous-code scanner. NemoClaw's own skill install paths should go through the same scanner (don't bypass).
- Remember: `skills.entries.*.env` and `skills.entries.*.apiKey` run on the **host process** (not sandbox). NemoClaw sandbox posture must assume host skill entries can access host secrets.
