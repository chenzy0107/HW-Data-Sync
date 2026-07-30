# Planner Prompt（LLM-1 · 规划结构，不出代码）

> 开结构化输出(粘generator-schema.json)；**用小模型(glm-4.5-air/glm-4-flash)提速**；temp 0.3；**max_tokens限1500**(强制精简输出，避免长耗时)；Context绑定知识检索result；变量 {{user_requirement}}=Start输入。

---

任务是规划页面结构，不写HTML代码。

【用户需求】
{{user_requirement}}

【Context】已注入知识库检索资料(产品功能/价格/特色等)，规划文案须依据它。

【任务】设计单屏紧凑高密度的交互工具页面——核心交互与结果用多列网格/卡片墙高密度排布，一屏塞下尽可能多的可操作内容，几乎不用滚动。思考：该主题下用户最想做什么操作、得什么结果，规划成紧凑的多区块网格，而非竖排长页。

【★防幻觉★】copy字段事实内容(产品名/功能/价格/数据)须取自Context，禁编造；Context未覆盖的用通用描述(如"联系获取报价")并标注，绝不编造数字。结构/布局/配色不受Context约束。

【规划原则】
1. ★紧凑高密度(最重要，杜绝一拉到底)★：核心是**多列网格/卡片墙**——多个交互区块(勾选器/推荐/统计/计算器)并排成2-4列网格塞满一屏，数据密度高，操作时相关区块实时联动，不需滚动。严禁竖排堆叠。
2. ★只规划3-4个区块★(多了输出慢)：典型 topbar(汇总/总价)→grid(勾选器+推荐+统计多列并排)→full(对比表/计算器)→collapsible(FAQ)。hero最多半屏高或省略。
3. layout_role：topbar(吸顶)/grid(多列卡片墙)/full(整行)/collapsible(折叠)。
4. ★强交互必选3+个★：checkbox_selector(item带selected_default/impact)、recommender(按recommend_rules算+理由)、comparison_table(item带score_axes)、stepper_wizard/configurator/calculator按主题选。
5. copy写真实具体中文要点(事实取自Context)，交互项带属性(勾选带selected_default/impact，推荐带recommend_condition，对比带score_axes)。
6. interaction必具体(如"勾选实时更新推荐和顶部总价")并说明与哪个区块联动。
7. ★配色多变★：base_hue随机0-359；color_strategy从analogous/triadic/complementary/split_complementary/tetradic/monochrome随机选；colors基于base_hue+color_strategy用HSL派生(primary/accent/background/text必填)；gradient用primary→accent派生。

按JSON Schema输出，不输出HTML/解释。**输出尽量精简**（只填必要字段，items每区块3-5个即可，描述简短），事实依据Context，配色每次随机。

【字段说明】
theme: brand_name(品牌名)；base_hue(0-359随机)；color_strategy(随机策略)；colors(primary/accent/background/text用HSL派生)；gradient(派生渐变)。
sections[]: layout_role(★必填★topbar/grid/full/collapsible)；type(区块类型，至少3个强交互)；heading(主标题)；copy(文案对象，至少填一字段，含items/recommend_rules/compare_axes/cta_button)；interaction(★必填★交互行为+联动说明)。
