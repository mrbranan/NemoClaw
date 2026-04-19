<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Plugins (Architecture, Manifest, SDK)

Condensed from `docs/vendor/openclaw-llms-full.txt`.

## Capability Model

(openclaw-llms-full.txt:28489-30160)

**Native plugins register against one or more capability types.** Each capability has a dedicated registration method on the plugin API:

| Capability | Registration method | Example plugins |
|------------|--------------------|-----------------|
| Text inference | `api.registerProvider(...)` | `openai`, `anthropic` |
| CLI inference backend | `api.registerCliBackend(...)` | `openai`, `anthropic` |
| Speech | `api.registerSpeechProvider(...)` | `elevenlabs`, `microsoft` |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider(...)` | `openai` |
| Realtime voice | `api.registerRealtimeVoiceProvider(...)` | `openai` |
| Media understanding | `api.registerMediaUnderstandingProvider(...)` | `openai`, `google` |
| Image generation | `api.registerImageGenerationProvider(...)` | `openai`, `google`, `fal`, `minimax` |
| Music generation | `api.registerMusicGenerationProvider(...)` | `google`, `minimax` |
| Video generation | `api.registerVideoGenerationProvider(...)` | `qwen` |
| Web fetch | `api.registerWebFetchProvider(...)` | `firecrawl` |
| Web search | `api.registerWebSearchProvider(...)` | `google` |
| Channel / messaging | `api.registerChannel(...)` | `msteams`, `matrix` |

A plugin that registers zero capabilities but provides hooks/tools/services is a **legacy hook-only plugin**. Still fully supported.

### Plugin Shapes

OpenClaw classifies plugins by actual registration behavior (not metadata):

- **plain-capability** — exactly one capability type (e.g. `mistral`)
- **hybrid-capability** — multiple capability types (e.g. `openai` owns text + speech + media-understanding + image-gen)
- **hook-only** — only hooks, no capabilities/tools/commands/services
- **non-capability** — tools/commands/services/routes but no capabilities

Inspect: `openclaw plugins inspect <id>`.

### Compatibility Signals

`openclaw doctor` and `openclaw plugins inspect <id>` emit:

| Signal | Meaning |
|--------|---------|
| config valid | Config parses fine, plugins resolve |
| compatibility advisory | Plugin uses supported-but-older pattern (e.g. `hook-only`) |
| legacy warning | Plugin uses `before_agent_start` (deprecated) |
| hard error | Config invalid or plugin failed to load |

Neither `hook-only` nor `before_agent_start` breaks today — advisory/warning only.

## Capability Ownership Model

**Plugin = ownership boundary. Capability = core contract that multiple plugins can implement or consume.**

- A company plugin should usually own that company's OpenClaw-facing surfaces (e.g. bundled `openai` plugin owns OpenAI text + speech + realtime-voice + media-understanding + image-gen).
- A feature plugin should usually own the full feature surface (e.g. `voice-call` owns call transport, tools, CLI, routes, Twilio bridging — but **consumes** shared speech + realtime-transcription/voice capabilities).
- Channels consume shared core capabilities; they do not re-implement provider behavior ad hoc.

When a new domain (e.g. video) arrives: **first define the core capability contract, then register vendor implementations against it.**

### Capability Layering

- **Core capability layer** — shared orchestration, policy, fallback, config merge rules, delivery semantics, typed contracts.
- **Vendor plugin layer** — vendor APIs, auth, model catalogs, speech synthesis, image gen, usage endpoints.
- **Channel/feature plugin layer** — Slack/Discord/voice-call/etc. consuming core capabilities.

## Architecture Layers

1. **Manifest + discovery** — OpenClaw finds candidate plugins from configured paths, workspace roots, global extension roots, and bundled extensions. Reads native `openclaw.plugin.json` manifests + supported bundle manifests **without executing plugin code**.
2. **Enablement + validation** — core decides enabled/disabled/blocked or exclusive-slot selection (e.g. memory).
3. **Runtime loading** — native plugins loaded in-process via `jiti`, register capabilities into a central registry. Compatible bundles normalized into registry records without importing runtime code.
4. **Surface consumption** — rest of OpenClaw reads the registry to expose tools, channels, provider setup, hooks, HTTP routes, CLI commands, services.

### Plugin CLI discovery is split:

- **Parse-time metadata** — `registerCli(..., { descriptors: [...] })` so root command names can be reserved.
- **Lazy runtime** — real plugin CLI module stays lazy and registers on first invocation.

### Key Design Boundary

- Discovery + config validation works from manifest/schema metadata only (no plugin code execution).
- Native runtime behavior comes from the plugin module's `register(api)` path.

## Channel Plugins and the Shared Message Tool

Channel plugins do NOT register a separate send/edit/react tool. OpenClaw keeps one shared `message` tool in core; channel plugins own the channel-specific discovery and execution behind it.

Boundary:

- Core owns the shared `message` tool host, prompt wiring, session/thread bookkeeping, execution dispatch.
- Channel plugins own scoped action discovery, capability discovery, channel-specific schema fragments, provider-specific session/conversation grammar, and execute the final action through their action adapter.
- SDK surface: `ChannelMessageActionAdapter.describeMessageTool(...)` returns visible actions, capabilities, and schema contributions together.
- If a message-tool param carries a media source (local path or remote media URL), plugin returns `mediaSourceParams` from `describeMessageTool(...)`. Core applies sandbox path normalization + outbound media-access hints. Use **action-scoped maps**, not one channel-wide flat list.

Runtime scope passed into discovery: `accountId`, `currentChannelId`, `currentThreadTs`, `currentMessageId`, `sessionKey`, `sessionId`, `agentId`, trusted inbound `requesterSenderId`.

Polls:

- `outbound.sendPoll` — shared baseline for channels that fit the common poll model.
- `actions.handleAction("poll")` — preferred path for channel-specific poll semantics. Core defers shared poll parsing until after plugin poll dispatch declines.

## Plugin Manifest (`openclaw.plugin.json`)

(openclaw-llms-full.txt:31396-32039)

**Every native OpenClaw plugin MUST ship `openclaw.plugin.json` in the plugin root.** OpenClaw uses it to validate configuration **without executing plugin code**. Missing/invalid manifests block config validation.

### Uses

- plugin identity
- config validation (`configSchema`)
- auth/onboarding metadata without booting runtime
- activation hints control-plane surfaces can inspect
- setup descriptors
- alias/auto-enable metadata
- shorthand model-family ownership
- static capability ownership snapshots
- QA runner metadata
- channel-specific config metadata
- config UI hints

### Do NOT use for

- registering runtime behavior
- declaring code entrypoints
- npm install metadata

Those belong in plugin code + `package.json`.

### Minimal

```json
{
  "id": "voice-call",
  "configSchema": { "type": "object", "additionalProperties": false, "properties": {} }
}
```

### Key fields

| Field | Required | Type | Meaning |
|-------|----------|------|---------|
| `id` | Yes | string | Canonical plugin id used in `plugins.entries.<id>` |
| `configSchema` | Yes | object | Inline JSON Schema for plugin config |
| `enabledByDefault` | No | `true` | Marks bundled plugin as enabled by default |
| `legacyPluginIds` | No | string[] | Legacy ids normalizing to this canonical id |
| `autoEnableWhenConfiguredProviders` | No | string[] | Provider ids that auto-enable this plugin when mentioned in auth/config/model refs |
| `kind` | No | `"memory"` \| `"context-engine"` | Declares exclusive plugin kind (`plugins.slots.*`) |
| `channels` | No | string[] | Channel ids owned by plugin |
| `providers` | No | string[] | Provider ids owned by plugin |
| `modelSupport` | No | object | Shorthand model-family metadata for auto-load |
| `providerEndpoints` | No | object[] | Endpoint host/baseUrl metadata for route classification |
| `cliBackends` | No | string[] | CLI inference backend ids |
| `syntheticAuthRefs` | No | string[] | Refs whose synthetic auth hook is probed during cold model discovery |
| `nonSecretAuthMarkers` | No | string[] | Placeholder API key values representing non-secret local/OAuth/ambient state |
| `commandAliases` | No | object[] | Plugin-owned command names for plugin-aware diagnostics |
| `providerAuthEnvVars` | No | `Record<string, string[]>` | Provider-auth env metadata (no code load) |
| `providerAuthAliases` | No | `Record<string, string>` | Provider ids that reuse another provider's auth lookup |
| `channelEnvVars` | No | `Record<string, string[]>` | Channel env metadata |
| `providerAuthChoices` | No | object[] | Auth-choice metadata for onboarding pickers |
| `activation` | No | object | Cheap activation hints for provider/command/channel/route triggers |
| `setup` | No | object | Setup/onboarding descriptors |
| `qaRunners` | No | object[] | QA runner descriptors (`openclaw qa` host) |
| `contracts` | No | object | Static bundled capability snapshot for speech/transcription/voice/media-understanding/image-gen/music-gen/video-gen/web-fetch/web-search/tool ownership |
| `channelConfigs` | No | `Record<string, object>` | Channel config metadata merged into discovery/validation |
| `skills` | No | string[] | Skill directories relative to plugin root |
| `name` | No | string | Human-readable name |
| `description` | No | string | Short summary for plugin surfaces |
| `version` | No | string | Informational version |
| `uiHints` | No | `Record<string, object>` | UI labels/placeholders/sensitivity hints |

### Bundled vs Native Manifest Files

OpenClaw auto-detects compatible bundle layouts but does NOT validate them against `openclaw.plugin.json` schema:

- Codex bundle: `.codex-plugin/plugin.json`
- Claude bundle: `.claude-plugin/plugin.json` or default Claude component layout
- Cursor bundle: `.cursor-plugin/plugin.json`

For compatible bundles OpenClaw currently reads metadata + declared skill roots + Claude command roots + Claude bundle `settings.json` defaults + Claude bundle LSP defaults + supported hook packs when the layout matches.

## Plugin SDK (Imports + Registration)

(openclaw-llms-full.txt:34002-34517)

### Import convention

Always import from a specific subpath — each is a small self-contained module to keep startup fast and avoid circular deps:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

### Key entry-point subpaths

| Subpath | Key exports |
|---------|-------------|
| `plugin-sdk/plugin-entry` | `definePluginEntry` |
| `plugin-sdk/core` | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema` (root `openclaw.json` Zod schema) |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
| `plugin-sdk/channel-core` | channel plugin entry helpers |

