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

### Blocking 响应（response_mode: blocking）

```json
{
  "task_id": "...",
  "workflow_run_id": "...",
  "data": {
    "status": "succeeded",
    "outputs": {
      "html": "```html\n<!DOCTYPE html>...\n</html>\n```"
    },
    "total_tokens": 12345,
    "elapsed_time": 12.3
  }
}
```

→ 完整 HTML 在 **`data.outputs.html`**（`html` 就是你 Output 节点里命名的变量名）。

### Streaming 响应（response_mode: streaming，SSE）

逐事件返回，类型包括 `node_started` / `node_finished` / `text_chunk` / `workflow_finished` / `ping`。

**最终结果**：在最后一个 **`workflow_finished`** 事件里取 `data.outputs.html`。

```jsonc
{
  "event": "workflow_finished",
  "task_id": "...",
  "data": {
    "status": "succeeded",
    "outputs": { "html": "```html\n<!DOCTYPE html>...\n</html>\n```" },
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
    iframe { width:100%; height:75vh; border:1px solid #ddd; border-radius:12px; background:#fff; }
    .status { color:#666; font-size:13px; }
    a.dl { padding:8px 12px; background:#e5e7eb; color:#111; border-radius:8px; text-decoration:none; font-size:13px; }
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
  <iframe id="preview" sandbox="allow-scripts"></iframe>

<script>
const API_KEY = 'app-xxxxxxxxxxxxxxxx';      // ← 换成你的 Dify API Key
const BASE = 'https://api.dify.ai/v1';       // ← 自托管改成你的地址

const $ = id => document.getElementById(id);
let lastHtml = '';

async function run() {
  const q = $('req').value.trim();
  if (!q) return alert('请输入需求');
  $('run').disabled = true;
  $('status').textContent = '生成中…（首次约 15-40 秒）';
  $('download').style.display = 'none';

  try {
    const resp = await fetch(`${BASE}/workflows/run`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${API_KEY}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ inputs: { user_requirement: q }, response_mode: 'blocking', user: 'web' })
    });
    const json = await resp.json();
    const html = json?.data?.outputs?.html;
    if (!html) throw new Error('未取到 html，完整返回：' + JSON.stringify(json).slice(0, 300));
    // 剥掉 ```html 围栏/前言，截到 </html>
    const clean = extractHtml(html);
    lastHtml = clean;
    $('preview').srcdoc = clean;
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

function extractHtml(s) {
  let t = s.trim();
  // 模型输出被 ```html ... ``` 代码块包裹：剥掉首尾围栏
  const fence = t.match(/```(?:html)?\s*\n([\s\S]*?)\n?```\s*$/i);
  if (fence) t = fence[1];
  // 兜底：去掉残留的首尾单围栏
  t = t.replace(/^```(?:html)?\s*/i, '').replace(/```\s*$/, '').trim();
  // 截取 <!DOCTYPE html> 到 </html> 之间的纯净 HTML
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
      const html = evt.data.outputs.html;
      $('preview').srcdoc = extractHtml(html);
    }
  }
}
```
