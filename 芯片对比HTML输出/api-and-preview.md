# 发布、API 调用与安全预览

> 工作流搭好后，发布为应用，通过 API 调用，并在前端用 iframe 安全预览生成的 HTML。

---

## 一、发布并获取 API Key

1. 工作流编辑器右上角 → **「发布 (Publish)」**。
2. 进入应用「**访问 API (API Access)**」页（左侧菜单）。
3. 创建一个 **API 密钥**（每个应用独立）。
4. 记下：
   - **Base URL**：Dify Cloud 为 `https://api.dify.ai/v1`；自托管用你的实例地址 + `/v1`。
   - **API Key**：`Bearer <API_KEY>`。

---

## 二、调用 API

### Endpoint

`POST /v1/workflows/run`

> Workflow 类型应用用这个。Chat/Chatflow 类型用 `/v1/chat-messages`（本方案推荐 Workflow 类型，结果直接在 `data.outputs`，最干净）。

### 请求头

```
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

### 请求体

```json
{
  "inputs": {
    "user_requirement": "帮我做一个 AI 写作助手的落地页，要有定价和用户评价"
  },
  "response_mode": "streaming",
  "user": "user-001"
}
```

- `inputs.user_requirement`：**必须与 Start 节点变量名一致**（我们叫 `user_requirement`）。
- `response_mode`：`blocking`（等全部生成完一次性返回）或 `streaming`（SSE 流式）。单页 HTML 推荐 `streaming`，可在前端展示"正在生成"。
- `user`：终端用户标识（必填，自定义字符串）。

> **RAG 说明**：接入私有知识库后，**API 入参不变**（仍只传 `user_requirement`）——知识检索在工作流内部自动完成。生成的内容会**严格依据你知识库里的产品/业务事实**。注意：生成质量取决于知识库内容与召回设置，首次调用会多经过 Query 改写 + 检索 + 两个 LLM，延迟略增（约多 5-15 秒）。

### Blocking 响应（response_mode: blocking）

```json
{
  "task_id": "...",
  "workflow_run_id": "...",
  "data": {
    "status": "succeeded",
    "outputs": {
      "summary": "本次为面向中小企业的在线客服 SaaS 生成落地页，含首屏、特性、定价、FAQ 等区块……（≤500 字）",
      "html": "<!DOCTYPE html>...</html>"
    },
    "total_tokens": 12345,
    "elapsed_time": 18.5
  }
}
```

→ **`data.outputs.summary`** 是 ≤500 字中文设计说明；**`data.outputs.html`** 是完整 HTML（Code 节点已剥掉 markdown 围栏，可直接渲染）。变量名 `summary`/`html` 对应你在 Output 节点里命名的变量。

### Streaming 响应（response_mode: streaming，SSE）

逐事件返回，类型包括 `node_started` / `node_finished` / `text_chunk` / `workflow_finished` / `ping`。

**最终结果**：在最后一个 **`workflow_finished`** 事件里取 `data.outputs.summary` 和 `data.outputs.html`。

```jsonc
{
  "event": "workflow_finished",
  "task_id": "...",
  "data": {
    "status": "succeeded",
    "outputs": {
      "summary": "本次为面向中小企业的在线客服 SaaS 生成落地页……",
      "html": "<!DOCTYPE html>...</html>"
    },
    "total_tokens": 12345
  }
}
```

> 流式期间若想要"边生成边显示"，可读 `text_chunk` 事件里的分片拼接到一个代码视图；但**预览渲染必须等 `workflow_finished` 拿到完整 HTML**（半截 HTML 无法渲染）。

### curl 示例（blocking）

```bash
curl -X POST 'https://api.dify.ai/v1/workflows/run' \
  -H 'Authorization: Bearer <API_KEY>' \
  -H 'Content-Type: application/json' \
  -d '{
    "inputs": { "user_requirement": "做一个面向咖啡爱好者的订阅盒落地页" },
    "response_mode": "blocking",
    "user": "demo"
  }'
