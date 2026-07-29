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

## 主路线（3 节点）

### 节点 1：Start（开始）

- 在画布上点击 **「开始节点」**。
- 添加输入字段：
  - 字段名：`user_requirement`
  - 类型：**段落 (Paragraph)** 或 **文本 (Text)**
  - 必填：是
  - 标签/提示：例如「用一句话描述你想要的网站」

> 之后所有节点的 `{{user_requirement}}` 都引用这里。

### 节点 2：LLM-1 Planner（规划结构）

在 Start 后面加一个 **「LLM」节点**，命名 `Planner`。

**① 模型**
- 选 `glm-4.6`（首选）或 `glm-4.5`。

**② SYSTEM（系统提示词）**
- 粘贴 `prompts/system-prompt.md` 里「直接复制下面的内容」整段。

**③ USER / 提示词（用户消息）**
- 粘贴 `prompts/planner-prompt.md` 的 USER Prompt 整段。
- 其中 `{{user_requirement}}` 需要绑定变量：在编辑器里选中该占位 → 选择「变量」→ 引用 **Start 节点的 `user_requirement`**。（直接敲 `{{user_requirement}}` 文本也可以，Dify 会识别。）

**④ 结构化输出（Structured Output / JSON Schema）**
- 打开「**结构化输出**」开关。
- 选 **JSON Schema** 方式，把 `prompts/generator-schema.json` 的**全部内容**粘进输入框。
- 输出变量名默认为节点输出（后续 Generator 引用它）。

**⑤ 参数**
- `temperature`：**0.3**（结构要稳）。
- `max_tokens`：默认即可（Planner 输出的是简短 JSON）。

> 验证：点节点右上角「运行」，输入 `做一个 AI 写作助手的落地页`，应返回一段 JSON，含 `theme` 与 `sections[]`。

### 节点 3：LLM-2 Generator（生成完整 HTML）

在 Planner 后面加一个 **「LLM」节点**，命名 `Generator`。

**① 模型**
- 选 `glm-4.6`（首选）。

**② SYSTEM**
- 粘贴 `prompts/system-prompt.md` 整段（与 Planner 共用）。

**③ USER / 提示词**
- 粘贴 `prompts/generator-prompt.md` 的 USER Prompt 整段。
- 变量绑定：
  - `{{user_requirement}}` → Start 的 `user_requirement`
  - `{{planner_output}}` → **Planner 节点的输出**（在变量选择器里选 Planner 节点 → output）。⚠️ 关键：必须引用 Planner 的结构化输出，不要手敲。

**④ 结构化输出**
- **关闭**（Generator 要输出纯 HTML 字符串，不是 JSON）。

**⑤ 参数**
- `temperature`：**0.6 ~ 0.7**。
- `max_tokens`：**拉到 UI 允许的上限**（GLM-4.6 可设很大；务必远大于默认 4K，否则单页 HTML 会截断）。

### 节点 4：Output（输出 / End）

在 Generator 后面加 **「输出 (Output / End)」节点**。

- 添加输出变量：
  - 变量名：`html`
  - 值：选择 **Generator 节点的输出文本**（Generator 的 `text` / `output`）。
- 这个 `html` 就是 API 最终返回的完整 HTML 字符串（`data.outputs.html`）。

> 注意：当前 Dify 版本终点节点叫 **Output 节点**（旧称 End）。你给变量取的名字 `html` 会成为 API 响应里的 key。

### 测试

- 画布右上角「运行」，输入：*「帮我做一个面向中小企业的在线客服 SaaS 产品落地页，要包含定价」*。
- 预期：Output 节点输出一整段被 ```html 代码块包裹的完整 HTML（剥掉围栏后以 `<!DOCTYPE html>` 开头、`</html>` 结尾，且不含任何注释）。
- 剥掉首尾的 ```html / ``` 后存成 `.html` 文件用浏览器打开，应是一个设计完整、内容丰富的落地页。

---

## 兜底路线（超长页面 · Iteration 分块）

> 仅当单页内容极多（如 10+ 个重内容 section），GLM 单次生成仍截断时启用。日常用主路线即可。

拓扑：

```
Start → Planner(输出 sections 数组)
      → Iteration(输入=Planner.sections, 内部 LLM 逐段生成 HTML 片段) → 输出片段数组
      → Template 节点(join 拼接) → Output
