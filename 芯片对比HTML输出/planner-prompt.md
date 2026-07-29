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
1. **区块数量充足且多为可交互**：根据主题规划 **5 ~ 9 个区块（section）**，让页面饱满。**至少 3-4 个区块要是可交互的**，让用户能点能玩，而不是只能滚动浏览。可选区块类型（含交互类）：
   - **展示类**：`hero`（首屏）、`logos`/`trust`（信任背书）、`features`（特性）、`showcase`（案例）、`steps`（流程）、`testimonials`（评价）、`pricing`（价格）、`cta`（号召行动）、`footer`（页脚）。
   - **★交互类（多选几个，让页面可玩）★**：`tabs`（标签页切换）、`interactive_faq`（可折叠问答）、`gallery_filter`（可筛选图库）、`filterable_list`（搜索/筛选列表）、`sortable_list`（拖拽排序）、`calculator`（计算器，如价格估算/BMI/贷款）、`before_after`（前后对比拉杆）、`color_playground`（配色演示）。
   你不必全部使用，按主题挑选最合适的组合，**但务必让页面有交互**。
2. **每个区块给足写作素材**：`copy` 字段要写出**真实、具体**的中文文案要点（标题、若干条要点、按钮文字等），**事实部分取自 Context**。交互类区块的条目要带可交互属性（如 `category` 用于筛选、`tabs` 分组）。
3. **明确交互行为**：每个 section 的 `interaction` 字段必须**具体描述**该区块的交互（如"点击 tab 按钮切换面板""输入框实时过滤""拖拽重排""滑块重算"），纯展示区块填"无(纯展示)"。Generator 会严格据此实现。
4. **明确视觉意图**：`visual_intent` 写清该区块的视觉处理（如"左文右图卡片网格""三栏卡片带图标"），Generator 据此落地布局。
5. **★配色多变规划（每次生成不同）★**：`theme` 里：
   - `base_hue`：**随机生成一个 0-359 的整数**作为基准色相（每次不同）。
   - `color_strategy`：从 analogous/triadic/complementary/split_complementary/tetradic/monochrome 里**随机选一个**派生策略（每次不同）。
   - `mood`：按主题选一个调性（vibrant/calm/professional/playful/elegant/tech/warm/cool）。
   - `colors`：**基于 base_hue 与 color_strategy 派生**出 primary/secondary/accent/background/surface/text/muted（用 HSL 表达，如 `hsl(210 70% 50%)`），不可套用固定模板。
   - `dark_mode`：给出一组同色相、降明度的暗色覆盖色。
   - `gradient`：用 primary→accent 派生的渐变。

【输出要求】
严格按照给定的 JSON Schema 输出，不要输出任何 HTML、不要输出任何解释性文字。文案内容必须**贴合「用户需求」且事实部分**依据 Context**，而非通用模板或编造内容。配色每次都要随机派生、互不相同。

【字段填写说明】
- `theme.brand_name`：网站/品牌名称（**取自 Context**；若 Context 无，按用户主题起一个）。
- `theme.tagline`：一句话品牌标语（贴合主题）。
- `theme.base_hue`：0-359 随机整数基准色相（每次不同）。
- `theme.color_strategy`：从 analogous/triadic/complementary/split_complementary/tetradic/monochrome 里随机选一个派生策略（每次不同）。
- `theme.mood`：按主题选一个调性（vibrant/calm/professional/playful/elegant/tech/warm/cool），影响饱和度/明度。
- `theme.colors.*`：基于 base_hue + color_strategy 派生的 HSL 颜色（primary/secondary/accent/background/surface/text/muted，必填前 5 个）。
- `theme.dark_mode`：暗色覆盖色（至少给 background/surface/text 三个，同色相降明度）。
- `theme.gradient`：用 primary→accent 派生的渐变。
- `theme.font_heading / font_body`：标题与正文中文字体建议。
- `sections[].order`：从 1 开始的顺序号，决定页面从上到下的排列。
- `sections[].type`：上面列出的区块类型之一（鼓励交互类）。
- `sections[].heading`：该区块主标题。
- `sections[].copy`：该区块的文案要点（对象，**至少填一个字段**，含 subheading / items[] / cta_button 等，按区块类型灵活组织；**事实性内容取自 Context**）。
- `sections[].interaction`：★必填★ 该区块的交互行为具体描述（如"点击 tab 切换面板""拖拽重排""滑块重算"），纯展示区块填"无(纯展示)"。
- `sections[].visual_intent`：视觉布局意图描述。
