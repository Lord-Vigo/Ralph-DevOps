# HTML5 Best Practices

**Type:** General
**Maintained by:** Carl

---

## Document Structure

- One `<h1>` per page — it is the primary heading of that page's content, not the site name (the site name belongs in `<title>` and the logo/nav)
- Heading levels must be sequential and not skip ranks: `h1 → h2 → h3`. Never skip from `h1` to `h3` for visual sizing — use CSS instead
- `<main>` appears exactly once per page; it wraps the page's unique content (not the header or footer)
- `<section>` should have a heading; if you can't give it one, use `<div>` instead
- Never use `<table>` for layout — only for tabular data (e.g., the order summary table)
- Never use `<br>` for spacing — use CSS margin/padding

---

## Forms

- Every `<input>` and `<textarea>` must have an associated `<label>` using `for`/`id` pairing — never use `placeholder` as a substitute for a label
- Required fields: use the `required` attribute for semantics even when using custom JS validation with `novalidate`
- Input types: use the most specific type available (`type="email"`, `type="tel"`) — this triggers appropriate mobile keyboards and provides implicit validation hints
- Group related form controls with `<fieldset>` and `<legend>` (e.g., a shipping address block)
- Associate error messages with their input: `<input aria-describedby="error-email">` + `<span id="error-email" role="alert">`

---

## Links and Buttons

- Use `<a href="...">` for navigation (goes somewhere) and `<button>` for actions (does something) — never swap them
- Never use `<a>` without an `href` to create a clickable element — use `<button>` instead
- `<button>` inside a `<form>` defaults to `type="submit"` — always set `type="button"` on buttons that should not submit the form
- External links that open in a new tab: always add `rel="noopener noreferrer"` to prevent the opened page from accessing `window.opener`
  ```html
  <a href="https://external.com" target="_blank" rel="noopener noreferrer">Link</a>
  ```
- Avoid `javascript:` URIs in `href` — use event listeners instead

---

## Images

- Every `<img>` must have an `alt` attribute — empty string (`alt=""`) for purely decorative images, descriptive text for meaningful images
- Specify `width` and `height` on `<img>` to prevent layout shift (CLS) — the browser reserves space before the image loads
- Use `loading="lazy"` for below-the-fold images; use `loading="eager"` (or omit `loading`) for hero/above-the-fold images
- Prefer `<picture>` with `<source>` for art direction or format negotiation (e.g., serving WebP where supported) — not required for initial implementation but beneficial for performance
- Do not use `<img>` for decorative backgrounds — use CSS `background-image` instead

---

## Performance

- Place `<link rel="stylesheet">` in `<head>` — stylesheets in `<body>` cause flash of unstyled content
- Place `<script>` tags at the end of `<body>` OR use `type="module"` (deferred automatically) OR use `defer` attribute — scripts block HTML parsing if placed in `<head>` without these
- Minimize `<link rel="preload">` usage — overuse causes the browser to download resources eagerly that may not be needed
- Use `<link rel="preconnect">` for external font hosts (e.g., Google Fonts) to reduce connection latency:
  ```html
  <link rel="preconnect" href="https://fonts.googleapis.com">
  ```

---

## Meta Tags

- Always include `<meta name="viewport" content="width=device-width, initial-scale=1.0">` — without it, mobile browsers render at desktop width and then scale down
- Always include `<meta charset="UTF-8">` as the first tag in `<head>` — must appear within the first 1024 bytes of the document
- `<meta name="description">` is optional but beneficial for SEO and social sharing

---

## Avoiding Common Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| Using `<div>` for everything | Use semantic elements (nav, main, article, section, etc.) |
| `<b>` and `<i>` for styling | Use CSS `font-weight` and `font-style`; use `<strong>` and `<em>` for semantic emphasis |
| `<br><br>` for paragraph spacing | Use `<p>` elements with CSS margin |
| `onclick="..."` inline handlers | Use `addEventListener` in JS files |
| Empty `href="#"` on links | Use `<button>` for actions; give links a real destination |
| Nesting `<a>` inside `<a>` | Invalid HTML — restructure the markup |
| `<script>` without `type="module"` blocking render | Use `type="module"` or `defer` |

---

## Validation

- Validate HTML using the W3C Markup Validator (or browser developer tools) before finalizing any page
- Common validation errors to watch for: unclosed tags, duplicate `id` attributes, invalid nesting (e.g., `<p>` inside `<p>`), missing `alt` on `<img>`
