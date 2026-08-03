Generator Prompt

任务：据 Planner 大纲与检索事实，生成**能在浏览器直接、完整渲染**的对标 Gemini Canvas 水准的单页交互工具 HTML。角色与硬性契约见 SYSTEM，你负责把 Planner 的结构真正落地为达标的页面。

【最高目标】
产出的 HTML **必须能渲染**：Tailwind 样式生效、所有 JS 不报错、交互真能联动。任何取舍下"能渲染"优先级最高——宁可少一个动效，也不白屏/JS 崩溃/裸奔。

【契约：6 条成功标准须逐条兑现（与 SYSTEM 同源）】
1. 密度：每区块 ≥6 数据单元（据 Planner 大纲与 Context 展开真实/示例数据），一屏饱满，无大片空白/空 section（观感不空洞）。
2. 可操作：≥3 个联动交互且 JS 真能跑（点了对结果有影响，非死按钮）；联动逻辑写成纯函数（如 `recommend(selected)`），状态存 JS 内存变量，选项变化实时改结果。
3. 惊艳：质感排版 + 阴影/渐变/玻璃态 + 每次不同配色；含 ≥1 玻璃态/渐变背景 + ≥1 动效 + 字体三级层级。
4. 清晰：文字 ≥14px、对比度足、选中态不遮挡；图表/对比表真有数据（非空壳）。
5. 完整：到 `</html>` 无占位符、无"…"、无截断。
6. 可信：真实数据严格来自 Context、示例已视觉标注且未与真实混排、含演示数据免责声明。

【用户需求】{{userinput.query}}
【页面结构大纲（参考，可有限优化）】{{planner_output}}
【Context】已注入知识库检索资料，作为事实来源。

【优化边界】
- 允许：调整视觉风格、配色、排版、动效以更惊艳。
- 禁止：删减 Planner 已规划的区块数、改动 Context 事实、低于 6 条成功标准任一硬下限、违反下方「可渲染 HTML 骨架」。

【可渲染 HTML 骨架（页面能否渲染的根本）】
**严格按 SYSTEM【技术骨架】的 8 条硬性顺序约束与骨架模板输出**，此处不重复全文，只重申 3 条最易崩的渲染铁律（违反任一页面必白屏/JS 失效）：
1. Tailwind CDN `<script src="https://cdn.tailwindcss.com"></script>` 必须在 `<head>` 内、所有自定义脚本之前（漏了 → 所有 Tailwind class 失效 → 裸奔）。
2. 所有自定义 JS 集中在 `</body>` 之前单个 `<script>`，整体被 `document.addEventListener('DOMContentLoaded', () => { ... })` 包裹（漏了 → 元素未渲染查询到 null → JS 第一行就崩 → 所有交互失效）。
3. 状态存 JS 内存变量，禁用 `localStorage`/`sessionStorage`/`IndexedDB`/`cookie`/`alert` 等沙箱敏感 API（用了 → SecurityError → JS 中断）。

【数据策略（密度优先，细则见 SYSTEM）】事实取自 Context 不衍生数字；缺覆盖用标注"示例"的结构化数据；示例须可视化标注、禁混排；页面含免责声明。

【运行安全（违反会崩）】禁用 `localStorage`/`sessionStorage`/`IndexedDB`/`cookie`/`alert`/`confirm`/`prompt`（改用内存变量 + 页面内 toast）；禁用调试语句（`console.*`/`debugger`）与调试文字（test/TODO/调试中）；思考过程不进页面。

【输出格式】两部分**无分隔**、连续输出：
1. 开头 **≤200 字**核心功能说明（纯文本，列举页面功能）；
2. 紧接 ```html 代码块，内容是上述可渲染骨架的完整 HTML（`<!DOCTYPE html>` … `</html>`）。

代码块必须以 `<!DOCTYPE html>` 开头、以 `</html>` 结尾、以闭合的 ``` 收尾，不夹带任何解释文字。

<自检>
【生成前自检（关闭 thinking 后必须逐项打勾，任一不达标即先修正再写）】
- [ ] 密度：4-6 区块、每区块 ≥6 数据单元，无大片空白/空 section（观感不空洞）
- [ ] 可操作：≥3 联动交互且 JS 真能跑（非死按钮）
- [ ] 惊艳：质感 + 每次不同配色、有层次；含 ≥1 玻璃态/渐变背景 + ≥1 动效 + 字体三级层级
- [ ] 清晰：文字 ≥14px、对比度足、选中态不遮挡
- [ ] 完整：到 `</html>` 无占位符/无"…"/无截断
- [ ] 可信：真实数据来自 Context、示例已标注且未混排、含免责声明
- [ ] ★可渲染（核心）：Tailwind CDN 在 head 最前；自定义 JS 全在单个 `<script>` 且整体被 DOMContentLoaded 包裹；无沙箱敏感 API（localStorage/sessionStorage/IndexedDB/cookie/alert 等）；无内联 onclick；无注释、无调试代码
- [ ] ★闭合：代码块以 `<!DOCTYPE html>` 开头、以 `</html>` 结尾、闭合 ``` 收尾
</自检>

内容真实饱满，不写占位符。先写功能说明，紧接 ```html 完整 HTML，事实依据 Context。
