# Pi Coding Agent 代码质量优化源码导航

这份文档不是 Pi 的完整设计说明，而是面向“提升 coding 质量”时的源码索引。

重点回答三个问题：

1. Pi 如何减少错误修改、并发写入和不完整工具调用。
2. 哪些质量来自 system prompt、工具实现和运行时保护。
3. 哪些质量门禁 Pi 没有默认提供，需要在自己的项目中补齐。

## 先看整体调用链

```text
用户请求
  -> AgentSession 组装 system prompt、项目上下文和当前工具
  -> Agent loop 请求模型
  -> 校验工具调用参数、权限和工具是否可用
  -> 执行 read / grep / edit / write / bash 等工具
  -> 返回结构化结果、错误和 diff
  -> Agent 根据结果继续规划或执行验证命令
  -> 长任务触发 compaction，保留目标、进度、文件和关键上下文
```

建议按这个顺序阅读：

1. [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)
2. [agent.ts](../packages/agent/src/agent.ts)
3. [agent-loop.ts](../packages/agent/src/agent-loop.ts)
4. [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)
5. [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) 和 [edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)
6. [bash.ts](../packages/coding-agent/src/core/tools/bash.ts)
7. [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)

## 按优化目标查找

### 我要让 Agent 遵循仓库规范

- [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)：发现并加载全局、项目和父目录中的 `AGENTS.md`、`CLAUDE.md` 等上下文文件，并按目录范围合并。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：`_rebuildSystemPrompt` 把项目上下文放入本轮 system prompt；上下文变更后可重新构建。
- [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)：把工作目录、项目规则、技能和工具说明组合成最终 prompt。
- [tool-system-prompt-contributions.test.ts](../packages/coding-agent/test/tool-system-prompt-contributions.test.ts)：检查内置工具的 prompt 片段和工具定义保持一致。

借鉴重点：把代码风格、测试命令、禁止修改的目录、完成标准写进仓库规则，而不是只依赖模型临时记忆。规则能约束行为，但不能替代工具层校验。

### 我要设计更可靠的 System Prompt 和工具说明

- [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)：默认 coding assistant 角色、工作目录、工具使用指导、项目上下文和技能说明。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：`_rebuildSystemPrompt` 负责把当前激活工具和工具提示注入 prompt。
- [tools](../packages/coding-agent/src/core/tools)：每个内置工具可以提供自己的 `promptSnippet` 和 `promptGuidelines`。
- [tool-system-prompt-contributions.test.ts](../packages/coding-agent/test/tool-system-prompt-contributions.test.ts)：验证工具声明与 prompt 贡献没有漂移。

借鉴重点：稳定的通用规则放在核心 prompt，工具的输入限制和使用时机放在工具自身的 guideline。不要把“必须这样做”只写在 prompt 中；能在代码中拒绝的错误，应放到参数校验、权限钩子或工具执行层。

### 我要让文件修改更可靠

- [edit.ts](../packages/coding-agent/src/core/tools/edit.ts)：编辑工具要求 `path` 加一组 `oldText/newText`，并在写入前准备和校验参数。
- [edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)：负责查找旧文本、检查唯一性、检查编辑区间重叠、拒绝空操作，并生成 unified diff。
- [write.ts](../packages/coding-agent/src/core/tools/write.ts)：用于完整写入或创建文件，和针对局部修改的 `edit` 分工不同。
- [path-utils.ts](../packages/coding-agent/src/core/tools/path-utils.ts)：统一路径解析和工作目录边界检查。

这里的质量保护包括：

- 旧文本找不到时失败，而不是静默写入错误位置。
- 旧文本重复时失败，要求模型扩大匹配范围。
- 多个编辑重叠时失败，避免后一个编辑覆盖前一个编辑的意图。
- 没有实际变化时失败，避免产生无意义写入。
- 保留 BOM 和换行风格，并返回变更 diff 和首个变更行。

借鉴重点：让模型提交“基于原文的精确变更”，不要默认接受整文件重写。`write` 适合新文件或明确的完整替换；`edit` 适合已有文件的局部修改。

### 我要防止并发、部分或错误写入

- [file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)：按真实路径对同一文件串行化 `edit` 和 `write`，不同文件可以并行；符号链接别名也会归并到同一队列。
- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：默认可并行执行独立工具；工具或配置声明 `executionMode: "sequential"` 时切换为顺序执行。
- [file-mutation-queue.test.ts](../packages/coding-agent/test/file-mutation-queue.test.ts)：覆盖同文件串行、不同文件并行、符号链接和 abort 后的锁释放。
- [agent-loop.test.ts](../packages/agent/test/agent-loop.test.ts)：覆盖工具执行顺序、并行和顺序模式。

借鉴重点：并发策略应由运行时控制，而不是要求模型“记得不要并发”。文件级互斥只能保证写入安全，不能保证两个修改在语义上相容，因此仍需要后续测试和 diff 检查。

### 我要给工具调用加权限和安全边界

- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：`prepareToolCall` 检查工具是否存在于当前上下文，执行参数准备、参数校验和 `beforeToolCall`；被阻止的调用不会执行。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：安装扩展的 before/after tool hooks，可阻止调用、终止任务或改写工具结果。
- [project-trust.ts](../packages/coding-agent/src/core/project-trust.ts) 和 [trust-manager.ts](../packages/coding-agent/src/core/trust-manager.ts)：管理项目是否被信任。
- [permission-gate.ts](../packages/coding-agent/examples/extensions/permission-gate.ts)：按工具或参数增加审批门禁。
- [protected-paths.ts](../packages/coding-agent/examples/extensions/protected-paths.ts)：保护敏感路径。
- [rpc-demo.ts](../packages/coding-agent/examples/extensions/rpc-demo.ts)：展示外部控制和工具调用拦截的扩展方式。

