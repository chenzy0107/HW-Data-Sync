# Dify Workflow 逐步配置指南

> 照着本文从上到下操作，约 15 分钟搭好主路线。所有 Prompt 在 `prompts/` 目录，直接复制。

---

## 0. 准备

1. 登录 Dify（Dify Cloud 或自托管）。
2. 「设置 → 模型供应商」接入 **GLM**：
   - 智谱开放平台或 Z.ai 申请 API Key。
   - 在 Dify 配置好 `glm-4.6`（或 `glm-4.5`、`glm-4-plus` 等可用模型名，以你账号实际可见为准）。
3. 进入「工作室 / 创建空白应用 → **工作流 (Workflow)**」。

---

## 主路线（5 节点 · 含私有 RAG 知识库）

> 拓扑：Start → LLM-0 Query Rewriter → 知识检索 → LLM-1 Planner → LLM-2 Generator → Output

### 节点 1：Start（开始）

- 在画布上点击 **「开始节点」**。
- 添加输入字段：
  - 字段名：`user_requirement`
  - 类型：**段落 (Paragraph)** 或 **文本 (Text)**
  - 必填：是
  - 标签/提示：例如「用一句话描述你想要的网站」

> 之后所有节点的 `{{user_requirement}}` 都引用这里。

### 前置：建好私有知识库

1. 进入 Dify 顶部「**知识库 (Knowledge)**」页 → 「创建知识库」。
2. 上传你的**产品/业务资料**（PDF / Word / TXT / 网页链接 / Markdown 均可），如产品手册、功能清单、价格表、公司介绍。
3. 选择分段策略（推荐「自动分段」或按需自定义），等**索引完成**（状态变成就绪）。
4. 记下这个知识库名称，后面「知识检索节点」要选它。

> 检索质量很大程度取决于知识库内容质量。建议：一份资料讲清一类事实（如单独一份「产品功能」「价格方案」），分段不要太长。

### 节点 2：LLM-0 Query Rewriter（查询改写）

在 Start 后面加一个 **「LLM」节点**，命名 `QueryRewriter`。这是**轻量节点**，用最便宜的小模型即可。

**① 模型**
- 选**小模型**：`glm-4-flash` 或 `glm-4.5-air`（无需大模型，省钱省时间）。

**② SYSTEM / USER**
- 粘贴 `prompts/query-rewriter-prompt.md` 里的 SYSTEM 与 USER 内容。
- `{{user_requirement}}` 绑定 Start 的 `user_requirement`。

**③ 结构化输出**
- **关闭**（输出纯文本检索词）。

**④ 参数**
- `temperature`：**0**（要稳定）。
- `max_tokens`：**512**（只输出几行查询词）。

**⑤ 输出变量**
- 该节点的输出文本就是检索查询词，记下输出变量名（如节点的 `text` / `output`），下一步知识检索节点引用它。

> 验证：输入 `做个 AI 写作助手落地页`，应输出类似 `AI写作助手 产品功能 价格 特色 适用场景 目标用户` 的关键词。

### 节点 3：知识检索（Knowledge Retrieval）

在 QueryRewriter 后面加一个 **「知识检索 (Knowledge Retrieval)」节点**，命名 `KnowledgeRetrieval`。

**① Query（查询变量）**
- 选 **变量** 输入，绑定 **QueryRewriter 节点的输出**（上一步的检索查询词）。
- ⚠️ 不要直接绑 `user_requirement`（那是生成指令，检索效果差）。

**② 知识库（Dataset）**
- 勾选你**前置步骤建好的私有知识库**（可多选，节点会同时检索多个库再合并）。

**③ 检索设置**
- **检索模式**：推荐 **混合 (Hybrid)**（关键词+语义）；纯语义也可。
- **Top K**：建议 **5**（重排后返回的片段数）。
- **Score Threshold（分数阈值）**：建议 **0.5**（仅返回相似度 ≥ 阈值的结果，过滤无关召回；调高更严格、调低更宽松）。
- **Rerank（重排序）**：推荐开启——选 **加权分数 (Weighted Score)**（调语义/关键词权重）或接一个 **Rerank 模型**（如 Jina/Cohere，质量更好）。

**④ 输出**
- 节点输出变量是 **`result`**（一个**对象数组**，每元素含 `content / title / score / metadata`）。
- ⚠️ **不需要加 join 节点**！下一步直接把 `result` 绑定到 LLM 节点的 **Context 字段**，Dify 自动序列化为文本。

> 验证：点节点「运行」，看 `result` 里是否召回了你预期的资料片段；若召回不准，调 Query 改写词 / Top K / 阈值 / 知识库内容。

### 节点 4：LLM-1 Planner（规划结构）

