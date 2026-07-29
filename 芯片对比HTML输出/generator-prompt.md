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
4. **版式与视觉创意可自由发挥**：版式布局、动效、图标、组件组合不受 Context 约束。但**配色必须严格依据大纲 `theme` 派生**（见下方「设计要求」），不要脱离 base_hue/color_strategy 自由改色。

【硬性要求 · 必须全部满足】

1. **输出分两段**（先总结后 HTML，见下方「输出结构」）：第一段是 ≤500 字中文设计说明，第二段是被 ```html 代码块包裹的单个完整 HTML 文件。HTML 整体结构如下（注意：代码块内外均不得出现任何注释）：
   ```html
   <!DOCTYPE html>
   <html lang="zh-CN" data-theme="light">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>...</title>
     <script src="https://cdn.tailwindcss.com"></script>
     <script>tailwind.config = { theme: { extend: { colors: { brand: 'var(--c-primary)', accent: 'var(--c-accent)' }, fontFamily: {...} } } }</script>
     <style>
       :root{
         --c-primary: hsl(...); --c-secondary: hsl(...); --c-accent: hsl(...);
         --c-bg: ...; --c-surface: ...; --c-text: ...; --c-muted: ...;
       }
       :root[data-theme="dark"]{
         --c-bg: ...; --c-surface: ...; --c-text: ...;
       }
     </style>
   </head>
   <body>
     <button id="theme-toggle">🌙</button>
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
   - **★配色多变（每次生成不同，禁止套用固定模板）★**：
     - 严格依据大纲 `theme.base_hue`（基准色相）与 `theme.color_strategy`（派生策略：analogous/triadic/complementary 等）**派生**所有颜色，不可凭空套一套固定配色。
     - 派生算法：以 `base_hue` 为 H0，按策略角度（triadic 用 +120/+240、complementary 用 +180、analogous 用 ±15 等）算出 secondary/accent 的色相；同色相内用 ±明度造中性灰阶（背景/卡片/边框）。推荐用 HSL 写 CSS 变量，如 `--c-accent: hsl(285 70% 55%)`。
     - 所有颜色通过 `:root` 的 CSS 变量下发（`--c-primary/--c-accent/--c-bg/--c-surface/--c-text/--c-muted`），Tailwind config 与自定义样式统一引用。
     - **必须实现亮/暗双主题**：`:root[data-theme="dark"]` 覆盖一组（同色相、降低明度），页面右上角放一个**一键切换按钮**（toggle `document.documentElement.dataset.theme`）。
     - 首屏或 CTA 使用 `theme.gradient`。
   - 强对比的标题字重（如 `font-black text-4xl md:text-6xl`）、合理的圆角、阴影、留白。
   - 全程响应式（`md:` / `lg:` 断点），移动端单列、桌面端多列。
   - 图标用内联 SVG 或 Lucide CDN。

5. **★丰富的页面级交互（让用户可点可玩，不是只能看）★**：
   - 每个区块都要严格实现大纲里 `interaction` 字段描述的交互行为，**一个都不能漏**。
   - **必须实现的交互清单**（按大纲区块类型对应落地）：
     - **导航类**：移动端汉堡菜单、平滑锚点滚动（`scrollIntoView({behavior:'smooth'})`）、滚动渐入（IntersectionObserver）、回到顶部按钮。
     - **tab 标签页**：点击 tab 按钮切换对应面板（toggle `.active` class）。
     - **手风琴/可折叠问答**：点击标题展开/折叠答案（`max-height` 过渡）。
     - **筛选/搜索**：分类 chips 点击过滤带 `data-category` 的项；搜索框 `input` 事件实时 `textContent.includes()` 过滤。
     - **可拖拽排序**：用原生 HTML5 Drag API（`dragstart/dragover/drop`）或 pointer 事件实现卡片重排。
     - **计算器**：表单输入 `input` 事件实时重算并显示结果（贷款/BMI/价格估算等，按大纲 `calc_config`）。
     - **前后对比拉杆**：拖动滑块对比两张图/两种状态（`clip-path` 或 `width` 联动）。
     - **暗色模式切换**：见上。
     - 按钮 hover/active 过渡、卡片悬浮上浮、点赞计数、复制到剪贴板（`navigator.clipboard`）等微反馈。
   - **交互实现硬约束（违反会导致运行报错，务必遵守）**：
     - ✅ 状态一律存在 **JS 内存变量**（闭包内变量）。
     - ❌ **严禁使用 `localStorage` / `sessionStorage` / `IndexedDB` / `document.cookie`**——页面在 `iframe sandbox` 下运行，这些 API 会抛 `SecurityError`。需要"记住"的就用内存变量。
     - ❌ 严禁用 `alert()` / `confirm()` / `prompt()`——用页面内 toast/浮层替代。
     - 表单提交一律 `e.preventDefault()` 在前端处理，不要真提交跳转。
     - 用**事件委托**（在父元素监听）统一处理动态元素的事件。
   - 所有交互 JS 内联在页面底部的 `<script>` 里，无外部 JS 文件。

