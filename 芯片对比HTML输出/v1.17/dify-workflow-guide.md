# Dify Workflow 逐步配置指南

> 照着本文从上到下操作，约 15 分钟搭好主路线。所有 Prompt 在 `prompts/` 目录，直接复制。

---

## 0. 准备

1. 登录 Dify（Dify Cloud 或自托管）。
2. 「设置 → 模型供应商」接入 **GLM**：
   - 智谱开放平台或 Z.ai 申请 API Key。
   - 在 Dify 配置好 `glm-4.6`（或 `glm-4.5`、`glm-4-plus` 等可用模型名，以你账号实际可见为准）。
3. 进入「工作室 / 创建空白应用 → **工作流 (Workflow)**」。

---

## 主路线（9 节点 · 含私有 RAG 知识库 + 独立审查 + 自动修复）

> 拓扑：Start → LLM-0 Query Rewriter → 知识检索 → LLM-1 Planner → LLM-2 Generator(功能说明+HTML) → LLM-3 Review(独立审查,出JSON) → IF 条件分支(Review.passed?) →┬ true → Output(用 Generator 的 html)
>                                                                                                                  └ false → LLM-4 Fixer(据Review意见修复) → Output(用 Fixer 的 html)
>
> 节点编号对照：下面「节点 1~9」分别对应 Start、LLM-0、知识检索、LLM-1 Planner、LLM-2 Generator、LLM-3 Review、IF 条件分支、LLM-4 Fixer、Output。
>
> 💡 **Loop Engineering 档位 2（Review + 单次修复）**：三个独立 agent 各司其职——Generator 生成(作者)、Review 审查(审查者)、Fixer 修复(修复者)。生成者不能诚实审查自己、审查者不负责修,所以拆成三个独立视角。Review 输出结构化 JSON(passed + issues),条件分支据 passed 路由:通过则直接输出 Generator 的 HTML,不通过则交 Fixer 据 issues 修复后输出。**只修一次不回环**(避免死循环和"越改越坏";要迭代审查升级档位 3)。



### 节点 1：Start（开始）

- 在画布上点击 **「开始节点」**。
- **本工作流统一使用 Start 节点内置的 `userinput.query` 字段**（对话型 / Chatflow 应用自带；若你用的是 Workflow 且尚无该字段，则把唯一的文本输入字段命名为 `userinput.query` 即可），**无需再手动添加额外的输入字段**。
- 确认 Start 的输入变量名为 `userinput.query`，用户在那一句输入里描述想要的网站（如「用一句话描述你想要的网站」）。

> 之后所有节点（QueryRewriter / Planner / Generator）的 `{{userinput.query}}` 都引用这里——**全链路必须统一引用同一个字段名 `userinput.query`**，否则会报"变量不存在"。

### 前置：建好私有知识库

1. 进入 Dify 顶部「**知识库 (Knowledge)**」页 → 「创建知识库」。
2. 上传你的**产品/业务资料**（PDF / Word / TXT / 网页链接 / Markdown 均可），如产品手册、功能清单、价格表、公司介绍。
3. 选择分段策略（推荐「自动分段」或按需自定义），等**索引完成**（状态变成就绪）。
4. 记下这个知识库名称，后面「知识检索节点」要选它。

> 检索质量很大程度取决于知识库内容质量。建议：一份资料讲清一类事实（如单独一份「产品功能」「价格方案」），分段不要太长。

### 节点 2：LLM-0 Query Rewriter（查询改写）

在 Start 后面加一个 **「LLM」节点**，命名 `QueryRewriter`。这是**轻量节点**，用最便宜的小模型即可。

**① 模型**
- 选**小模型**：`glm-4-flash` 或 `glm-4.5-air`（无需大模型，省钱省时间）。

**② SYSTEM / USER**
- 粘贴 `prompts/query-rewriter-prompt.md` 里的 SYSTEM 与 USER 内容。
- `{{userinput.query}}` 绑定 Start 的 `userinput.query`。

**③ 结构化输出**
- **关闭**（输出纯文本检索词）。

**④ 参数**
- `temperature`：**0**（要稳定）。
- `max_tokens`：**512**（只输出几行查询词）。
- 关闭思考模式：若 GLM 支持深度思考，设 `thinking: {"type":"disabled"}` 提速（注意是嵌套对象，扁平的 `thinking: disabled` 无效，会被忽略）。

**⑤ 输出变量**
- 该节点的输出文本就是检索查询词，记下输出变量名（如节点的 `text` / `output`），下一步知识检索节点引用它。

> 验证：输入 `做个 AI 写作助手落地页`，应输出类似 `AI写作助手 产品功能 价格 特色 适用场景 目标用户` 的关键词。

### 节点 3：知识检索（Knowledge Retrieval）

在 QueryRewriter 后面加一个 **「知识检索 (Knowledge Retrieval)」节点**，命名 `KnowledgeRetrieval`。

**① Query（查询变量）**
- 选 **变量** 输入，绑定 **QueryRewriter 节点的输出**（上一步的检索查询词）。
- ⚠️ 不要直接绑 `userinput.query`（那是生成指令，检索效果差）。

**② 知识库（Dataset）**
- 勾选你**前置步骤建好的私有知识库**（可多选，节点会同时检索多个库再合并）。

**③ 检索设置**
- **检索模式**：推荐 **混合 (Hybrid)**（关键词+语义）；纯语义也可。
- **Top K**：建议 **5**（重排后返回的片段数）。
- **Score Threshold（分数阈值）**：建议 **0.5**（仅返回相似度 ≥ 阈值的结果，过滤无关召回；调高更严格、调低更宽松）。
- **Rerank（重排序）**：推荐开启——选 **加权分数 (Weighted Score)**（调语义/关键词权重）或接一个 **Rerank 模型**（如 Jina/Cohere，质量更好）。

