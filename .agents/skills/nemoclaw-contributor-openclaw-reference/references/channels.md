<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Channels: Routing, Pairing, and Per-Channel Setup

Condensed from `docs/vendor/openclaw-llms-full.txt`. Focus: what NemoClaw blueprint/policy-preset authors need to know.

## Channels & Routing

(openclaw-llms-full.txt:877-1015)

**OpenClaw routes replies back to the channel where a message came from.** The model does not choose a channel; routing is deterministic and controlled by host config.

### Key terms

- **Channel:** `telegram`, `whatsapp`, `discord`, `irc`, `googlechat`, `slack`, `signal`, `imessage`, `line`, plus extension channels. `webchat` is internal, not configurable as outbound.
- **AccountId:** per-channel account instance (multi-account).
- Channel default: `channels.<channel>.defaultAccount` picks account when outbound doesn't specify `accountId`. In multi-account setups, set an explicit default — else fallback picks the first normalized account ID.
- **AgentId:** isolated workspace + session store ("brain").
- **SessionKey:** bucket key storing context + controlling concurrency.

### Session Key Shapes

DMs collapse to agent's main session:

- `agent:<agentId>:<mainKey>` (default: `agent:main:main`)

Groups/channels stay isolated per channel:

- Groups: `agent:<agentId>:<channel>:group:<id>`
- Channels/rooms: `agent:<agentId>:<channel>:channel:<id>`

Threads:

- Slack/Discord: append `:thread:<threadId>` to base key.
- Telegram forum topics: embed `:topic:<topicId>` in group key.

Examples: `agent:main:telegram:group:-1001234567890:topic:42`, `agent:main:discord:channel:123456:thread:987654`.

### Main DM Route Pinning

When `session.dmScope: "main"`, multiple DMs may share one main session. To prevent `lastRoute` from being overwritten by non-owner DMs, OpenClaw infers a **pinned owner** from `allowFrom` when:

- `allowFrom` has exactly one non-wildcard entry.
- That entry normalizes to a concrete sender ID for the channel.
- The inbound sender does not match that pinned owner.

In mismatch case, inbound session metadata still recorded; main session `lastRoute` is NOT updated.

### Routing Rules (agent selection, in order)

1. **Exact peer match** — `bindings` with `peer.kind` + `peer.id`.
2. **Parent peer match** — thread inheritance.
3. **Guild + roles match** (Discord) — `guildId` + `roles`.
4. **Guild match** (Discord) — `guildId`.
5. **Team match** (Slack) — `teamId`.
6. **Account match** — `accountId` on the channel.
7. **Channel match** — any account on that channel (`accountId: "*"`).
8. **Default agent** — `agents.list[].default`, else first list entry, fallback `main`.

When a binding includes multiple match fields (`peer`, `guildId`, `teamId`, `roles`), **all provided fields must match**.

### Broadcast Groups

Run multiple agents for the same peer when OpenClaw would normally reply:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

### Config Overview

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

### Session Storage

- `~/.openclaw/agents/<agentId>/sessions/sessions.json` + JSONL transcripts alongside.
- Override path via `session.store` with `{agentId}` templating.
- Gateway and ACP session discovery scans disk-backed agent stores under default `agents/` root and templated `session.store` roots.
- Discovered stores must stay inside resolved agent root and use a regular `sessions.json` file. Symlinks and out-of-root paths are ignored.

### WebChat Behavior

WebChat attaches to the selected agent and defaults to that agent's main session, giving cross-channel context for one agent in one place.

### Reply Context (cross-channel)

Inbound replies include `ReplyToId`, `ReplyToBody`, `ReplyToSender` when available. Quoted context is appended to `Body` as `[Replying to ...]` block.

## Pairing

(openclaw-llms-full.txt:7319-7445)

"Pairing" is OpenClaw's explicit **owner approval** step for:

1. **DM pairing** — who can talk to the bot.
2. **Node pairing** — which devices/nodes join the gateway network.

### DM Pairing

Channel with `dmPolicy: "pairing"` gives unknown senders a short code; the message is **not processed** until you approve.

**Pairing codes:**

- 8 characters, uppercase, no ambiguous chars (`0O1I`).
- **Expire after 1 hour.** Pairing message only sent on new request (≈ once per hour per sender).
- Pending DM pairing requests **capped at 3 per channel**; additional requests ignored until one expires or is approved.

**Approve:**

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

**Supported channels:** `bluebubbles`, `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `openclaw-weixin`, `signal`, `slack`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

**State:**

- Pending: `~/.openclaw/credentials/<channel>-pairing.json`
- Approved allowlist:
  - Default account: `~/.openclaw/credentials/<channel>-allowFrom.json`
  - Non-default: `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json`
- Non-default accounts read/write only their scoped allowlist.
- Default account uses channel-scoped unscoped allowlist.

Treat as sensitive — gates access to the assistant.

**Important:** this store is DM-access only. Group authorization is **separate**. DM approval does NOT allow the sender to run group commands or control the bot in groups. Use channel's explicit group allowlists (`groupAllowFrom`, `groups`, per-group/per-topic overrides).

### Node Device Pairing

Nodes connect to Gateway as devices with `role: node`. Gateway creates a device pairing request requiring approval.

Pair via Telegram (`device-pair` plugin):

1. Message bot: `/pair`
2. Bot replies with instruction + separate **setup code** message (copy-paste friendly).
3. iOS app → Settings → Gateway → paste setup code → connect.
4. In Telegram: `/pair pending`, approve.

Setup code = base64-encoded JSON containing:

- `url` — Gateway WS URL (`ws://` or `wss://`)
- `bootstrapToken` — short-lived single-device bootstrap token for initial handshake

