# Code 节点：拆分 summary 与 html

> 作用：Generator 现在按 `===SUMMARY===` / `===HTML===` 两段输出。本 Code 节点把它拆成 **`summary`** 和 **`html`** 两个独立字符串变量，供下游 Output 节点分别引用。
>
> 为什么需要它：Dify 的 LLM 节点默认只产出一个文本流，Output 节点没法"自动知道"哪段是总结、哪段是 HTML。用一个轻量 Code 节点按分隔标记拆开，是最稳、最可控的方式（不依赖 JSON Schema，避免长 HTML 字符串解析失败）。

---

## Dify Code 节点配置

- 节点类型：**代码执行 (Code)** → 语言选 **JavaScript**（Python 也行，下面两个版本都给）。
- 位置：放在 **Generator 之后、Output 之前**。
- **输入变量**：
  - 变量名：`raw`
  - 值：绑定 **Generator 节点的输出**（Generator 的 text / output）。

## 输出变量（在 Code 节点的「输出」里定义这两个）

| 变量名 | 类型 |
|---|---|
| `summary` | String |
| `html` | String |

---

## JavaScript 版（粘进代码框）

```javascript
function main({ raw }) {
  let summary = '';
  let html = '';

  const text = (raw || '').toString();

  const htmlIdx = text.indexOf('===HTML===');
  if (htmlIdx >= 0) {
    const before = text.slice(0, htmlIdx);
    const after = text.slice(htmlIdx + '===HTML==='.length);

    const sumIdx = before.indexOf('===SUMMARY===');
    summary = sumIdx >= 0 ? before.slice(sumIdx + '===SUMMARY==='.length) : before;
    summary = summary.trim();

    let h = after.trim();
    const fence = h.match(/```(?:html)?\s*\n([\s\S]*?)\n?```\s*$/i);
    if (fence) h = fence[1];
    h = h.replace(/^```(?:html)?\s*/i, '').replace(/```\s*$/, '').trim();
    const start = h.toLowerCase().indexOf('<!doctype html>');
    const end = h.toLowerCase().lastIndexOf('</html>');
    if (start >= 0 && end > start) h = h.slice(start, end + '</html>'.length);
    html = h;
  } else {
    html = text.trim();
    const fence = html.match(/```(?:html)?\s*\n([\s\S]*?)\n?```\s*$/i);
    if (fence) html = fence[1];
    html = html.replace(/^```(?:html)?\s*/i, '').replace(/```\s*$/, '').trim();
    const start = html.toLowerCase().indexOf('<!doctype html>');
    const end = html.toLowerCase().lastIndexOf('</html>');
    if (start >= 0 && end > start) html = html.slice(start, end + '</html>'.length);
  }

  return { summary: summary, html: html };
}
```

> **入参说明（重要）**：Dify Code 节点 JS 的入参对象的**键名 = 你定义的输入变量名**。本脚本假设输入变量名为 `raw`，所以解构写成 `{ raw }`。务必保证：① 节点「输入变量」里把变量名填为 `raw`，值绑定 Generator 输出；② 函数签名与变量名一致。若你把变量名设为别的（如 `text`），请同步把签名改成 `function main({ text })` 并把函数体内 `raw` 全部替换。

## Python 版（备选）

```python
import re

def main(raw: str) -> dict:
    text = raw or ''
    summary = ''
    html = ''

    if '===HTML===' in text:
        before, after = text.split('===HTML===', 1)
        if '===SUMMARY===' in before:
            summary = before.split('===SUMMARY===', 1)[1]
        else:
            summary = before
        summary = summary.strip()
        h = after.strip()
    else:
        h = text.strip()

    m = re.search(r'```(?:html)?\s*\n([\s\S]*?)\n?```\s*$', h, re.IGNORECASE)
    if m:
        h = m.group(1)
    h = re.sub(r'^```(?:html)?\s*', '', h, flags=re.IGNORECASE)
    h = re.sub(r'```\s*$', '', h).strip()
    low = h.lower()
    s = low.find('<!doctype html>')
    e = low.rfind('</html>')
    if s >= 0 and e > s:
        h = h[s:e + len('</html>')]

    html = h
    return {'summary': summary, 'html': html}
```

> Python 版同理：函数参数名 `raw` 必须与你在节点「输入变量」里定义的变量名一致。

---

## 验证

- 点 Code 节点「运行」，输入测试文本（含两段 + 分隔标记），应输出：
  - `summary`：纯中文设计说明文本（无 `===` 标记、无围栏）。
  - `html`：以 `<!DOCTYPE html>` 开头、`</html>` 结尾的纯净 HTML（无围栏、无总结文字）。
- 若 `summary` 为空：检查 Generator 是否真的输出了 `===SUMMARY===` 标记行。
- 若 `html` 含围栏/总结文字：分隔标记没对上，检查 Generator 输出格式。