**④ 输出**
- 节点输出变量是 **`result`**（一个**对象数组**，每元素含 `content / title / score / metadata`）。
- ⚠️ **不需要加 join 节点**！下一步直接把 `result` 绑定到 LLM 节点的 **Context 字段**，Dify 自动序列化为文本。

> 验证：点节点「运行」，看 `result` 里是否召回了你预期的资料片段；若召回不准，调 Query 改写词 / Top K / 阈值 / 知识库内容。

**⚠️ result 一致为空的排查 SOP（按概率排序）**

> 现象：`KnowledgeRetrieval.result` 数组长度为 0，下游 LLM 拿不到任何知识。按以下顺序排查，多数情况落在 ① 或 ④。

- **① 分数阈值过高（最常见）**：`Score Threshold` 设 0.5 时，若 Query 改写词与文档匹配度低，所有片段都会被过滤成空。→ 临时把阈值调到 **0**（关闭过滤）再运行；若能召回，说明是阈值问题，再逐步上调到"既能召回又不引入噪声"（通常 0.2~0.3，而非 0.5）。
- **② 知识库未就绪 / 索引未完成**：进「知识库」页，确认该库状态为**就绪**。索引中、索引失败、或文档仍在处理的库，检索必为空。
- **③ 检索节点没勾选知识库**：双击知识检索节点，确认「Dataset」里确实勾选了你建的库（至少勾一个；可多选但漏勾则空）。
- **④ Query 绑定错误 / QueryRewriter 输出空**：单独"运行" `QueryRewriter` 节点，确认它产出的是检索词文本（非空、非 JSON）。再确认检索节点的 **Query 变量**绑的是 `QueryRewriter 的输出`，而非误绑成 `userinput.query`（空）或绑错变量名。
- **⑤ 文档本身无文本**：打开知识库文档的「分段预览」，确认分段有实际 `content` 文本。扫描版 PDF / 纯图片无 OCR → 分段空 → 检索空。
- **⑥ Rerank / 向量库异常**：开启了 Rerank 但 Rerank 模型未配置或报错时，节点可能返回空或报错。先**关闭 Rerank** 验证；若节点显示红色报错而非单纯空结果，按报错修（多半是 Rerank/向量库连接问题）。
- **⑦ Embedding 模型缺失/不可用**：知识库创建时若未配置 Embedding 模型（或所选模型不可用），无法向量化检索 → 空。重进知识库设置确认 Embedding 模型已选且可用。

**快速定位法（先做一次，能省 80% 时间）**：点知识检索节点「运行」，在 Query 里**手动输入一个确定存在于库中的关键词/短语**（绕过 QueryRewriter），看 `result` 是否非空：

- **手动能召回、自动（绑 QueryRewriter）为空** → 问题在 ④（改写词质量 / 变量绑定）；
- **手动也空** → 问题在 ①②③⑤⑥⑦（库本身 / 阈值 / 文档 / 配置）。

### 节点 4：LLM-1 Planner（规划结构，不出代码）

在 KnowledgeRetrieval 后面加一个 **「LLM」节点**，命名 `Planner`。

**① 模型**
- 选**默认不开启思考模式**的快速模型。⚠️ **DeepSeek V4 系列（含 V4-Flash）默认开启 Thinking Mode**：模型先输出一段 `reasoning_content`（思维链）再给 JSON，且这段 reasoning **不计入 max_tokens**——所以你明明设了 1500 上限却看到 5.2k 总输出、并卡 2-3 分钟，根源就是思考链（Planner 只产 JSON 大纲，完全不需要思考）。
- 继续用 DeepSeek-V4-Flash 时，必须显式关闭 thinking（见 ⑥）；若你的 Dify 无自定义参数入口传不了 `{"thinking":{"type":"disabled"}}`，用以下绕过之一：① 改**直连 DeepSeek API**（见 `api-and-preview.md`）在请求体写死该参数；② 升级 Dify 或装支持注入模型参数的插件；③ 换成**默认不思考**的模型（如 DeepSeek-V3 系列、`glm-4.7-flash`）。

**② SYSTEM（系统提示词）**
- 粘贴 `prompts/system-prompt.md` 的 **【最高目标】+【角色】+【6 条成功标准】+【数据策略】+【运行安全】** 五段（Planner 不写代码，可省【技术骨架】与【设计自由】以减 prefill）。

**③ USER / 提示词**
- 粘贴 `prompts/planner-prompt.md` 的整段内容。
- `{{userinput.query}}` 绑定 Start 的 `userinput.query`。

**④ ★Context（关键，RAG 注入）★**
- 在 CONTEXT 字段绑定 **KnowledgeRetrieval 的 `result`**。检索到的知识库事实自动注入。**不要**在 prompt 正文里再手写 `{{KnowledgeRetrieval.result}}`（Context 已处理）。

**⑤ 结构化输出（Structured Output / JSON Schema）**
- 打开「**结构化输出**」开关，选 **JSON Schema**，把 `prompts/planner-schema.json` 全部内容粘进输入框。

**⑥ 参数**
- `temperature`：**0.5**（结构要稳）。
- `max_tokens`：**限 1500**（强制精简输出，避免长耗时）。
- **关闭思考模式（首要提速项）**：DeepSeek V4 系列默认开启 thinking，须显式传 `thinking: {"type":"disabled"}（嵌套对象；扁平的 thinking: disabled / false 无效，会报类型错误）` 提速。注意 reasoning 内容**不计入 max_tokens**，所以关掉前你看到的总输出 5.2k 里大部分是思维链，**max_tokens=1500 只限制了最终 JSON**，并非限制失效。若 Dify 无自定义参数入口传不了该参数，见 ① 的绕过方案。

