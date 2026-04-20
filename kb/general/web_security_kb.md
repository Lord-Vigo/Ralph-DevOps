# Web Security Best Practices

**Type:** General
**Maintained by:** Carl
**Applies to:** HTML5, CSS, JavaScript — all client-side web projects

---

## Cross-Site Scripting (XSS)

XSS is the most common and impactful security risk for client-side web applications. Dynamic content from any source (user input, APIs, configuration files) must be escaped before inserting into the DOM.

### The Rule
**Never insert untrusted or dynamic strings directly into `innerHTML`, `outerHTML`, or attribute values without escaping.**

### Safe DOM Insertion Methods

| Method | XSS Safe? | Notes |
|--------|-----------|-------|
| `element.textContent = str` | ✅ Yes | Treats string as text, never parses HTML |
| `element.setAttribute('alt', str)` | ✅ Yes | Browser handles attribute encoding |
| `element.innerHTML = escapeHTML(str)` | ✅ Yes | Safe only if `escapeHTML` is applied first |
| `element.innerHTML = str` | ❌ No | Never do this with dynamic data |
| `element.insertAdjacentHTML('...', str)` | ❌ No | Same risk as innerHTML |

### Required Helper Functions

Every project that uses `innerHTML` with dynamic data must implement these:

```javascript
function escapeHTML(str) {
  if (str === null || str === undefined) return '';
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

function escapeAttr(str) {
  // Same escaping — safe for use in HTML attribute values
  return escapeHTML(str);
}
```

### Practical Application

```javascript
// Building an HTML string — escape all dynamic values:
const html = `
  <h2 class="title">${escapeHTML(item.title)}</h2>
  <p class="description">${escapeHTML(item.description)}</p>
  <button data-value="${escapeAttr(item.value)}"
          aria-label="${escapeAttr(ariaLabel)}">
    Action
  </button>
`;
container.innerHTML = html;  // safe because all dynamic values are escaped

// Even simpler — use textContent for plain text nodes:
titleEl.textContent = item.title;  // no escaping needed at all
```

### What XSS Can Do
If untrusted input is inserted unescaped into the DOM, an attacker can:
- Execute arbitrary JavaScript in the visitor's browser
- Steal session data or form input
- Redirect the visitor to a malicious site
- Deface the page

---

## URL Safety

### Avoid `javascript:` URLs
Never use `javascript:` in `href` or any other URL attribute — it executes JavaScript when the link is activated and bypasses Content Security Policy:
```html
<!-- Never do this -->
<a href="javascript:doSomething()">Click</a>

<!-- Correct approach -->
<button type="button" id="my-btn">Click</button>
<script>document.getElementById('my-btn').addEventListener('click', doSomething);</script>
```

### URL Encoding
Always encode dynamic values before inserting them into URLs:
```javascript
// Wrong: special chars in subject or body break the URI
const mailto = `mailto:${email}?subject=${subject}&body=${body}`;

// Correct: encodeURIComponent handles all special characters
const mailto = `mailto:${email}`
  + `?subject=${encodeURIComponent(subject)}`
  + `&body=${encodeURIComponent(body)}`;
```

### Open Redirect
If building any link with a user-supplied destination URL, validate it points to your own domain — don't allow arbitrary redirect destinations.

---

## External Links

Always add `rel="noopener noreferrer"` to links with `target="_blank"`:
```html
<a href="https://example.com" target="_blank" rel="noopener noreferrer">Link</a>
```
- **`noopener`**: prevents the opened page from accessing `window.opener` — without this, the opened page can redirect your page
- **`noreferrer`**: prevents sending the HTTP `Referer` header to the destination

---

## Content Security Policy (CSP)

CSP is an HTTP response header (or `<meta>` tag) that restricts what resources the browser will load and execute. It is the primary defense against XSS.

### CSP as a Meta Tag
For static sites, a minimal CSP can be set as a `<meta>` tag:
```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data:; script-src 'self';">
```

