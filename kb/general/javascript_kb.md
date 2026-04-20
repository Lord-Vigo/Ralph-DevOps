# JavaScript Knowledge Base

**Type:** General
**Maintained by:** Carl

---

## ES Modules

Use ES modules for all JavaScript. Each file is a module:

```javascript
// Exporting
export function myFunction() { ... }
export const MY_CONST = 42;
export default class MyClass { ... }

// Importing
import { myFunction, MY_CONST } from './utils.js';
import MyClass from './MyClass.js';
```

**In HTML**: `<script type="module" src="js/main.js"></script>` — modules are deferred automatically.

---

## Async/Await and Fetch

```javascript
// Fetch a resource and get its text
async function loadFile(path) {
  const response = await fetch(path);
  if (!response.ok) {
    throw new Error(`Failed to load ${path}: ${response.status}`);
  }
  return response.text();
}

// Usage with proper error handling
try {
  const text = await loadFile('data/config.json');
} catch (err) {
  console.warn('Could not load file:', err.message);
}
```

**`Promise.allSettled` vs `Promise.all`:**
- `Promise.all`: rejects immediately if any promise rejects — use when ALL must succeed
- `Promise.allSettled`: waits for all, collects results including rejections — use when partial failure is acceptable (e.g., loading items where one might be missing)

```javascript
const results = await Promise.allSettled(items.map(item => loadItem(item)));
const loaded = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value);
```

---

## DOM Manipulation

```javascript
// Selecting elements
const el = document.getElementById('my-id');
const els = document.querySelectorAll('.my-class');

// Creating and inserting elements (preferred for complex HTML)
const div = document.createElement('div');
div.className = 'card';
div.textContent = 'Hello';
parent.appendChild(div);

// Building HTML strings (faster for large blocks, use escapeHTML!)
container.innerHTML = buildHTML(data);  // MUST escape user/dynamic data

// Toggling hidden attribute
el.setAttribute('hidden', '');       // hide
el.removeAttribute('hidden');        // show
const isHidden = el.hasAttribute('hidden');

// Toggling CSS classes
el.classList.add('active');
el.classList.remove('active');
el.classList.toggle('active');
el.classList.contains('active');     // returns boolean
```

---

## XSS Prevention (innerHTML Safety)

When using `innerHTML` with dynamic data, always escape:

```javascript
function escapeHTML(str) {
  if (!str) return '';
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

function escapeAttr(str) {
  return escapeHTML(str);  // same escaping works for attribute values
}

// Safe usage
el.innerHTML = `<p>${escapeHTML(userInput)}</p>`;
el.innerHTML = `<img alt="${escapeAttr(title)}">`;
```

---

## Event Handling

```javascript
// Add listener
element.addEventListener('click', handler);

// Remove listener (requires named function reference, not anonymous)
element.removeEventListener('click', handler);

// Event delegation (handle events from many children at parent level)
list.addEventListener('click', (event) => {
  const item = event.target.closest('.list-item');
  if (item) { handleItemClick(item); }
});

// Custom events
document.dispatchEvent(new CustomEvent('app:stateChanged', { detail: { count: 3 } }));
document.addEventListener('app:stateChanged', (event) => {
  console.log(event.detail.count);
});
```

---

## Focus Management

```javascript
// Move focus to an element (modal open)
modalCloseBtn.focus();

// Return focus after modal close
const previousFocus = document.activeElement;
// ... open modal ...
// ... close modal ...
previousFocus.focus();

// Focus trap within a modal
function trapFocus(modalEl, event) {
  if (event.key !== 'Tab') return;
  const focusable = modalEl.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last = focusable[focusable.length - 1];

  if (event.shiftKey) {
    if (document.activeElement === first) { event.preventDefault(); last.focus(); }
  } else {
    if (document.activeElement === last) { event.preventDefault(); first.focus(); }
  }
}
```

---

## Keyboard Events

```javascript
document.addEventListener('keydown', (event) => {
  switch (event.key) {
    case 'Escape':   closeModal(); break;
    case 'ArrowLeft':  showPrev(); break;
    case 'ArrowRight': showNext(); break;
    case 'Enter':
    case ' ':        // space bar
      activateSelected(); break;
  }
});
```

---

## URL and Clipboard APIs

```javascript
// Mailto URI — use config values, never hardcode
import { config } from './config.js';
const mailto = `mailto:${config.contactEmail}`
  + `?subject=${encodeURIComponent(config.emailSubject)}`
  + `&body=${encodeURIComponent(messageBody)}`;
window.location.href = mailto;

// Clipboard (modern, requires HTTPS or localhost)
navigator.clipboard.writeText('text to copy')
  .then(() => { console.log('Copied!'); })
  .catch(() => { /* fallback: select the text */ });
```

---

## `DOMContentLoaded` vs `load`

```javascript
// DOMContentLoaded: fires when HTML is parsed and DOM is ready (before images/stylesheets)
document.addEventListener('DOMContentLoaded', init);

// load: fires when everything (images, stylesheets) is fully loaded — rarely needed
window.addEventListener('load', init);

// With ES modules: the script is deferred so DOM is already ready
// DOMContentLoaded is still good practice for clarity
```

---

## String Utilities

```javascript
// Slug to title (e.g., 'desert-light' → 'Desert Light')
function slugToTitle(slug) {
  return slug
    .split(/[-_]/)
    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ');
}

// Format price
function formatPrice(price) {
  if (price === null || price === undefined || isNaN(price)) return '—';
  return '$' + (Number.isInteger(price) ? price : price.toFixed(2));
}

// Replace 'x' with × for dimensions
function formatSize(size) {
  return size.replace('x', '\u00D7');  // × U+00D7
}
```

---

## `window.location` for Page Detection

```javascript
// Get current page filename
const page = window.location.pathname.split('/').pop();
// e.g., 'gallery.html'

// Check if on a specific page
const isGallery = window.location.pathname.includes('gallery');
const isOrdering = window.location.pathname.includes('order');
```