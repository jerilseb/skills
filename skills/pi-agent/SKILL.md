---
name: pi-agent
description: Reference manual for the Pi agent framework. Use when working with @earendil-works/pi-ai, @earendil-works/pi-agent-core, @earendil-works/pi-coding-agent, Pi agent tools, agent loops, sessions, compaction, extensions, or OAuth/model APIs.
---

# Pi Agent Framework

Use this skill when building, reviewing, or modifying code that uses the Pi agent framework.

The Pi ecosystem has three relevant packages:

1. **`@earendil-works/pi-ai`**: stateless LLM provider abstraction for models, tools, streaming/completion, utilities, and OAuth.
2. **`@earendil-works/pi-agent-core`**: stateful agent loop, agent tools, queues, and agent events.
3. **`@earendil-works/pi-coding-agent`**: higher-level coding-agent SDK with sessions, built-in coding tools, extensions, resources, skills, settings, and compaction.

## Classify The Task First

Before editing, decide which API surface you are working with:

- Provider/model discovery, LLM calls, streaming events, tool schema validation, utilities, or OAuth
- Agent tools, the `Agent` class, event handling, steering/follow-up queues, or low-level loops
- Built-in coding tools, sessions, extensions, prompt templates, skills, tree navigation, settings, or compaction

Read only the matching reference files before making changes.

## Core Working Rules

- Use `StringEnum` instead of `Type.Enum` for tool enums so schemas remain compatible with Gemini.
- Validate tool calls and arguments with the Pi validation helpers rather than trusting raw LLM JSON.
- When implementing `AgentTool.execute`, throw `Error` on failure; do not return error strings as normal content.
- Return `terminate: true` from tools only when the agent should not automatically continue; in a tool batch, every finalized result must request termination for early stop.
- Preserve tool-call/result pairing when pruning, transforming, or compacting context.
- Treat context overflow as a first-class agent-loop condition and compact rather than blindly retrying.
- Use `@earendil-works/pi-coding-agent` for sessions, built-in coding tools, extensions, and compaction; these are not exported by `pi-agent-core` in version 0.74.0.

## Load The Matching Reference

- Read `references/pi-ai.md` for `@earendil-works/pi-ai`: models/providers, API provider registry, env API keys, tool schemas, streaming/completion, utilities, and OAuth.
- Read `references/agent-core.md` for `@earendil-works/pi-agent-core`: `AgentTool`, `Agent`, agent events, steering/follow-up behavior, and low-level loops.
- Read `references/coding-agent-infrastructure.md` for `@earendil-works/pi-coding-agent`: sessions, SDK entry points, built-in coding tools, extensions, skills/resources, tree navigation, and compaction/summarization.