**⚡ Planner 性能优化（若单次 >30 秒）**
- **首要：关掉思考模式**（见 ⑥）。DeepSeek-V4-Flash 默认开启，关掉后通常从 2-3 分钟降到秒级——你实测的 5.2k 输出里绝大多数是思维链（不计入 max_tokens），关掉即消失。
- 减 Context 注入：在「知识检索」节点把 Top K 调小（如 3）、提高相似度阈值，或仅在 Planner 注入精简后的 `result`，缩短 prefill（关掉思考后的次要耗时来源）。
- 精简 SYSTEM：Planner/Generator 的 SYSTEM 按节点 4/5 ② 的段名清单粘贴（见上文，Generator 含【技术骨架】、Planner 不含）；不要把整份 `system-prompt.md` 全粘，去掉与本节点无关的段以缩短 prefill、加速首 token。
- `max_tokens` 限 1500 足够；若关掉 thinking 后 JSON 仍 >2k（模型过度展开，把长文案/示例数据塞进 JSON），收紧 planner-prompt 的 copy 字段只给标题级要点。

> 验证：点节点「运行」，应返回 JSON（**不应再含 reasoning/思维链**），含 `theme` 与 `sections[]`，事实性文案取自检索结果。

### 节点 5：LLM-2 Generator（生成 HTML）

在 Planner 后面加一个 **「LLM」节点**，命名 `Generator`。

**① 模型**
- 选能承载整页 HTML 的模型（如 `glm-4.6`、或你实际用的 `DeepSeek-V4-Flash`）。⚠️ 模型标注的「64k」是**上下文窗口（输入+输出合计）**，不是最大输出长度；单页 HTML 实际能输出多少，由下面 ⑥ 的 `max_tokens` 与模型**硬输出上限**决定。DeepSeek V4 系列同样**默认开启 thinking**，会挤占 64k 上下文、导致 HTML 被截断（见 ⑥）。

**② SYSTEM**
- 粘贴 `prompts/system-prompt.md` 的 **【最高目标】+【角色】+【6 条成功标准】+【设计自由】+【数据策略】+【运行安全】+【技术骨架】** 七段（【设计自由】与【技术骨架】均不可省，前者保证视觉表达自由、后者保证可渲染）。⚠️ **【技术骨架】不可省**——它规定了 Tailwind CDN 引入顺序与 JS 的 DOMContentLoaded 包裹，是 HTML 能正常渲染的根本；省掉等于撤销渲染修复。

**③ USER / 提示词**
- 粘贴 `prompts/generator-prompt.md` 的整段内容。
- 变量绑定：
  - `{{userinput.query}}` → Start 的 `userinput.query`
  - `{{planner_output}}` → **Planner 节点的输出**（变量选择器选 Planner → output）。⚠️ 必须引用 Planner 结构化输出，不要手敲。
