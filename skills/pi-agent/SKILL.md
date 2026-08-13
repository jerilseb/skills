---
name: pi-agent
description: Reference manual for the Pi agent framework. Use when working with @earendil-works/pi-ai, @earendil-works/pi-agent-core, @earendil-works/pi-coding-agent, Pi agent tools, agent loops, sessions, compaction, extensions, telemetry, or model/auth APIs.
---

# Pi Agent Framework

Use this skill when building, reviewing, or modifying code that uses the Pi agent framework. References were verified against **`0.84.1`** (the version installed in this repo).

The Pi ecosystem has three packages:

1. **`@earendil-works/pi-ai`**: stateless LLM provider abstraction — the `Models` collection, model catalog, auth/login, tools, streaming/completion, utilities.
2. **`@earendil-works/pi-agent-core`**: the agent loop (`Agent`, `AgentTool`, events, queues) **plus a harness layer** (durable sessions, compaction, skills/prompt templates, an execution environment with read/write/edit/bash tools, and telemetry).
3. **`@earendil-works/pi-coding-agent`**: the coding-agent product SDK — `AgentSession`, the full built-in tool set, extensions, settings, resources, and the interactive/RPC/print modes.

## Classify The Task First

Before editing, decide which API surface you are working with:

- Providers/model discovery, LLM calls, streaming events, tool schema validation, auth/login, utilities → `pi-ai`
- Agent tools, the `Agent` class, event handling, steering/follow-up queues, low-level loops → `pi-agent-core` (loop)
- Durable sessions, compaction, skills/prompt templates, sandboxed filesystem/shell tools, telemetry spans → `pi-agent-core` (harness)
- `AgentSession`, extensions, settings, `~/.pi` resources, grep/find/ls tools, TUI/RPC modes → `pi-coding-agent`

Read only the matching reference files before making changes.

## Core Working Rules

- Use `StringEnum` instead of `Type.Enum` for tool enums so schemas remain compatible with Gemini.
- Validate tool calls and arguments with the Pi validation helpers rather than trusting raw LLM JSON.
- When implementing `AgentTool.execute`, throw `Error` on failure; do not return error strings as normal content.
- Return `terminate: true` from tools only when the agent should not automatically continue; in a tool batch, every finalized result must request termination for early stop.
- **`Agent` requires a `streamFn`** — pass `(m, c, o) => models.streamSimple(m, c, o)` from a `pi-ai` `Models` collection (it resolves auth internally, so `getApiKey` is unnecessary), or install a process-wide default with `setDefaultStreamFn()`. The low-level loop functions take `streamFn` as a required positional argument too.
- Build **one `Models` collection per process** and reuse it; never reach for the root-level `complete`/`getModel` globals (compat-only) or a per-call `apiKey`.
- **OAuth is collection-owned**: `models.login(providerId, type, interaction)` / `logout` / `checkAuth`. The `@earendil-works/pi-ai/oauth` subpath is **type-only** — the old `loginAnthropic`/`getOAuthApiKey`/`registerOAuthProvider` value exports are gone.
- For a graceful stop, use the `shouldStopAfterTurn` hook (available on both `AgentOptions` and `AgentLoopConfig`) rather than aborting from a `turn_end` subscriber.
- Preserve tool-call/result pairing when pruning, transforming, or compacting context.
- Treat context overflow as a first-class agent-loop condition and compact rather than blindly retrying.
- Harness-layer APIs return `Result<T, E>` with tagged errors instead of throwing — check `.ok` and use `matchError` / `getOrThrow`.
- `compact`, `Skill`, `PromptTemplate` and friends exist in **both** `pi-agent-core` and `pi-coding-agent` with **different signatures**. Confirm which package a symbol comes from before calling it.

## Load The Matching Reference

- Read `references/pi-ai.md` for `@earendil-works/pi-ai`: the `Models` collection, catalog reads, auth/login/credentials, tool schemas, generation + deferred responses, stream events, and utilities.
- Read `references/agent-core.md` for `@earendil-works/pi-agent-core`'s loop: `AgentTool`, `Agent`, agent events, steering/follow-up behavior, stop hooks, and the low-level loop functions.
- Read `references/agent-harness.md` for `@earendil-works/pi-agent-core`'s harness: `AgentHarness`, durable sessions, compaction, execution environment + harness tools, skills/prompt templates, `Result` errors, and telemetry.
- Read `references/coding-agent-infrastructure.md` for `@earendil-works/pi-coding-agent`: `AgentSession`, SDK entry points, built-in coding tools, extensions, skills/resources, settings, and session management.
