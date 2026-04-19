<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Model Providers, Failover, and Authentication

Condensed from `docs/vendor/openclaw-llms-full.txt`. Focus: what NemoClaw's inference routing layer needs to know.

## Model Providers Overview

(openclaw-llms-full.txt:27386-28088)

Covers **LLM/model providers**, not chat channels.

### Quick Rules

- Model refs: `provider/model` (e.g. `anthropic/claude-sonnet-4-6`).
- If `agents.defaults.models` is set, it becomes the allowlist.
- CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- `models.providers.*.models[].contextWindow` = native model metadata; `contextTokens` = effective runtime cap.
- Provider plugins can inject model catalogs via `registerProvider({ catalog })` — OpenClaw merges into `models.providers` before writing `models.json`.
- Provider manifests can declare `providerAuthEnvVars` and `providerAuthAliases` so generic env-based auth probes and provider variants don't need to load plugin runtime.

### Bundled `codex` Provider Note

The bundled `codex` provider is paired with the bundled Codex agent harness. Use `codex/gpt-*` for Codex-owned login, model discovery, native thread resume, and app-server execution. Plain `openai/gpt-*` continues to use the OpenAI provider with normal OpenClaw transport.

Codex-only deployments can disable automatic PI fallback with `agents.defaults.embeddedHarness.fallback: "none"`.

## Plugin-Owned Provider Behavior

Provider plugins can own most provider-specific logic while OpenClaw keeps the generic inference loop. Typical extension points:

- **Auth flow:** `auth[].run`, `auth[].runNonInteractive` — onboarding/login for `openclaw onboard`, `openclaw models auth`, headless setup.
- **Wizard:** `wizard.setup`, `wizard.modelPicker` — auth-choice labels, legacy aliases, onboarding allowlist hints.
- **Catalog:** provider entry in `models.providers`.
- **Model ID normalization:** `normalizeModelId`, `normalizeResolvedModel`, `resolveDynamicModel`, `prepareDynamicModel`.
- **Transport/config normalization:** `normalizeTransport`, `normalizeConfig`, `applyNativeStreamingUsageCompat`.
- **Auth resolution:** `resolveConfigApiKey`, `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`, `formatApiKey`, `refreshOAuth`, `buildAuthDoctorHint`, `prepareRuntimeAuth`, `resolveUsageAuth`.
- **Capabilities / tooling quirks:** `capabilities`, `normalizeToolSchemas`, `inspectToolSchemas`, `contributeResolvedModelCompat`.
- **Stream behavior:** `createStreamFn`, `wrapStreamFn`, `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`.
- **Reasoning/thinking:** `resolveReasoningOutputMode`, `isBinaryThinking`, `supportsXHighThinking`, `resolveDefaultThinkingLevel`, `prepareExtraParams`.
- **Error classification:** `matchesContextOverflowError`, `classifyFailoverReason`, `buildMissingAuthMessage`, `suppressBuiltInModel`.
- **Catalog lifecycle:** `augmentModelCatalog`, `isCacheTtlEligible`, `isModernModelRef`, `applyConfigDefaults`, `onModelSelected`.
- **Usage/quota:** `fetchUsageSnapshot`, `resolveUsageAuth`.
- **Embedding:** `createEmbeddingProvider`.

> Runtime `capabilities` is shared runner metadata (provider family, transcript/tooling quirks, transport/cache hints). **Not** the public capability model used by `api.registerProvider(...)` (which describes what the plugin registers: text inference, speech, etc.).

## Bundled Provider Examples