### Don't add these

- Provider-named convenience seams like `plugin-sdk/slack`, `plugin-sdk/discord`, `plugin-sdk/signal`, `plugin-sdk/whatsapp`. Bundled plugins should compose generic subpaths inside their own `api.ts` / `runtime-api.ts` barrels.
- Core should NOT import channel-specific convenience barrels. If core needs cross-channel behavior, promote it to a narrow generic SDK contract.

### Subpath categories (200+ total; see `scripts/lib/plugin-sdk-entrypoints.json`)

- Plugin entry, channel, provider — as above.
- Auth & security: `plugin-sdk/command-auth`, `approval-*`, `ssrf-policy`, `ssrf-dispatcher`, `ssrf-runtime`, `secret-input`, `webhook-ingress`, `security-runtime`.
- Runtime & storage: `runtime`, `runtime-env`, `plugin-runtime`, `hook-runtime`, `lazy-runtime`, `process-runtime`, `gateway-runtime`, `config-runtime`, `reply-runtime`, `session-store-runtime`.
- Channel internals: `channel-setup`, `channel-pairing`, `channel-reply-pipeline`, `channel-location`, `channel-logging`, `channel-contract`, `interactive-runtime`, `channel-mention-gating`.
- Provider internals: `provider-setup`, `provider-auth-*`, `provider-catalog-shared`, `provider-stream`, `provider-transport-runtime`, `provider-web-fetch`, `provider-web-search`, `provider-tools`, `provider-usage`.

