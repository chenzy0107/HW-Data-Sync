# Dify + GLM 单页 HTML 生成工具

> 一句话 → 一个内容丰富、设计精美、可直接运行的单页 HTML 网站。对标 v0.dev / Lovable / Gemini Canvas，用 **Dify Workflow + GLM-4.6** 落地。

---

## 一、为什么这么设计（核心原理）

让 LLM 一次性生成"内容丰富的整页 HTML"有 **3 个独立的失败根因**，本方案逐一击破：

| 失败根因 | 表现 | 本方案的解法 |
|---|---|---|
| **① 物理截断** | 输出 token 到上限被硬切，HTML 断在标签中间 | GLM-4.6 拥有 **128K 输出窗口**，单页富 HTML 通常仅 8K~20K token，绰绰有余；超长页走兜底分块 |
| **② 模型「偷懒占位符」** | 输出 `...` / `[content here]` / `<!-- 其余略 -->` | Generator Prompt 用 v0 同款**强约束**（零占位符、零注释、必须用 ```html 代码块包裹、必须完整） |
| **③ 长文质量衰减** | 一次直出整页，后半段走样、内容空洞 | **Plan-then-Code**：先让 Planner 规划真实文案大纲，Generator 再据此成文 |
| **④ RAG 幻觉/编造** | 接入知识库后仍瞎编价格、功能、数据 | Planner 与 Generator 都绑定 Context，Prompt 加「事实严格遵循、未覆盖不捏造」硬约束 |

这就是 v0.dev、Lovable、Bolt.new 等已被验证的工业做法（Planner→Generator 分层 + Tailwind CDN 骨架 + 强约束 Prompt），本方案把它迁移到 Dify Workflow 上。

> 选 **GLM-4.6/4.5** 的关键理由：**输出窗口大**（128K/96K，业界最大档之一）且**成本低**（社区反馈约为 Claude 的 1/7），是"单次出整页"最划算的选择。Claude Sonnet 4.5(64K)、GPT-4.1(32K) 也可用；DeepSeek(8K) 输出窗口太小，**必须**走兜底分块路线。

---

## 二、架构总览

### 主路线（推荐，5 节点 · 含私有 RAG 知识库）— 适用于绝大多数单页

```
┌─────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────┐
│  Start  │▶│  LLM-0       │▶│  知识检索    │▶│  LLM-1       │▶│  LLM-2         │▶│  Output  │
│ 一句话   │  │  Query       │  │  (查私有库)  │  │  Planner     │  │  Generator     │  │  html    │
│ 需求     │  │  Rewriter    │  │  输出 result │  │  Context=    │  │  Context=      │  │          │
└─────────┘  │  (生成检索词)│  │  数组        │  │  检索结果    │  │  检索结果      │  └──────────┘
             └──────────────┘  └──────────────┘  │  (出大纲JSON)│  │  (出完整HTML)  │
             小模型, temp=0    TopK/阈值/Rerank   temp=0.3       temp=0.6~0.7
                                                  关结构化输出   关结构化输出,max_tokens拉满
```

- **LLM-0 Query Rewriter**：把用户的"生成指令"（如「做个 AI 写作助手落地页」）改写成"检索关键词"（如「AI 写作助手 功能 价格 特色 用户群」）。用**最便宜的小模型**即可。**为什么需要**：直接拿生成指令去检索命中率差，改写后召回质量显著提升。
- **知识检索节点**：用改写后的查询查私有知识库，输出 `result`（对象数组）。配置 Top K、Score Threshold（仅返回高于阈值的，过滤无关召回）、Rerank。
- **LLM-1 Planner**：**Context 绑定检索结果**，规划页面结构（哪些 section、文案要点、视觉意图、配色），输出 JSON。**事实类文案严格依据检索结果，禁止编造**。
- **LLM-2 Generator**：**Context 同样绑定检索结果**，把 Planner 大纲作为结构约束，**单次**生成完整 HTML。事实内容严格依据检索结果。
- **Output 节点**：定义输出变量 `html`，API 返回 `data.outputs.html`。

> ★**RAG 注入的关键简化**：知识检索节点输出的是**数组**，但**不需要加 join 节点**——下游 LLM 节点有专门的 **Context 字段**，把 `result` 绑定进去即可，Dify 自动序列化为文本并启用引用追踪。[LLM 节点文档](https://docs.dify.ai/en/guides/workflow/node/llm)

**为什么主路线不分 section 迭代？** GLM 单次窗口够大，单次生成**结构一致性最好**（无拼接错位）、**最快**、**最省 token**。只有"超长页（10+ 个重内容 section）"才需要兜底路线。

### 兜底路线（超长页面）— Iteration 分块

```
Start → LLM-1 Planner(输出 section 数组)
      → Iteration 节点(输入=section数组, 内部一个 LLM 逐段生成 HTML 片段) → 片段数组
      → Template/Code 节点(把数组 join 成完整 HTML) ★Dify硬限制,不可省
      → Output 节点
