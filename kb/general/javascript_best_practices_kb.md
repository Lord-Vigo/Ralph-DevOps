# JavaScript Best Practices

**Type:** General
**Maintained by:** Carl

---

## Code Quality Fundamentals

### Variables
- Use `const` by default; use `let` when reassignment is needed; never use `var`
  - `var` is function-scoped (not block-scoped), has hoisting behavior, and allows accidental redeclaration — all sources of bugs
- Give variables descriptive names — `photoObject` not `p`, `galleryIndex` not `arr`
- Constants that are truly fixed values (not reassigned): name in `UPPER_SNAKE_CASE` (e.g., `PRINT_SIZES`) — this signals intent to other developers

### Functions
- Keep functions small and single-purpose — if a function needs a paragraph to describe what it does, split it
- Name functions as verbs: `loadPhoto`, `renderCarousel`, `validateContactForm` — not `photo`, `carousel`, `form`
- Prefer named function declarations or named `const` arrow functions — anonymous functions are harder to read in stack traces:
  ```javascript
  // Prefer: named, readable in stack traces
  const handleOrderButtonClick = (event) => { ... };

  // Avoid: anonymous, shows as <anonymous> in errors
  btn.addEventListener('click', (event) => { ... });
  ```
- Limit function parameters to 3 or fewer — if you need more, pass an object:
  ```javascript
  // Hard to call correctly: what order are the args?
  buildOrderRow(title, size, price, isDisabled, ariaLabel)

  // Better: named, order doesn't matter
  buildOrderRow({ title, size, price, isDisabled, ariaLabel })
  ```

### Modules
- Each JS file is a module. Export only what needs to be consumed externally.
- Avoid module-level side effects (code that runs immediately on import other than registering event listeners that rely on `DOMContentLoaded`).
- Import `config` from `js/site.config.js` at the top of any module that uses configuration values — never hardcode email addresses or domain strings.

---

## Error Handling

- Use `try/catch` around `await` calls or wrap in `.catch()` — unhandled promise rejections are silent bugs in production
- Never swallow errors silently:
  ```javascript
  // Wrong: error disappears
  try { await loadPhoto(path); } catch (e) {}

  // Correct: log at minimum, handle gracefully
  try {
    await loadPhoto(path);
  } catch (e) {
    console.warn(`Could not load photo at ${path}:`, e.message);
    // show fallback UI or skip the photo
  }
  ```
- Use `Promise.allSettled` (not `Promise.all`) when loading a collection where individual failures should not abort the whole operation (e.g., `loadGallery`)
- Log errors with `console.warn` (recoverable) or `console.error` (unexpected) — `console.log` is for debug output only and should be removed before production

---

## DOM Manipulation

- **Prefer `textContent` over `innerHTML`** when inserting plain text — `textContent` does not parse HTML and is immune to XSS:
  ```javascript
  el.textContent = userSuppliedString;   // safe, always
  el.innerHTML = userSuppliedString;     // only safe with escapeHTML applied first
  ```
- **Batch DOM writes** — avoid reading then writing then reading then writing the DOM in a loop. Read all values first, then write all at once, to prevent layout thrashing (browser forced to recalculate layout on every read after a write)
- **Cache DOM references** — don't call `document.querySelector` in a loop or on every event; query once and store:
  ```javascript
  const carousel = document.querySelector('.carousel__stage');  // once
  // use `carousel` in all subsequent operations
  ```
- Use `element.setAttribute('hidden', '')` / `element.removeAttribute('hidden')` to show/hide elements semantically — not `element.style.display`
- When building a list of elements, use a `DocumentFragment` for large lists:
  ```javascript
  const fragment = document.createDocumentFragment();
  items.forEach(item => fragment.appendChild(buildItem(item)));
  container.appendChild(fragment);  // one DOM insertion instead of N
  ```

---

## Event Handling