6. **技术约束**：
   - 仅依赖 Tailwind Play CDN（+ 可选 Lucide CDN），不引入其它第三方库，不使用本地资源，不需要构建。
   - 所有 CSS / JS 内联在文件内。
   - 中文 `lang="zh-CN"`，正文用清晰的中文字体栈。

【输出结构 · 必须严格遵守】

你的完整输出由**两段**组成，按以下**精确格式**输出（注意分隔标记行必须**独占一行、原样输出**，系统靠它拆分两段）：

````
===SUMMARY===
（这里是一段不超过 500 字的中文设计说明，纯文本，不要 markdown 标题/列表符号，不要代码块包裹）
===HTML===
```html
<!DOCTYPE html>
...完整 HTML...
</html>
```
````

格式细则：
1. 第一行必须是 `===SUMMARY===`（独占一行）。
2. 紧跟其后的若干行是**设计说明**（≤500 字中文，见下方「设计说明」要求）。
3. 然后一行必须是 `===HTML===`（独占一行）。
4. 然后是 ```html 代码块（以 ```html 开头、以 ``` 结尾），里面是完整 HTML。
5. 除上述两段及分隔标记外，**不输出任何其它内容**（无前言、无结语）。

【设计说明（===SUMMARY=== 段）要求 · 设计说明型】
写一段**不超过 500 字**的中文设计说明，帮助用户理解这次生成的页面。内容应包含：
- 这个页面**做了什么**：主题、定位、面向人群。
- 包含了**哪些主要区块**及其作用（如首屏突出核心卖点、特性区列出 N 大功能、价格区给出 X 档方案）。
- **设计思路**：配色与视觉风格的选择理由、布局逻辑、关键的交互/动效设计。
- 若事实内容依据了 Context 知识库，简要说明**采用了哪些关键资料**（如产品名、核心功能、价格档位）。
要求：自然流畅的中文段落，**不要**用 markdown 标题或列表符号（用顿号、分号组织即可），**不要**超出 500 字。

【绝对禁令】
- ✅ **必须按上述 `===SUMMARY===` / `===HTML===` 结构输出**，两个分隔标记行原样独占一行。
- ✅ HTML 部分**必须用 ```html 代码块包裹**（以 ```html 开头、以 ``` 结尾）。
- ❌ **HTML 代码块内外禁止任何注释**（`<!-- -->`、`/* */`、`//` 一律不许出现，哪怕一行也不行）。
- ❌ 总结段**禁止**用 markdown 代码块包裹、禁止超出 500 字。
- ❌ **禁止编造 Context 中没有的事实性数据**（价格、规格、功能、公司信息等）。
- ❌ 不要省略任何区块或内容，不要使用任何占位符。
- ❌ 不要在代码块中间截断——必须写完并闭合代码块。

现在开始输出：先 `===SUMMARY===` 和设计说明，再 `===HTML===` 和被 ```html 包裹的完整 HTML。把大纲里的每一个区块都写完整、写丰富，事实性内容严格依据 Context，全程不得包含任何注释。