在 KnowledgeRetrieval 后面加一个 **「LLM」节点**，命名 `Planner`。

**① 模型**
- 选 `glm-4.6`（首选）或 `glm-4.5`。

**② SYSTEM（系统提示词）**
- 粘贴 `prompts/system-prompt.md` 里「直接复制下面的内容」整段。

**③ USER / 提示词（用户消息）**
- 粘贴 `prompts/planner-prompt.md` 的 USER Prompt 整段。
- `{{user_requirement}}` 绑定 Start 的 `user_requirement`。

**④ ★Context（关键，RAG 注入）★**
- 在 LLM 节点的 **CONTEXT / 上下文** 字段里，点「+ 添加」，选择 **KnowledgeRetrieval 节点的 `result`**。
- 这样检索到的知识库事实会自动注入，Dify 还会启用**引用追踪**。**不要**在 prompt 正文里再手写 `{{KnowledgeRetrieval.result}}`（Context 已处理，重复会冗余）。

**⑤ 结构化输出（Structured Output / JSON Schema）**
- 打开「**结构化输出**」开关。
- 选 **JSON Schema** 方式，把 `prompts/generator-schema.json` 的**全部内容**粘进输入框。

**⑥ 参数**
- `temperature`：**0.3**（结构要稳）。
- `max_tokens`：默认即可（Planner 输出的是简短 JSON）。

> 验证：点节点右上角「运行」，应返回一段 JSON，含 `theme` 与 `sections[]`，且**事实性文案取自检索结果**（检查价格/功能是否与你知识库一致）。

### 节点 5：LLM-2 Generator（生成完整 HTML）

在 Planner 后面加一个 **「LLM」节点**，命名 `Generator`。

**① 模型**
- 选 `glm-4.6`（首选）。

**② SYSTEM**
- 粘贴 `prompts/system-prompt.md` 整段（与 Planner 共用）。

**③ USER / 提示词**
- 粘贴 `prompts/generator-prompt.md` 的 USER Prompt 整段。
- 变量绑定：
  - `{{user_requirement}}` → Start 的 `user_requirement`
  - `{{planner_output}}` → **Planner 节点的输出**（在变量选择器里选 Planner 节点 → output）。⚠️ 关键：必须引用 Planner 的结构化输出，不要手敲。

**④ ★Context（关键，RAG 注入）★**
- 同 Planner：CONTEXT 字段里**也绑定 KnowledgeRetrieval 的 `result`**。
- 让 Generator 也能看到原始检索事实，确保文案细节准确（Planner 可能压缩了细节）。

**⑤ 结构化输出**
- **关闭**（Generator 要输出纯 HTML 字符串，不是 JSON）。

**⑥ 参数**
- `temperature`：**0.6 ~ 0.7**。
- `max_tokens`：**拉到 UI 允许的上限**（GLM-4.6 可设很大；务必远大于默认 4K，否则单页 HTML 会截断）。

### 节点 6：Output（输出 / End）

在 Generator 后面加 **「输出 (Output / End)」节点**。

- 添加输出变量：
  - 变量名：`html`
  - 值：选择 **Generator 节点的输出文本**（Generator 的 `text` / `output`）。
- 这个 `html` 就是 API 最终返回的完整 HTML 字符串（`data.outputs.html`）。

> 注意：当前 Dify 版本终点节点叫 **Output 节点**（旧称 End）。你给变量取的名字 `html` 会成为 API 响应里的 key。

### 测试

- 画布右上角「运行」，输入：*「帮我做一个面向中小企业的在线客服 SaaS 产品落地页，要包含定价」*。
- 预期：Output 节点输出一整段被 ```html 代码块包裹的完整 HTML（剥掉围栏后以 `<!DOCTYPE html>` 开头、`</html>` 结尾，且不含任何注释）。
- 剥掉首尾的 ```html / ``` 后存成 `.html` 文件用浏览器打开，应是一个设计完整、内容丰富的落地页。
- ★**事实核查（RAG 专项）**：对比生成页面里的产品名/功能/价格与你的知识库资料，确认**严格一致、未编造**；若有出入，加强 Planner/Generator 的 Context 绑定与防幻觉约束，或检查知识库召回是否命中。

---

## 兜底路线（超长页面 · Iteration 分块）

> 仅当单页内容极多（如 10+ 个重内容 section），GLM 单次生成仍截断时启用。日常用主路线即可。

拓扑：

```
Start → Planner(输出 sections 数组)
      → Iteration(输入=Planner.sections, 内部 LLM 逐段生成 HTML 片段) → 输出片段数组
      → Template 节点(join 拼接) → Output
