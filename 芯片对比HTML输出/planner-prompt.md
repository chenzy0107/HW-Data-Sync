# Planner Prompt（LLM-1 · 只规划结构，不出代码 · 基于 RAG 事实）

> 配置要点：本节点**开启「结构化输出 / Structured Output」**，Schema 见同目录 `generator-schema.json`（粘进 Dify 的 JSON Schema 输入框）。`temperature` 设为 **0.3**（结构要稳）。模型用 GLM-4.6 / GLM-4.5。
>
> ★RAG 注入（关键）：本节点绑定 **Context = 知识检索节点的 `result`**（在 LLM 节点的 Context 字段里选「知识检索 → result」）。Dify 会自动把检索到的知识库片段序列化进上下文。
>
> 变量：
> - `{{user_requirement}}` → Start 节点输入
> - 检索到的事实知识 → 通过 **Context 字段**自动注入

---

## USER Prompt（直接复制下面的内容到 LLM-1 的用户消息）

你现在的任务是**规划页面结构**，不写任何 HTML 代码。

【用户需求】
{{user_requirement}}

【可用的知识库事实（Context）】
系统已通过 Context 为你注入了从私有知识库检索到的相关资料（产品功能、价格、特色等）。这是**权威事实来源**，规划文案时必须依据它。

【你的任务】
基于上面的需求，设计一个**内容丰富、结构专业**的一页式网站结构。你要像一个资深产品设计师那样思考：这个主题的网站应该包含哪些区块？每个区块讲什么？视觉风格与配色如何？

【★事实遵循规则 · 防幻觉，必须遵守★】
1. **文案要点必须依据 Context 真实资料**：`copy` 字段里写的产品名、功能、价格、特色、数据等**事实性内容**，必须**取自 Context** 检索结果，**不得编造**。
2. **Context 没有的内容留白处理**：若某个区块需要某类事实（如价格）但 Context 里**没有**，则在 `copy` 里写"用通用/不涉及具体数字的描述"（如"联系我们获取专属报价"），并在 `visual_intent` 或备注里标注"Context 无此数据，已用通用描述"，**绝不编造具体数字**。
3. **结构可自由设计**：区块的选择、数量、视觉布局不受 Context 约束，按主题自由规划最佳结构。

【规划原则】
1. **区块数量充足**：根据主题规划 **5 ~ 9 个区块（section）**，让页面饱满。典型可选区块类型：
   - `hero`（首屏：大标题 + 副标题 + 主行动按钮 + 视觉元素）
   - `logos` / `trust`（信任背书：合作机构、数据统计）
   - `features`（核心特性/服务，3~6 项）
   - `showcase` / `gallery`（案例展示 / 作品集）
   - `steps` / `how-it-works`（使用流程，3~4 步）
   - `testimonials`（用户评价，2~4 条）
   - `pricing`（价格方案，2~4 档）
   - `faq`（常见问题，4~6 条）
   - `cta`（号召行动）
   - `footer`（页脚：导航 + 版权 + 联系方式）
   你不必全部使用，按主题挑选最合适的组合。
2. **每个区块给足写作素材**：`copy` 字段要写出**真实、具体**的中文文案要点（标题、若干条要点、按钮文字等），**事实部分取自 Context**，让 Generator 直接可抄写。
3. **明确视觉意图**：`visual_intent` 写清该区块的视觉处理（如"左文右图卡片网格""深色背景大数字""三栏卡片带图标"），Generator 据此落地布局。
4. **配色与字体统一**：`theme` 里给出一套协调的主色 / 辅色 / 强调色 / 渐变 / 字体建议，所有区块共用。

【输出要求】
严格按照给定的 JSON Schema 输出，不要输出任何 HTML、不要输出任何解释性文字。文案内容必须**贴合「用户需求」且事实部分**依据 Context**，而非通用模板或编造内容。

【字段填写说明】
- `theme.brand_name`：网站/品牌名称（**取自 Context**；若 Context 无，按用户主题起一个）。
- `theme.colors.primary / secondary / accent`：主色 / 辅色 / 强调色，给 hex 值。
- `theme.gradient`：用于首屏或 CTA 的渐变（如 `linear-gradient(135deg,#667eea,#764ba2)`）。
- `theme.font_heading / font_body`：标题与正文中文字体建议。
- `sections[].order`：从 1 开始的顺序号，决定页面从上到下的排列。
- `sections[].type`：上面列出的区块类型之一。
- `sections[].heading`：该区块主标题。
- `sections[].copy`：该区块的文案要点（对象，含 subheading / items[] / cta_button 等，按区块类型灵活组织；**事实性内容取自 Context**）。
- `sections[].visual_intent`：视觉布局意图描述。
