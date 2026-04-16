# CSS Knowledge Base

**Type:** General
**Maintained by:** Carl

---

## CSS Custom Properties (Design Tokens)

Define all reusable values as custom properties on `:root`:

```css
:root {
  --color-accent: #c8a96e;
  --font-size-lg: 1.25rem;
  --space-4: 16px;
}
```

Reference them everywhere else:
```css
.btn { background: var(--color-accent); padding: var(--space-4); }
```

**Never hardcode color values, font sizes, or spacing outside of `variables.css`.** This makes global design changes a one-file edit.

---

## CSS Reset Pattern (Modern)

```css
*, *::before, *::after { box-sizing: border-box; }
* { margin: 0; padding: 0; }
html { font-size: 16px; }
body { min-height: 100vh; line-height: 1.5; }
img, video { max-width: 100%; display: block; }
input, button, textarea { font: inherit; }
```

---

## Flexbox Quick Reference

```css
/* Horizontal row, centered */
.row { display: flex; align-items: center; gap: 1rem; }

/* Vertical stack */
.stack { display: flex; flex-direction: column; gap: 1rem; }

/* Space between (header pattern) */
.header { display: flex; justify-content: space-between; align-items: center; }

/* Push footer to bottom */
body { display: flex; flex-direction: column; min-height: 100vh; }
main { flex: 1; }
```

---

## CSS Grid Quick Reference

```css
/* Auto-fit responsive grid */
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 1.5rem; }

/* Fixed 3-column */
.three-col { display: grid; grid-template-columns: 1fr 2fr 1fr; gap: 2rem; }

/* Full-width overlay (image + text) */
.overlay-grid { display: grid; }
.overlay-grid > * { grid-area: 1 / 1; }  /* all children on same cell */
```

---

## Responsive Design

### Mobile-First Approach
Write base styles for small screens, add `@media (min-width: ...)` for larger:
```css
/* Base (mobile) */
.card { font-size: 1rem; }

/* Tablet and up */
@media (min-width: 768px) { .card { font-size: 1.125rem; } }

/* Desktop */
@media (min-width: 1024px) { .card { font-size: 1.25rem; } }
```

### `clamp()` for Fluid Sizing
```css
/* Scales between 1.5rem (small) and 3rem (large), responsive in between */
h1 { font-size: clamp(1.5rem, 4vw, 3rem); }

/* Fluid padding */
.container { padding-inline: clamp(1rem, 5vw, 4rem); }
```

### Content Container Pattern
```css
.container {
  max-width: 1280px;
  margin-inline: auto;
  padding-inline: clamp(1rem, 5vw, 4rem);
}
```

---

## Object-Fit for Images

```css
/* Fill container, crop to fit (use for thumbnails, covers) */
.thumbnail { width: 100%; height: 100%; object-fit: cover; object-position: center; }

/* Fit inside container, preserve aspect ratio (use for lightboxes) */
.lightbox-img { max-width: 100%; max-height: 100%; object-fit: contain; }
```

---

## Aspect Ratio

```css
/* Modern approach (CSS 2021+, all modern browsers) */
.card-image { aspect-ratio: 3 / 4; overflow: hidden; }
.card-image img { width: 100%; height: 100%; object-fit: cover; }
```

---

## Sticky Positioning

```css
/* Sticky header */
header { position: sticky; top: 0; z-index: 100; }

/* Sticky sidebar (stays visible while content scrolls) */
.sidebar { position: sticky; top: calc(72px + 2rem); align-self: start; }
```

Note: `position: sticky` requires the parent not to have `overflow: hidden` or `overflow: auto` — this breaks stickiness.

---

## Focus Visible (Keyboard Navigation)

```css
/* Only show focus ring for keyboard users, not mouse clicks */
:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* Remove default focus outline only if :focus-visible is used */
:focus:not(:focus-visible) { outline: none; }
```

---

## CSS Transitions

```css
/* Always use transition on the base state, not the hover state */
.btn { background: var(--color-accent); transition: background 150ms ease; }
.btn:hover { background: var(--color-accent-hover); }

/* Multiple properties */
.card { transition: transform 300ms ease, box-shadow 300ms ease; }
.card:hover { transform: scale(1.02); box-shadow: 0 8px 30px rgba(0,0,0,0.2); }
```

---

## Modal Overlay Pattern

```css
.modal-overlay {
  position: fixed;
  inset: 0;                          /* shorthand for top/right/bottom/left: 0 */
  z-index: 300;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);       /* optional: frosted glass effect */
}

.modal {
  position: relative;
  max-width: 90vw;
  max-height: 90vh;
  overflow: auto;
  background: var(--color-surface);
  border-radius: 8px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
}
```

---

## Hiding Scrollbars While Keeping Scroll

```css
.scroll-container {
  overflow-x: auto;
  scrollbar-width: none;          /* Firefox */
}
.scroll-container::-webkit-scrollbar {
  display: none;                   /* Chrome, Safari, Edge */
}
```

---

## WCAG Color Contrast

- **Normal text** (< 18px or < 14px bold): minimum contrast ratio **4.5:1**
- **Large text** (≥ 18px or ≥ 14px bold): minimum contrast ratio **3:1**
- **UI components and graphics**: minimum **3:1**

Tool: Use a browser extension or online tool (e.g., WebAIM Contrast Checker) to verify contrast ratios after setting the color palette.