- 该 Prompt 让 Generator **先输出 ≤200 字核心功能说明，再紧接 ```html 代码块**（无分隔标记）；`summary`/`html` 由调用方/前端按代码块边界拆分（本方案不依赖 Dify Code 节点，见 `api-and-preview.md` 的 `sanitizeHtml`）。

**④ ★Context（关键，RAG 注入）★**
- 同 Planner：CONTEXT 字段**也绑定 KnowledgeRetrieval 的 `result`**（Generator 看原始检索事实，确保文案细节准确，Planner 可能压缩了细节）。

**⑤ 结构化输出**
- **关闭**（Generator 输出"功能说明+HTML"文本流，不是 JSON）。

**⑥ 参数**
- `temperature`：**0.8 ~ 0.9**。
- `max_tokens`：**拉到 UI 上限**（单页 HTML 通常 8K~20K token；若平台/模型硬上限低于 HTML 体积仍会截断，见下方排查）。
- **关闭思考模式（防截断首要项）**：DeepSeek V4 系列默认开启 thinking，reasoning 会塞满 64k 上下文、挤掉 HTML 输出空间导致截断，须传 `thinking: {"type":"disabled"}（嵌套对象；扁平写法无效）`。若 Dify 无自定义参数入口传不了，用绕过方案（三个 LLM 节点同理）：
  - 换**默认不思考**的模型（`glm-4.7-flash` / DeepSeek-V3 系列）。
  - 改用**直连 API**（见 `api-and-preview.md`）在请求体写死 `"thinking":{"type":"disabled"}` 并把 `max_tokens` 拉满。
  - 思考模式是**服务端推理行为**，提示词关不掉。

**✂️ Generator 内容截断排查（HTML 没到 `</html>` 就被切）**
- **① 关掉 thinking**（上条）：64k 窗口下 HTML 被挤掉的主因，关掉后 HTML 通常能完整输出。
- `max_tokens` 拉到 UI 上限；若仍截，说明 HTML 体积超过模型**硬输出上限**（与 64k 上下文是两回事），此时用**兜底路线**（见文档下方「兜底路线」接 Iteration 逐区块生成）——它专为"单次生成放不下"设计。
- 减 Generator 的 Context 注入：知识检索 Top K 调小/提阈值，或只注入精简 `result`，腾上下文给 HTML（事实性已由 Planner 规划，影响有限）。
- 收紧 HTML 体积：把 generator-prompt 的密度从"每区块≥6 数据单元、4-6 区块"适度下调，或强化"复用 Tailwind 工具类、抽公共 `<style>`"，把单页压进输出上限。

⚖️ **关闭思考模式的质量权衡**：思考链提升结构/配色/交互的推理深度，关掉后速度大升但质量可能略降。若质量不可接受，按优先级恢复（都不必重开 thinking）：
1. **强化 Prompt 硬规则**：两 prompt 已内置以 6 条成功标准为骨架的「规划自检 / 生成前自检」（关闭 thinking 后须显式打勾），可继续加具体反例（见下）。
2. **微调温度**：Generator `0.8~0.9` → `0.7`（更收敛、少随机崩坏），Planner `0.5` → `0.3~0.4`（结构更稳）；代价是"每次不同"略减弱。
3. **换更高质的非思考模型**：如 `DeepSeek-V4-Pro`、或 `glm-4.6` 关 thinking（质量上限更高，仍不触发思考链）。

### 节点 6：LLM-3 Review（独立审查 · Loop Engineering 档位 2）

在 Generator 后面加一个 **「LLM」节点**，命名 `Review`。这是**独立审查 agent**，与 Generator 不同视角——Generator 是作者（有盲区），Review 是审查者（挑刺）。

**① 模型**
- 选**中端模型**即可（如 `glm-4.5-flash` / `DeepSeek-V3` / 同级），审查不需要旗舰——能读懂 HTML、对照 Context 找问题就行。**关闭思考模式**（审查是扫读，不需要长链推理）。

**② SYSTEM / Prompt**
- 粘贴 `prompts/review-prompt.md` 全文（含【角色】【为什么需要你】【审查依据 A-H】【审查方法】【输出】）。该 Prompt 定义了独立审查者视角，对照 SYSTEM 的 A-H 硬约束逐条核查，输出结构化问题清单。

**③ USER / 输入变量**
- `{{userinput.query}}` → Start 节点的 `userinput.query`。
- `{{generator_output}}` → **Generator 节点的输出**（变量选择器选 Generator → output）。⚠️ 必须引用 Generator 的完整输出（功能说明 + HTML），Review 要审查的是这份 HTML。
- CONTEXT 字段绑定 **KnowledgeRetrieval 的 `result`**——Review 要拿原始 Context 对照核查数据真实性（A），这是它比 Generator 自检强的地方。

**④ 结构化输出**
- **打开**「结构化输出」，选 **JSON Schema**，schema 如下（供下游条件分支判断 + Fixer 消费）：
  ```json
  {
    "type": "object",
    "properties": {
      "passed": { "type": "boolean" },
      "issues": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "severity": { "type": "string", "enum": ["致命", "质量"] },
            "constraint": { "type": "string", "enum": ["A","B","C","D","E","F","G","H"] },
            "problem": { "type": "string" },
            "fix_suggestion": { "type": "string" }
          },
          "required": ["severity", "constraint", "problem", "fix_suggestion"]
        }
      }
    },
    "required": ["passed", "issues"]
  }
  ```

**⑤ max_tokens / 参数**
- `max_tokens`：**2000**（问题清单通常几百字，够用）。
- 关闭思考模式（同 Generator 的关 thinking 方法）。

**⑥ 输出变量**
- 该节点输出结构化 JSON（`passed` + `issues`），记下输出变量名。下一步**条件分支节点**判断 `passed`，决定路由到 Fixer 还是直接输出。

> 验证：点节点「运行」，看输出是 JSON：无问题时 `{"passed": true, "issues": []}`；有问题时 `passed: false` 且 issues 含具体问题。若输出非 JSON 或缺 passed 字段，检查 ④ 结构化输出 schema 是否粘对。

**⑦ 下游衔接**
- Review 之后接**条件分支（IF）节点**，判断 `Review.passed == true`：是 → 直接接 Output（用 Generator 的 html）；否 → 接 Fixer 节点（据 issues 修复）。见节点 7、8。

### 节点 7：IF 条件分支（判断 Review.passed）

在 Review 后面加一个 **「条件分支 (IF/ELIF)」节点**。

- **IF 条件**：选 **Review 节点的输出** → `passed` → `等于` → `true`（或 `为真`）。
  - **IF（真）分支** → 直接接 **Output 节点**（用 Generator 的 html，无需修复）。
- **ELSE（假）分支** → 接 **Fixer 节点**（节点 8）→ 再接 **Output 节点**。

> 这就是 Loop Engineering 的"条件路由"——Review 通过则跳过修复，不通过才触发 Fixer。只修一次，不回环。

### 节点 8：LLM-4 Fixer（独立修复 · 据问题清单修复 HTML）

在条件分支的 **ELSE 分支**后面加一个 **「LLM」节点**，命名 `Fixer`。这是**第三个独立 agent**——拿着 Review 的诊断报告做手术，只改指出的问题，不动其他地方。

**① 模型**
- 选**与 Generator 同级或更强**的模型（修复 HTML 需要输出完整文件，能力要够）。关闭思考模式（修复是定向操作，不需要长链推理）。

**② SYSTEM / Prompt**
- 粘贴 `prompts/fixer-prompt.md` 全文。核心理念：**精准修复，不重新生成**——只改 Review 指出的问题，未被指出的部分保持原样，避免"越改越坏"。

**③ USER / 输入变量**
- `{{generator_output}}` → **Generator 节点的输出**（原 HTML，待修复）。
- `{{review_output}}` → **Review 节点的输出**（结构化 JSON 问题清单，Fixer 据此修复）。
- CONTEXT 字段绑定 **KnowledgeRetrieval 的 `result`**（修复数据真实性问题 A 时需对照 Context）。

**④ 结构化输出**
- **关闭**（Fixer 输出修复后的完整 HTML 文本流，不是 JSON）。

**⑤ max_tokens / 参数**
- `max_tokens`：**与 Generator 一致**（拉到 UI 上限 / ≥16000，保证修复后的完整 HTML 不被截断）。
- 关闭思考模式。

**⑥ 输出变量**
- 该节点输出修复后的完整 HTML（`<!DOCTYPE html>` … `</html>`），记下输出变量名。下一步 Output 节点的 `html` 变量在 ELSE 分支引用它。

> 验证：构造一个 Review 不通过的场景（如让 Generator 故意生成带注释的 HTML），看 Fixer 是否据 issues 删除了注释、且未改动其他部分。若 Fixer 推倒重写（改了 Review 没指出的地方），强化 fixer-prompt 的"只改指出问题"约束。
>
> ⚠️ **只修一次不回环**：Fixer 修复后直接接 Output，不再 Review 第二次。这是档位 2 的边界——避免死循环和耗时爆炸。若需迭代审查直到通过，升级档位 3（Chatflow 基于对话历史迭代）。

### 节点 9：Output（输出 / End）

条件分支的两条分支（IF 真 / ELSE 经 Fixer）**都汇合到这一个 Output 节点**。本方案不依赖 Dify Code 拆分节点——`summary`/`html` 由调用方或前端按 ```html 代码块边界拆分（见 `api-and-preview.md` 的 `sanitizeHtml`）。输出**三个**变量：

