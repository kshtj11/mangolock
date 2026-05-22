# Architecture — mango launcher prototype

For replicating this prototype or building new ones in the same style.

---

## Stack

- **Plain HTML/CSS/JS** — no framework, no bundler, no dependencies
- Single `index.html` (self-contained, works in preview panels over file://)
- `js/` and `style.css` exist as source reference but are NOT loaded — everything is inlined
- No `type="module"` — ES modules break on file:// (preview panels). Use a global namespace (`window.M`) if splitting files

---

## Canvas

```css
#app {
  position: fixed; top: 0; left: 0;
  width: 480px; height: 480px;
  overflow: hidden;
  touch-action: none; /* critical — disables all browser gesture handling */
}
```

All screens are `position: absolute; inset: 0` children of `#app`. Transitions are CSS `transform` only.

---

## Visual Language

**Philosophy:** near-black backgrounds, dark tiles slightly lighter than bg, no borders except subtle search bar, large border-radius, no text labels on tiles.

| Token | Value | Usage |
|---|---|---|
| Background (deep) | `#050505` / `#080808` | Homepage, Recents |
| Background (mid) | `#0d0d0d` / `#0e0e0e` | Lock, App List |
| Background (drawer) | `#111` | Settings Drawer |
| Tile | `#1e1e1e` | App icons, QS tiles |
| Card | `#252525` | Recents app cards |
| Border subtle | `#2c2c2c` | Search bar only |
| Text dim | `#555` / `#666` | Inactive tabs, icons |
| Text active | `#ccc` | Status bar, active tab |
| Border-radius (icon) | `24px` | App icons (103×103) |
| Border-radius (card) | `24px` | Recents cards |
| Border-radius (tile) | `20px` | QS tiles |
| Border-radius (search) | `22px` | Search bar (full pill) |
| Border-radius (tab) | `10px` | Settings tabs |

**Lock screen dot grid:**
```css
background-color: #0d0d0d;
background-image: radial-gradient(circle, #2a2a2a 1px, transparent 1px);
background-size: 18px 18px;
```

---

## Screen System

Each screen is `position: absolute; inset: 0` with a CSS transition on `transform`. State is managed by JS setting `el.style.transform` directly — no class toggling for screen state.

```
z-index stack (within #app):
  20  Settings Drawer   ← slides from top
  10  Recents           ← slides from bottom
   3  Lock              ← slides up on unlock
   2  App List          ← slides in from right/bottom
   1  Homepage          ← slides in from left
```

**Transition:** `transform 0.4s cubic-bezier(0.4, 0, 0.2, 1)` on `.screen`

**Default transforms (off-screen positions):**
```
Lock:            translateY(0)       → unlock → translateY(-100%)
App List:        translateY(100%)    → unlock → translateX(0)
Homepage:        translateX(-100%)   → nav    → translateX(0)
Recents:         translateY(100%)    → open   → translateY(0)
Settings Drawer: translateY(-100%)   → open   → translateY(0)
```

---

## Gesture System

**Rule:** All gestures use Pointer Events. Never mix mouse + touch + pointer events on the same element.

### Core pattern for every gesture zone

```js
el.addEventListener('pointerdown', e => {
  startX = e.clientX; startY = e.clientY;
  el.setPointerCapture(e.pointerId); // keeps tracking when finger leaves element
  e.preventDefault();
});
el.addEventListener('pointerup', e => {
  const dx = e.clientX - startX;
  const dy = e.clientY - startY;
  if (/* threshold met */) doAction();
});
el.addEventListener('pointercancel', () => {
  // always clean up drag state — iOS fires this on system interruptions
  isDragging = false;
});
```

### Gesture zones (invisible overlays, `position: absolute`)

| Zone | Size | z-index | Purpose |
|---|---|---|---|
| `.top-zone` | 100% × 30px, top | 30 | Swipe-down → Settings |
| `.bottom-zone` | 100% × 44px, bottom | 20 | Swipe-up / long-press → Recents |
| `#edge-zone` | 40px × 100%, left | 10 | Swipe-right → Homepage |

**Long press pattern:**
```js
let moved = false, timer = null;
el.addEventListener('pointerdown', e => {
  moved = false;
  el.setPointerCapture(e.pointerId);
  timer = setTimeout(() => { if (!moved) trigger(); }, 500);
});
el.addEventListener('pointermove', e => {
  if (Math.abs(e.clientX - sx) > 8 || Math.abs(e.clientY - sy) > 8) {
    moved = true; clearTimeout(timer);
  }
});
el.addEventListener('pointerup', e => {
  clearTimeout(timer);
  if (sy - e.clientY > 30) trigger(); // also fires on swipe-up
});
```

### Drag-scroll with momentum

Used on: App List grid, Recents cover flow.

```js
el.addEventListener('pointerdown', e => {
  isDragging = true;
  dragStartX = e.clientX;
  scrollAtStart = el.scrollLeft;
  lastX = e.clientX; lastT = performance.now(); velocity = 0;
  el.setPointerCapture(e.pointerId);
  e.preventDefault();
});
el.addEventListener('pointermove', e => {
  if (!isDragging) return;
  const now = performance.now();
  velocity = (lastX - e.clientX) / (now - lastT || 1) * 16; // px per frame
  lastX = e.clientX; lastT = now;
  el.scrollLeft = scrollAtStart + (dragStartX - e.clientX);
});
el.addEventListener('pointerup', () => {
  isDragging = false;
  applyMomentum();
});

function applyMomentum() {
  velocity *= 0.92; // friction
  if (Math.abs(velocity) < 0.3) return;
  el.scrollLeft += velocity;
  requestAnimationFrame(applyMomentum);
}
```

### Cover flow (Recents)

Cards are `position: absolute`. A float `progress` (0 to n-1) controls which card is centered. Drag updates `progress` in real time, release snaps to nearest integer with eased animation.

```js
const CARD_W = 300, CARD_OFFSET = 240, VIEW_CX = 240;
const SCALE_BIG = 1.0, SCALE_SMALL = 0.72;

function updateCards() {
  cards.forEach((card, i) => {
    const dist = i - progress;
    const scale = SCALE_BIG - (SCALE_BIG - SCALE_SMALL) * Math.min(Math.abs(dist), 1);
    card.style.left = (VIEW_CX + dist * CARD_OFFSET - CARD_W / 2) + 'px';
    card.style.transform = `scale(${scale})`;
  });
}

// Snap with ease-out cubic
function snapTo(target) {
  const from = progress, t0 = performance.now(), dur = 320;
  (function step(now) {
    const t = Math.min(1, (now - t0) / dur);
    progress = from + (target - from) * (1 - Math.pow(1 - t, 3));
    updateCards();
    if (t < 1) requestAnimationFrame(step);
  })(performance.now());
}
```

**Dismiss triggers on Recents:**
- Tap outside cards: `!e.target.closest('.app-card')` + no movement
- Swipe down: `dy > 50 && dy > Math.abs(dx)`

---

## App Grid Layout

```css
.app-grid {
  display: grid;
  grid-template-rows: repeat(2, 103px); /* fixed height = square icons */
  grid-auto-flow: column;               /* fills columns, not rows */
  grid-auto-columns: 103px;
  gap: 12px;
  width: max-content;
}
```

Icon size: `(480 - 32px padding - 3×12px gap) / 4 = 103px`

Scrollbar is custom — synced via `scrollLeft / (scrollWidth - clientWidth)` on every scroll event.

---

## Settings Drawer Grid

```css
.qs-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
}
.qs-tile       { height: 103px; }      /* square */
.qs-tile.wide  { grid-column: span 2; height: 85px; } /* 2-wide */
```

Layout: 2 rows of 4 square tiles + 1 row of 2 wide tiles + tab bar.

---

## Touch Checklist (for new prototypes)

- [ ] `touch-action: none` on the root container
- [ ] All gesture handlers use Pointer Events only (no mouse/touch mix)
- [ ] Every drag uses `el.setPointerCapture(e.pointerId)`
- [ ] Every drag cleans up on `pointercancel`
- [ ] `e.preventDefault()` on `pointerdown` for any draggable element
- [ ] No `window.addEventListener('pointermove')` — use capture instead
- [ ] Viewport meta: `width=device-width, initial-scale=1, user-scalable=no`
