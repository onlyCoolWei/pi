# 基于 PI 的 Agent Loop Token 成本治理方案

状态：`draft`

调研快照：2026-08-19，PI `main` 分支

## 1. 文档目标

本文基于 [Agent Loop Token 成本优化改造文档](../../wegic-agents/docs/plans/agent-loop-token-cost-optimization.md) 中识别的问题，调研 PI 项目是否已有可复用方案，并给出 Wegic Agents 的落地设计。

本文所在项目即 [`earendil-works/pi`](../README.md)。除 Wegic 的基线材料外，源码和项目文档均使用仓库内相对路径；Issue/PR 使用官方外部链接。

本方案的原则是借鉴 PI 已验证的机制，不引入第二套 Agent Loop，不替换 Wegic 当前的 OpenAI Agents SDK Runner。

## 2. 结论

PI 对以下问题有较成熟的解决方案：

- 在工具源头限制输出体积，并给模型明确的继续读取方式。
- 根据内容类型选择保留头部或尾部，完整输出转存文件。
- 在唯一的模型请求边界转换上下文，允许发送前清理而不破坏持久化历史。
- 按真实 usage 和最近消息估算上下文 token。
- 按 token 保留最近上下文，而不是只按消息条数保留。
- 支持压缩超长单轮的前缀，不要求完整保护整个当前轮次。
- 压缩摘要显式保存已读文件和已修改文件。
- 同一 assistant 响应中的独立工具默认并行执行，同时保持结果顺序稳定。
- 提供 `shouldStopAfterTurn`，可以在完成当前工具批次后安全停止下一次模型调用。
- 根据实际启用的工具构建 System Prompt，避免暴露无关工具说明。

PI 没有完整解决以下问题：

- 没有内置的单轮累计 input token、模型调用次数和只读工具次数预算。
- 没有内置的相同文件重复区间或高重叠区间检测。
- 没有内置的单轮图片数量、图片 URL/hash 去重和图片历史保留预算。
- 没有内置 `inspect -> execute -> verify` 阶段机。
- PI Coding Agent 当前仍未把自动压缩可靠地接入长工具循环的每个 turn 边界。

因此推荐采用“PI 基础机制 + Wegic 预算治理”的组合，而不是直接移植 PI Coding Agent 的完整实现。

## 3. 问题映射

| Wegic 问题                      | PI 现有机制                                          | 采用结论                                          |
| ------------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| 单次工具结果过大                | `truncateHead`、`truncateTail`，默认 2,000 行或 50KB | 采用算法，Wegic 使用更小的 token 预算             |
| `read_file` 截断后反复猜 offset | Read Tool 返回实际行号和精确 `nextOffset`            | 直接借鉴                                          |
| Bash/日志输出挤占上下文         | 只保留尾部，完整输出写临时文件                       | 直接借鉴，完整输出写项目级 artifact               |
| 旧工具结果反复重放              | `transformContext` 在模型请求前转换上下文            | 借用扩展点思路，接入 Wegic `callModelInputFilter` |
| 当前轮过大但不能压缩            | Split-turn compaction 可总结单轮前缀并保留后缀       | 借鉴算法，适配 SDK Tool Call/Result 协议          |
| 压缩后忘记文件状态              | 摘要累计保存 `readFiles`、`modifiedFiles`            | 直接借鉴并扩展验证状态                            |
| 工具串行导致模型调用多          | 同一 assistant 响应的工具批次默认并行                | 保留 SDK 并行能力，加强 Prompt 和观测             |
| 需要安全停止下一次模型调用      | `shouldStopAfterTurn`                                | 借鉴为 Round Budget 的硬停止边界                  |
| 工具 schema 固定开销大          | PI 按 selected tools 构建工具列表和 Prompt           | 借鉴为 Wegic Tool Profile                         |
| 图片 token 过大                 | 图片读取自动 resize、Tool Result 图片 normalize      | 只借鉴预处理；数量预算和去重需自研                |
| 同一文件重复读取                | 无内置方案                                           | Wegic 自研 Read Ledger                            |
| 单轮成本硬预算                  | 无内置方案                                           | Wegic 自研 Round Budget Manager                   |
| 探索阶段不收敛                  | 无内置阶段机                                         | Wegic 自研轻量阶段状态                            |