```

### 步骤

1. **Planner** 与主路线相同，确保其 `sections` 输出是数组。
2. 加 **「迭代 (Iteration)」节点**：
   - **输入**：选 Planner 输出的 `sections` 数组（必须是 array 类型）。
   - **模式**：并行（Parallel），注意并行上限 10。
   - **循环内部**：放一个 **LLM 节点**，Prompt 大意：
     > 你是前端工程师。请为下面这**一个**页面区块生成 HTML 片段（仅该区块的 `<section>...</section>`，**不要** `<!DOCTYPE>`、不要 `<html><head><body>`，不要 Tailwind CDN——这些由外层骨架提供）。
     > 当前区块信息：`{{items}}`（迭代内置变量，即数组当前元素）。主题色：`{{theme}}`。
     > 要求：完整、真实文案、无占位符、用 Tailwind class、**不得包含任何注释**（`<!-- -->`/`/* */`/`//` 都不行）。
   - 内部 LLM 的输出即"一段 HTML 片段"。
   - Iteration 节点最终输出 = **片段数组**（每个元素是一段 HTML 字符串）。
3. ⚠️ **加 Template 节点（关键，不可省）**：
   - Dify 的 LLM 节点只能吃字符串，Iteration 输出的数组**不能直接喂下游**。
   - 加 **「模板 (Template / Jinja2)」节点**，内容（不含任何注释，并用 ```html 代码块包裹，与主路线输出格式一致）：
     ```jinja
     ```html
     <!DOCTYPE html>
     <html lang="zh-CN">
     <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>{{ title }}</title>
       <script src="https://cdn.tailwindcss.com"></script>
       <style>:root{--c-primary:{{primary}};}</style>
     </head>
     <body>
     {% for sec in sections %}{{ sec }}
     {% endfor %}
     <footer>{{ footer_copy }}</footer>
     </body>
     </html>
     ```
     ```
     （把 Iteration 的片段数组用 `{% for %}` 或 `| join` 拼进骨架。`title` / `primary` / `footer_copy` 等可从 Planner 的 theme 取。各 section 片段本身也不得含注释。）
   - 或用 **Code 节点**（JS）：`return { html: "<!DOCTYPE html>..." + inputs.sections.join("\n") + "..." }`。注意 Code 节点字符串上限 40 万字符。
4. **Output 节点**：输出 Template/Code 的 `html`。

### 兜底路线的注意点
- Iteration 内部 LLM 也要设高 `max_tokens`（单段也可能较长）。
- 拼接顺序 = 数组顺序 = Planner 里 `order` 排序后的顺序；确保 Planner 输出按 `order` 升序。
- 骨架（`<head>`、Tailwind CDN、`<style>`、`</body>`）只在 Template 里出现**一次**，各 section 片段只含 `<section>`，避免重复引入 CDN。

---

## 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| Generator 引用 `{{planner_output}}` 报错 | 没用变量选择器绑定，或绑错了节点。删掉占位重新插入「变量 → Planner → output」。 |
| Generator 输出被截断（代码块未闭合，没有结尾 ```） | `max_tokens` 太小。拉到上限；或改走兜底 Iteration 路线。 |
| Generator 输出**没有** ```html 围栏、或夹带"好的这是…" | System Prompt 的禁令没生效，确认 system-prompt.md 已正确粘贴到 SYSTEM 位（围栏是**必须**的）。 |
| 页面出现 `...` / 占位符 | 温度过高 或 max_tokens 不足迫使省略。降温度到 0.6、加大 max_tokens。 |
| Iteration 下游 LLM 报"需要字符串" | 没加 Template/Code 转换。见兜底路线步骤 3。 |
| 配色丑/不协调 | 强化 Planner 的 theme 要求，或预设几套色板让其选。 |
| **★生成的产品/价格与知识库不符（编造）** | 防幻觉未生效。① 确认 Planner 和 Generator 都在 **Context 字段绑定了 KnowledgeRetrieval.result**；② 确认检索确实召回了相关资料（看 `result` 是否为空）；③ 调低 Score Threshold 提高召回。 |
| **★检索没召回到内容（result 为空）** | Query 改写词不准；或 Score Threshold 太高；或知识库分段过大/过小。检查 QueryRewriter 输出、降低阈值、优化知识库分段。 |
| **★LLM 报"未提供 context"** | Context 字段没绑定，或绑定成了空数组。在 Planner/Generator 的 CONTEXT 字段重新选 KnowledgeRetrieval → result。 |
| 不知道在哪绑 Context | LLM 节点配置里有独立的 **CONTEXT / 上下文** 区域（和 SYSTEM/USER 平级），点「+」选「知识检索 → result」。**不要**在 prompt 正文里写 `{{result}}`。 |