```

### 步骤

1. **Planner** 与主路线相同，确保其 `sections` 输出是数组。
2. 加 **「迭代 (Iteration)」节点**：
   - **输入**：选 Planner 输出的 `sections` 数组（必须是 array 类型）。
   - **模式**：并行（Parallel），注意并行上限 10。
   - **循环内部**：放一个 **LLM 节点**，Prompt 大意：
     > 你是前端工程师。请为下面这**一个**页面区块生成 HTML 片段（仅该区块的 `<section>...</section>`，**不要** `<!DOCTYPE>`、不要 `<html><head><body>`，不要 Tailwind CDN——这些由外层骨架提供）。
     > 当前区块信息：`{{items}}`（迭代内置变量，即数组当前元素）。主题色：`{{theme}}`。
     > 要求：完整、真实文案、无占位符、用 Tailwind class、**不得包含任何注释**（`<!-- -->`/`/* */`/`//` 都不行）。
   - 内部 LLM 的输出即"一段 HTML 片段"。
   - Iteration 节点最终输出 = **片段数组**（每个元素是一段 HTML 字符串）。
3. ⚠️ **加 Template 节点（关键，不可省）**：
   - Dify 的 LLM 节点只能吃字符串，Iteration 输出的数组**不能直接喂下游**。
   - 加 **「模板 (Template / Jinja2)」节点**，内容（不含任何注释，并用 ```html 代码块包裹，与主路线输出格式一致）：
     ```jinja
     ```html
     <!DOCTYPE html>
     <html lang="zh-CN">
     <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>{{ title }}</title>
       <script src="https://cdn.tailwindcss.com"></script>
       <style>:root{--c-primary:{{primary}};}</style>
     </head>
     <body>
     {% for sec in sections %}{{ sec }}
     {% endfor %}
     <footer>{{ footer_copy }}</footer>
     </body>
     </html>
     ```
     ```
     （把 Iteration 的片段数组用 `{% for %}` 或 `| join` 拼进骨架。`title` / `primary` / `footer_copy` 等可从 Planner 的 theme 取。各 section 片段本身也不得含注释。）
   - 或用 **Code 节点**（JS）：`return { html: "<!DOCTYPE html>..." + inputs.sections.join("\n") + "..." }`。注意 Code 节点字符串上限 40 万字符。
4. **Output 节点**：输出 Template/Code 的 `html`。

### 兜底路线的注意点
- Iteration 内部 LLM 也要设高 `max_tokens`（单段也可能较长）。
- 拼接顺序 = 数组顺序 = Planner 里 `order` 排序后的顺序；确保 Planner 输出按 `order` 升序。
- 骨架（`<head>`、Tailwind CDN、`<style>`、`</body>`）只在 Template 里出现**一次**，各 section 片段只含 `<section>`，避免重复引入 CDN。

---

## 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| Generator 引用 `{{planner_output}}` 报错 | 没用变量选择器绑定，或绑错了节点。删掉占位重新插入「变量 → Planner → output」。 |
| Generator 输出被截断（代码块未闭合，没有结尾 ```） | `max_tokens` 太小。拉到上限；或改走兜底 Iteration 路线。 |
| Generator 输出里有 ```html 围栏或"好的这是…" | System Prompt 的禁令没生效，确认 system-prompt.md 已正确粘贴到 SYSTEM 位。 |
| 页面出现 `...` / 占位符 | 温度过高 或 max_tokens 不足迫使省略。降温度到 0.6、加大 max_tokens。 |
| Iteration 下游 LLM 报"需要字符串" | 没加 Template/Code 转换。见兜底路线步骤 3。 |
| 配色丑/不协调 | 强化 Planner 的 theme 要求，或预设几套色板让其选。 |
