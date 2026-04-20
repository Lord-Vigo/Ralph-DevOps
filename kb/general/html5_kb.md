# HTML5 Knowledge Base

**Type:** General
**Maintained by:** Carl

---

## Semantic Structure

Use semantic HTML5 elements to convey meaning:

| Element | Use Case |
|---------|----------|
| `<header>` | Site header or section header |
| `<nav>` | Navigation blocks — add `aria-label` to distinguish multiple navs |
| `<main>` | Primary page content — one per page |
| `<article>` | Self-contained content (bio, post, product) |
| `<section>` | Thematic grouping within a page |
| `<aside>` | Tangentially related content |
| `<footer>` | Site footer or section footer |
| `<figure>` / `<figcaption>` | Images with captions |
| `<blockquote>` | Extended quotations |

Avoid using `<div>` where a semantic element is appropriate.

---

## Document Head Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Page Title | Site Name</title>
  <link rel="stylesheet" href="css/reset.css">
  <link rel="stylesheet" href="css/variables.css">
  <link rel="stylesheet" href="css/layout.css">
  <link rel="stylesheet" href="css/components.css">
</head>
```

---

## ES Modules in HTML

Use `type="module"` for script tags to enable ES6 import/export:
```html
<script type="module" src="js/main.js"></script>
```
Modules are deferred automatically — they do not block rendering. No need for `defer` attribute when using `type="module"`.

---

## The `hidden` Attribute

The `hidden` attribute is semantically meaningful for hiding content that is not relevant yet (modals, conditional sections). Prefer it over CSS `display: none` for toggling visibility in JavaScript-controlled UI:

```html
<!-- hidden: element not shown, not in accessibility tree -->
<div id="modal" hidden>...</div>

<!-- show it -->
element.removeAttribute('hidden');

<!-- hide it -->
element.setAttribute('hidden', '');
```

Add `[hidden] { display: none !important; }` to CSS to ensure `hidden` always wins over other display rules.

---

## ARIA Patterns

| Pattern | Implementation |
|---------|----------------|
| Current page nav link | `aria-current="page"` on active link |
| Modal dialog | `role="dialog"` + `aria-modal="true"` + `aria-label` or `aria-labelledby` |
| Alert/error message | `role="alert"` — announced automatically when made visible |
| Interactive `<div>` | Add `role="button"` + `tabindex="0"` + keyboard handlers |
| List without bullets | `<ul role="list">` — removes default list role but keeps semantics |

---

## Image Best Practices

```html
<!-- Always provide alt text -->
<img src="photo.jpg" alt="Descriptive text about the image">

<!-- Decorative images (no alt needed) -->
<img src="decoration.svg" alt="">

<!-- Responsive images -->
<img src="photo.jpg" alt="..." loading="lazy" width="800" height="600">
```

- Specify `width` and `height` on `<img>` to reserve layout space and prevent layout shift (CLS)
- Use `loading="lazy"` for below-the-fold images
- Use `loading="eager"` for above-the-fold/hero images

---

## Form Accessibility

```html
<!-- Always associate labels with inputs -->
<label for="email">Email Address</label>
<input type="email" id="email" name="email" autocomplete="email" required>

<!-- Use novalidate to take over validation -->
<form novalidate>

<!-- Error messages with role="alert" -->
<span role="alert" id="error-email" hidden>Please enter a valid email.</span>
```

---

## Common Unicode Characters

| Character | Unicode | HTML Entity | Usage |
|-----------|---------|-------------|-------|
| × (multiply) | U+00D7 | `&times;` | Dimensions: 10×8 |
| · (middle dot) | U+00B7 | `&middot;` | Separators |
| ✓ (check) | U+2713 | `&#x2713;` | Success states |
| → (arrow) | U+2192 | `&rarr;` | Navigation |
| ← (arrow) | U+2190 | `&larr;` | Navigation |
| … (ellipsis) | U+2026 | `&hellip;` | Truncation |

Preferred: use Unicode characters directly in HTML/JS source files (UTF-8 encoding). HTML entities are acceptable alternatives.