```

> ⚠️ **Dify 硬限制**（官方社区高频踩坑点）：Iteration 的输出是**数组**，而 LLM 节点只接受**字符串**。中间**必须**加 Template(`{{ sections | join('\n') }}`) 或 Code 节点转换，否则下游报错。详见 `dify-workflow-guide.md` 的「兜底路线」章节。

---

## 三、文件说明

```
html-generator/
├── README.md                       ← 本文件：方案总览、原理、模型选型、优化建议
├── dify-workflow-guide.md          ← Dify 每个节点的逐步配置（照着点就行）
├── prompts/
│   ├── system-prompt.md            ← 全局 System Prompt（含 RAG 防幻觉约束，各 LLM 共用）
│   ├── query-rewriter-prompt.md    ← LLM-0 查询改写（生成指令 → 检索关键词）
│   ├── planner-prompt.md           ← LLM-1 Planner 的 User Prompt（Context=检索结果）
│   ├── generator-prompt.md         ← LLM-2 Generator 的 User Prompt（核心，Context=检索结果）
│   └── generator-schema.json       ← Planner 的结构化输出 JSON Schema（粘进 Dify）
└── api-and-preview.md             ← 发布后 API 调用 + iframe 安全预览
```

**最快上手路径**：先读 `dify-workflow-guide.md` 照着搭主路线（3 节点），Prompt 直接从 `prompts/` 复制。

---

## 四、模型与关键参数

| 节点 | 模型 | temperature | max_tokens | 结构化输出 | 其它 |
|---|---|---|---|---|---|
| LLM-0 Query Rewriter | **小模型**（glm-4-flash / glm-4.5-air） | **0** | 512 | 关 | 只出检索词，越便宜越快 |
| LLM-1 Planner | GLM-4.6 / 4.5 | **0.3** | 默认即可 | **开**（粘 generator-schema.json） | Context 绑定检索 result；只要结构稳 |
| LLM-2 Generator | GLM-4.6（首选）| **0.6~0.7** | **尽量拉满**（≥16000，按 UI 上限） | **关**（输出纯 HTML） | Context 绑定检索 result；要创意但不发散 |

- 接入：Dify「设置 → 模型供应商」配置 Z.ai / 智谱 GLM 的 API Key。
- **`max_tokens` 务必拉满**：单页富 HTML 8K~20K token，用默认值（很多模型默认仅 4K）必截断。

---

## 五、落地步骤（精简版，详见 dify-workflow-guide.md）

1. 在 Dify 新建 **Workflow 类型**应用。
2. Start 节点：加一个文本输入变量 `user_requirement`。
3. **建好私有知识库**：「知识库」页上传你的产品/业务资料（PDF/Word/网页/文本），等索引完成。
4. 加 **LLM-0 Query Rewriter**：粘 query-rewriter-prompt，小模型，temp=0，输出变量 `search_query`。
5. 加 **知识检索节点**：Query 选 LLM-0 的 `search_query`，勾选你的知识库，设 Top K（如 5）、Score Threshold（如 0.5）、Rerank。
6. 加 **LLM-1 Planner**：粘 system-prompt + planner-prompt，**Context 绑定检索 result**，开结构化输出粘 schema，temp=0.3。
7. 加 **LLM-2 Generator**：粘 system-prompt + generator-prompt，**Context 也绑定检索 result**，把 Planner 输出拖进 `{{planner_output}}`，temp=0.7，max_tokens 拉满，关结构化输出。
8. 加 **Output 节点**：选 Generator 的输出，命名为 `html`。
9. 右上角「运行」用一句话测试，如：*「帮我做一个 AI 写作助手的落地页」*。
10. 发布 → 访问 API（见 api-and-preview.md）。

---

## 六、优化与排错建议

- **压测建议**：首版只做主路线，用三类 prompt 压测——①落地页 ②产品介绍 ③个人简历/作品集。单页稳定后再考虑兜底分块。
- **Generator 偶发截断**（极少见，因 GLM 窗口大）：
  - 方案 A：在调用端检测输出是否以闭合的 ``` 代码块结尾（且含 `</html>`），没有则触发续写循环（把已生成内容作为 assistant 消息回传 + "continue exactly where you left off"）。
  - 方案 B：需求明显超长（如"做一份超长的产品文档页"）时，切换到兜底 Iteration 路线。
- **占位符没防住**：说明 Generator Prompt 的强约束没生效，检查是否 ①温度过高 ②max_tokens 被默认值限制了导致"被迫省略"。
- **配色/设计不佳**：Planner 的 `theme` 字段质量决定上限，可在 Planner Prompt 里强化"配色要协调、对比要强"的要求，或固定几套优质主题色板让模型选。
- **成本**：GLM 单次整页生成 token 低、速度快；兜底 Iteration 并行上限 10，用于真正长页。

---

## 七、可扩展方向

- **多轮修改**：把应用类型从 Workflow 改成 Chatflow，让用户对话式微调（"把首屏改成深色""加一个价格表"）。
- **模板库**：在 Planner 前加一个 IF/ELSE 或知识库检索，按主题选预设主题色板与区块组合，进一步提升稳定性与一致性。
- **图片**：接入一个图片生成节点（或占位图服务如 Unsplash Source / picsum），在 Generator 里用真实图片 URL 替换占位，视觉更丰满。
- **导出**：前端「下载 .html」「复制代码」按钮，配合 iframe 预览。