## Legacy Hooks

- `before_agent_start` remains supported as compatibility path for hook-only plugins. Many real plugins still depend on it.
- Direction: keep working, mark legacy, **prefer `before_model_resolve`** for model/provider override work, **prefer `before_prompt_build`** for prompt mutation.
- Remove only after usage drops and fixture coverage proves migration safety.

## Blueprint-Relevant Takeaways

- NemoClaw ships as an OpenClaw plugin (`nemoclaw/openclaw.plugin.json`). The manifest MUST follow the schema above — treat it as the contract.
- For sandbox lifecycle work, NemoClaw registers against `plugins.entries.openshell.config.*` (see the OpenShell backend in `sandboxing.md`). The NemoClaw plugin consumes that plugin-owned config surface.
- Use capability registration (`api.registerProvider`, `api.registerChannel`, etc.) for new NemoClaw capabilities — not hook-only plugins. Hook-only is advisory-only and drifts from the preferred model.
- For security-sensitive work (SSRF validation in `nemoclaw/src/blueprint/ssrf.ts`) use `plugin-sdk/ssrf-policy`, `plugin-sdk/ssrf-dispatcher`, or `plugin-sdk/ssrf-runtime` rather than rolling custom host-allowlist logic.
- For secret handling, use `plugin-sdk/secret-ref-runtime` (`coerceSecretRef`) and `plugin-sdk/secret-input` — do NOT inline plaintext secrets in config.
- Channel policy presets should rely on the shared `message` tool (core-owned) rather than per-channel tools.
- Plugin manifest config validation happens BEFORE plugin code runs — NemoClaw must make config-validation failures surface cleanly without requiring a sandbox bring-up.
- `autoEnableWhenConfiguredProviders` is the right mechanism for NemoClaw inference routing to auto-activate the right provider plugin when onboarding configures it.