- 添加输出变量 1：
  - 变量名：`summary`
  - 值：选 **Generator 节点的输出**（功能说明始终来自 Generator，Fixer 只改 HTML 不改说明）。
- 添加输出变量 2：
  - 变量名：`html`
  - 值：⚠️ **路由关键**——IF 真分支选 **Generator 节点的输出**；ELSE 分支选 **Fixer 节点的输出**。Dify 里两条分支汇合到同一 Output 时，`html` 变量按实际执行的分支取值：通过审查→用原 HTML；未通过→用修复后 HTML。（若 Dify 版本不支持分支汇合到单 Output 的变量选择，改用两个 Output 节点：IF 真→Output-A（Generator html），ELSE→Fixer→Output-B（Fixer html），前端按 API 返回的 key 区分。）
- 添加输出变量 3：
  - 变量名：`review`
  - 值：选 **Review 节点的输出**（JSON 问题清单，透明展示审查结果 + 修复了哪些）。
- API 最终返回 `data.outputs.summary`（设计说明）、`data.outputs.html`（完整 HTML，可能经修复）、`data.outputs.review`（审查清单）。

> 注意：当前 Dify 版本终点节点叫 **Output 节点**（旧称 End）。你给变量取的名字会成为 API 响应里的 key。

### 测试

- 画布右上角「运行」，输入：*「帮我做一个面向中小企业的在线客服 SaaS 产品落地页，要包含定价」*。
- 预期：Output 节点输出**两个**变量——`summary`、`html`，**两者都指向 Generator 的完整输出**（含 ≤200 字功能说明前缀 + ` ```html ` 围栏）。API 原始 `data.outputs.html` 不是纯净 HTML；需前端 `sanitizeHtml` 按 ` ```html ` 围栏 / `<!DOCTYPE>`~`</html>` 边界提取后，才是「以 `<!DOCTYPE html>` 开头、以 `</html>` 结尾、无注释」的完整 HTML（净化后存成 `.html` 用浏览器打开应是设计完整、内容丰富的落地页）。
- 把 `html` 存成 `.html` 文件用浏览器打开，应是一个设计完整、内容丰富的落地页；`summary` 应是一段通顺的中文设计说明。
- ★**事实核查（RAG 专项）**：对比生成页面里的产品名/功能/价格与你的知识库资料，确认**严格一致、未编造**；若有出入，加强 Generator 的 Context 绑定与防幻觉约束，或检查知识库召回是否命中。

---

## 输出速度优化（token/s 偏慢，如比 Gemini 慢数倍）

> 适用：实测 token 输出速度明显慢于基线（如 Gemini Flash）时。先定位瓶颈（thinking 隐性成本 / 中转 / 长输入长输出），再对症优化。

**⚠️ 提示词对速度的影响机制（先理解，再优化）**
- 提示词**不改变模型的生成吞吐（tokens/s）**——一旦开始生成，每秒出几个 token 由模型与推理引擎决定，与提示词内容无关。"换个更聪明的提示词让模型生成更快"是误区。
- 但提示词会显著影响**两件事**，从而改变总耗时：
  - **输出总长度（绝对耗时）**：要求越含糊 / 越详尽，输出 token 越多、越慢。好提示词用硬约束（限长度、禁解释文字、只输出代码、禁重复）压缩输出；坏提示词导致过度展开、冗余解释、反复重申 → 更长更慢。
  - **输入长度 → 首 token 延迟（TTFT）**：提示词越长、注入的上下文（system / 检索 Context）越多，prefill 越慢，首字越晚出。精简提示词（如只粘必要三段）直接降 TTFT。
- 好提示词还**减少返工**：约束清晰（格式 / 禁注释 / 禁占位符）让模型一次成型，避免生成中途格式错、截断重来等额外耗时。
- **反直觉点**：「提示词更好」≠「更快」。优化**清晰度 / 约束**（减冗余、禁解释）通常更快且质量不降；优化**丰富度 / 质量**（加密度、加交互、加惊艳）往往让输出更长 → 更慢。这正是「质量 ↔ 速度」权衡的本质（见节点 5 的质量权衡）。

**① 先确认 thinking 真关掉了（最大嫌疑）**
- 思考是**服务端推理行为**，prompt 里写"不要思考"无效。若 Dify 无自定义参数入口、你只是以为关了却没传 `thinking: {"type":"disabled"}`，模型仍在吐 `reasoning_content` 思维链——它不计入 `max_tokens`，但**照样消耗生成时间**，是"输出慢数倍"最常见的隐性元凶。
- 验证：看节点原始输出是否含 reasoning / 思维链文本。有 → 没关掉。必须：直连 API 写死该参数、或换默认不思考模型（`DeepSeek-V3` / `glm-4.7-flash`）。