```

---

## 三、前端预览（iframe + 安全）

### 最小预览页（单文件，直接存成 .html 打开）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>HTML 生成预览</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 0; padding: 16px; background:#f5f5f7; }
    .row { display:flex; gap:8px; margin-bottom:12px; }
    input { flex:1; padding:10px; font-size:14px; border:1px solid #ccc; border-radius:8px; }
    button { padding:10px 18px; font-size:14px; border:0; border-radius:8px; background:#2563eb; color:#fff; cursor:pointer; }
    button:disabled { background:#9aa0a6; }
    .bar { display:flex; gap:8px; align-items:center; margin-bottom:8px; }
    iframe { width:100%; height:70vh; border:1px solid #ddd; border-radius:12px; background:#fff; }
    .status { color:#666; font-size:13px; }
    a.dl { padding:8px 12px; background:#e5e7eb; color:#111; border-radius:8px; text-decoration:none; font-size:13px; }
    summary-box { display:block; background:#fff; border:1px solid #e5e7eb; border-radius:12px; padding:12px 16px; margin-bottom:10px; font-size:14px; line-height:1.7; color:#333; white-space:pre-wrap; }
    summary-box:empty{ display:none; }
  </style>
</head>
<body>
  <div class="row">
    <input id="req" placeholder="一句话描述你要的网站，如：AI 瑜伽教练 App 落地页" />
    <button id="run">生成</button>
  </div>
  <div class="bar">
    <span class="status" id="status"></span>
    <a class="dl" id="download" style="display:none" download="page.html">下载 .html</a>
  </div>
  <summary-box id="summary"></summary-box>
  <iframe id="preview" sandbox="allow-scripts"></iframe>

<script>
const API_KEY = 'app-xxxxxxxxxxxxxxxx';      // ← 换成你的 Dify API Key
const BASE = 'https://api.dify.ai/v1';       // ← 自托管改成你的地址

const $ = id => document.getElementById(id);

async function run() {
  const q = $('req').value.trim();
  if (!q) return alert('请输入需求');
  $('run').disabled = true;
  $('status').textContent = '生成中…（含检索+总结，约 20-50 秒）';
  $('download').style.display = 'none';
  $('summary').textContent = '';

  try {
    const resp = await fetch(`${BASE}/workflows/run`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${API_KEY}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ inputs: { user_requirement: q }, response_mode: 'blocking', user: 'web' })
    });
    const json = await resp.json();
    const out = json?.data?.outputs;
    const html = out?.html;
    const summary = out?.summary;
    if (!html) throw new Error('未取到 html，完整返回：' + JSON.stringify(json).slice(0, 300));
    // 工作流里 Code 节点已剥掉围栏、截到 </html>；这里做兜底清洗，确保纯净
    const clean = sanitizeHtml(html);
    $('preview').srcdoc = clean;
    $('summary').textContent = summary || '';
    $('status').textContent = '完成 ✓';
    // 下载链接
    const blob = new Blob([clean], { type: 'text/html' });
    $('download').href = URL.createObjectURL(blob);
    $('download').style.display = 'inline-block';
  } catch (e) {
    $('status').textContent = '出错：' + e.message;
  } finally {
    $('run').disabled = false;
  }
}

function sanitizeHtml(s) {
  let t = (s || '').trim();
  // 兜底：剥掉可能的 ```html 围栏（Code 节点通常已处理，这里防万一）
  const fence = t.match(/```(?:html)?\s*\n([\s\S]*?)\n?```\s*$/i);
  if (fence) t = fence[1];
  t = t.replace(/^```(?:html)?\s*/i, '').replace(/```\s*$/, '').trim();
  const start = t.toLowerCase().indexOf('<!doctype html>');
  const end = t.toLowerCase().lastIndexOf('</html>');
  if (start >= 0 && end > start) t = t.slice(start, end + '</html>'.length);
  return t;
}

