# Pi Coding Agent 优化源码导航

这份文档不是 Pi 的完整原理介绍，而是一个源码索引：当你要优化自己的 Coding Agent 时，先根据问题找到 Pi 对应的实现。

## 先看整体调用链

```text
用户输入
  -> packages/coding-agent/src/core/agent-session.ts
  -> packages/agent/src/agent.ts
  -> packages/agent/src/agent-loop.ts
  -> packages/coding-agent/src/core/model-runtime.ts
  -> packages/ai/src/api/*
```

建议阅读顺序：

1. [coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：应用层 session、工具和 Prompt 组装
2. [agent/src/agent.ts](../packages/agent/src/agent.ts)：有状态 Agent、prompt/continue、队列和生命周期
3. [agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)：LLM -> tool call -> tool result 的通用循环
4. [coding-agent/src/core/model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)：模型和 provider runtime
5. [ai/src/api](../packages/ai/src/api)：不同 provider 的请求、缓存和 usage 适配

## 按优化目标查找

### 我要理解或修改 Agent Loop

先看：

- [packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)
  - `runAgentLoop()`：带新 prompt 开始一轮
  - `runAgentLoopContinue()`：从现有 user/tool result 继续
  - `runLoop()`：处理多轮 assistant、tool call、steering 和 follow-up
  - `streamAssistantResponse()`：把 AgentMessage 转成 LLM 请求
  - `executeToolCalls()`：选择串行或并行执行工具
- [packages/agent/src/agent.ts](../packages/agent/src/agent.ts)
  - `prompt()`、`continue()`
  - `runPromptMessages()`、`runContinuation()`
  - `createLoopConfig()`：把队列、hook、停止条件传给 loop

对应测试：

- [packages/agent/test/agent-loop.test.ts](../packages/agent/test/agent-loop.test.ts)
- [packages/agent/test/agent.test.ts](../packages/agent/test/agent.test.ts)
- [packages/coding-agent/test/suite/agent-session-prompt.test.ts](../packages/coding-agent/test/suite/agent-session-prompt.test.ts)
- [packages/coding-agent/test/suite/agent-session-queue.test.ts](../packages/coding-agent/test/suite/agent-session-queue.test.ts)

借鉴重点：把 Loop 的“继续条件”和“停止条件”写成 runtime 逻辑，不要只依赖模型自行说完成。

### 我要限制长 Loop、重复调用和重试

先看：

- [packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)：tool result 后是否继续下一轮
- [packages/agent/src/agent.ts](../packages/agent/src/agent.ts)：`shouldStopAfterTurn`、`prepareNextTurn`、steering/follow-up 队列
- [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：自动重试、错误恢复、`_handlePostAgentRun()`
- [packages/coding-agent/src/core/output-guard.ts](../packages/coding-agent/src/core/output-guard.ts)：输出和执行保护

需要在自己的项目中额外记录：

```text
maxTurns
maxToolCalls
maxRetriesPerTool
maxSameToolCall
maxSameError
wall time
```

对应回归测试可参考：

- [packages/coding-agent/test/suite/regressions/5998-blocked-tool-terminate.test.ts](../packages/coding-agent/test/suite/regressions/5998-blocked-tool-terminate.test.ts)
- [packages/coding-agent/test/suite/agent-session-retry-events.test.ts](../packages/coding-agent/test/suite/agent-session-retry-events.test.ts)

### 我要设计 System Prompt

先看：

- [packages/coding-agent/src/core/system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)
  - `buildSystemPrompt()`：生成最终 system prompt
- [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)
  - `_rebuildSystemPrompt()`：根据工具和资源重建 prompt
  - `_baseSystemPromptOptions`：Prompt 的输入来源
- [packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)
  - 项目上下文、Skills、Extensions 等资源加载
- [packages/coding-agent/src/core/skills.ts](../packages/coding-agent/src/core/skills.ts)
  - Skills 的发现和校验

对应测试：

- [packages/coding-agent/test/tool-system-prompt-contributions.test.ts](../packages/coding-agent/test/tool-system-prompt-contributions.test.ts)
- [packages/agent/test/harness/system-prompt.test.ts](../packages/agent/test/harness/system-prompt.test.ts)

推荐学习的结构：

```text
稳定核心规则
  -> 工具使用规则
  -> 项目规则
  -> 当前任务状态
```

不要每个 turn 重写全部 Prompt。把稳定规则放在前面，当前工具和任务状态作为低频或末尾的动态部分。

### 我要根据任务动态激活 Tools

先看：

- [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)
  - `getActiveToolNames()`：当前暴露给模型的工具
  - `getAllTools()`：registry 中的全部工具
  - `setActiveToolsByName()`：切换 active tools，并重建 system prompt
  - `_refreshToolRegistry()`：合并内置工具和 Extension 工具
- [packages/coding-agent/src/core/tools/index.ts](../packages/coding-agent/src/core/tools/index.ts)
  - 内置工具 registry 和工具集合
- [packages/coding-agent/src/core/extensions/loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)
  - `registerTool()`、`setActiveTools()` 的 Extension 接口
- [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts)
  - `ToolDefinition`、`registerTool()`、`setActiveTools()` 类型

重要区别：Pi 默认不是每轮由模型加载工具，而是启动时注册工具、选择一组 active tools。模型驱动的按需激活是扩展能力。

可直接参考：

- [examples/extensions/kimi-deferred-tools.ts](../packages/coding-agent/examples/extensions/kimi-deferred-tools.ts)：`tool_search` 激活 `Calculator`
- [examples/extensions/dynamic-tools.ts](../packages/coding-agent/examples/extensions/dynamic-tools.ts)：动态工具示例
- [examples/extensions/plan-mode/index.ts](../packages/coding-agent/examples/extensions/plan-mode/index.ts)：计划模式切换工具集合
- [packages/coding-agent/test/suite/regressions/6162-extension-active-tools-next-turn.test.ts](../packages/coding-agent/test/suite/regressions/6162-extension-active-tools-next-turn.test.ts)：切换在下一轮生效

建议：小工具集合保持稳定；只有大型、低频、专业工具才按需激活。

### 我要优化工具设计和 Tool Result

工具总入口：

- [packages/coding-agent/src/core/tools/index.ts](../packages/coding-agent/src/core/tools/index.ts)

内置工具：

| 目标 | 文件 | 作用 |
| --- | --- | --- |
| 读取文件 | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | 已知路径和行范围读取 |
| 搜索内容 | [grep.ts](../packages/coding-agent/src/core/tools/grep.ts) | 在不知道位置时定位匹配 |
| 查找文件 | [find.ts](../packages/coding-agent/src/core/tools/find.ts) | 按 glob 查找路径 |
| 列目录 | [ls.ts](../packages/coding-agent/src/core/tools/ls.ts) | 了解目录结构 |
| 局部修改 | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | `oldText -> newText` 精确替换 |
| 整体写入 | [write.ts](../packages/coding-agent/src/core/tools/write.ts) | 创建或完整覆盖文件 |
| 执行命令 | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | 测试、构建和 shell 操作 |

结果限制：

- [tools/truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)：行数和字节上限
- [tools/output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts)：流式输出累计和截断
- [tools/file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)：文件修改串行化
- [tools/edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)：edit 的匹配、重叠检查和 diff

重点借鉴：

```text
有限结果 + 截断标记 + continuation 方式
局部 edit 优先于完整 write
只读工具可并行，写入工具串行
```

对应测试：

- [packages/agent/test/harness/truncate.test.ts](../packages/agent/test/harness/truncate.test.ts)
- [packages/coding-agent/test/suite/regressions/5303-bash-output-truncation.test.ts](../packages/coding-agent/test/suite/regressions/5303-bash-output-truncation.test.ts)
- [packages/coding-agent/test/file-mutation-queue.test.ts](../packages/coding-agent/test/file-mutation-queue.test.ts)

### 我要优化 Context、Compaction 和长会话

先看：

- [packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)
  - token 估算
  - `shouldCompact()`
  - compaction summary 生成
- [packages/coding-agent/src/core/compaction/branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts)
  - session 分支摘要
- [packages/coding-agent/src/core/compaction/utils.ts](../packages/coding-agent/src/core/compaction/utils.ts)
  - 上下文保留和 entry 处理
- [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)
  - 自动触发、重试和 compaction 后恢复
- [packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)
  - JSONL session 和消息持久化

对应测试：

- [packages/coding-agent/test/agent-session-compaction.test.ts](../packages/coding-agent/test/agent-session-compaction.test.ts)
- [packages/coding-agent/test/suite/agent-session-compaction.test.ts](../packages/coding-agent/test/suite/agent-session-compaction.test.ts)
- [packages/agent/test/harness/compaction.test.ts](../packages/agent/test/harness/compaction.test.ts)

重点记录：

```text
context tokens
context window
reserve tokens
compaction boundary
compaction retry
```

### 我要优化 Prompt Cache 和 Token 成本

先看 Pi 的抽象：

- [packages/ai/src/types.ts](../packages/ai/src/types.ts)
  - `sessionId`
  - `cacheRetention`
  - `cacheRead` / `cacheWrite`
  - provider cache capability
- [packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts)
  - `sessionManager.getSessionId()` 传入 Agent/runtime
- [packages/agent/src/agent.ts](../packages/agent/src/agent.ts)
  - `sessionId` 保存在 Agent 并传入 loop config
- [packages/ai/src/api/openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)
  - OpenAI usage 中 cached tokens 的解析
- [packages/ai/src/api/openai-prompt-cache.ts](../packages/ai/src/api/openai-prompt-cache.ts)
  - OpenAI prompt cache key 处理
- [packages/ai/src/api/anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts)
  - Anthropic cache control 适配
- [packages/ai/src/api/mistral-conversations.ts](../packages/ai/src/api/mistral-conversations.ts)
  - session affinity 和 prompt cache key

判断动态 Prompt/Tools 是否划算时，不要只看 cache 命中率，记录：

```text
system prompt hash
active tool names
tool schema hash
input tokens
cache read tokens
cache write tokens
output tokens
tool turns
total task cost
```

建议：

- 小工具集合固定，避免每轮切换
- 稳定规则放在 Prompt 前部
- 动态任务状态放在后部
- 只低频激活大型专业工具
- 用真实 usage 比较“全量工具、任务开始选择、按需加载”三种策略

### 我要增加或覆盖自定义工具

先看：

- [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts)：工具接口
- [packages/coding-agent/src/core/extensions/loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)：注册流程
- [packages/coding-agent/src/core/extensions/wrapper.ts](../packages/coding-agent/src/core/extensions/wrapper.ts)：Extension tool 包装
- [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：工具 registry、hook 和 Prompt guideline

示例：

- [examples/extensions/hello.ts](../packages/coding-agent/examples/extensions/hello.ts)：最小自定义工具
- [examples/extensions/tool-override.ts](../packages/coding-agent/examples/extensions/tool-override.ts)：覆盖内置工具
- [examples/extensions/subagent/index.ts](../packages/coding-agent/examples/extensions/subagent/index.ts)：通过工具启动子 agent
- [examples/extensions/truncated-tool.ts](../packages/coding-agent/examples/extensions/truncated-tool.ts)：自定义截断结果

## 推荐的优化顺序

如果你的项目目前 Token 高、Loop 长，按以下顺序阅读和实现：

1. [agent-loop.ts](../packages/agent/src/agent-loop.ts)：明确每个继续和停止分支
2. [core/tools/truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)：给所有 Tool Result 加预算和 continuation
3. [core/tools/edit.ts](../packages/coding-agent/src/core/tools/edit.ts)：实现精确局部修改
4. [core/system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)：拆分稳定规则和动态规则
5. [core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)：加入上下文估算和压缩
6. [core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：接入重试、工具切换和 session 生命周期
7. [ai/src/api](../packages/ai/src/api)：最后再根据 provider usage 优化 Prompt Cache

## 最小运行指标

每个 Agent Turn 至少记录：

```text
turn number
input tokens
output tokens
system prompt tokens
tool schema tokens
tool name
tool args size
tool result size
context size
cache read tokens
cache write tokens
retry count
```

没有这些数据，不要先猜是 Prompt、Tools、Cache 还是 Loop 导致成本高。

## 一句话总结

```text
先从 agent-loop 看控制流；
从 tools 看上下文体积和文件修改；
从 system-prompt 看模型行为；
从 compaction 看长会话；
从 ai/src/api 看 provider cache 和真实成本。
```