借鉴重点：把“能不能做”与“怎么做”分开。模型负责提出调用，运行时负责检查工具、路径、权限和参数。

### 我要让 Agent 验证修改质量

- [bash.ts](../packages/coding-agent/src/core/tools/bash.ts)：执行测试、lint、类型检查、构建、格式化和 `git diff`；支持超时、abort、进程树终止和输出截断。
- [truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)：限制超长命令输出，同时保留完整输出文件路径，避免模型被噪声淹没。
- [output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts)：在流式执行期间累积和截断输出。
- [5303-bash-output-truncation.test.ts](../packages/coding-agent/test/suite/regressions/5303-bash-output-truncation.test.ts)：回归测试命令输出截断行为。

重要边界：Pi 默认提供“执行验证命令”的能力，但没有强制每次 edit 后自动运行测试，也没有通用的语义验收门禁、回滚事务或默认 review agent。若质量要求高，应在项目规则中定义完成标准，或通过扩展在结束前检查测试、diff 和必改文件。

### 我要保证长任务中不会丢失代码状态

- [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)：在上下文接近上限时提取已读文件、已修改文件和工具结果，并生成结构化摘要。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：处理上下文溢出、compaction 和有限重试；compaction 后重新注入摘要继续任务。
- [agent-session-compaction.test.ts](../packages/coding-agent/test/agent-session-compaction.test.ts)：测试 session 层 compaction。
- [compaction.test.ts](../packages/agent/test/harness/compaction.test.ts)：测试 agent loop 的 compaction 行为。

摘要重点包括 Goal、Constraints、Progress、Key Decisions、Next Steps 和 Critical Context，并要求保留精确文件路径、函数名和错误信息。借鉴时不要只保留自然语言总结，否则后续 edit 很容易失去定位依据。

### 我要处理不完整或截断的工具调用

- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：当模型输出因 `length` 截断时，`failToolCallsFromTruncatedMessage` 不执行可能不完整的参数，而是返回错误工具结果，让模型重新发起调用。
- [agent-loop.test.ts](../packages/agent/test/agent-loop.test.ts)：覆盖截断工具调用和错误结果回传。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：区分上下文溢出和普通瞬时错误；前者先 compaction，后者按限制重试并退避。

借鉴重点：永远不要“猜测补全”被截断的 JSON 或 shell 命令。失败结果必须明确告诉模型发生了什么，让下一轮重新生成完整调用。

### 我要添加自定义质量门禁或 review 流程

- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：扩展可以监听 `beforeToolCall`、`afterToolCall` 和下一轮刷新事件。
- [extensions](../packages/coding-agent/examples/extensions)：权限、受保护路径、工具覆盖和 RPC 示例。
- [edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)：可复用 diff 生成逻辑，用于在提交前检查变更范围。

Pi 的默认结构是单个 coding agent 加工具循环，不是强制的“主 agent + review 子 agent”。如果需要 review，可以在完成编辑后由扩展或独立 agent 检查 diff、测试结果和验收标准；这属于额外质量流程，不是 Pi loop 的默认步骤。

## Pi 已有的质量保障与缺口

| 机制 | 主要源码 | 默认覆盖范围 |
| --- | --- | --- |
| 项目规则和上下文 | `resource-loader.ts`、`system-prompt.ts` | 让模型知道仓库约束 |
| 精确局部编辑 | `edit.ts`、`edit-diff.ts` | 防止错位、重复、重叠和无效修改 |
| 文件写入串行化 | `file-mutation-queue.ts` | 防止同一文件并发写坏 |
| 工具参数和权限校验 | `agent-loop.ts`、session hooks | 阻止未知工具、非法参数和未授权操作 |
| 截断调用保护 | `agent-loop.ts` | 不执行不完整工具参数 |
| 命令验证能力 | `bash.ts` | 能运行测试和检查，但默认不强制运行 |
| 长任务上下文保留 | `compaction.ts` | 保留目标、进度、文件和关键错误 |
| 语义正确性证明 | 无统一默认实现 | 需要项目测试、验收脚本或 review 流程 |
| 自动回滚 | 无统一默认实现 | 需要自行设计事务、备份或 git 工作流 |

## 推荐的优化顺序

1. 先加载并强化项目规则：明确修改边界、测试命令和完成标准。
2. 采用 `edit` 的精确匹配和 diff 反馈，减少整文件重写。
3. 在运行时加入参数、路径、权限和敏感操作校验。
4. 对同文件修改做串行化，对独立只读查询保留并行。
5. 建立固定验证契约：修改后运行针对性测试、lint、类型检查和 `git diff`。
6. 让工具结果只返回可行动信息，保留错误原因、退出码和完整输出位置。
7. 为长任务设计结构化 compaction，保留精确路径、符号和未完成事项。
8. 最后再增加 review agent 或语义质量门禁；它不能替代前面的基础保护。

## 最小质量指标

建议在自己的 coding agent 中记录：

- 每次任务的验收标准，以及是否全部满足。
- 修改文件列表、每个文件的 edit 次数、失败原因和最终 diff 统计。
- 工具调用被阻止、参数校验失败、权限拒绝和截断调用次数。
- 测试、lint、类型检查和构建命令的退出码与摘要。
- 重试、compaction、未完成事项和最终未解决错误。
- 任务结束时是否存在未验证的修改。

## 一句话总结

Pi 的 coding 质量主要来自一条分层保护链：项目规则负责约束意图，`edit-diff` 负责精确修改，agent loop 和 mutation queue 负责安全执行，`bash` 负责验证，compaction 负责长任务连续性；真正的语义正确性仍需要项目自己的测试和验收门禁。
