<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Configuration

Condensed from `docs/vendor/openclaw-llms-full.txt`. Full schema: `openclaw config schema`.

## Source of Truth

OpenClaw reads optional **JSON5** (comments + trailing commas) config from `~/.openclaw/openclaw.json`. Missing file → safe defaults. (openclaw-llms-full.txt:53491-54177)

- **Canonical schema:** `openclaw config schema` — same JSON Schema used by validation and Control UI, with bundled/plugin/channel metadata merged in.
- **Drill-down:** `config.schema.lookup` returns one path-scoped schema node (`title`, `description`, `type`, `enum`, `const`, bounds, UI hints + immediate child summaries).
- **Docs drift check:** `pnpm config:docs:check` (and `config:docs:gen`) compare the baseline hash against current schema surface.

## Minimal Config

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Editing Config

| Method | How |
|--------|-----|
| Interactive wizard | `openclaw onboard`, `openclaw configure` |
| CLI one-liners | `openclaw config get <path>`, `openclaw config set <path> <value>`, `openclaw config unset <path>` |
| Control UI | `http://127.0.0.1:18789` → Config tab (form from live schema + Raw JSON escape hatch) |
| Direct edit | Edit file; Gateway watches and hot-reloads |

## Strict Validation

**OpenClaw only accepts configurations that fully match the schema.** Unknown keys, malformed types, or invalid values cause the Gateway to **refuse to start**. Only root-level exception: `$schema` (string) for editor metadata.

When validation fails:

- Gateway does not boot.
- Only diagnostic commands work: `openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`.
- `openclaw doctor` shows exact issues; `openclaw doctor --fix` (or `--yes`) applies repairs.

Schema metadata (`title`, `description`) carries into generated schema for editor and form tooling. Nested object, wildcard (`*`), and array-item (`[]`) entries inherit docs metadata from matching field docs. `anyOf` / `oneOf` / `allOf` composition branches inherit too.

## Top-Level Config Surfaces

Full reference at openclaw-llms-full.txt:54807-58648. High-level map:

| Key | Owns |
|-----|------|
| `agents.defaults.*` | Default workspace, model, heartbeat, sandbox, timeouts, compaction, memorySearch, skills, tools, llm, bootstrap limits, blockStreaming |
| `agents.list[]` | Per-agent overrides for any `agents.defaults.*` key, plus `id`, `groupChat.mentionPatterns` |
| `channels.<provider>` | Per-channel config (WhatsApp, Telegram, Discord, Feishu, Google Chat, MS Teams, Slack, Signal, iMessage, Mattermost, BlueBubbles, Matrix, Tlon, IRC, LINE, Nostr, Nextcloud Talk, QQ Bot, Synology Chat, Twitch, WeChat, Zalo). |
| `channels.defaults.*` | Shared defaults (`groupPolicy`, heartbeat behavior). |
| `channels.modelByChannel.<provider>.<id>` | Pin specific channel IDs to a model (takes `provider/model` or configured alias). Applies when session has no model override. |
| `gateway.*` | Bind host/port, auth mode, TLS, health monitoring thresholds, push relay |
| `session.*` | `dmScope`, `threadBindings`, `reset` policy, identity links |
| `tools.*` | Global/per-provider/sandbox tool policy, elevated gates |
| `plugins.entries.<name>` | Plugin enablement + plugin-owned config (e.g. `plugins.entries.openshell.config.*`) |
| `skills.*` | Skill loading, `extraDirs`, gating |
| `memory.*` | Memory engine knobs (full reference separate) |
| `docker.*` | Bind mounts, network (Docker backend only) |
| `nodes.*` | Node-specific config |

