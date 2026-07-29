# Generator Prompt（LLM-2 · 核心 · 输出完整 HTML · 基于 RAG 事实）

> 配置要点：本节点**关闭「结构化输出」**（要输出纯 HTML，不是 JSON）。`temperature` 设为 **0.6 ~ 0.7**。`max_tokens` **尽量拉满**（GLM-4.6 可设到上限，单页富 HTML 通常 8K~20K token，务必给足避免截断）。模型 GLM-4.6 / GLM-4.5。
>
> ★RAG 注入（关键）：本节点绑定 **Context = 知识检索节点的 `result`**（在 LLM 节点的 Context 字段里选「知识检索 → result」）。Dify 会自动把检索到的知识库片段序列化进上下文，并启用引用追踪。
>
> 变量：
> - `{{user_requirement}}` → Start 节点输入
> - `{{planner_output}}` → Planner 节点的结构化输出
> - 检索到的事实知识 → 通过 **Context 字段**自动注入（不要在 prompt 里再手写 `{{retrieval.result}}`，Context 已处理）

---

## USER Prompt（直接复制下面的内容到 LLM-2 的用户消息）

【用户原始需求】
{{user_requirement}}

【页面结构大纲（必须严格遵守）】
{{planner_output}}

【可用的知识库事实（Context）】
系统已通过 Context 为你注入了从私有知识库检索到的相关资料（产品功能、价格、特色等）。这是**权威事实来源**。

【你的任务】
基于上面的**页面结构大纲**，生成**一个完整、可直接运行、内容丰富、设计精美**的单页 HTML 网站。严格按大纲里 `sections` 的 `order` 顺序逐个实现每个区块。

【★事实遵循规则 · 防幻觉，必须遵守★】
1. **事实类内容必须严格依据 Context 检索结果**：产品名称、功能描述、价格、规格、数据、适用场景、公司/品牌信息等**事实性内容**，必须**逐字或贴近采用** Context 里给出的真实信息，**不得编造、不得臆测、不得张冠李戴**。
2. **Context 没覆盖的事实，禁止捏造**：若大纲要求某个事实（如某个价格、某项功能）但 Context 中**没有**相应资料，则用**通用、不涉及具体数字/承诺**的描述代替（如"联系我们获取专属报价""多档方案灵活可选"），**绝不编造具体数字**。
3. **润色允许、杜撰禁止**：你可以把检索到的事实润色成更具吸引力的营销文案，但不能改变其事实含义、不能添加未提及的特性或数据。
4. **风格与布局可自由发挥**：配色、版式、动效、图标、视觉创意不受 Context 约束，尽情发挥设计能力。

【硬性要求 · 必须全部满足】

1. **必须是单个完整 HTML 文件**，并被 ```html 代码块包裹。整体结构如下（注意：代码块内外均不得出现任何注释）：
   ```html
   <!DOCTYPE html>
   <html lang="zh-CN">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>...</title>
     <script src="https://cdn.tailwindcss.com"></script>
     <script>tailwind.config = { theme: { extend: { colors: {...}, fontFamily: {...} } } }</script>
     <style>:root{ --c-primary:...; --c-secondary:...; --c-accent:...; }</style>
   </head>
   <body>
     <section>...</section>
     <section>...</section>
     <footer>...</footer>
     <script></script>
   </body>
   </html>
   ```

2. **逐区块实现，一个都不能少**：大纲里有几个 `sections`，你就必须输出几个 `<section>`，顺序、类型、文案与大纲完全对应。**严禁省略任何区块**。

3. **内容必须真实且饱满**：
   - 事实性文案**取自 Context**（按上面的事实遵循规则）。
   - 每个 `<section>` 内部要有足够的子元素（多张卡片、多个列表项、图标、数据、描述文字），让页面看起来内容充实、专业。
   - **绝对禁止**出现"示例文字""待补充""Lorem Ipsum""……""[content here]"等任何占位符或省略。

4. **设计要求（对标 v0 / Lovable 的水准）**：
   - 配色严格使用 `theme.colors`，通过 Tailwind config 与 CSS 变量统一；首屏或 CTA 使用 `theme.gradient`。
   - 强对比的标题字重（如 `font-black text-4xl md:text-6xl`）、合理的圆角、阴影、留白。
   - 至少包含这些细节：移动端汉堡菜单、按钮 hover 过渡、卡片悬浮上浮、首屏滚动渐入(IntersectionObserver)、数字计数动画（如有数据统计区块）、平滑锚点滚动。
   - 全程响应式（`md:` / `lg:` 断点），移动端单列、桌面端多列。
   - 图标用内联 SVG 或 Lucide CDN。

5. **技术约束**：
   - 仅依赖 Tailwind Play CDN（+ 可选 Lucide CDN），不引入其它第三方库，不使用本地资源，不需要构建。
   - 所有 CSS / JS 内联在文件内。
   - 中文 `lang="zh-CN"`，正文用清晰的中文字体栈。

【绝对禁令】
- ✅ **必须用 ```html 代码块包裹整个输出**（以 ```html 开头、以 ``` 结尾）。
- ❌ **代码块内外禁止任何注释**（`<!-- -->`、`/* */`、`//` 一律不许出现，哪怕一行也不行）。
- ❌ **禁止编造 Context 中没有的事实性数据**（价格、规格、功能、公司信息等）。
- ❌ 不要输出任何解释、前言、结语（如"这是为您生成的网站"）。
- ❌ 不要省略任何区块或内容，不要使用任何占位符。
- ❌ 不要在代码块中间截断——必须写完并闭合代码块。

现在，用一个 ```html 代码块包裹，输出这个完整的 HTML 文件（从 `<!DOCTYPE html>` 开始，到 `</html>` 结束，再闭合代码块）。全程不得包含任何注释。把大纲里的每一个区块都写完整、写丰富，事实性内容严格依据 Context。
