# CSS Best Practices

**Type:** General
**Maintained by:** Carl

---

## Organization and Architecture

- **One source of truth:** All color, font, spacing, and z-index values live in `css/variables.css` as custom properties. No other file hardcodes these values.
- **File responsibility:** Each CSS file has a clear, narrow scope:
  - `reset.css` — baseline normalization only, no design decisions
  - `variables.css` — custom properties only, no rules
  - `layout.css` — structural layout (header, footer, container, grid, responsive)
  - `components.css` — shared component styles (buttons, modals, forms)
  - Page-specific files (e.g., `gallery.css`) — styles unique to one page
- **Order of imports matters:** Load in this order: reset → variables → layout → components → page-specific. Later files override earlier ones.
- **No `@import` inside CSS files** for performance — use `<link>` tags in HTML instead (CSS `@import` blocks parallel loading)

---

## Specificity Management

- Keep specificity as low as possible — prefer class selectors (`.btn`) over ID selectors (`#submit`) and element+class chains (`div.btn`)
- Avoid inline styles (`style="..."`) except for values that must be set dynamically from JavaScript (e.g., `element.style.height = ...`)
- Reserve `!important` for genuine utility classes (e.g., `[hidden] { display: none !important }`) — never use it to override specificity battles; fix the specificity instead
- Do not over-qualify selectors: write `.nav-link` not `nav ul li a.nav-link` — the shorter form is easier to override and faster to match

---

## Custom Properties (Design Tokens)

- Always define custom properties on `:root` — they cascade to all elements
- Provide a fallback value in `var()` when a custom property might be undefined:
  ```css
  color: var(--color-accent, #c8a96e);  /* fallback shown during FOUC */
  ```
- Group custom properties with comments (colors, typography, spacing, z-index, transitions) — makes the file scannable
- Custom properties in CSS are live — updating a value on `:root` in JavaScript updates all elements that reference it:
  ```javascript
  document.documentElement.style.setProperty('--color-accent', '#ff0');
  ```

---

## Layout

- **Use CSS Grid for two-dimensional layouts** (rows AND columns simultaneously), **Flexbox for one-dimensional** (row OR column)
- Avoid setting explicit heights on containers that hold text — let content define height; use `min-height` when a minimum is needed
- Never use `position: absolute` for layout that would work with normal flow — absolute positioning removes elements from flow and causes maintenance headaches
- Avoid magic numbers (e.g., `margin-top: 37px`) — use spacing variables or calculations that relate to known values
- For centering: `margin-inline: auto` centers a block; `display: grid; place-items: center` centers within a container

---

## Responsive Design

- **Mobile-first:** write base (small screen) styles first, use `@media (min-width: ...)` to add complexity — not the reverse
- Never use `px` for font sizes in media queries — use `em` so they respect browser font size preferences:
  ```css
  @media (min-width: 48em) { ... }  /* 768px equivalent */
  ```
- Avoid fixed-pixel widths on layouts — use `max-width`, percentages, `fr` units, or `clamp()`
- Test at real device sizes, not just by dragging the browser window — use browser DevTools device emulation
- Do not suppress `zoom` in the viewport meta tag (`user-scalable=no`) — this harms accessibility for low-vision users

---

## Naming Conventions

- Use **BEM-inspired** naming: `.block`, `.block__element`, `.block--modifier`
  - Block: `.gallery-card`
  - Element: `.gallery-card__image`, `.gallery-card__label`
  - Modifier: `.gallery-card--featured`, `.btn--primary`, `.btn--disabled`
- All class names use `kebab-case` (hyphens) — no `camelCase` or `snake_case`
- State classes: prefix with verb for dynamic states — `.nav-menu--open`, `.thumbnail--active`, `.btn--added`
- JavaScript hooks: if a class is only used by JS (not styled), prefix with `js-`: `.js-carousel-trigger` — this signals it should not be renamed during CSS refactors

---

## Performance

- Avoid `*` (universal selector) in complex selectors — it forces the browser to check every element
- Prefer `transform` and `opacity` for animations — they run on the GPU and don't trigger layout reflow:
  ```css
  /* Good: GPU-composited */
  .card:hover { transform: scale(1.02); opacity: 0.9; }

  /* Avoid for animation: triggers layout */
  .card:hover { width: 105%; margin-top: -5px; }
  ```
- Use `will-change: transform` sparingly on elements that will animate — it hints to the browser to promote the element to its own layer. Overuse wastes GPU memory.
- Avoid `@import` in CSS — use `<link>` tags in HTML for parallel loading
- Use `content-visibility: auto` on long off-screen sections to defer rendering (advanced; use only if performance profiling shows need)

---

## Transitions and Animations

- Always apply transitions to the **base state**, not just the `:hover` or `:active` state — otherwise there's no transition when leaving the state:
  ```css
  /* Correct */
  .btn { background: blue; transition: background 200ms ease; }
  .btn:hover { background: darkblue; }

  /* Wrong: no transition on mouse-out */
  .btn { background: blue; }
  .btn:hover { background: darkblue; transition: background 200ms ease; }
  ```
- Use the project's transition duration variables (`--transition-fast`, `--transition-normal`, `--transition-slow`) — consistent timing feels intentional
- Respect `prefers-reduced-motion` for users who are sensitive to motion:
  ```css
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  }
  ```

---

## Accessibility in CSS

- Never use `display: none` or `visibility: hidden` to hide content that should be readable by screen readers — use the `hidden` attribute or appropriate ARIA instead
- Visually hidden but screen-reader accessible (e.g., skip links, icon button labels):
  ```css
  .sr-only {
    position: absolute;
    width: 1px; height: 1px;
    padding: 0; margin: -1px;
    overflow: hidden;
    clip: rect(0,0,0,0);
    white-space: nowrap;
    border: 0;
  }
  ```
- Never use `outline: none` without providing an equivalent focus indicator — keyboard users rely on visible focus
- Color must not be the only means of conveying information (e.g., an error state must use more than just red — also use an icon or text)

---

## Common Mistakes to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| `outline: none` on focused elements | Use `:focus-visible` with a custom outline |
| `position: fixed` without `z-index` | Always pair fixed/sticky/absolute with explicit `z-index` from variables |
| Animating `width`, `height`, `top`, `left` | Animate `transform` and `opacity` instead |
| Setting `height: 100%` on a child without `height` on the parent | Use `min-height: 100vh` on the parent or flexbox instead |
| Duplicating color values across files | All colors go in `--variables.css`; reference via `var()` everywhere else |
| Using `px` for media query breakpoints | Use `em` or `rem` to respect browser font settings |
