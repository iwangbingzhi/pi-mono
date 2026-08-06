# Agent Interaction Guide

This document explains how the Agent in this project interacts with various tools and Large Language Models (LLMs).

## 1. Core Execution Flow: CLI to Interaction Loop
The agent's execution starts in the `coding-agent` package and moves into the core `agent` package.
- **Entry Point**: `packages/coding-agent/src/cli.ts` -> `main.ts`. This parses arguments and creates an `AgentSession`.
- **Session Management**: `packages/coding-agent/src/core/agent-session.ts`. It wraps the `Agent` class and handles session state, auto-compaction, and retry logic.
- **Core Loop**: **`packages/agent/src/agent-loop.ts`**. The `runLoop` function manages the cycle of model generation, tool calling, and result processing.

## 2. Interaction with LLMs
The `ai` package provides a unified abstraction layer for multiple providers (OpenAI, Anthropic, Google, etc.).
- **Abstraction Layer**: `packages/ai/src/stream.ts`. The agent calls `streamSimple` or `complete` to get responses.
- **Message Transformation**: In `agent-loop.ts`, `streamAssistantResponse` converts `AgentMessage[]` to LLM-compatible formats using `config.convertToLlm`.
- **Provider Implementations**: Found in `packages/ai/src/providers/` (e.g., `anthropic.ts`).

## 3. Interaction with Tools
Tools are defined as plugins and executed dynamically.
- **Execution Logic**: In `agent-loop.ts`, `executeToolCalls` does the following:
    1. Matches the LLM's `toolCall` to a registered `AgentTool`.
    2. Validates arguments at runtime using **TypeBox** schemas (`validateToolArguments`).
    3. Invokes the tool's `execute` method.
- **Tool Implementations**: Located in `packages/coding-agent/src/core/tools/` (e.g., `ls.ts`, `bash.ts`).
- **Registry**: All tools are aggregated in `packages/coding-agent/src/core/tools/index.ts`.

## 4. Key Data Structures
- **`packages/agent/src/types.ts`**: Defines `AgentMessage`, `AgentTool`, and `AgentLoopConfig`.

## 5. Recommended Reading Order
1. `packages/agent/test/agent-loop.test.ts`: Shows how to mock LLM responses and trigger tools.
2. `packages/agent/src/agent-loop.ts`: The heart of the interaction logic.
3. `packages/coding-agent/src/core/tools/ls.ts`: Example of a simple tool with schema validation.
4. `packages/ai/src/providers/anthropic.ts`: Example of how messages are adapted for a specific API.
