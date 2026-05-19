# Content Security Policy

CSP defends against XSS and data injection. Set via `<meta>` tag since Electron loads local files (no HTTP headers):

```html
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'">
```

## Rules

- Start strict, loosen only what's necessary
- Avoid `'unsafe-eval'` — bundlers sometimes emit `eval()`-like code (Webpack `devtool: 'eval'`). Use `devtool: 'source-map'` instead
- `new Function()` and `setTimeout('string')` are treated as eval; refactor to arrow functions
- Use `Content-Security-Policy-Report-Only` during development to detect violations without breaking the app
- CSP violations appear in DevTools console

## Bundler Considerations

| Bundler | Safe devtool | Unsafe devtool |
|---------|-------------|----------------|
| Webpack | `source-map` | `eval`, `cheap-eval-source-map` |
| Vite | `build.sourcemap: true` | default dev server is safe |
| esbuild | `--sourcemap` | n/a |

## Common Gotchas

- Third-party libraries may internally use `eval()` or `new Function()` — audit them
- Enabling `nodeIntegration: true` in renderer exposes Node's `eval` path — don't
- If CSP blocks something unexpectedly, check DevTools Console for the specific directive violation

> **Ref:** [Security: CSP](https://www.electronjs.org/docs/latest/tutorial/security) · [MDN CSP Guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