**② 减少绝对输出 token 数（直接降耗时）**
- Generator 的 HTML 体积（8K~20K）是主要耗时来源。（注：此段降配建议会违反 6 条成功标准的硬下限，仅作应急，日常仍按 4-6 区块 / ≥6 数据单元）速度优先时适度下调密度：每区块 `6→4` 数据单元、区块 `4-6→3-4`，或强化"复用 Tailwind 工具类、抽公共 `<style>`"压体积。这是质量↔速度的权衡。
- `max_tokens` 的两套语境（不矛盾，按场景取）：**默认防截断**（节点 5）→ 拉到 UI 上限 / ≥16000，保证 HTML 不被切；**速度优化**（本章，仅当已稳定不截断、想压耗时）→ 不必盲目拉满，按实际 HTML 体积设略有余量即可（拉满不会更快，设过大反而给模型"展开"空间）。先求不截断，再谈提速。

**③ 减少 prefill 输入（降首 token 延迟 TTFT）**
- 精简 SYSTEM：不必粘整份 `system-prompt.md`。Planner 粘【最高目标】+【角色】+【6 条成功标准】+【数据策略】+【运行安全】五段（不写代码，省【技术骨架】/【设计自由】）；Generator **必须保留【技术骨架】**（六段），否则撤销渲染修复。去掉与本节点无关的段以缩短输入、加速首 token。
- 减 Context：知识检索 Top K 调小（如 3）、提高 Score Threshold，或仅注入精简后的 `result`，缩短 RAG 注入文本。

**④ 减少中转 / 供应商开销**
- 经 Dify 的 Z.ai 等集成调用，可能比官方**直连 API** 慢（多一层代理/排队）。改直连 DeepSeek / Gemini（见 `api-and-preview.md`）：去掉中转层、直接控制 `thinking` 与 `max_tokens`，吞吐通常更稳更快。
- 若你本就以 Gemini 为速度基线且有可用 key，可直接在 Generator 用 `gemini-flash`（原生输出速度极高），Dify 的 Google 供应商或直连均可。

**⑤ 用 streaming 模式（改善感知速度）**
- API 设 `response_mode: "streaming"`（SSE），前端边收边渲染（半截 HTML 不可渲染，但代码视图可先显示），体验上更快（不提升实际生成速度）。

## 缩短整体耗时（真·提高生成吞吐）

> 目标：缩短「工作流跑完、拿到完整 HTML」的**整体耗时（wall-clock）**。先建立公式，再给可操作的真杠杆。本章与「输出速度优化」互补——那里讲"节流（减 token）"，这里讲"提吞吐（换引擎 + 并行）"。

**① 公式：整体耗时 = 总输出 token ÷ tokens/s（单流）**

- **tokens/s（单流生成速度）**由**模型 + 推理引擎/硬件**决定，与你调 prompt、改参数无关（见上一节 ⚠️ 提示词机制）。所以"提高 token 输出速度"这件事，你**只能在选型层动手**，不能在调用层靠提示词动手。
- 缩短整体耗时有**三条互不冲突的路径**，按性价比排序：

| 路径 | 改的是什么 | 是否真提 tokens/s | 操作层 |
|---|---|---|---|
| A. 换更快引擎/模型 | 单流 tokens/s 本身 | ✅ 是 | 选型（换模型、直连、本地部署） |
| B. 减总输出 token | 公式分子 | ❌ 否（节流） | 提示词/密度 |
| C. 并行分块生成 | 单流 → 多流同时 | ✅ 是（整体吞吐） | 工作流拓扑 |

**② 路径 A：换更快的引擎/模型（最直接提 tokens/s）**

- **换模型变体**：同系列优先选更快档（如 `gemini-flash` ≫ `gemini-pro`；`DeepSeek-V3` 比带重推理的变体快）。原生 `gemini-flash` 输出速度极高，是你当前"比 Gemini 慢 5 倍"的最直接解法。
- **去中转、直连官方 API**：经 Dify 集成的供应商调用，可能比**官方直连**多一层代理/排队，吞吐更差。直连 DeepSeek / Gemini（见 `api-and-preview.md`）去掉中转、直接控参，tokens/s 通常更稳更快。
- **本地/自建推理（进阶）**：用 `vLLM` + 高显存 GPU 自托管同源模型，可把 tokens/s 拉到硬件上限，且零排队——适合高频/商用。这是"提 tokens/s"的物理天花板手段。

**③ 路径 B：减总输出 token（见上一节，此处收口）**

- 密度 `6→4`、区块 `4-6→3-4`、复用 Tailwind 抽公共 `<style>`。公式分子变小，耗时线性下降，但牺牲部分丰富度。（注：此降配违反成功标准硬下限，仅应急）

**④ 路径 C：并行分块生成（唯一从架构缩短 wall-clock 的手段）**

> 这是"提高整体生成吞吐"的关键——单流 tokens/s 你改不了，但可以让**多个 section 同时生成**，把"串行求和"变成"并行取最大"。

- **原理**：整体耗时 = Σ(各 section token) ÷ 单流 tokens/s（串行）。若把页面拆成 N 个 `<section>`，用 **N 个并行 LLM 节点**同时生成（都引用同一个共享 design token 保证配色/CSS 变量一致），整体耗时 ≈ **最慢那一段的耗时**，而非 N 段之和。N 段页面理论上可压到约 `1/N` 的生成时间。
- **Dify 落地约束**：
  - Dify 支持**并行分支**（一个节点 fan-out 到多个下游同时跑），可用它并行生成各 section。
  - **注意**：`Iteration` 节点**是否并行取决于 Dify 版本**——默认多为串行逐项，用它逐段生成**通常没有并行收益**；若确认所用版本支持迭代并发，可开启并行模式（见兜底路线步骤 2）；否则要并行必须显式拉出多个并行 LLM 节点（每个对应一个 section 槽位）。