## 4. PI 可直接借鉴的机制

### 4.1 工具输出在源头截断【已实现】

PI 把工具输出截断放在 Tool 实现中，而不是等到上下文接近上限后才处理。公共实现同时限制行数和 UTF-8 字节数，默认上限为 2,000 行或 50KB，先达到哪个上限就按哪个截断。

代码位置：

- [`packages/coding-agent/src/core/tools/truncate.ts`](../packages/coding-agent/src/core/tools/truncate.ts#L1-L42)：公共上限和 `TruncationResult`。
- [`truncateHead()`](../packages/coding-agent/src/core/tools/truncate.ts#L67-L150)：保留头部，适合文件、搜索和目录结果。
- [`truncateTail()`](../packages/coding-agent/src/core/tools/truncate.ts#L152-L225)：保留尾部，适合日志和命令错误。
- [`truncateLine()`](../packages/coding-agent/src/core/tools/truncate.ts#L246-L258)：限制单个 grep 匹配行。

这比统一截断更合理，因为不同工具的高价值信息位置不同：文件和搜索结果通常需要开头，构建错误和日志通常需要结尾。

Wegic 不应直接使用 PI 的 50KB 默认值。对于连续 Tool Loop，单个结果 50KB 仍然偏大，建议以 token 为主要口径：

| 工具类型                   | 建议默认上限 | 保留策略           |
| -------------------------- | -----------: | ------------------ |
| `read_file`                | 2,000 tokens | head               |
| `grep/glob/list_directory` | 1,500 tokens | head               |
| `exec_command/read_log`    | 2,500 tokens | tail               |
| 写入型工具结果             |   500 tokens | 结构化摘要         |
| 错误结果                   | 2,000 tokens | tail，保护最近一次 |

### 4.2 Read Tool 返回可执行的续读位置【已实现】

PI 的 Read Tool 在截断时计算实际输出的结束行，并在发给模型的文本提示中返回准确的 `nextOffset`：

- [`read.ts`](../packages/coding-agent/src/core/tools/read.ts#L286-L317)：先应用用户的 `limit`，再执行统一截断，并基于实际输出行数生成 `Use offset=<nextOffset> to continue`。

PI 的核心价值不只是文本提示，而是 `nextOffset` 来源于真实截断结果，模型不需要猜测下一段起点。PI 内部的 `details.truncation` 用于 UI 和程序逻辑，不会把整套截断元数据重复发给模型。

Wegic 应将模型可见的返回值结构化为最小协议：

```typescript
interface ReadFileResult {
  range: { start: number; end: number } | null;
  totalLines: number;
  nextOffset?: number;
  content: string;
}
```

请求路径和范围已存在于 Tool Call，`truncated` 可由 `nextOffset` 是否存在表达，截断原因不改变模型的续读动作。`filePath + fileVersion + returnedRange` 保留在工具内部的 Read Ledger 中，用于重复区间检测，不重复占用模型上下文。

### 4.3 命令输出只保留尾部，完整结果旁路存储

PI 的 `OutputAccumulator` 在流式接收命令输出时只维护有界尾部；输出超过限制后，将完整内容写入临时文件，发给模型的结果只包含尾部和完整文件位置。

代码位置：

- [`output-accumulator.ts`](../packages/coding-agent/src/core/tools/output-accumulator.ts#L28-L62)：有界内存、尾部保留和临时文件设计。
- [`OutputAccumulator.snapshot()`](../packages/coding-agent/src/core/tools/output-accumulator.ts#L91-L118)：使用 `truncateTail` 并返回 `fullOutputPath`。
- [`bash.ts`](../packages/coding-agent/src/core/tools/bash.ts)：Bash Tool 对 accumulator 的接入。

Wegic 不应把完整输出写入宿主机 `/tmp` 后直接暴露给模型。应写入当前项目沙箱中的受控 artifact，并返回 artifact ID 或沙箱相对路径：

```text
[Command output truncated]
showing=last 2,500 tokens
full_output_artifact=agent-output/01J...log
```

只有模型明确需要时才读取该 artifact 的指定范围。

### 4.4 唯一的模型请求上下文转换边界

PI 保留完整的 `AgentMessage[]` 作为应用状态，只在模型调用前执行：

```text
AgentMessage[]
  -> transformContext
  -> convertToLlm
  -> provider Context
```

代码位置：

- [`agent-loop.ts: buildProviderContext()`](../packages/agent/src/agent-loop.ts#L277-L298)。
- [`agent.ts`](../packages/agent/src/agent.ts#L96-L122)：`transformContext` 作为 Agent 配置项暴露。

这个边界适合执行确定性的上下文清理：发送给模型的是紧凑副本，持久化历史仍保持完整，用户回放和审计不受影响。

Wegic 已有等价入口 `AgentStreamHandler.callModelInputFilter`。推荐顺序：

```text
sanitize tool protocol
  -> compact old tool results in current round
  -> clean/deduplicate images
  -> enforce round budget
  -> apply cache markers
  -> call model
```

不能在 Cache 标记之后清理，否则会导致缓存点落在随后被替换的消息上。

### 4.5 基于 usage 的上下文计算

PI 计算上下文时优先使用 provider usage，并将非缓存输入、输出、cache read 和 cache write 都计入真实上下文；最后一个 usage 之后新增的 Tool Result 再通过估算补齐。

代码位置：

- [`compaction.ts: calculateContextTokens()`](../packages/coding-agent/src/core/compaction/compaction.ts)：计算完整 context token。
- [`compaction.ts: estimateContextTokens()`](../packages/coding-agent/src/core/compaction/compaction.ts)：最后一次真实 usage 加 trailing message 估算。
- [`compaction.ts: shouldCompact()`](../packages/coding-agent/src/core/compaction/compaction.ts)：使用 `contextWindow - reserveTokens` 作为触发条件。

PI 默认配置是：

```json
{
  "enabled": true,
  "reserveTokens": 16384,
  "keepRecentTokens": 20000
}
```

来源：[`packages/coding-agent/docs/compaction.md`](../packages/coding-agent/docs/compaction.md#when-it-triggers)。

Wegic 应借鉴“绝对 reserve token”而不是只使用窗口百分比。200K 上下文的 96% 才执行压缩过于接近上限，也不能表达需要为输出、工具 schema 和图片预留多少空间。

推荐触发条件：

```typescript
shouldCompact = estimatedNextRequestTokens > contextWindow - reserveTokens;
```

其中 `estimatedNextRequestTokens` 必须包含刚产生的 Tool Result 和待注入图片，而不是只读取上一次模型响应的 input usage。

### 4.6 Split-turn compaction

PI 压缩不是简单地“保护整个当前轮次”。当一个用户轮次本身超过 `keepRecentTokens` 时，PI 可以在 assistant message 边界切开当前轮次：

```text
user + early assistant/tool work + recent assistant/tool work
       \________ summary ______/ \_______ keep verbatim ______/
```

代码位置：

- [`compaction.ts: findCutPoint()`](../packages/coding-agent/src/core/compaction/compaction.ts)：倒序累计 token，选择合法切点。
- [`compaction.ts: prepareCompaction()`](../packages/coding-agent/src/core/compaction/compaction.ts)：拆分 `messagesToSummarize`、`turnPrefixMessages` 和保留后缀。
- [`TURN_PREFIX_SUMMARIZATION_PROMPT`](../packages/coding-agent/src/core/compaction/compaction.ts)：专门总结超长单轮前缀。
- [PI Compaction 文档的 Split Turns 说明](../packages/coding-agent/docs/compaction.md#split-turns)。

这正好解决 Wegic 当前“压缩只处理当前轮之前的历史，当前轮完整保护”的缺陷。

Wegic 适配时必须遵守 provider 协议：

- 不能在 Tool Call 和对应 Tool Result 之间切开。
- 合法切点只能是 user message、无悬挂调用的 assistant message，或完整 Tool Call/Result 批次之后。
- 最近一次失败验证及相关命令必须一起保留。
- 写入结果与其文件状态摘要不能分离。

### 4.7 结构化摘要和文件状态累积

PI 从 assistant Tool Call 中确定性提取 `read`、`write` 和 `edit` 的文件路径，并在压缩摘要后追加：

```xml
<read-files>
path/to/read-file.ts
</read-files>

<modified-files>
path/to/changed-file.ts
</modified-files>
```

代码位置：

- [`compaction/utils.ts: extractFileOpsFromMessage()`](../packages/coding-agent/src/core/compaction/utils.ts#L26-L56)。
- [`computeFileLists()`](../packages/coding-agent/src/core/compaction/utils.ts#L58-L67)。
- [`formatFileOperations()`](../packages/coding-agent/src/core/compaction/utils.ts#L69-L82)。

生成压缩摘要时，PI 还会把每个 Tool Result 限制为 2,000 字符，避免“为了压缩而再次发送完整历史”：

- [`TOOL_RESULT_MAX_CHARS`](../packages/coding-agent/src/core/compaction/utils.ts#L84-L99)。
- [`serializeConversation()`](../packages/coding-agent/src/core/compaction/utils.ts#L101-L150)。

Wegic 应扩展为确定性 `WorkState`：

```typescript
interface WorkState {
  readFiles: Array<{ path: string; version: string; ranges: LineRange[] }>;
  modifiedFiles: Array<{ path: string; version: string }>;
  completedActions: string[];
  pendingActions: string[];
  lastValidation?: { command: string; passed: boolean; artifactId?: string };
}
```

LLM 只负责总结目标、决策和进度；文件路径、版本、验证状态由代码生成，避免摘要幻觉。

### 4.8 工具并行执行与稳定结果顺序

PI 对同一个 assistant message 中的多个工具调用默认并行执行；如果全局配置为 sequential，或批次中任一 Tool 标记为 sequential，则整个批次顺序执行。并行完成后，Tool Result 仍按模型原始调用顺序进入 transcript。

代码位置：

- [`agent-loop.ts`](../packages/agent/src/agent-loop.ts#L417-L435)：选择 sequential 或 parallel。
- [`executeToolCallsParallel()`](../packages/agent/src/agent-loop.ts#L498-L558)：并行执行并稳定输出顺序。
- [`types.ts`](../packages/agent/src/types.ts#L240-L248)：Tool execution 配置契约。

Wegic 使用的 SDK 已支持单次 assistant response 返回多个 Tool Call，因此不需要移植 PI Loop。需要做的是：

- 在 Prompt 中要求独立读取一次性并行发出。
- 确认 Tool 层允许并行读取但串行写入。
- 记录“每个 assistant response 的 tool call 数”，观察模型是否真正批量调用。
- 对混合读写批次使用 workspace lock 或强制 sequential。

并行主要降低模型调用数和延迟，不会自动减少单次 Tool Result 的 token，仍需配合输出预算。

### 4.9 `shouldStopAfterTurn` 作为安全背压边界

PI 低层 Loop 在 assistant response 和 Tool Result 全部完成后调用 `shouldStopAfterTurn`。返回 true 时，不再轮询 steering/follow-up，也不启动下一次模型调用。

代码位置：

- [`agent-loop.ts`](../packages/agent/src/agent-loop.ts#L247-L257)：安全停止判断。
- [`agent.ts: AgentOptions`](../packages/agent/src/agent.ts#L96-L122)：对上层暴露 hook。
- [`agent.ts: createLoopConfig()`](../packages/agent/src/agent.ts#L433-L470)：把 hook 传入 Loop。
- [PR #7367](https://github.com/earendil-works/pi/pull/7367)：2026-08-03 合入 Agent 层暴露能力。

Wegic 可在现有 `executeSingleStreamRound()` 完成后使用相同思想：

```text
turn 完整结束
  -> 更新 Round Budget
  -> 计算包含 Tool Result 的下一请求体积
  -> warn / compact-and-resume / graceful stop
```

严禁在工具执行中途用硬 abort 实现预算停止，否则可能留下不确定的写入状态。

### 4.10 动态工具集合

PI 的 System Prompt 根据实际 selected tools 生成 Available Tools 和对应 guideline，而不是始终描述完整工具集合。

代码位置：

- [`system-prompt.ts`](../packages/coding-agent/src/core/system-prompt.ts)：`selectedTools`、`visibleTools` 和动态 guideline 构建。

Wegic 推荐定义 Tool Profile：

| Profile    | 工具范围                                       |
| ---------- | ---------------------------------------------- |
| `chat`     | 无文件和执行工具                               |
| `inspect`  | `glob/grep/read/analyze`                       |
| `code`     | inspect + `write/search_replace/exec/read_log` |
| `visual`   | code + 图片查看/生成/编辑                      |
| `research` | 搜索和研究工具                                 |

Agent Factory 按任务类型构造运行时 clone，只暴露本轮需要的工具。这会降低固定 Tool Schema token，也减少模型选择错误工具的概率。

### 4.11 图片预处理

PI Read Tool 会在图片进入模型前执行 resize，AgentSession 的 `afterToolCall` 还会 normalize 扩展 Tool 返回的图片：

- [`read.ts`](../packages/coding-agent/src/core/tools/read.ts#L253-L269)：`processImage()`。
- [`agent-session.ts`](../packages/coding-agent/src/core/agent-session.ts#L473-L501)：`normalizeToolResultImages()`。

这能减少单张超大图片带来的请求体积，但 PI 没有提供单轮图片计数、URL/hash 去重或旧图片淘汰策略。Wegic 仍需实现 `ImageBudgetManager`。

## 5. PI 当前方案的关键缺口

### 5.1 Coding Agent 的自动压缩仍不是可靠的 mid-loop guard

PI 当前 `AgentSession._checkCompaction()` 主要在 Agent Run 完成后或发送新用户 Prompt 前调用：

- [`agent-session.ts: _handlePostAgentRun()`](../packages/coding-agent/src/core/agent-session.ts#L1015-L1042)。
- [`agent-session.ts: prompt()` preflight](../packages/coding-agent/src/core/agent-session.ts#L1133-L1138)。
- [`agent-session.ts: _checkCompaction()`](../packages/coding-agent/src/core/agent-session.ts#L1886-L1978)。

低层 `Agent` 已经暴露 `shouldStopAfterTurn`，但当前 `AgentSession` 没有搜索到将它接入 auto-compaction 的实现。因此长 Tool Loop 中，刚产生的大 Tool Result 仍可能直接进入下一次模型调用。

PI 官方 [Issue #5512](https://github.com/earendil-works/pi/issues/5512) 对该问题给出了相同分析。虽然 Issue 记录中的“Agent 未暴露 hook”部分已由 PR #7367 修复，但 Coding Agent 的 compaction wiring 仍不能作为 Wegic 的现成实现照搬。

Wegic 必须把检查放到每个 Tool Turn 结束、下一次 provider request 之前，并使用“上次真实 usage + 本次新增 Tool Result/图片”的估算。

### 5.2 PI 的默认工具上限仍偏大

50KB 大约可能对应 10K 以上文本 token。它能防止单个工具彻底撑爆上下文，但不足以控制连续 10-20 次工具调用的累计成本。

Wegic 应保留 PI 的通用截断算法，但默认限制降到 1.5K-2.5K tokens，并增加单轮累计工具结果预算。

### 5.3 没有重复读取账本

PI 的 `read` 提供精确 `nextOffset`，但不会阻止模型再次读取完全相同或高度重叠的范围。[`demo.json`](../../wegic-agents/docs/plans/demo.json) 中的读取模式仍可能在 PI 中发生。

Wegic 需要独立维护：

```text
normalizedPath + fileVersion + returnedRange
```

完全重复时直接返回已有读取记录；重叠超过阈值时只允许读取未覆盖区间。

### 5.4 没有成本型硬预算

PI 的 compaction 以防止上下文溢出为主要目标，不限制累计账单成本。即使每次请求都没有接近 context window，20 次不断增长的请求仍会产生高累计 input token。

Wegic 仍需限制模型调用次数、只读次数、累计 input tokens 和首次写入前探索成本。

### 5.5 没有图片数量治理

PI 能缩放图片，但不能阻止一次查看 10 张、单轮累计查看 29 张，也不会按 URL 或内容 hash 去重。

Wegic 需要单独限制：

- 每个 Tool Call 最多 6 张。
- 每轮最多 12 张。
- URL、image ID、内容 hash 三层去重。
- 默认只在模型上下文保留最近一批原图。
- 已形成文字结论的旧图片替换为结构化观察摘要。

## 6. Wegic 目标架构

### 6.1 Tool 执行边界

```text
Tool Call
  -> 参数和权限校验
  -> Round Budget 预检
  -> Read Ledger / Image Ledger 去重
  -> 执行 Tool
  -> PI-style head/tail truncation
  -> 完整大输出写 artifact
  -> 更新 WorkState 和预算
  -> 返回有界 Tool Result
```

### 6.2 下一次模型调用边界

```text
完整持久化历史
  -> sanitize Tool Call/Result
  -> PI-style transformContext
       -> 旧 Tool Result 确定性压缩
       -> 当前轮前缀按需 compact
       -> 图片去重和淘汰
       -> 动态 Tool Profile
  -> Round Budget 最终检查
  -> apply cache markers
  -> provider request
```

### 6.3 Turn 完成边界

```text
assistant + tool batch 完成
  -> 记录真实 usage
  -> 加上 trailing Tool Result/image 估算
  -> 未达预算：继续
  -> 达 warning：注入一次收敛提示后继续
  -> 达 context guard：split-turn compact 后继续
  -> 达 cost hard limit：安全停止并输出 budget_exceeded
```

## 7. 模块设计

### 7.1 `RoundBudgetManager`

```typescript
interface RoundBudgetState {
  modelCalls: number;
  readOnlyToolCalls: number;
  consecutiveReadOnlyCalls: number;
  toolResultTokens: number;
  cumulativeInputTokens: number;
  imageCount: number;
  firstMutationAtCall?: number;
  warningEmitted: boolean;
}
```

建议初始阈值沿用原改造文档：

```text
maxModelCalls=12
maxReadOnlyToolCalls=8
maxConsecutiveReadOnlyCalls=6
maxToolResultTokensPerCall=2000
maxToolResultTokensPerRound=12000
maxRoundInputTokens=60000
maxImagesPerCall=6
maxImagesPerRound=12
warningRatio=0.8
```

这里的 `maxRoundInputTokens` 是累计成本预算，不是当前上下文大小。当前上下文安全另由 `contextWindow - reserveTokens` 控制。

### 7.2 `ToolResultCompactor`

参考 PI `transformContext` 的位置，在 `callModelInputFilter` 中对发送副本处理：

- 最近 2 个 Tool Result 保留完整内容。
- 更早的只读结果替换成路径、范围、关键符号和 artifact 引用。
- 写入结果替换成文件、版本和操作状态，不删除成功事实。
- 最近失败验证保持完整。
- 清理后重新执行 Tool Call/Result 协议校验。

第一阶段使用确定性规则，不为每次模型调用增加额外总结模型。

### 7.3 `ReadLedger`

```typescript
interface ReadLedgerEntry {
  path: string;
  fileVersion: string;
  ranges: Array<{ start: number; end: number }>;
  symbols?: string[];
}
```

规则：

- 完全重复范围不执行 Tool。
- 重叠率超过 60% 时返回未覆盖区间建议。
- 文件写入后更新 `fileVersion`，旧范围失效。
- 全文件内容曾因 token 限制截断时，仍允许从 PI-style `nextOffset` 继续。

### 7.4 `ImageBudgetManager`

```typescript
interface ImageLedgerEntry {
  imageId?: string;
  normalizedUrl?: string;
  contentHash?: string;
  firstSeenCall: number;
  observation?: string;
}
```

处理顺序：去重、resize、单批限制、单轮限制、旧图替换为观察摘要。

### 7.5 `CurrentTurnCompactor`

借鉴 PI split-turn：

1. 从最新消息倒序累计 token，保留最近 12K-20K tokens。
2. 只在完整 Tool Call/Result 批次边界切分。
3. 将早期当前轮转换成 `CurrentTurnSummary + WorkState`。
4. 保留用户原始请求、最近工作、最近错误和待执行动作。
5. 用压缩后的输入继续同一个 Run。

这替代当前“完整保护当前轮”的策略。

### 7.6 `ToolProfileResolver`

根据意图和项目状态选择运行时工具集合。不要只隐藏 Prompt 文本，实际传给 provider 的 Tool Schema 也必须裁剪。

第一阶段可先定义静态 profile，并保留配置回退到完整工具集合。

## 8. 推荐实施顺序

### 阶段一：低风险直接收益

- 将 PI-style head/tail truncation 引入 `read_file`、`grep`、`glob`、`exec_command`、`read_log`。
- Read Tool 返回结构化范围和准确 `nextOffset`。
- 大命令输出写沙箱 artifact。
- `callModelInputFilter` 正式接入现有 `ToolContentCleanerService`。
- 开启模型调用前图片清理和 URL/image ID 去重。

### 阶段二：单轮预算与账本

- 新增 `RoundBudgetManager`。
- 新增 `ReadLedger` 和 `ImageBudgetManager`。
- 在 Tool Turn 完成后、下一次模型请求前执行预算检查。
- warning 阶段只注入一次收敛消息。
- hard limit 使用安全停止，不中断正在执行的 Tool。

### 阶段三：当前轮压缩

- 将压缩切点从“仅历史轮次”扩展为 PI-style split-turn。
- 生成确定性 `WorkState`。
- 压缩 LLM 只接收已截断的 Tool Result。
- 压缩后自动恢复当前任务。

### 阶段四：动态工具和成本调优

- 引入 Tool Profile。
- 按任务类型设置不同预算。
- 用 [`demo.json`](../../wegic-agents/docs/plans/demo.json) 和线上 Trace 回放校准阈值。
- 评估 contact sheet 或低成本视觉分析模型。

## 9. 验证方案

### 9.1 PI 机制移植测试

- UTF-8 多字节内容按完整字符截断，不产生损坏文本。
- Read 截断向模型返回准确 `range` 和 `nextOffset`，内部 Ledger 保留 `fileVersion` 和实际范围。
- Bash 只保留尾部，完整 artifact 可按需读取。
- Tool Call/Result 在上下文清理和 split-turn 后仍严格配对。
- 压缩摘要的文件列表由代码生成且跨多次压缩累积。
- 并行只读工具结果以调用原始顺序进入历史。

### 9.2 Wegic 补充治理测试

- 完全重复读取不实际执行。
- 高重叠读取只返回未覆盖范围建议。
- 文件修改后允许重新读取。
- 图片 URL、ID 和 hash 去重正确。
- warning 只注入一次。
- 达到 hard limit 后不启动下一次模型调用。
- 写入型工具运行中不会被预算 abort。
- 审查类任务不会被强制要求写文件。

### 9.3 回放验收

以 [`demo.json`](../../wegic-agents/docs/plans/demo.json) 为固定基线：

| 指标               | 基线 |         目标 |
| ------------------ | ---: | -----------: |
| 模型调用数         |   21 |        <= 13 |
| 工具调用数         |   19 |        <= 12 |
| 同文件高重叠读取   |   7+ | 0 次实际执行 |
| 图片数             |   29 |        <= 12 |
| 累计 input tokens  | 100% |       <= 50% |
| 首次写入前只读调用 |   19 |         <= 8 |
| Tool 协议错误      |    0 |            0 |
| 任务完成率         | 基线 |       不下降 |

## 10. 风险

### 10.1 PI 当前 main 仍在快速演进

PI 在 2026-08 刚合入 Agent 层 `shouldStopAfterTurn`，Coding Agent 的 mid-loop compaction 仍可能继续变化。实现时应复制设计思想和测试，不依赖 PI 内部未稳定 API。

### 10.2 压缩和 Prompt Cache 冲突

每次动态改写早期历史可能降低缓存命中。应保留稳定前缀，只压缩缓存点之后的 Tool Result，并同时观察 `cached_input_tokens` 和实际成本。

### 10.3 Artifact 生命周期

完整命令输出不能无限保留。需要绑定项目、session 和 message，设置过期时间，并在项目删除时联动清理。

### 10.4 过度预算导致完成率下降

先以 observe-only 和 warn-only 灰度，收集不同任务类型的 P50/P90，再开启 hard limit。复杂迁移和批量图片任务应使用单独 profile。

## 11. PI 代码位置索引

| 能力                       | 文件 / 符号                                                                                                             |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Agent Loop                 | [`packages/agent/src/agent-loop.ts: runLoop()`](../packages/agent/src/agent-loop.ts)                                    |
| 模型调用前转换             | [`buildProviderContext()`](../packages/agent/src/agent-loop.ts#L277-L298)                                               |
| 安全停止                   | [`shouldStopAfterTurn`](../packages/agent/src/agent-loop.ts#L247-L257)                                                  |
| 并行工具执行               | [`executeToolCallsParallel()`](../packages/agent/src/agent-loop.ts#L498-L558)                                           |
| Agent hook 暴露            | [`packages/agent/src/agent.ts`](../packages/agent/src/agent.ts#L96-L122)                                                |
| 通用 Tool 截断             | [`packages/coding-agent/src/core/tools/truncate.ts`](../packages/coding-agent/src/core/tools/truncate.ts)               |
| Read 分页                  | [`packages/coding-agent/src/core/tools/read.ts`](../packages/coding-agent/src/core/tools/read.ts#L286-L317)             |
| Bash 输出旁路              | [`output-accumulator.ts`](../packages/coding-agent/src/core/tools/output-accumulator.ts#L28-L118)                       |
| Compaction                 | [`packages/coding-agent/src/core/compaction/compaction.ts`](../packages/coding-agent/src/core/compaction/compaction.ts) |
| Split-turn                 | [`findCutPoint()` / `prepareCompaction()`](../packages/coding-agent/src/core/compaction/compaction.ts)                  |
| 文件状态摘要               | [`packages/coding-agent/src/core/compaction/utils.ts`](../packages/coding-agent/src/core/compaction/utils.ts#L26-L82)   |
| 总结输入截断               | [`serializeConversation()`](../packages/coding-agent/src/core/compaction/utils.ts#L84-L150)                             |
| Coding Agent 自动压缩      | [`AgentSession._checkCompaction()`](../packages/coding-agent/src/core/agent-session.ts#L1886-L1978)                     |
| 动态工具 Prompt            | [`packages/coding-agent/src/core/system-prompt.ts`](../packages/coding-agent/src/core/system-prompt.ts)                 |
| Compaction 文档            | [`packages/coding-agent/docs/compaction.md`](../packages/coding-agent/docs/compaction.md)                               |
| Mid-loop 缺口              | [Issue #5512](https://github.com/earendil-works/pi/issues/5512)                                                         |
| `shouldStopAfterTurn` 合入 | [PR #7367](https://github.com/earendil-works/pi/pull/7367)                                                              |

## 12. 最终建议

最值得从 PI 引入的不是它的完整 Coding Agent，而是以下五个机制：

1. Tool 输出源头限流，按工具语义选择 head 或 tail。
2. 模型调用前唯一的 context transform 边界。
3. 按 token 保留最近上下文，并允许 split-turn compaction。
4. 结构化保存文件和工作状态，避免压缩后重复劳动。
5. 在完整 Tool Turn 之后、下一次模型调用之前执行安全背压。

Wegic 在此基础上必须补充 Round Budget、Read Ledger、Image Budget 和阶段收敛。这样既能利用 PI 已验证的上下文工程设计，也能解决 [`demo.json`](../../wegic-agents/docs/plans/demo.json) 暴露的累计成本问题，而不引入双重 Agent Loop。