**What this policy means:**
- `default-src 'self'` — only load resources from the same origin by default
- `style-src 'self' https://fonts.googleapis.com` — allow stylesheets from self and Google Fonts
- `font-src 'self' https://fonts.gstatic.com` — allow fonts from self and Google Fonts CDN
- `img-src 'self' data:` — allow images from self and inline data URIs
- `script-src 'self'` — only allow scripts from the same origin (no inline scripts or external JS)

### CSP and Inline Scripts
A `'self'`-only `script-src` **blocks inline `<script>` blocks** and `onclick="..."` handlers. This forces all JavaScript into external `.js` files. If inline scripts are necessary, use a `nonce`:
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self' 'nonce-RANDOM_VALUE'">
<script nonce="RANDOM_VALUE">/* inline script */</script>
```

### CSP and `eval()`
Any `script-src` without `'unsafe-eval'` blocks `eval()`, `new Function()`, and `setTimeout('string', ...)`. This is desirable — see JavaScript Best Practices for why `eval()` should never be used.

---

## Sensitive Data Handling

- **Never put secrets in client-side code.** Any JavaScript sent to the browser is publicly readable — it may contain public configuration (email addresses, site names) but must never contain passwords, API keys, or private tokens.
- **Minimize data retention.** Only hold data in memory for as long as needed. Avoid writing sensitive data to `localStorage`, `sessionStorage`, cookies, or sending to servers unless necessary.
- **Form submission handling.** When forms submit data externally (e.g., via mailto URI), ensure data is not simultaneously sent to any server.

---

## HTTPS Requirements

Certain Web APIs require HTTPS (or `localhost`) to work:
- **`navigator.clipboard.writeText()`** — requires HTTPS; provide a fallback on HTTP
- **`fetch()`** — works on HTTP for same-origin requests but some security headers only apply over HTTPS
- **Service Workers** — require HTTPS

Production deployments must use HTTPS. Let's Encrypt provides free TLS certificates.

---

## `eval()` — Never Use

`eval(str)` executes an arbitrary string as JavaScript. It is a critical security vulnerability:
- If any part of `str` comes from user input or external data, it can execute attacker-controlled code
- It bypasses CSP `script-src` restrictions
- It is slower than equivalent explicit code (disables JavaScript engine optimizations)

**Never use:** `eval()`, `new Function(str)`, `setTimeout('string', ms)`, `setInterval('string', ms)`

```javascript
// Never
eval(userInput);
new Function('return ' + expression)();

// For parsing structured data, use JSON.parse:
const data = JSON.parse(jsonString);  // safe, not eval
```

---

## Dependency Security

For client-side projects, prefer minimal dependencies:
- Each dependency introduces a supply chain attack surface
- CDN scripts can be compromised or deliver different code over time

If dependencies are needed:
- Prefer CDN links with **Subresource Integrity (SRI)** hashes:
  ```html
  <script src="https://cdn.example.com/lib.js"
          integrity="sha384-[HASH]"
          crossorigin="anonymous"></script>
  ```
- Audit dependencies before adding: check license, maintenance status, and known vulnerabilities

---

## Clickjacking

Prevent the site from being embedded in `<iframe>` by another site (clickjacking) via an HTTP response header:
```
X-Frame-Options: DENY
```
Or via CSP:
```
Content-Security-Policy: frame-ancestors 'none';
```

This must be set server-side (server configuration, not a `<meta>` tag).

---

## Security Checklist

Before marking any feature complete, verify:

- [ ] No dynamic data is inserted into `innerHTML` without `escapeHTML()`
- [ ] No dynamic data is inserted into HTML attributes without `escapeAttr()`
- [ ] No `eval()`, `new Function()`, or string-based setTimeout/setInterval
- [ ] No `javascript:` URIs in `href` or event handlers
- [ ] All external `target="_blank"` links have `rel="noopener noreferrer"`
- [ ] All URL parameters are `encodeURIComponent()`-encoded
- [ ] No secrets (passwords, tokens) in any JS file