- **样式契约一致性（必做）**：
  - 先由一个轻量节点产出**共享 design token**（配色变量、字体、容器宽度、圆角/阴影规范），作为变量同时传入每个 section 生成节点；
  - 各 section 节点只输出 `<section>…</section>`，**不各自带 `<html>/<head>`**；
  - 最后用一个 **Code 节点** 拼装：`head 骨架(Tailwind CDN + <style> + 容器) + sections.join('') + 闭合标签`。
- **代价**：工作流变复杂（N 个并行节点 / 需管理 design token 变量）；section 间若需强上下文联动（如"上一节数据驱动下一节"）则不适合并行。适合**相互独立的内容区块**多的页面。

**⑤ 取舍建议（对应你的目标）**

- 想"提 tokens/s 本身" → **路径 A**：换 `gemini-flash` 或直连/本地部署，这是你慢 5 倍的根因级解法。
- 想"整体耗时最短且可接受架构改动" → **路径 C（并行）** 叠加 **路径 A**，物理上逼近最短。
- 想"零架构改动、先见效" → **路径 B（减 token）** + 去中转（路径 A 的直连）。
- 三者可叠加：并行（C）负责把长页面拆短、换快引擎（A）负责每流更快、减 token（B）负责每流更短。

## 分块流式渲染（head/body 拆分 ≠ 真·流式提速）

> 常见设想：把 HTML 拆成 head、body 两个 LLM 节点，让 head 先出、body 后出，以为能"流式提速"。下面澄清其真实效果与正确做法。

**① head/body 两节点的真实效果**
- **可行但串行**：Dify 工作流节点串行阻塞，head 节点先完整生成 → body 节点再生成（body 须输入 head 输出以保持配色 / CSS 变量 / 字体契约一致，否则 class 与配色对不上）。总耗时 ≈ head + body，**反而可能比单次生成（head+body 同一次调用）更晚**——多了一次 prefill 与一次调用开销。
- **不解决"首屏更早"**：前端按 `api-and-preview.md` 必须等 `workflow_finished` 拿到完整 HTML 才渲染（半截 HTML 不可渲染）。head 先流式到达也没有页面可显，除非前端自建"先渲染 head + 占位骨架、再 append body"的逻辑（实现成本高）。
- 真实价值不在提速，而在：① 缩短单次输出长度、降低截断概率；② 先定稿样式契约（head），body 以之为约束提升一致性。若为此目的，方案可行。

**② 真正"边生成边看到页面"的做法：section 级分块**
- 把页面按 `<section>` 拆成多个区块，**逐区块生成并流式追加**到 DOM，用户能看到页面"生长"——这才是有效的流式渲染。
- 这正对应本仓库的**兜底路线（Iteration 逐段生成）**：若所用 Dify 版本的流式 SSE 会暴露迭代的逐轮输出事件（每个 section 一个），前端可在收到时 append 到已渲染的 head 骨架上；否则仍需等 `workflow_finished` 拿到全部 sections 再一次性拼装渲染（半截 HTML 不可渲染，见「三、前端预览」）。
- 关键：必须有一个稳定的 head 骨架（Tailwind CDN + `<style>` + 容器），section 片段只含 `<section>...</section>`，**不能各自带 `<html>/<head>`**。

**③ 结论**
- 想"降单次长度 / 保样式契约" → head/body 两节点可以，但**别期望提速**。
- 想"真正流式看到页面" → 走 **section 级分块（Iteration）+ 前端 append**，而非 head/body 两节点。

## 兜底路线（超长页面 · Iteration 分块）

> 仅当单页内容极多（如 10+ 个重内容区块），Generator 单次生成仍截断时启用。日常用主路线即可。
>
> ⚠️ 兜底路线复用主路线的 **Planner**（输出区块数组）和 **Generator**（出 summary+html）。

拓扑：

```
Start → Query Rewriter → 知识检索 → Planner(输出 sections 数组)
      → Iteration(输入=Planner.sections, 内部LLM逐段生成HTML片段) → 片段数组
      → Template节点(拼成完整HTML骨架)
      → Generator(出summary+html,复用主路线Prompt)
      → Output(summary + html)   ← 无需 Code 拆分节点，由前端按 ```html 围栏提取