Bootstrap token carries the built-in pairing bootstrap profile:

- Primary handed-off `node` token stays `scopes: []`.
- Handed-off `operator` token is bounded to bootstrap allowlist: `operator.approvals`, `operator.read`, `operator.talk.secrets`, `operator.write`.
- Bootstrap scope checks are role-prefixed, not one flat pool.

Treat setup code like a password while valid.

**Approve via CLI:**

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Device retry with different auth (role/scopes/public key) supersedes the previous pending request with a new `requestId`.

**State:** `~/.openclaw/devices/{pending.json, paired.json}`.

## Common Channel Config Shape

Every channel shares the DM policy pattern:

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
| `pairing` (default) | Unknown senders get pairing code |
| `allowlist` | Only `allowFrom` or paired store |
| `open` | All DMs (requires `allowFrom: ["*"]`) |
| `disabled` | Ignore all DMs |

| Group policy | Behavior |
|--------------|----------|
| `allowlist` (default) | Only groups matching configured allowlist |
| `open` | Bypass group allowlists (mention-gating still applies) |
| `disabled` | Block all group/room messages |

## Per-Channel Notes (Blueprint-Relevant)

Each channel has its own config section `channels.<provider>` and its own network egress requirements. Full per-channel reference in `docs/vendor/openclaw-llms-full.txt`. Highlights:

| Channel | Lines | Blueprint relevance |
|---------|-------|--------------------|
| **Slack** | 7978-8999 | Bot + user tokens, Socket Mode vs Events API, `teamId`-based routing. Policy preset must allow `slack.com`, Events API webhook if using. |
| **Discord** | 1021-2232 | Bot token, `guildId`-based routing, threads support (`sessions_spawn` with `thread: true`), `channels.discord.threadBindings.*`. Policy preset allows `discord.com`, `gateway.discord.gg` (WSS). |
| **Telegram** | 9189-10270 | Bot API via grammY; forum topics use `:topic:<id>` in session key. Webhook vs long-poll. Policy preset allows `api.telegram.org`. |
| **WhatsApp** | 11258-11720 | Baileys WebSocket session — **exactly one Gateway per host**. `allowFrom` strongly recommended. Policy preset allows WhatsApp Web endpoints. |
| **Signal** | 7642-7776 | `signal-cli`-backed. Runs as process, not HTTP. |
| **BlueBubbles** | 1-436 | macOS REST server. Webhook password required; auth checked before body parse. |
| **iMessage** (legacy `imsg`) | 3439-3844 | macOS AppleScript path — BlueBubbles is recommended instead. |
| **Matrix** | 4423-5483 | Homeserver URL + access token. |
| **Mattermost** | 5489-5954 | Personal access token or bot token. |
| **Microsoft Teams** | 5955-6915 | Bot Framework endpoint + ngrok/Tailscale funnel for messaging endpoint. |
| **Google Chat** | 2672-2916 | Chat API via GCP service account. |
| **Feishu / Lark** | 2233-2671 | App token + verification. |
| **LINE, IRC, Nostr, Nextcloud Talk, QQ Bot, Synology Chat, Twitch, WeChat, Zalo, Tlon** | various | See source. |

**Common pattern:** NemoClaw network policy presets (`nemoclaw-blueprint/policies/presets/<channel>.yaml`) should allow:

- Channel API endpoints (REST).
- Channel WebSocket/gateway endpoints (if used).
- OAuth endpoints for channels that use OAuth (Slack, Discord, Google, Microsoft).
- Webhook inbound port/path (if channel pushes to the Gateway).

Stay narrow: no broad TLDs. Channel policy presets should only allow what the channel actually needs.

## Channel Location Parsing

(openclaw-llms-full.txt:4373-4422) — parsing helpers (`plugin-sdk/channel-location`) for `[Location: ...]` parsing in inbound channels.

## Channel Troubleshooting

(openclaw-llms-full.txt:10566-10692) — per-channel diagnostics. Common: `openclaw channels logout && openclaw channels login --verbose` to relink when status 409–515 or `loggedOut` appears.

## Blueprint-Relevant Takeaways

- NemoClaw's policy presets live in `nemoclaw-blueprint/policies/presets/<channel>.yaml`. Each preset should allow only the specific endpoints that channel needs — no broad wildcards.
- **NemoClaw can't bypass pairing.** DM pairing state is the allowlist; respect the pairing flow in NemoClaw docs and don't pre-populate `<channel>-allowFrom.json` without operator consent.
- When NemoClaw's sandbox restarts, `~/.openclaw/credentials/` must be preserved (pairing store). Treat it as durable state.
- Telegram's policy preset pattern is already in the NemoClaw blueprint — model others on that shape.
- WhatsApp policy preset: **one Gateway per host only.** NemoClaw must not run multiple sandboxed Gateways sharing a WhatsApp session.
- Webhook-based channels (BlueBubbles, Teams, Nextcloud Talk) need inbound port exposure. The blueprint must expose the Gateway HTTP port (not the WS port) to the webhook source — with webhook password validation **before body parse**.
- For channels that use OAuth, the preset must allow OAuth endpoints (e.g. `login.microsoftonline.com` for Teams). Don't conflate with channel API endpoints.
- Broadcast groups need multiple agent binding entries — NemoClaw config generators should produce those explicitly, not try to infer.
