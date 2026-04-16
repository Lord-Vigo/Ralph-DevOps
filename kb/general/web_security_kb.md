# Web Security Best Practices

**Type:** General
**Maintained by:** Carl
**Applies to:** HTML5, CSS, JavaScript — all client-side web projects

---

## Cross-Site Scripting (XSS)

XSS is the most relevant security risk for this project because dynamic content (photo titles, descriptions, prices) from `info.md` files is injected into the DOM. Even though `info.md` is admin-controlled, defending against XSS is non-negotiable.

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

These are implemented in `js/content.js` for this project. Import or duplicate where needed.

### Practical Application

```javascript
// Building an HTML string — escape all dynamic values:
const html = `
  <h2 class="title">${escapeHTML(photo.title)}</h2>
  <p class="info">${escapeHTML(photo.info)}</p>
  <button data-title="${escapeAttr(photo.title)}"
          aria-label="${escapeAttr(ariaLabel)}">
    Order Print
  </button>
`;
container.innerHTML = html;  // safe because all dynamic values are escaped

// Even simpler — use textContent for plain text nodes:
titleEl.textContent = photo.title;  // no escaping needed at all
```

### What XSS Can Do
If user-controlled input is inserted unescaped into the DOM, an attacker can:
- Execute arbitrary JavaScript in the visitor's browser
- Steal session data or form input (e.g., the order form contact details)
- Redirect the visitor to a malicious site
- Deface the page

For this project the attack surface is small (admin-controlled `info.md` files), but good practice protects against accidental injection and future content sources.

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
// Wrong: special chars in name or body break the mailto URI
const mailto = `mailto:${email}?subject=Order - ${name}&body=${body}`;

// Correct: encodeURIComponent handles all special characters
const mailto = `mailto:${config.ORDER_EMAIL}`
  + `?subject=${encodeURIComponent(subject)}`
  + `&body=${encodeURIComponent(body)}`;
```

### Open Redirect
If building any link with a user-supplied destination URL, validate it points to your own domain — don't allow arbitrary redirect destinations. (Not directly applicable to this project, but important to know.)

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

### For This Project (Static Site)
A minimal CSP can be set as a `<meta>` tag in each page's `<head>`:
```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data:; script-src 'self';">
```

**What this policy means:**
- `default-src 'self'` — only load resources from the same origin by default
- `style-src 'self' https://fonts.googleapis.com` — allow stylesheets from self and Google Fonts
- `font-src 'self' https://fonts.gstatic.com` — allow fonts from self and Google Fonts CDN
- `img-src 'self' data:` — allow images from self and inline data URIs (for SVG placeholders)
- `script-src 'self'` — only allow scripts from the same origin (no inline scripts or external JS)

### CSP and Inline Scripts
A `'self'`-only `script-src` **blocks inline `<script>` blocks** and `onclick="..."` handlers. This is good — it forces all JavaScript into external `.js` files, which is already this project's approach. If inline scripts are absolutely necessary, use a `nonce`:
```html
<meta http-equiv="Content-Security-Policy" content="script-src 'self' 'nonce-RANDOM_VALUE'">
<script nonce="RANDOM_VALUE">/* inline script */</script>
```
For a static site, avoid inline scripts entirely and this complication is unnecessary.

### CSP and `eval()`
Any `script-src` without `'unsafe-eval'` blocks `eval()`, `new Function()`, and `setTimeout('string', ...)`. This is desirable — see JavaScript Best Practices for why `eval()` should never be used.

---

## Sensitive Data Handling

- **Never put secrets in client-side code.** Configuration in `js/site.config.js` is publicly readable — it may contain the order email address, site name, and domain, but must never contain passwords, API keys, or private tokens.
- **Order data is session-only.** The cart holds buyer name, email, phone, and address in JavaScript memory only — it is never written to `localStorage`, `sessionStorage`, cookies, or sent to a server. This matches the stated design (no data retention).
- **Form data is not retained.** The contact form data is assembled into a `mailto:` URI and handed to the email client — it is not sent to any server by the site. Once the email is sent (by the visitor's email client), the site has no record of it.

---

## HTTPS Requirements

Certain Web APIs require HTTPS (or `localhost`) to work:
- **`navigator.clipboard.writeText()`** — requires HTTPS; falls back to `.select()` on HTTP
- **`fetch()`** — works on HTTP for same-origin requests but some security headers only apply over HTTPS
- **Service Workers** — require HTTPS (not used in this project but worth knowing)

Production deployment of this site must use HTTPS. Let's Encrypt provides free TLS certificates.

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

This project intentionally has **zero third-party JavaScript dependencies** (no npm, no CDN scripts beyond an optional web font). This is a significant security advantage:
- No supply chain attack surface (no `npm install` that could pull malicious packages)
- No CDN script that could be compromised or deliver different code over time
- No versioning concerns

If a dependency is ever added in the future:
- Prefer CDN links with **Subresource Integrity (SRI)** hashes so the browser verifies the file hasn't changed:
  ```html
  <script src="https://cdn.example.com/lib.js"
          integrity="sha384-[HASH]"
          crossorigin="anonymous"></script>
  ```
- Audit the dependency before adding: check its license, maintenance status, and known vulnerabilities

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
This must be set server-side (Apache/Nginx config, not a `<meta>` tag — `X-Frame-Options` as a meta tag is ignored by browsers). Mention to the hosting admin when deploying.

---

## Security Checklist for Each IOC

Before marking any IOC complete, Chad should verify:

- [ ] No dynamic data is inserted into `innerHTML` without `escapeHTML()`
- [ ] No dynamic data is inserted into HTML attributes without `escapeAttr()`
- [ ] No `eval()`, `new Function()`, or string-based setTimeout/setInterval
- [ ] No `javascript:` URIs in `href` or event handlers
- [ ] All external `target="_blank"` links have `rel="noopener noreferrer"`
- [ ] All URL parameters (e.g., mailto body) are `encodeURIComponent()`-encoded
- [ ] No secrets (passwords, tokens) in any JS file
- [ ] `config.ORDER_EMAIL` used for email address — not hardcoded
