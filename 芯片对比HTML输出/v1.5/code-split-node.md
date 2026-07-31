# Code 节点：拆分 summary 与 html

> 作用：Generator 输出「核心功能说明 + ```html 代码块」两部分（自然衔接，**无分隔标记**）。本 Code 节点把它拆成 **`summary`**（说明文字）和 **`html`**（纯 HTML）两个独立字符串变量，供下游 Output 节点分别引用。
>
> 拆分原理：**按 ```html 代码块的边界**拆——代码块之前的文本是 summary，代码块内部（剥掉 ```html / ``` 围栏、截到 `</html>`）是 html。比依赖 `===标记===` 更稳（不要求模型输出特定标记行）。

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

  const block = text.match(/```(?:html)?\s*\n([\s\S]*)\n?```/i);

  if (block) {
    const blockIdx = text.search(/```(?:html)?\s*\n/i);
    summary = blockIdx > 0 ? text.slice(0, blockIdx).trim() : '';

    let h = block[1].trim();
    const start = h.toLowerCase().indexOf('<!doctype html>');
    const end = h.toLowerCase().lastIndexOf('</html>');
    if (start >= 0 && end > start) h = h.slice(start, end + '</html>'.length);
    html = h;
  } else {
    html = text.trim();
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

    m = re.search(r'```(?:html)?\s*\n([\s\S]*)\n?```', text, re.IGNORECASE)
    if m:
        idx = re.search(r'```(?:html)?\s*\n', text, re.IGNORECASE)
        summary = text[:idx.start()].strip() if idx else ''
        h = m.group(1).strip()
    else:
        h = text.strip()

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

- 点 Code 节点「运行」，输入测试文本（开头一段功能说明 + 紧接 ```html 代码块），应输出：
  - `summary`：开头的功能说明文本（如"本页面包含：需求勾选器、智能推荐、方案对比表……"）。
  - `html`：以 `<!DOCTYPE html>` 开头、`</html>` 结尾的纯净 HTML（无围栏、无说明文字）。
- 若 `summary` 为空：检查 Generator 开头是否真的写了功能说明文字。
- 若 `html` 含围栏/说明文字：检查 Generator 是否正确用 ```html 代码块包裹、代码块是否闭合。
- 若两者都为空/错乱：检查 `raw` 是否正确绑定了 Generator 输出。
