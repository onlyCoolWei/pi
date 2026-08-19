# Pi 的 Token 效率机制与批量生图优化方案

## 1. 背景

在网站生成场景中，用户可能一次要求为大量产品生成主图、颜色图、细节图和场景图。例如：

- 18 个产品；
- 每个产品 3 个颜色；
- 每个产品还需要细节图和使用场景图。

仅颜色图就需要 54 张。如果 Agent 为每张图片分别输出完整的生图 prompt、工具参数和结果，再在后续回合持续携带这些历史，会产生大量编排 Token。

这里需要区分两类成本：

1. **Agent 编排 Token**：模型理解需求、输出工具调用、读取工具结果以及继续执行时消耗的输入和输出 Token。
2. **图片生成成本**：图片模型实际生成图片的成本。只要图片数量和质量要求不变，这部分不能通过上下文压缩直接消除。

Pi 的主要价值是降低第一类成本，并减少错误、重复生成和无意义重试间接造成的第二类成本。

## 2. Pi 的总体做法

Pi 没有依赖单一的“省 Token”功能，而是在 Agent Loop、工具系统、上下文转换、结果截断、缓存和 compaction 等层面共同控制成本。

整体链路如下：

```text
用户输入
  -> AgentSession 组装当前 system prompt 和 active tools
  -> Agent Loop 构建本轮 provider context
  -> transformContext 裁剪或转换消息
  -> 模型产生一个或多个工具调用
  -> runtime 串行或并行执行工具
  -> 工具只返回模型继续工作所需的结果
  -> 上下文接近上限时执行 compaction
```

核心原则是：

> 让模型负责判断和决策，让确定性程序负责重复展开、并发执行、存储和汇总。

## 3. Pi 已有的 Token 效率机制

### 3.1 在一次模型回合中并行执行独立工具

Pi 的 Agent Loop 会收集当前 assistant message 中的全部工具调用。如果没有工具要求顺序执行，runtime 会进入并行执行路径。

源码：

- [`packages/agent/src/agent-loop.ts`](../packages/agent/src/agent-loop.ts)，`executeToolCalls()`
- [`packages/agent/src/agent-loop.ts`](../packages/agent/src/agent-loop.ts)，`executeToolCallsParallel()`

并行工具调用的主要 Token 收益不是让单个工具参数变短，而是减少模型回合数。

```text
低效方式：
模型 -> 生成图片 1 -> 模型 -> 生成图片 2 -> 模型 -> ...

Pi 支持的方式：
模型 -> 一次给出多个工具调用 -> runtime 并行执行 -> 模型读取一次汇总结果
```

如果每次工具调用后都重新请求模型，完整 system prompt、历史消息和工具定义会被重复计入输入。并行执行可以避免这一部分重复。

### 3.2 在发送请求前转换上下文

Pi 在构建 provider context 时先调用可选的 `transformContext`，然后再执行 `convertToLlm`。

源码：

- [`packages/agent/src/agent-loop.ts`](../packages/agent/src/agent-loop.ts)，`buildProviderContext()`

这提供了一个明确的上下文治理边界：session 可以保留完整记录，但发送给模型的内容可以更精简。

适合在这一层实现的策略包括：

- 把已完成批次的几十条工具结果折叠成一条摘要；
- 移除后续推理不再需要的成功日志；
- 保留失败项、manifest 路径和未完成任务；
- 将大块历史状态替换为结构化 checkpoint。

### 3.3 动态控制 active tools

Pi 将“已注册工具”和“当前暴露给模型的工具”分开。`setActiveToolsByName()` 可以切换 active tools，并根据当前工具集合重新构建 system prompt。

源码：

- [`packages/coding-agent/src/core/agent-session.ts`](../packages/coding-agent/src/core/agent-session.ts)，`getActiveToolNames()`
- [`packages/coding-agent/src/core/agent-session.ts`](../packages/coding-agent/src/core/agent-session.ts)，`setActiveToolsByName()`
- [`packages/coding-agent/src/core/agent-session.ts`](../packages/coding-agent/src/core/agent-session.ts)，`_rebuildSystemPrompt()`
- [`packages/coding-agent/examples/extensions/kimi-deferred-tools.ts`](../packages/coding-agent/examples/extensions/kimi-deferred-tools.ts)

工具定义本身会占用上下文。对于专门的生图阶段，只暴露读取产品数据、批量生图、检查结果和更新 manifest 等相关工具，可以避免每轮重复发送大量无关 schema 和工具说明。

Pi 的 deferred-tool 示例进一步展示了先暴露一个搜索工具，再按需求激活低频工具的模式。

### 3.4 限制工具输出，并保留完整结果的外部位置

Pi 的工具输出有统一的行数和字节上限。默认上限为 2000 行或 50 KB，先达到哪个就按哪个截断。

源码：