## DM and Group Access (all channels)

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["tg:123"],
    },
  },
}
```

| DM policy | Behavior |
|-----------|----------|
| `pairing` (default) | Unknown senders get one-time pairing code; owner approves |
| `allowlist` | Only senders in `allowFrom` (or paired allow store) |
| `open` | Allow all inbound DMs (requires `allowFrom: ["*"]`) |
| `disabled` | Ignore all inbound DMs |

| Group policy | Behavior |
|--------------|----------|
| `allowlist` (default) | Only groups matching allowlist |
| `open` | Bypass group allowlists (mention-gating still applies) |
| `disabled` | Block all group/room messages |

Pairing codes expire after 1 hour. Pending DM pairing requests capped at **3 per channel**. If a provider block is absent entirely, runtime group policy fails closed to `allowlist` with a startup warning.

## Model Configuration

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["openai/gpt-5.4"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "openai/gpt-5.4": { alias: "GPT" },
      },
      imageMaxDimensionPx: 1200, // transcript/tool image downscaling (default 1200)
    },
  },
}
```

- `agents.defaults.models` defines the catalog + allowlist for `/model`.
- Model refs use `provider/model` format. OpenRouter-style IDs need explicit prefix (`openrouter/moonshotai/kimi-k2`).

## Group Mention Gating

```json5
{
  agents: {
    list: [
      { id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } },
    ],
  },
  channels: {
    whatsapp: { groups: { "*": { requireMention: true } } },
  },
}
```

- Metadata mentions (native @-mentions, WhatsApp tap-to-mention, Telegram `@bot`) + text patterns (safe regex in `mentionPatterns`).

## Skills per Agent

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" },                        // inherits defaults
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] },       // no skills
    ],
  },
}
```

- Omit `agents.defaults.skills` for unrestricted default.
- `[]` = no skills.

## Gateway Channel Health

```json5
{
  gateway: {
    channelHealthCheckMinutes: 5,       // 0 disables
    channelStaleEventThresholdMinutes: 30, // must be ≥ check interval
    channelMaxRestartsPerHour: 10,
  },
  channels: {
    telegram: {
      healthMonitor: { enabled: false },
      accounts: {
        alerts: { healthMonitor: { enabled: true } },
      },
    },
  },
}
```

## Session Controls

```json5
{
  session: {
    dmScope: "per-channel-peer", // main | per-peer | per-channel-peer | per-account-channel-peer
    threadBindings: { enabled: true, idleHours: 24, maxAgeHours: 0 },
    reset: { mode: "daily", atHour: 4, idleMinutes: 120 },
  },
}
```

## Sandbox

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",   // off | non-main | all
        scope: "agent",     // session | agent | shared
      },
    },
  },
}
```

Build Docker image first: `scripts/sandbox-setup.sh`.

## Relay-Backed Push (Official iOS Builds)

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000, // default 10000
        },
      },
    },
  },
}
```

- Lets the gateway send `push.test`, wake nudges, reconnect wakes through external relay.
- Uses registration-scoped send grant forwarded by paired iOS app. Gateway does not need deployment-wide relay token.
- Binds each relay registration to the paired gateway identity.
- Only applies to official/TestFlight builds that registered through the relay. Local/manual builds use direct APNs.

## Deep References (Not Inlined Here)

- **Memory config:** `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations`, `plugins.entries.memory-core.config.dreaming.*` — see upstream `/reference/memory-config`.
- **Slash command catalog:** `/tools/slash-commands`.
- **Per-channel/per-plugin command surfaces:** owned by each channel/plugin page.

## Blueprint-Relevant Takeaways

- NemoClaw's blueprint must produce a config that passes strict schema validation — unknown keys block Gateway boot. Always diff against `openclaw config schema` output before shipping.
- Hot reload is a feature: NemoClaw can write to `~/.openclaw/openclaw.json` and Gateway will pick up changes without restart. Good for runtime mutations but risky — validation happens on reload too.
- `agents.defaults.model.fallbacks` is the mechanism for inference routing fallbacks; NemoClaw's inference routing should populate this via the primary-provider preset.
- When NemoClaw runs on `backend: "openshell"`, the plugin config at `plugins.entries.openshell.config.*` is the NemoClaw-facing surface — keep that aligned with the NemoClaw blueprint's OpenShell expectations.
- For multi-channel deployments, `channels.modelByChannel.<provider>.<id>` is how NemoClaw can route different channels to different models without per-agent duplication.
- `gateway.channelMaxRestartsPerHour` (default 10) will throttle NemoClaw sandbox restart loops — watch for suppression if NemoClaw is itself auto-restarting.
