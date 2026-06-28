---
description: Add or update a feature in the RITA Mobile PWA (riia-jun-release/mobileapp/index.html)
---

You are an Engineer agent adding or updating a feature in the RITA Mobile PWA.

**Task:** $ARGUMENTS

---

## IMPORTANT — Relocation vs Feature Tasks

Before writing any code, determine which type of task this is:

**Type A — File relocation / restructuring** (e.g. "move files", "restructure folders", "relocate screens"):
- "Move" always means: copy to new path + update all references (main.py, specs, CLAUDE.md) + add old path to `.gitignore` + `git rm --cached -r <old-path>/`
- Do NOT just copy files and leave the originals tracked. That is not a move.
- Confirm with the user if the scope is ambiguous: "To confirm — move means copy to new location, update all references, then remove old path from git tracking. Correct?"

**Type B — Feature addition / modification** (adding a screen, binding API data, modifying UI):
- Proceed with the instructions below.

---

## IMPORTANT — Never Read index.html in Full

`index.html` is 4,000+ lines. Always use targeted grep + read only the section you need.

```bash
# Find a screen's start line
grep -n "id=\"s3\"" rita-build-portfolio/android-mobile-app/index.html

# Find a bind function
grep -n "function bind" rita-build-portfolio/android-mobile-app/index.html

# Find a DOM element by id
grep -n "id=\"my-element\"" rita-build-portfolio/android-mobile-app/index.html
```

---

## Read the Spec First

Read `Specs/Spec_Mobile_App.md` before touching any code. It contains:
- Architecture overview (single-file PWA, no ES modules, `goTo(n)` navigation)
- Screen inventory (s0–s8, what each screen shows, which `bind*()` function drives it)
- All API fetch functions and their endpoints
- `initLiveData()` coordinator flow
- Signal thresholds, factor bar mapping, sparkline pattern
- Live toggle logic

---

## Key Architecture Facts (memorise before editing)

- **Single file** — all HTML, CSS, and JS in `index.html`. No new `.js` or `.css` files.
- **No ES modules** — all functions are global. No `import`/`export`.
- **10 screens**, carousel navigation. Track width = `calc(10 * 100vw)`.
- **`LIVE_MODE` guard** — never call API functions if `LIVE_MODE` is false.
- **Fallback required** — every API binding must `if (!data) return` before accessing fields.
- **`API_BASE` constant** — always prefix fetch URLs with `API_BASE`. Never hardcode `http://localhost:8000`.

---

## Adding a New Screen

1. Add `<div class="screen" id="sN"><div class="screen-inner">...</div></div>` inside `.carousel-track`
2. Write `function bindMyScreen(data) { ... }` — DOM bindings only; no fetch inside this function
3. Add the fetch call to `initLiveData()` — call `bindMyScreen(data)` from there
4. Link from home cards or existing screens: `onclick="goTo(N)"`
5. If adding beyond screen 9: update carousel track — `width: calc(N * 100vw)`

---

## Modifying an Existing Screen

1. `grep -n "id=\"sN\"" index.html` → get start line
2. Read only that screen's section (~80–200 lines)
3. `grep -n "function bindMyScreen" index.html` → find the bind function
4. Read + modify only that bind function

---

## API Fetch Pattern

```js
async function fetchMyData() {
    try {
        const r = await fetch(`${API_BASE}/api/v1/my-endpoint`);
        return r.ok ? r.json() : null;
    } catch {
        return null;           // never throw — app must not break on API failure
    }
}
```

Then call it in `initLiveData()` alongside the existing parallel fetches.

---

## Touch + CSS Rules

- Tap target minimum: `40px × 40px`
- All clickable elements: `-webkit-tap-highlight-color: transparent; user-select: none; cursor: pointer`
- Safe area: `padding-bottom: max(env(safe-area-inset-bottom), 8px)` for bottom elements
- Always use CSS variables — never hex literals:

| Var | Use |
|---|---|
| `--build` | Positive / primary action |
| `--danger` | Negative / bearish |
| `--warn` | Warning / neutral |
| `--t3` | Muted label text |
| `--surface` | Card background |
| `--border` | Card border |

---

## DOM Binding Pattern

```js
function bindMyScreen(data) {
    if (!data) return;                    // always guard
    function g(id) { return document.getElementById(id); }

    g('my-value').textContent = data.some_field ?? '—';
    g('my-pct').innerHTML = `<span style="color:var(--build)">${(data.pct * 100).toFixed(1)}%</span>`;
}
```

No `setEl()` wrapper exists in mobile — use `g(id).textContent` or `g(id).innerHTML` directly.

---

## Sparklines (SVG — not Chart.js)

```js
function pricesToPolyline(prices, w=200, h=60) {
    const min = Math.min(...prices), max = Math.max(...prices);
    return prices.map((p, i) =>
        `${(i/(prices.length-1)*w).toFixed(1)},${(h - (p-min)/(max-min)*h).toFixed(1)}`
    ).join(' ');
}
// Use in HTML: <polyline points="${pricesToPolyline(data.prices)}" stroke="var(--run)" stroke-width="1.5" fill="none"/>
```

Chart.js is too heavy for mobile — use inline SVG for all mini charts.

---

## Files to Touch

| File | Action |
|---|---|
| `rita-build-portfolio/android-mobile-app/index.html` | Edit — targeted section only |
| `rita-build-portfolio/android-mobile-app/sw.js` | Edit — add new endpoint to cache list if offline support needed |
| `Specs/Spec_Mobile_App.md` | Edit — add screen to inventory or update bind function table |

---

## Definition of Done

- [ ] Only targeted sections of `index.html` were read — not the full file
- [ ] `if (!data) return` guard present in every bind function
- [ ] No hardcoded URLs — `API_BASE` used for all fetch calls
- [ ] `LIVE_MODE` check respected — API calls only when live mode is on
- [ ] Touch targets ≥ 40px, tap highlight suppressed on all interactive elements
- [ ] CSS variables used — no hex literals in new HTML
- [ ] New screen registered in `initLiveData()` if it fetches data
- [ ] `sw.js` updated if new endpoint should be cached offline
- [ ] `Specs/Spec_Mobile_App.md` screen inventory updated if new screen added