- [`packages/coding-agent/src/core/tools/truncate.ts`](../packages/coding-agent/src/core/tools/truncate.ts)
- [`packages/coding-agent/src/core/tools/output-accumulator.ts`](../packages/coding-agent/src/core/tools/output-accumulator.ts)
- [`packages/coding-agent/src/core/tools/bash.ts`](../packages/coding-agent/src/core/tools/bash.ts)

对于被截断的命令输出，Pi 可以保存完整输出，并只把截断提示和完整输出路径交给模型。这一模式同样适用于批量生图：

```text
不要返回：
54 个完整 prompt + 54 个 URL + 54 份元数据 + 所有成功日志

只返回：
Generated: 51/54
Manifest: assets/product-images.json
Failed: product-03/sand, product-11/detail, product-16/black
```

完整结果保存在文件、数据库或对象存储中，需要时再按产品或失败项读取。

### 3.5 区分展示数据、运行时详情和模型上下文

Pi 的 custom message 包含 `content`、`display` 和 `details`。转换为 LLM message 时，真正进入模型上下文的是 `content`，运行时详情不需要全部序列化给模型。

源码：

- [`packages/coding-agent/src/core/messages.ts`](../packages/coding-agent/src/core/messages.ts)，`CustomMessage`
- [`packages/coding-agent/src/core/messages.ts`](../packages/coding-agent/src/core/messages.ts)，`convertToLlm()`

这说明 UI、审计记录和模型上下文不必使用同一份数据结构。批量图片的完整任务详情可以供 UI 展示和故障追踪，而模型只接收继续决策所需的紧凑摘要。

### 3.6 结构化 compaction

当上下文使用量超过 `contextWindow - reserveTokens` 时，Pi 可以触发 compaction。默认配置会预留输出空间，并保留一部分最近消息。

源码：

- [`packages/coding-agent/src/core/compaction/compaction.ts`](../packages/coding-agent/src/core/compaction/compaction.ts)，`DEFAULT_COMPACTION_SETTINGS`
- [`packages/coding-agent/src/core/compaction/compaction.ts`](../packages/coding-agent/src/core/compaction/compaction.ts)，`shouldCompact()`
- [`packages/coding-agent/src/core/compaction/compaction.ts`](../packages/coding-agent/src/core/compaction/compaction.ts)，`findCutPoint()`

Pi 的摘要不是一段无结构的自然语言，而是保留：

- Goal；
- Constraints & Preferences；
- Progress；
- Key Decisions；
- Next Steps；
- Critical Context。

这种结构能删除旧工具日志，同时保留继续任务需要的文件路径、错误和未完成事项。

Compaction 是长任务的安全网，不应作为第一优化手段。它本身需要一次摘要模型调用，而且无法阻止模型最初输出几十份重复的长工具参数。

### 3.7 Provider Prompt Cache

Pi 的 provider 层支持 prompt cache retention，并在支持的 provider 上为稳定内容设置 cache control。

源码：

- [`packages/ai/src/types.ts`](../packages/ai/src/types.ts)，`cacheRetention`
- [`packages/ai/src/api/anthropic-messages.ts`](../packages/ai/src/api/anthropic-messages.ts)，`buildParams()`

适合缓存的内容包括：

- 稳定的 system prompt；
- 工具定义；
- 品牌视觉规则；
- 通用摄影规范；
- 不频繁变化的会话前缀。

缓存可以降低重复输入的计费，但不能降低模型输出长工具参数所需的输出 Token。因此应先减少重复输出，再利用缓存优化不可避免的稳定输入。

### 3.8 图片进入历史前统一缩放

Pi 会规范化工具返回的图片，并在默认情况下缩放过大的图片，避免大体积 base64 数据持续进入 session history 和后续 provider 请求。

源码：

- [`packages/coding-agent/src/utils/tool-result-images.ts`](../packages/coding-agent/src/utils/tool-result-images.ts)
- [`packages/coding-agent/src/utils/image-process.ts`](../packages/coding-agent/src/utils/image-process.ts)

批量生图任务不应把所有原图重新注入模型。更合适的方法是生成 contact sheet 或缩略图，只让模型查看整体一致性；单独回传失败或异常图片进行修正。

## 4. Pi 机制不能自动解决的问题

Pi 的并行执行可以减少模型回合数，但如果模型仍然输出如下内容 54 次，输出 Token 依然很高：

```text
Photorealistic commercial e-commerce product photography...
natural lighting...
realistic material texture...
clean off-white background...
no cartoon, no illustration, no text, no watermark...
```

因此，仅启用并行调用、Prompt Cache 或 compaction 还不够。必须把重复 prompt 的展开从 LLM 移到确定性代码中。

## 5. 面向批量生图的迁移方案

以下方案是基于 Pi 机制的领域扩展，不是 Pi 当前内置的批量生图功能。

### 5.1 增加批量领域工具

设计一个 `batch_generate_product_images` 工具，让模型只描述统一策略和差异数据：

```json
{
  "styleProfile": "momopaw-product-v1",
  "productsSource": "src/data/products.ts",
  "shots": ["color", "detail", "lifestyle"],
  "colorsPerProduct": 3,
  "concurrency": 6
}
```