$('run').addEventListener('click', run);
$('req').addEventListener('keydown', e => { if (e.key === 'Enter') run(); });
</script>
</body>
</html>
```

> 把 `API_KEY` 和 `BASE` 换成你的即可用。若 Dify 与前端不同域，确保 Dify 侧允许跨域（自托管时可配置 CORS；Dify Cloud 的 API 默认可被浏览器调用）。

### 安全要点（iframe 渲染不可信 HTML）

生成的 HTML 含 **Tailwind Play CDN**（会执行外部脚本），因此预览 iframe 必须：

1. **`sandbox="allow-scripts"`** —— 只放开脚本权限（Tailwind 需要）。
2. **绝对不要加 `allow-same-origin`** —— `allow-scripts` + `allow-same-origin` 组合会让沙箱内脚本访问父页面 DOM、逃逸沙箱。只给 `allow-scripts` 时，iframe 会被强制为唯一 opaque origin，同源策略生效。
3. **用 `srcdoc` 而非 `src=同源URL`** —— `srcdoc` 默认继承父源，必须靠 `sandbox` 把它降为 opaque origin。
4. **父子通信用 `postMessage`**，不要把父 `window` 暴露给 iframe。
5. **叠加 CSP（纵深防御）**：在父页设 `Content-Security-Policy`，`script-src` 只放行 `cdn.tailwindcss.com` 与自身，禁止 `unsafe-inline` 之外的未知源。

> 这是 web.dev 与 Google SafeContentFrame 一致的建议：从最小权限的空 sandbox 开始，按需逐个放开；不可信内容理想情况下放到独立 origin 渲染。

### 交互的 API 边界（升级到可交互页面后必读）

`sandbox="allow-scripts"`（不带 `allow-same-origin`）下，iframe 是 opaque origin，绝大多数交互能用，**但有几类 API 必定失败**：

| API | 能否用 | 说明 |
|---|---|---|
| click / input / select / drag / tab / modal / Canvas / IntersectionObserver / `navigator.clipboard` | ✅ 能 | 不依赖 origin |
| 暗色切换（`dataset.theme`）/ CSS 变量 | ✅ 能 | 纯 DOM/CSS |
| **`localStorage` / `sessionStorage` / `IndexedDB` / `document.cookie`** | ❌ 抛 `SecurityError` | opaque origin 无存储 |
| `alert()` / `confirm()` / `prompt()` | ❌ 被阻止 | 需 `allow-modals`（不建议加） |

**对策**（已写进 Generator Prompt 硬约束）：
- 状态存 **JS 内存变量**（不持久化，刷新即丢，单页交互通常可接受）。
- 弹窗用页面内 **toast/浮层**替代 `alert`。
- 若需跨刷新持久化：iframe 内用 `window.parent.postMessage(data, '*')` 把数据交给**父页面**，父页面存自己的 `localStorage`，再 `postMessage` 回传。监听示例（父页面）：
  ```js
  window.addEventListener('message', e => {
    if (e.data?.type === 'save') localStorage.setItem(e.data.key, JSON.stringify(e.data.value));
    if (e.data?.type === 'load') {
      const v = localStorage.getItem(e.data.key);
      document.getElementById('preview').contentWindow.postMessage({ type: 'load', value: v }, '*');
    }
  });
  ```

---

## 四、流式版本（可选，体验更好）

把 `response_mode` 改为 `streaming`，用 `fetch` 读 SSE 流：

- 读 `text_chunk` 事件可做"代码区边生成边显示"。
- **但 iframe 预览要等 `workflow_finished`**（拿到完整 `data.outputs.html` 后再 `srcdoc`），因为半截 HTML 无法渲染。
- 解析 SSE：按 `\n\n` 分割事件块，每块里取 `event:` 与 `data:` 行，`data` 是 JSON 字符串再 `JSON.parse`。

伪代码：

```js
const resp = await fetch(`${BASE}/workflows/run`, { /* ... response_mode:'streaming' */ });
const reader = resp.body.getReader();
const dec = new TextDecoder();
let buf = '';
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buf += dec.decode(value, { stream: true });
  const parts = buf.split('\n\n'); buf = parts.pop();
  for (const part of parts) {
    const line = part.split('\n').find(l => l.startsWith('data:'));
    if (!line) continue;
    const evt = JSON.parse(line.slice(5).trim());
    if (evt.event === 'workflow_finished') {
      const { summary, html } = evt.data.outputs;
      $('summary').textContent = summary || '';
      $('preview').srcdoc = sanitizeHtml(html);   // sanitizeHtml 见上方预览页
    }
  }
}
```
