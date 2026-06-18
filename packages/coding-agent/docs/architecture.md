# Architecture

This document describes the internal architecture of Pi and how the Agent interacts with Large Language Models (LLMs) and tools.

## System Overview

Pi is built as a layered monorepo with a clear separation of concerns:

| Package | Role | Key Components |
|---------|------|----------------|
| **`@earendil-works/pi-ai`** | **LLM Abstraction** | `ApiRegistry`, `Provider` implementations, `streamSimple` |
| **`@earendil-works/pi-agent-core`** | **Agent Runtime** | `agentLoop`, `runLoop`, `Agent` class |
| **`@earendil-works/pi-coding-agent`** | **Application Layer** | `AgentSession`, `ExtensionRunner`, Built-in Tools |
| **`@earendil-works/pi-tui`** | **Terminal UI** | Differential rendering engine, TUI components |

## The Core Loop

The "brain" of the agent is the `runLoop` function in `packages/agent/src/agent-loop.ts`. It orchestrates a cycle of perception, thought, and action:

1.  **Context Preparation**: `AgentMessage[]` are transformed and converted to LLM-compatible `Message[]`.
2.  **LLM Call**: The loop calls `pi-ai` to stream a response from the model.
3.  **Event Emission**: Lifecycle events (`turn_start`, `message_delta`, etc.) are emitted for the UI and extensions.
4.  **Tool Execution**: If the model requests tool calls, the loop executes them and appends results to the context.
5.  **Iteration**: The loop continues until the model stops or a termination condition is met.

## Life of a Tool Call

When an LLM decides to use a tool, the following sequence occurs:

1.  **Detection**: `pi-ai` provider (e.g., Anthropic) parses the `tool_use` block from the stream.
2.  **Preparation**:
    *   Agent loop identifies the tool in the `AgentContext`.
    *   Arguments are validated against the tool's **TypeBox** schema.
    *   `beforeToolCall` hooks run (allowing extensions to block or modify calls).
3.  **Execution**:
    *   The tool's `execute` method is called.
    *   Tools like `bash` or `read` perform their side effects (filesystem access, process execution).
    *   Optional: Tools can stream progress updates via `onUpdate`.
4.  **Finalization**:
    *   `afterToolCall` hooks run (allowing result modification).
    *   A `toolResult` message is created and appended to the history.
5.  **Feedback**: The agent loop automatically triggers a follow-up LLM call with the tool results, unless `terminate: true` was returned.

## Tool Execution Modes

The Agent supports two modes for executing multiple tool calls in a single turn:

*   **Parallel (Default)**: Preflights all calls, then executes non-blocking tools concurrently.
*   **Sequential**: Executes calls one-by-one. Forced if any tool in the batch is marked as `executionMode: "sequential"` (like `edit` which must be serial to avoid file conflicts).

## Extension System

The `ExtensionRunner` in `pi-coding-agent` acts as an interceptor. It wraps every tool call and message event, allowing extensions to:
-   Add custom tools.
-   Modify tool arguments before execution.
-   Post-process tool outputs (e.g., for pretty-printing or security filtering).
-   Inject system prompt fragments.

## Navigation & Compaction

The `AgentSession` manages higher-level state like:
-   **Session Tree**: Branching and navigating through conversation history.
-   **Compaction**: Summarizing old context when the token limit is reached, handled via `compact()` in `pi-coding-agent`.
-   **Auto-Retry**: Exponential backoff for transient LLM errors.

## Key Source Files to Explore

-   **Orchestration**: `packages/agent/src/agent-loop.ts`
-   **LLM API**: `packages/ai/src/stream.ts`
-   **Anthropic Adapter**: `packages/ai/src/providers/anthropic.ts`
-   **Tool Base**: `packages/coding-agent/src/core/tools/index.ts`
-   **Session Logic**: `packages/coding-agent/src/core/agent-session.ts`