- **anthropic** — Claude forward-compat fallback, auth repair hints, usage fetching, cache-TTL + provider-family metadata.
- **amazon-bedrock** — provider-owned context-overflow matching + failover classification for Bedrock-specific throttle/not-ready. Shared `anthropic-by-model` replay family.
- **anthropic-vertex** — Claude-only replay-policy guards on Anthropic-message traffic.
- **openai** — GPT-5.4 forward-compat fallback, direct OpenAI transport normalization, Codex-aware missing-auth hints, synthetic OpenAI/Codex catalog rows, usage-token alias normalization, bundled image-gen for `gpt-image-1`, bundled video-gen for `sora-2`. Owns both `openai` and `openai-codex` provider ids.
- **google** / **google-gemini-cli** — Gemini 3.1 forward-compat, native Gemini replay validation, bootstrap replay sanitation, tagged reasoning-output mode, image-gen (Gemini image-preview models), video-gen (Veo models). Gemini CLI OAuth owns auth-profile token formatting, usage-token parsing, quota endpoint fetching.
- **openrouter** — pass-through model ids, request wrappers, provider capability hints, Gemini thought-signature sanitation, proxy reasoning injection via `openrouter-thinking` stream family, routing metadata forwarding, cache-TTL policy.
- **github-copilot** — device login, forward-compat model fallback, Claude-thinking transcript hints, runtime token exchange, usage endpoint fetching.
- **moonshot**, **kilocode**, **zai**, **xai**, **mistral**, **qwen**, **minimax**, **together**, **alibaba**, **byteplus**, **fal**, **runway**, **cloudflare-ai-gateway**, **huggingface**, **kimi**, **nvidia**, **qianfan**, **stepfun**, **synthetic**, **venice**, **vercel-ai-gateway**, **volcengine**, **xiaomi** — each with varying combinations of catalogs, transport quirks, video-gen, image-gen, and usage auth.

## API Key Rotation

Generic provider rotation for selected providers:

- `OPENCLAW_LIVE_<PROVIDER>_KEY` — single live override, highest priority.
- `<PROVIDER>_API_KEYS` — comma or semicolon list.
- `<PROVIDER>_API_KEY` — primary key.
- `<PROVIDER>_API_KEY_*` — numbered list (e.g. `<PROVIDER>_API_KEY_1`).
- Google providers: `GOOGLE_API_KEY` also included as fallback.

Rules:

- Key selection preserves priority and deduplicates values.
- Retried with next key **only on rate-limit responses** (429, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, periodic usage-limit messages).
- Non-rate-limit failures fail immediately; no key rotation.
- All keys fail → final error returned from last attempt.

## Model Failover

(openclaw-llms-full.txt:27043-27385)

**Two stages:**

1. **Auth profile rotation** within the current provider.
2. **Model fallback** to next model in `agents.defaults.model.fallbacks`.

### Runtime Flow

Candidate order for a text run:

1. Currently selected session model.
2. Configured `agents.defaults.model.fallbacks` in order.
3. Configured primary model at the end (when the run started from an override).

Within each candidate, try auth-profile failover before advancing to the next model.

Sequence:

1. Resolve active session model + auth-profile preference.
2. Build model candidate chain.
3. Try current provider with auth-profile rotation/cooldown rules.
4. If exhausted with failover-worthy error → next model candidate.
5. Persist selected fallback override **before** retry starts (so other session readers see the same provider/model the runner is about to use).
6. If fallback fails, roll back only fallback-owned session override fields when they still match that failed candidate.
7. If all fail → `FallbackSummaryError` with per-attempt detail + soonest cooldown expiry.

Reply runner only persists model-selection fields it owns for fallback: `providerOverride`, `modelOverride`, `authProfileOverride`, `authProfileOverrideSource`, `authProfileOverrideCompactionCount`. Prevents failed fallback retry from overwriting unrelated mutations (manual `/model`, session rotation).

## Auth Storage (Keys + OAuth)

OpenClaw uses **auth profiles** for both API keys and OAuth.

- **Secrets:** `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (legacy: `~/.openclaw/agent/auth-profiles.json`).
- **Runtime auth-routing state:** `~/.openclaw/agents/<agentId>/agent/auth-state.json`.
- Config `auth.profiles` / `auth.order` = **metadata + routing only** (no secrets).
- Legacy import-only OAuth file: `~/.openclaw/credentials/oauth.json` (imported into `auth-profiles.json` on first use).

### Credential Types

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` for some providers)

### Profile IDs

OAuth logins create distinct profiles so multiple accounts coexist:

- Default: `provider:default` when no email available.
- OAuth with email: `provider:<email>` (e.g. `google-antigravity:user@gmail.com`).

### Rotation Order

1. **Explicit config:** `auth.order[provider]` if set.
2. **Configured profiles:** `auth.profiles` filtered by provider.
3. **Stored profiles:** entries in `auth-profiles.json`.

No explicit order → round-robin:

