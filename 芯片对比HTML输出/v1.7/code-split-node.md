# Code 节点：拆分 summary 与 html

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
