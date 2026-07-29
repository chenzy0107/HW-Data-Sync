# Generator Prompt（LLM-2 · 核心 · 输出完整 HTML）

> 配置要点：本节点**关闭「结构化输出」**（要输出纯 HTML，不是 JSON）。`temperature` 设为 **0.6 ~ 0.7**。`max_tokens` **尽量拉满**（GLM-4.6 可设到上限，单页富 HTML 通常 8K~20K token，务必给足避免截断）。模型 GLM-4.6 / GLM-4.5。
>
> 变量：`{{planner_output}}` 引用上游 Planner 节点的结构化输出（在 Dify 里把 Planner 节点的输出变量拖进来）。`{{user_requirement}}` 引用 Start 节点的输入。

---

## USER Prompt（直接复制下面的内容到 LLM-2 的用户消息）

【用户原始需求】
{{user_requirement}}

【页面结构大纲（必须严格遵守）】
{{planner_output}}

【你的任务】
基于上面的**页面结构大纲**，生成**一个完整、可直接运行、内容丰富、设计精美**的单页 HTML 网站。严格按大纲里 `sections` 的 `order` 顺序逐个实现每个区块，**逐字采用**大纲里 `copy` 字段提供的文案（可润色但不得删减/泛化），并按 `visual_intent` 落地视觉布局，按 `theme` 统一配色与字体。

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
   - 每个区块的文案采用大纲 `copy` 的内容，并扩展为**完整的中文段落/列表**。
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
- ❌ 不要输出任何解释、前言、结语（如"这是为您生成的网站"）。
- ❌ 不要省略任何区块或内容，不要使用任何占位符。
- ❌ 不要在代码块中间截断——必须写完并闭合代码块。

现在，用一个 ```html 代码块包裹，输出这个完整的 HTML 文件（从 `<!DOCTYPE html>` 开始，到 `</html>` 结束，再闭合代码块）。全程不得包含任何注释。把大纲里的每一个区块都写完整、写丰富。