工具内部负责：

1. 读取产品数据；
2. 根据产品类型选择 prompt 模板；
3. 将产品、颜色、角度和材质填入模板；
4. 展开全部生图任务；
5. 按并发上限执行；
6. 保存 URL、状态、prompt hash 和错误；
7. 生成 manifest；
8. 只返回简短摘要。

这样，模型不再为每张图重复输出摄影规范和完整 JSON。

### 5.2 使用模板和差异字段

统一摄影要求应由应用维护为带版本的 style profile：

```text
momopaw-product-v1:
  photorealistic commercial product photography
  clean off-white studio background
  natural soft lighting
  realistic material texture and proportions
  no text, illustration, fantasy, or watermark
```

模型只需要给出差异：

```json
{
  "product": "adjustable dog harness",
  "material": "padded oxford nylon",
  "hardware": ["D-ring", "quick-release buckles"],
  "colors": ["black", "olive", "sand"]
}
```

固定模板由程序展开，可以同时降低 Token、减少风格漂移，并便于统一升级摄影规范。

### 5.3 样品先行，再批量展开

在生成 54 张图片之前，先为每个产品类别生成一个样品：

```text
产品规格提取
  -> 每类生成 1 张样品
  -> 检查结构、材质和摄影风格
  -> 通过后批量展开颜色和角度
```

这种方式可能增加一个模型回合，但能减少大批图片方向错误后产生的重试成本。对于图片成本高于文本 Token 成本的场景，这是值得的。

### 5.4 使用内容哈希避免重复生成

为每个任务计算稳定哈希：

```text
hash(styleProfileVersion + productSpec + color + shotType + aspectRatio)
```

哈希已存在且资产仍有效时直接复用。只有输入规格或模板版本变化时才重新生成。

这可以防止：

- Agent 重跑整个任务；
- 页面改动意外触发生图；
- 相同颜色或角度被重复请求；
- 重试逻辑重新生成已成功资产。

### 5.5 使用 manifest 作为单一事实来源

批量工具输出一个结构化 manifest：

```json
{
  "batchId": "momopaw-2026-08-19",
  "styleProfile": "momopaw-product-v1",
  "products": {
    "mp-h001": {
      "black": "assets/mp-h001-black.webp",
      "olive": "assets/mp-h001-olive.webp",
      "sand": "assets/mp-h001-sand.webp"
    }
  },
  "failures": []
}
```

后续页面代码只读取 manifest。模型无需在上下文中记住几十个 URL，也不需要从历史工具结果中重新寻找对应关系。

### 5.6 只把异常返回模型

批量执行结束后，成功项由程序直接记录。模型只处理：

- 生成失败；
- 内容审核失败；
- 产品结构明显错误；
- 同一产品不同颜色结构不一致；
- 需要重新生成的少数异常项。

将正常路径做成确定性流程，把模型留给异常路径，通常比对所有结果逐条推理更省 Token。

## 6. 推荐落地顺序

按收益和实现成本排序：

1. **批量领域工具**：一次接收模板或 profile 与产品清单，在工具内部展开任务。
2. **紧凑工具结果**：完整数据写入 manifest，只返回计数、失败项和路径。
3. **哈希缓存**：跳过已存在且输入未变化的图片。
4. **样品先行**：每类先验证一张，再批量生成。
5. **动态 active tools**：生图阶段只暴露必要工具。
6. **Prompt Cache**：缓存稳定 system prompt、工具定义和 style profile。
7. **上下文转换与 compaction**：折叠已完成批次，为长任务提供兜底。
8. **缩略图与 contact sheet**：避免全部原图进入模型历史。

## 7. 预期效果

假设原方案由模型输出 54 个近似的生图工具调用，改为一次批量工具调用后：

- 工具调用参数从几十份长 JSON 降为一份短 manifest 或配置；
- 模型回合数保持在少数几个阶段；
- 成功日志和图片 URL 不再持续进入上下文；
- 重复任务可以由哈希缓存直接跳过；
- 模型只处理样品判断和异常重试。

在需求不变的情况下，这通常可以减少 70% 到 95% 的 Agent 编排 Token。实际图片生成数量没有变化时，图片模型本身的成本不会按相同比例下降；图片侧的主要节省来自缓存、样品验证和减少错误重试。

## 8. 结论

Pi 的 Token 效率来自分层控制：并行工具执行减少回合数，active tools 缩短工具上下文，transformContext 和工具输出截断减少历史噪声，Prompt Cache 降低稳定输入成本，compaction 保证长任务可以继续。

对于批量生图，最重要的进一步改造是：

> 不让 LLM 逐张编写完整生图调用，而是让 LLM 生成一次策略和紧凑规格，由批处理工具确定性展开、并发执行、缓存并汇总。

这是比单独缩短 prompt、增加并发或更频繁 compaction 更直接的降本方式。
