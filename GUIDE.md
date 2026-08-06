# Pi 项目 Agent 与 Tool/LLM 交互源码阅读指南

想要了解本项目中 Agent 是如何与 Tool 以及大模型 (LLM) 交互的，建议按照以下核心模块和文件顺序进行阅读。

## 1. 核心架构概览

项目采用了分层架构：
- **`@mariozechner/pi-agent-core` (packages/agent)**: 负责 Agent 的生命周期、状态管理、工具调用逻辑以及对话循环。
- **`@mariozechner/pi-ai` (packages/ai)**: 统一的 LLM 抽象层，屏蔽了不同供应商（Anthropic, OpenAI, Google 等）的 API 差异。

---

## 2. Agent 对话循环 (Agent Loop)

这是 Agent 的“大脑”所在，负责协调 LLM 响应和工具执行。

### 关键文件：`packages/agent/src/agent-loop.ts`
- **`agentLoop` / `agentLoopContinue`**: 启动或继续 Agent 循环的入口。
- **`runLoop`**: 核心循环函数。它包含一个 `while(true)` 循环，不断执行以下步骤：
    1. **调用 LLM**: 通过 `streamAssistantResponse` 获取模型输出。
    2. **处理工具调用**: 如果模型返回了 `toolCall`，则调用 `executeToolCalls`。
    3. **状态更新**: 将工具执行结果作为 `toolResult` 存回上下文。
    4. **中断检查 (Steering)**: 检查是否有用户输入的“转向”消息（例如中途打断）。

---

## 3. 与大模型 (LLM) 的交互

交互是通过 `pi-ai` 包完成的，它提供了一个统一的流式接口。

### 关键文件：`packages/ai/src/stream.ts`
- **`streamSimple`**: Agent 调用的统一接口。它会根据 `model.api` 自动路由到对应的 Provider。

### Provider 实现：`packages/ai/src/providers/`
- 例如 **`anthropic.ts`**:
    - 负责将标准的 `Message` 和 `Tool` 转换为 Anthropic API 要求的格式。
    - 处理流式响应，将其解析为 `text_delta`、`toolcall_start` 等标准事件。

---

## 4. Tool 的执行机制

### 关键文件：`packages/agent/src/types.ts`
- **`AgentTool` 接口**: 扩展了基础的 `Tool` 接口，增加了核心的 `execute` 方法。

### 关键文件：`packages/agent/src/agent-loop.ts` 中的 `executeToolCalls`
- **`executeToolCalls`**:
    1. 遍历模型返回的所有 `toolCall`。
    2. 根据名称在 `tools` 列表中找到对应的 `AgentTool`。
    3. 校验参数（使用 TypeBox）。
    4. 执行 `tool.execute(...)` 并获取结果。
    5. 发送执行状态事件（`tool_execution_start`/`end`）。

---

## 5. 高层封装：Agent 类

为了方便使用，项目提供了一个 `Agent` 类来封装上述循环。

### 关键文件：`packages/agent/src/agent.ts`
- **`Agent` 类**:
    - 管理 `AgentState`（消息历史、当前模型、工具列表等）。
    - 提供 `prompt()` 方法发送新消息。
    - 处理消息队列（`steeringQueue` 和 `followUpQueue`），支持在 Agent 运行过程中注入新消息。
    - 通过订阅 `agentLoop` 的事件流来实时更新内部状态。

---

## 源码阅读顺序建议

1. **`packages/agent/src/types.ts`**: 先看 `AgentMessage` 和 `AgentTool` 的定义，理解基本数据结构。
2. **`packages/agent/src/agent-loop.ts`**: 重点研读 `runLoop`，这是理解整个交互逻辑的关键。
3. **`packages/ai/src/types.ts`**: 了解 `pi-ai` 定义的标准消息格式。
4. **`packages/ai/src/providers/anthropic.ts`**: 作为一个例子，看 Provider 是如何桥接具体 API 的。
5. **`packages/coding-agent/src/core/tools/`**: 查看具体的工具（如 `bash.ts` 或 `read.ts`）是如何实现 `AgentTool` 接口的。