```

### 步骤

1. **Planner** 与主路线相同，确保其 `sections` 输出是数组。
2. 加 **「迭代 (Iteration)」节点**：
   - **输入**：选 Planner 输出的 `sections` 数组。
   - **模式**：并行（上限 10）。
   - **循环内部**：放一个 **LLM 节点**，Prompt 大意：
     > 你是前端工程师。请为下面这**一个**区块生成 HTML 片段（仅该区块的 `<section>...</section>`，不要 `<!DOCTYPE>`/`<html>`/`<head>`/Tailwind CDN——由外层骨架提供）。
     > 当前区块：`{{items}}`。要求：完整真实文案、无占位符、Tailwind class、禁注释、禁localStorage。
   - Iteration 输出 = 片段数组。
3. ⚠️ **加 Template 节点（关键，不可省）**：Iteration 输出数组不能直接喂下游 LLM。加「模板(Jinja2)」节点把片段拼进骨架（用 4 反引号示意，实际按 Jinja2 语法填，外层不套 markdown 围栏）：

   ````jinja
   <!DOCTYPE html>
   <html lang="zh-CN">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width,initial-scale=1.0">
     <title>{{ brand_name }}</title>
     <script src="https://cdn.tailwindcss.com"></script>
     <style>:root{--c-primary:{{primary}};}</style>
   </head>
   <body>
   {% for sec in sections %}{{ sec }}
   {% endfor %}
   </body>
   </html>
   ````

   - 用 `{% for %}` 拼片段；模板变量需从 Planner 输出**显式绑定**：`brand_name` ← `Planner.theme.brand_name`，`primary` ← `Planner.theme.colors.primary`（顶层并无 `primary` 字段，必须映射到 `colors.primary`；若 Dify 模板不支持嵌套变量直接引用，可先用一个变量赋值/Code 节点把 `theme.colors.primary` 取出再传入）。`title` 字段不存在，已改 `brand_name`。Template 输出纯净 HTML 字符串，不要再外包围栏。
  - ⚠️ 此处的 `sections` 指**迭代节点的输出数组变量**，并非 Planner 的 `sections` 大纲——两者须用不同变量名（如把迭代输出命名为 `html_sections`，并改此处的 `{% for sec in html_sections %}`），否则会把大纲 JSON 当片段拼进去。
4. **复用 Generator**（出 summary + html）：把 Template 输出的完整 HTML 作为变量（如 `{{assembled_html}}`）注入主路线 Generator 的 USER Prompt（新增一段"已生成完整 HTML 如下，请基于它写功能说明并原样输出"），生成 ≤200 字功能说明 + ```html 代码块。
5. **Output**：与主路线完全相同（无需 Code 拆分节点，HTML 由前端按 ```html 围栏提取）。

### 兜底路线注意点
- Planner 用小模型 + max_tokens 限 1500 提速。
- Iteration 内部 LLM 也要高 max_tokens（单段可能较长）。
- 骨架（`<head>`/Tailwind CDN/`<style>`/`</body>`）只在 Template 出现一次，片段只含 `<section>`。
- **兜底路线必须也输出 summary**：别跳过步骤 4（复用 Generator），否则 Output 缺 summary。

---

---

## 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| Generator 引用变量报错 | 没用变量选择器绑定。删掉占位重新插入「变量 → Start → userinput.query」。 |
| Generator 输出被截断（代码块未闭合，没有结尾 ```） | `max_tokens` 太小。拉到上限；或改走兜底 Iteration 路线。 |
| Generator 输出**没有** ```html 围栏、或夹带"好的这是…" | System Prompt 的禁令没生效，确认 system-prompt.md 已正确粘贴到 SYSTEM 位（围栏是**必须**的）。 |
| 页面出现 `...` / 占位符 | 温度过高 或 max_tokens 不足迫使省略。降温度到 0.6、加大 max_tokens。 |
| Iteration 下游 LLM 报"需要字符串" | 没加 Template/Code 转换。见兜底路线步骤 3。 |
| 配色每次都一样/像固定模板 | 模型没放开发挥。强化 generator-prompt/planner-prompt 的「配色自主、每次风格迥异、不要总蓝紫」要求，提高 temp（Planner 0.5、Generator 0.8+）。 |
| **★页面一拉到底/没有强交互** | Generator 规划的还是展示型区块。① 强化 generator-prompt 的「紧凑高密度+强交互」要求；② 必要时直接在 userinput.query 里点明要哪种交互，如"做一个能勾选需求并推荐方案的页面"。 |
| **★推荐/勾选联动不生效** | Generator 没实现联动逻辑。① 强化 generator-prompt 要求勾选项带 impact、推荐带 recommend_rules；② 推荐逻辑应是纯函数 `recommend(selected)`。 |
| **★页面交互失效/JS 报错（控制台 SecurityError）** | 模型用了 `localStorage`/`sessionStorage`/`alert` 等沙箱禁用 API。① 强化 generator-prompt 的禁令；② 在 iframe 渲染层这些 API 本就会被 `sandbox` 阻止，需让模型改用内存变量+页面内 toast。 |
| **★生成的产品/价格与知识库不符（编造）** | 防幻觉未生效。① 确认 Generator 在 **Context 字段绑定了 KnowledgeRetrieval.result**；② 确认检索确实召回了相关资料（看 `result` 是否为空）；③ 调低 Score Threshold 提高召回。 |
| **★检索没召回到内容（result 为空）** | Query 改写词不准；或 Score Threshold 太高；或知识库分段过大/过小。检查 QueryRewriter 输出、降低阈值、优化知识库分段。 |
| **★LLM 报"未提供 context"** | Context 字段没绑定，或绑定成了空数组。在 Generator 的 CONTEXT 字段重新选 KnowledgeRetrieval → result。 |
| 不知道在哪绑 Context | LLM 节点配置里有独立的 **CONTEXT / 上下文** 区域（和 SYSTEM/USER 平级），点「+」选「知识检索 → result」。**不要**在 prompt 正文里写 `{{result}}`。 |
| **★Output 里没有 summary，只有 html**（或反之） | Generator 输出格式不对。① 检查 Generator 原始输出是否为「开头功能说明 + 紧接 ```html 代码块」；② Output 节点两个变量都引用 Generator 输出（由前端/调用方按 ```html 代码块边界拆分 `summary`/`html`，见 `api-and-preview.md` 的 `sanitizeHtml`）。 |
| **★summary 和 html 混在一起 / 拆不开** | 未用前端/调用方按 ```html 代码块边界拆分。确认 `api-and-preview.md` 的 `sanitizeHtml` 已截取 `<!doctype … </html>`（它会自动剥离功能说明前缀与围栏）。 |
| summary 超过 200 字 | Generator Prompt 的字数约束没生效，强化提示或检查温度（功能说明部分温度宜低）。 |