- Primary key: profile type (**OAuth before API keys**).
- Secondary: `usageStats.lastUsed` (oldest first, within each type).
- Cooldown/disabled profiles moved to end, ordered by soonest expiry.

### Session Stickiness

OpenClaw **pins the chosen auth profile per session** to keep provider caches warm. Pinned profile reused until:

- Session is reset (`/new`, `/reset`)
- Compaction completes (compaction count increments)
- Profile in cooldown/disabled

Manual selection via `/model …@<profileId>` = **user override** for that session (not auto-rotated until new session).

Auto-pinned profiles = **preference**: tried first, but may rotate on rate limits/timeouts. User-pinned profiles stay locked; if they fail and model fallbacks are configured, OpenClaw moves to the next model instead of switching profiles.

### "OAuth can look lost"

Both OAuth + API-key profile for same provider → round-robin can switch between them unless pinned. Force single profile:

- Pin with `auth.order[provider] = ["provider:profileId"]`, or
- Use per-session override via `/model …` with profile override.

## Cooldowns

Triggered by auth/rate-limit errors (+ timeout-that-looks-like-rate-limiting). Broader than plain 429:

- `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, `throttled`, `resource exhausted`.
- Periodic usage-window limits: `weekly/monthly limit reached`.
- Format/invalid-request errors (e.g. Cloud Code Assist tool call ID validation) → same cooldowns.
- OpenAI-compatible stop-reason: `Unhandled stop reason: error`, `stop reason: error`, `reason: error`.
- Provider-scoped generic server text (e.g. Anthropic bare `An unknown error occurred`, JSON `api_error` with `internal server error`/`unknown error, 520`/`upstream error`/`backend error`) → treated as failover-worthy timeouts.
- OpenRouter-specific: bare `Provider returned error` treated as timeout only when provider context is actually OpenRouter.
- Generic internal fallback text `LLM request failed with an unknown error.` stays conservative and does NOT trigger failover.

### Model-Scoped Cooldowns

- `cooldownModel` recorded for rate-limit failures when failing model id is known.
- Sibling model on same provider can still be tried when cooldown is scoped to a different model.
- Billing/disabled windows still block the whole profile across models.

### Exponential Backoff

- 1 minute → 5 minutes → 25 minutes → 1 hour (cap)

State in `auth-state.json` under `usageStats`.

## Authentication (Model Providers)

(openclaw-llms-full.txt:52586-52765) — Covers env-var precedence, OAuth flow, auth-profile materialization, env-marker auth for config providers (non-secret local/OAuth markers in `auth-profiles.json`).

## Auth Credential Semantics

(openclaw-llms-full.txt:52502-52580) — Defines what counts as a "non-secret marker" vs a real secret. Plugin manifests can declare `nonSecretAuthMarkers` for placeholder API key values representing local/OAuth/ambient state that should NOT be treated as secrets.

## Blueprint-Relevant Takeaways

- NemoClaw's inference routing sets `agents.defaults.model.primary` + `agents.defaults.model.fallbacks`. The fallback list is the model failover order — keep it ordered by preference, not alphabetically.
- **Secrets live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`** — NemoClaw must preserve this across sandbox restarts, and never expose the full path to guest code.
- NemoClaw's NIM / Ollama / vLLM integrations are **provider plugins** that implement `resolveSyntheticAuth` (local server availability) + `normalizeTransport` (rewrite to the local endpoint). See `plugin-sdk/self-hosted-provider-setup`.
- Round-robin rotation + OAuth-first ordering means NemoClaw onboarding should NOT inject an API-key profile alongside OAuth unless operator explicitly asks — else round-robin will switch between them across messages.
- Cooldown backoff caps at 1 hour. NemoClaw health monitor restarts must not churn faster than this or it'll fight the cooldown state.
- Use `auth.order[provider] = ["provider:profileId"]` in NemoClaw blueprint to pin a specific profile when the operator wants deterministic routing (e.g. single corporate OAuth).
- `OPENCLAW_LIVE_<PROVIDER>_KEY` is the highest-priority env override; NemoClaw can inject this for CI/CD deployments that shouldn't touch `auth-profiles.json`.
- For any new NemoClaw-supported provider, prefer registering a provider plugin with `api.registerProvider(...)` + manifest `providerAuthEnvVars`/`providerAuthAliases` rather than hardcoding provider knowledge in NemoClaw itself.
