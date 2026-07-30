# Planner Prompt（LLM-1 · 规划结构，不出代码）

> 开结构化输出(粘generator-schema.json)；temp 0.3；Context绑定知识检索result；变量 {{user_requirement}}=Start输入。

---

任务是规划页面结构，不写HTML代码。

【用户需求】
{{user_requirement}}

【Context】已注入知识库检索资料(产品功能/价格/特色等)，规划文案须依据它。

【任务】设计单屏紧凑高密度的交互工具页面——核心交互与结果用多列网格/卡片墙高密度排布，一屏塞下尽可能多的可操作内容，几乎不用滚动。思考：该主题下用户最想做什么操作、得什么结果，规划成紧凑的多区块网格，而非竖排长页。

【★防幻觉★】copy字段事实内容(产品名/功能/价格/数据)须取自Context，禁编造；Context未覆盖的用通用描述(如"联系获取报价")并标注，绝不编造数字。结构/布局/配色不受Context约束。

【规划原则】
1. ★紧凑高密度(最重要，杜绝一拉到底)★：核心是**多列网格/卡片墙**——多个交互区块(勾选器/推荐/统计/计算器)并排成2-4列网格塞满一屏，数据密度高、信息量大，不浪费空间。操作时相关区块实时联动，不需滚动。严禁竖排堆叠的线性宣传页。
2. ★每section必填layout_role★：topbar(吸顶汇总/总价/导航，始终可见)；grid(紧凑网格区，核心交互用多列卡片墙，2-4列并排)；full(整行宽，对比表/计算器等需宽度的)；collapsible(可折叠，FAQ/详情默认收起)。
3. 典型结构：topbar(汇总/总价)→grid(勾选器+推荐+统计等多列并排)→full(对比表/计算器)→collapsible(FAQ)。可选半屏高hero。总区块4~6个。
4. ★强交互类型(必选3+个)★：checkbox_selector(勾选器，item带selected_default/impact)；recommender(智能推荐，配勾选器，按recommend_rules算+理由)；comparison_table(对比表，item带score_axes，放full)；stepper_wizard/configurator/calculator按主题选。
5. 常规交互(按需)：tabs/interactive_faq(放collapsible)/filterable_list/before_after/quiz。
6. 展示类极简或省略：hero最多半屏高；features/testimonials/cta尽量并入交互或省略；footer可省。宁可少区块也要保证紧凑核心区完整。
7. copy写真实具体中文要点(事实取自Context)，交互项带属性(勾选带selected_default/impact，推荐带recommend_condition，对比带score_axes)。
8. interaction必具体(如"勾选实时更新推荐和顶部总价""点表头排序")并说明与哪个区块联动，纯展示填"无(纯展示)"。
9. visual_intent写清布局(如"勾选器+推荐+统计三列卡片墙并排，顶部吸顶汇总")，与layout_role一致。
10. ★配色多变★：base_hue随机0-359整数；color_strategy从analogous/triadic/complementary/split_complementary/tetradic/monochrome随机选；mood按主题选(vibrant/calm/professional/playful/elegant/tech/warm/cool)；colors基于base_hue+color_strategy用HSL派生(primary/secondary/accent/background/surface/text/muted，必填前5)，单一主题无dark_mode；gradient用primary→accent派生。

按JSON Schema输出，不输出HTML/解释。文案贴合需求且事实依据Context。配色每次随机派生。

【字段说明】
theme.brand_name(Context取或按主题起)；tagline(标语)；base_hue(0-359随机)；color_strategy(随机策略)；mood(调性)；colors.*(HSL派生，必填前5)；gradient(派生渐变)；font_heading/font_body。
sections[].order(顺序号)；id(锚点id英文小写下划线，唯一)；layout_role(★必填★布局角色topbar/grid/full/collapsible)；type(区块类型，鼓励交互类)；heading(主标题)；copy(文案对象，至少填一字段)；interaction(★必填★交互行为+联动说明)；visual_intent(布局意图)。
