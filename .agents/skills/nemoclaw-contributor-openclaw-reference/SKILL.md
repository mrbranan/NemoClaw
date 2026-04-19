---
name: "nemoclaw-contributor-openclaw-reference"
description: "OpenClaw architecture and subsystem reference for NemoClaw contributors working on the blueprint, plugin, or policy presets. Use when building or modifying NemoClaw blueprint YAML, network policy presets, or plugin code that interacts with OpenClaw's runtime, gateway, sandbox, channels, model providers, or secrets. Covers agent runtime and loop, gateway architecture and protocol, sandboxing and network model, configuration keys, plugin manifest and SDK, delegate and sub-agent composition, channel integrations, model providers and failover, secrets management, and skills/hooks/tasks. Trigger keywords - openclaw architecture, openclaw gateway, openclaw sandbox, openclaw plugin, openclaw config, delegate architecture, sub-agents, channel preset, provider plugin, secrets, openclaw skills, hooks, standing orders."
---

<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# OpenClaw Reference for NemoClaw Contributors

Condensed reference of OpenClaw subsystems that NemoClaw contributors need to understand when working on the blueprint, plugin, or policy presets. Distilled from the upstream `llms-full.txt` snapshot at `docs/vendor/openclaw-llms-full.txt` (source: https://docs.openclaw.ai/llms-full.txt).

## When to Load This Skill

Use this skill when working in any of these areas:

- `nemoclaw-blueprint/` — YAML blueprint, `min_openclaw_version`, network policies, policy presets
- `nemoclaw/src/blueprint/` — runner, snapshot, SSRF validation
- `nemoclaw/src/commands/` — slash commands that wrap OpenClaw behavior
- `src/lib/inference*.ts` and `src/lib/preflight*.ts` — inference routing that must match OpenClaw's model provider expectations
- Adding or updating a channel policy preset in `nemoclaw-blueprint/policies/presets/`

Skip this skill for pure NemoClaw CLI or onboarding work that does not touch OpenClaw internals — use `nemoclaw-user-overview` or `nemoclaw-user-reference` instead.

## Topic Index

Each reference file is a focused summary of one OpenClaw subsystem, with line-number pointers back to `docs/vendor/openclaw-llms-full.txt` for full detail.

| Reference | Covers | Use when |
|-----------|--------|----------|
| [agent-runtime.md](references/agent-runtime.md) | Agent runtime, agent loop, workspace, context, session management, system prompt | Touching runner code, session state, or context plumbing |
| [gateway.md](references/gateway.md) | Gateway architecture, WebSocket protocol, lock, heartbeat, discovery, runbook, bridge protocol | Debugging gateway orchestration or changing transport semantics |
| [sandboxing.md](references/sandboxing.md) | Sandbox model, tool policy vs elevated, network model, OpenShell integration | Changing sandbox capabilities, egress policy, or OpenShell wiring |
| [configuration.md](references/configuration.md) | Configuration layers, examples, full configuration reference | Adding or mapping a config key that NemoClaw must set or preserve |
| [plugins.md](references/plugins.md) | Plugin internals, manifest, SDK, entry points, channel/provider plugins | Changing the `nemoclaw` plugin or `openclaw.plugin.json` |
| [delegate-subagents.md](references/delegate-subagents.md) | Delegate architecture, sub-agents, multi-agent routing | Designing multi-agent flows or policy around delegated auth |
| [channels.md](references/channels.md) | Channels & routing, pairing, Slack, Discord, Telegram, Signal | Building or updating a channel policy preset |
| [model-providers.md](references/model-providers.md) | Provider directory, failover, authentication, credential semantics | Adding or routing a new inference provider |
| [secrets-security.md](references/secrets-security.md) | Secrets management, apply-plan contract, security posture | Touching credential handling or the security threat model |
| [skills-hooks-tasks.md](references/skills-hooks-tasks.md) | OpenClaw skills, hooks, standing orders, task flow, background tasks | Surfacing OpenClaw automation primitives in NemoClaw |

## Provenance

- Source: https://docs.openclaw.ai/llms-full.txt (fetched and committed to this repo).
- Snapshot: `docs/vendor/openclaw-llms-full.txt` (97,009 lines, ~4 MB).
- Each reference file cites specific line ranges in the snapshot so you can read the full upstream text when the summary is insufficient.
- These references are **distillations**, not verbatim copies. When the upstream changes, re-fetch and regenerate or patch the relevant sections.

## Conventions Used in References

- **Line pointers** look like `(openclaw-llms-full.txt:13058-13181)` and refer to ranges in the snapshot.
- **Blueprint implication** callouts highlight where a concept affects the NemoClaw blueprint YAML, policy presets, or plugin code.
- Upstream code examples are paraphrased or omitted; read the snapshot for exact syntax.