- **Remove event listeners when they are no longer needed** — especially on elements that get replaced or destroyed (e.g., pop-out panel is rebuilt each time it opens; old listeners are auto-GC'd with the old elements, so no manual removal is needed in that case)
- **Use event delegation** for dynamically created elements — attach one listener to a stable parent:
  ```javascript
  // Good: one listener on stable parent
  container.addEventListener('click', (e) => {
    if (e.target.matches('.btn--order')) handleOrderClick(e);
  });

  // Fragile: listener lost when element is rebuilt
  document.querySelectorAll('.btn--order').forEach(btn =>
    btn.addEventListener('click', handleOrderClick)
  );
  ```
- Use `event.preventDefault()` explicitly when overriding browser defaults (form submit, link navigation) — don't rely on `return false`
- Don't use `event.stopPropagation()` casually — it silently breaks other listeners up the DOM tree. Only use it when genuinely necessary (e.g., preventing a modal close when clicking inside the modal)

---

## Async Patterns

- Always `await` async functions or attach `.catch()` — floating Promises cause silent failures
- Don't mix `async/await` and `.then()` chains in the same function — pick one style for readability
- For independent async operations, run in parallel:
  ```javascript
  // Sequential: slow (waits for each)
  const a = await fetchA();
  const b = await fetchB();

  // Parallel: fast (both run simultaneously)
  const [a, b] = await Promise.all([fetchA(), fetchB()]);
  ```
- For collections where partial failure is acceptable, use `Promise.allSettled`:
  ```javascript
  const results = await Promise.allSettled(paths.map(loadPhoto));
  const loaded = results.filter(r => r.status === 'fulfilled').map(r => r.value);
  ```

---

## Avoiding Common Pitfalls

### Never Use
| Avoid | Reason | Use Instead |
|-------|--------|-------------|
| `eval()` | Executes arbitrary strings as code — critical security risk | Parse data with JSON.parse or write explicit logic |
| `document.write()` | Overwrites the entire page if called after load | `element.innerHTML` or DOM methods |
| `with()` statement | Deprecated, ambiguous scoping | Explicit variable references |
| `var` | Function-scoped, hoisted, error-prone | `const` / `let` |
| `==` (loose equality) | Unexpected type coercion: `0 == ''` is `true` | `===` (strict equality) always |
| Inline event handlers `onclick="..."` | Mixes JS logic into HTML, can't be removed programmatically | `addEventListener` |
| Global variables | Pollute `window`, cause naming collisions across modules | Module-scoped variables and ES module exports |

### Type Coercion Traps
```javascript
// These are all true with ==, which surprises most developers:
0 == ''           // true
0 == false        // true
null == undefined // true
'1' == 1          // true

// Use === to avoid all of these:
0 === ''          // false — correct
```

### `this` Context
- Avoid using `this` in module-level code — ES modules don't have a meaningful `this`
- In event handlers, prefer `event.currentTarget` over `this` — arrow functions don't bind `this`, so `this` is unpredictable
- If you need to bind context, prefer arrow functions over `.bind()`:
  ```javascript
  class Gallery {
    handleClick = (event) => { this.doSomething(); }  // arrow: this is Gallery
  }
  ```

---

## Code Clarity

- **Comments explain WHY, not WHAT** — the code itself says what it does; comments explain why a non-obvious choice was made:
  ```javascript
  // Good: explains the why
  // Use Promise.allSettled instead of Promise.all so one missing info.md
  // doesn't abort the entire gallery load
  const results = await Promise.allSettled(paths.map(loadPhoto));

  // Unnecessary: restates the code
  // Loop through photos and load each one
  const results = await Promise.allSettled(paths.map(loadPhoto));
  ```
- Remove all `console.log` debug statements before considering an IOC complete
- Dead code (commented-out blocks that are no longer needed) should be deleted, not left commented — version control preserves history
- Magic numbers should be named constants or config values:
  ```javascript
  // Bad: what is 1500?
  setTimeout(restore, 1500);

  // Good: self-documenting
  const BUTTON_FEEDBACK_DURATION_MS = 1500;
  setTimeout(restore, BUTTON_FEEDBACK_DURATION_MS);
  ```
