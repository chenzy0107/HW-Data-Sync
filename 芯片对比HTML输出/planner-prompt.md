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
基于上面的需求，设计一个**工具型交互网站**的结构——不是一拉到底的宣传页，而是用户能勾选、对比、得到推荐/结果的决策工具。你要像一个资深产品设计师那样思考：这个主题下，用户最想**做什么操作、得到什么结果**？围绕这个核心交互来规划区块。视觉风格与配色如何？

【★事实遵循规则 · 防幻觉，必须遵守★】
1. **文案要点必须依据 Context 真实资料**：`copy` 字段里写的产品名、功能、价格、特色、数据等**事实性内容**，必须**取自 Context** 检索结果，**不得编造**。
2. **Context 没有的内容留白处理**：若某个区块需要某类事实（如价格）但 Context 里**没有**，则在 `copy` 里写"用通用/不涉及具体数字的描述"（如"联系我们获取专属报价"），并在 `visual_intent` 或备注里标注"Context 无此数据，已用通用描述"，**绝不编造具体数字**。
3. **结构可自由设计**：区块的选择、数量、视觉布局不受 Context 约束，按主题自由规划最佳结构。

【规划原则】
1. **★工具型优先，杜绝一拉到底（最重要的原则）★**：你规划的不能是一串"首屏→特性→价格→FAQ→页脚"的线性宣传页。要规划成一个**用户能操作、能决策的交互工具**。根据主题规划 **5 ~ 8 个区块**，其中**至少 3 个必须是强交互区块**。
2. **★强交互区块类型（必选 3+ 个，按主题挑最合适的组合）★**：
   - `checkbox_selector`（**勾选器，强烈推荐必选**）：一组可勾选的需求/选项卡片（如"你需要哪些功能？""你的团队规模？"），用户勾选时实时联动。每个 item 带 `selected_default`（是否默认勾选）和 `impact`（勾选影响，如"+¥200/月""适合团队"）。
   - `recommender`（**智能推荐，与勾选器配对必选**）：依赖勾选结果，按 `recommend_rules`（如"匹配标签最多的方案"）算出推荐方案并给理由。**必须紧邻勾选器之后**。
   - `comparison_table`（对比表）：多方案 × 多维度（`compare_axes`）横评，每个 item 带 `score_axes` 评分；支持勾选要对比的方案、点击排序、高亮最佳。
   - `stepper_wizard`（分步向导）：一步步选（`wizard_steps`），最后给结论/推荐。
   - `configurator`（配置器）：选项实时改变预览/价格（如选配产品）。
   - `calculator`（计算器）：输入实时算结果（贷款/BMI/ROI/价格估算）。
3. **常规交互区块（按需补充）**：`tabs`（标签页）、`interactive_faq`（折叠问答）、`gallery_filter`/`filterable_list`（筛选/搜索）、`sortable_list`（拖拽）、`before_after`（前后对比拉杆）、`quiz`（问答测试）。
4. **展示类区块（克制使用，1-3 个足矣）**：`hero`（首屏，必选）、`pricing`（价格，可与对比表二选一）、`features`、`testimonials`、`cta`、`footer`（必选）。**展示类区块越少越好**，把空间让给交互。
5. **★区块顺序与布局（避免一拉到底）★**：强交互区块要**紧跟首屏 hero**，让用户一进来就能操作；`recommender` 必须紧邻 `checkbox_selector`（勾选完立刻看到推荐）。可用 sticky 侧栏让"已选汇总"始终可见。典型结构：`hero → checkbox_selector → recommender → comparison_table/calculator → 少量展示 → footer`。
6. **每个区块给足写作素材**：`copy` 字段要写出**真实、具体**的中文文案要点（标题、若干条要点、按钮文字等），**事实部分取自 Context**。交互类区块的条目要带可交互属性（勾选项带 `selected_default`/`impact`、推荐项带 `recommend_condition`、对比项带 `score_axes`）。
7. **明确交互行为**：每个 section 的 `interaction` 字段必须**具体描述**该区块的交互（如"勾选卡片实时累计已选数和价格""点推荐按钮按匹配度排序方案""点击对比表表头排序"），纯展示区块填"无(纯展示)"。Generator 会严格据此实现。
8. **明确视觉意图**：`visual_intent` 写清该区块的视觉处理（如"左侧勾选卡片网格+右侧 sticky 已选汇总""分步进度条+大卡片选项"），Generator 据此落地布局。
9. **★配色多变规划（每次生成不同）★**：`theme` 里：
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
