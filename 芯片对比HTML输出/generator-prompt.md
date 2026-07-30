# Generator Prompt（LLM-2 · 核心）

> 关结构化输出；temp 0.6~0.7；max_tokens 拉满；Context 绑定知识检索 result；变量 {{user_requirement}}={{Start输入}}、{{planner_output}}=Planner输出。

---

【用户需求】
{{user_requirement}}

【页面结构大纲（必须严格遵守）】
{{planner_output}}

【Context】已注入知识库检索资料，作为事实来源。

【任务】按大纲生成单屏紧凑高密度的交互工具页面，严格按每个 section 的 `layout_role` 落地布局。

【★紧凑高密度（最高优先级，严禁一拉到底）★】
- 严禁竖向堆叠长页面。布局按 layout_role：`topbar`吸顶(sticky)放汇总/总价；`grid`核心交互区用**多列卡片墙**(2-4列grid/flex)把勾选器/推荐/统计/计算器等高密度并排塞满一屏，信息密度高、不浪费空间；`full`整行宽(对比表等需宽度的)；`collapsible`默认折叠。
- 核心区控制在一屏内，**几乎不用滚动**；操作时相关区块**实时联动**(勾选→推荐/汇总/统计立刻变，联动是核心)。
- **紧凑技巧**：用密集的小卡片/紧凑间距(gap-2/gap-3)、多列网格、小字号高信息量、卡片内塞数据/标签/图标，一屏呈现尽量多可操作内容。

【★事实遵循·防幻觉★】
1. 事实类内容(产品名/功能/价格/规格/数据/公司信息)必须严格依据 Context，不得编造臆测。
2. Context 未覆盖的事实，用不涉及具体数字的通用描述(如"联系我们获取报价")，绝不编造数字。
3. 可润色文案但不得改变事实含义或添加未提及特性。
4. 版式/动效/图标可自由发挥，但配色必须严格依据大纲 theme 派生。

【硬性要求】
1. 输出两部分(无分隔标记)：开头≤500字核心功能说明(列举页面核心功能)，紧接 ```html 代码块。HTML须按此完整结构(标签都要闭合)：
```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0">
<script src="https://cdn.tailwindcss.com"></script>
<style>:root{--c-primary:hsl(...);--c-accent:hsl(...);--c-bg:...;--c-text:...}</style>
</head>
<body>
（topbar/grid/full/collapsible 各 section 按 layout_role 布局，grid 区用多列卡片墙紧凑排布，此处省略具体内容）
<script>所有交互JS写在这里(事件委托、内存状态、recommend()纯函数)</script>
</body>
</html>
```
2. 逐区块实现，一个都不能少，顺序/类型/文案与大纲对应。
3. 内容真实饱满(多卡片/列表/图标/数据)，严禁占位符("…"/[content here]/Lorem等)。
4. 配色：严格依据 theme.base_hue 与 color_strategy 派生(以base_hue为H0，按策略角度算secondary/accent，同色相±明度造灰阶)，用HSL写CSS变量(--c-primary/--c-accent/--c-bg/--c-surface/--c-text/--c-muted)，只生成单一主题。标题强对比(font-black text-5xl)，仅PC端弹性填充(max-w-[1400px] mx-auto，不写md:/lg:断点)，图标用内联SVG/Lucide。
5. ★工具型强交互(至少3个)★：
   - checkbox_selector勾选器：勾选实时联动已选汇总/累计impact，状态存内存数组。
   - recommender智能推荐：依赖勾选结果，按recommend_rules算Top推荐+理由，放grid区与勾选器并排。
   - comparison_table对比表：多方案×compare_axes横评，支持勾选对比项/点表头排序/高亮最佳，数据取score_axes。
   - stepper_wizard/configurator：分步选(wizard_steps)最后给结论，或选项实时改预览/价格。
   - calculator：输入实时算。
   - 常规：顶部横向导航/锚点滚动/滚动渐入/回顶；tab切换；手风琴；筛选搜索(data-category)；拖拽排序(Drag API)；前后对比拉杆。
6. 技术约束：仅依赖Tailwind Play CDN(+可选Lucide)，无第三方库；CSS/JS全内联。

【交互硬约束】
- ✅状态存JS内存变量；推荐用纯函数 recommend(selected)。
- ❌严禁localStorage/sessionStorage/IndexedDB/cookie(iframe sandbox会抛SecurityError)；❌严禁alert/confirm/prompt(用toast)；表单preventDefault；用事件委托。

【禁令】
- ✅输出两部分：功能说明+紧接的```html代码块。
- ❌禁任何**注释**(HTML的`<!-- -->`、JS的`/* */`和`//`注释)；注意区分：JS正则(如/^\d+$/)、URL(https://)、字符串内的这些**不是注释，必须保留**，勿误删。
- ❌思考过程绝对禁止输出代码(只用自然语言规划，不贴标签/片段)；✅正确思考："勾选器+推荐+统计用3列卡片墙并排，base_hue=210三色派生，推荐按标签匹配度排序" ❌错误："`<div class='grid'>…querySelectorAll()…`"。
- ❌功能说明禁超500字、禁分隔标记、禁编造事实。
- ❌不省略、不截断。

先写核心功能说明(≤500字)，紧接 ```html 完整HTML。全程无注释，事实依据Context。
