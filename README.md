# PoG Mind Sync

A minimal iOS home-screen web app that converts page numbers between two physical editions of Iain M. Banks' **Player of Games** series. Built to solve a specific problem: reading the Folio Society hardback alongside notes or highlights made on a Kobo/XTEInk e-reader, where page numbers differ significantly between editions.

---

## What It Does

Enter a page number in either field — Folio Society print edition or XTEInk X4 e-reader section page — and the other field updates instantly. Bidirectional, no submit button, works offline once loaded.

---

## Files

```
mind-sync.html      # The app — single self-contained HTML file
mind-sync.png       # 180×180px Home Screen icon (place in same directory)
README.md           # This file
```

---

## Deployment

The app runs from any basic HTTP server. It does **not** need a backend.

### Local server (Mac)
```bash
# From the directory containing mind-sync.html
python3 -m http.server 8080
# Then navigate to http://<your-mac-ip>:8080/mind-sync.html
```

### Linux server (nginx / Apache)
Place `mind-sync.html` and `mind-sync.png` in your web root (e.g. `/var/www/html/`), then fix permissions if you hit a 403:
```bash
chmod 644 mind-sync.html mind-sync.png
chmod 755 /var/www/html/
```

### Adding to iPhone Home Screen
1. Open Safari and navigate to the file
2. Tap the **Share** button → **Add to Home Screen**
3. The app installs with the `mind-sync.png` icon and runs full-screen with no browser chrome

---

## Current Calibration — Book 2: Imperium

### Edition details
| Field | Value |
|---|---|
| Book | The Player of Games — **2. Imperium** |
| Print edition | Folio Society hardback |
| Print page range | 113 – 268 |
| E-reader edition | XTEInk X4 section pages |
| E-reader page range | 1 – 461 |

### Conversion formulas
```
XTEInk → Folio:   folio_page  = round(0.353 × ebook_page + 105)
Folio  → XTEInk:  ebook_page  = round((folio_page − 105) × 2.83)
```

These are linear fits derived from manually verified anchor points (see below). The relationship between the two editions is not perfectly linear across the entire book, so accuracy may drift slightly at the extremes — but it is close enough for quick navigation.

### Verified anchor points (ground truth)
These three points were manually confirmed and used to derive and validate the formulas above:

| XTEInk page | Folio page |
|---|---|
| 59 | 126 |
| 62 | 127 |
| 461 | 268 |

The seed value shown on app load is XTEInk **59** → Folio **126**.

---

## Adapting for a New Chapter or Book

When moving to a new section, chapter, or entirely different book, you need to recalibrate with at least **2–3 anchor points** (more is better). Follow these steps:

### Step 1 — Collect anchor points
Find 2–3 moments in the text that are unambiguous landmarks (start of a chapter, a named section break, a distinctive paragraph). Note the exact page number in **both** editions. Spread your anchors across the full range of the book — one near the start, one near the middle, one near the end.

| XTEInk page | Folio page |
|---|---|
| A | X |
| B | Y |
| C | Z |

### Step 2 — Derive the slope (scale factor)
Using two well-separated anchor points:
```
slope = (Z − X) / (C − A)
```
This gives you how many Folio pages correspond to one XTEInk page.

### Step 3 — Derive the offset
```
offset = X − (slope × A)
```

### Step 4 — Write the new formulas
```
XTEInk → Folio:   folio_page = round(slope × ebook_page + offset)
Folio  → XTEInk:  ebook_page = round((folio_page − offset) / slope)
```

### Step 5 — Update the HTML
In `mind-sync.html`, find the two formula functions in the `<script>` block and replace the constants:

```js
// Before (Book 2: Imperium)
function ebookToFolio(e) { return e ? Math.round(0.353 * e + 105) : ''; }
function folioToEbook(p) { return p ? Math.round((p - 105) * 2.83) : ''; }

// After (replace with your new slope and offset)
function ebookToFolio(e) { return e ? Math.round(SLOPE * e + OFFSET) : ''; }
function folioToEbook(p) { return p ? Math.round((p - OFFSET) * (1 / SLOPE)) : ''; }
```

Also update the following to keep the UI and documentation accurate:

```html
<!-- Header subtitle -->
<p class="header-sub">PLAYER OF GAMES<br>3. Whatever-Chapter-Title</p>

<!-- Page range limits on the inputs -->
<input id="folio" type="number" min="NEW_MIN" max="NEW_MAX" ...>
<input id="ebook" type="number" min="1"       max="NEW_MAX" ...>

<!-- Info box verified anchors -->
<div class="info">
    Calibrated to your Folio + XTEInk X4.<br>
    Verified: A → X &nbsp;|&nbsp; B → Y &nbsp;|&nbsp; C → Z
</div>

<!-- Seed value on load (set to your first anchor) -->
setTimeout(() => { ebookInput.value = A; updateFromEbook(); }, 300);
```

### Step 6 — Validate
Spot-check 3–5 random pages across the range. A good calibration should be within ±1–2 pages anywhere in the book. If drift is larger than that at the extremes, consider splitting the book into two ranges with separate formulas (a piecewise approach), and add a toggle or auto-detect based on which range the entered page falls into.

---

## iOS Web App Behaviour Notes

- `apple-mobile-web-app-capable` + `black-translucent` status bar: the app runs full-screen with a black status bar when launched from the Home Screen
- `position: fixed` + `overflow: hidden` on both `html` and `body`: eliminates iOS rubber-band/elastic scroll entirely
- `touch-action: none` on body, `touch-action: manipulation` on inputs: blocks scroll gestures while keeping tap-to-focus working on the number fields
- `inputmode="numeric"`: triggers the number pad keyboard on iPhone instead of the full keyboard
- Google Fonts (`Orbitron`, `Roboto Mono`) load when online; the app degrades gracefully to `Courier New` / `system-ui` when offline

---

## Cosmetic Reference

The visual design intentionally mimics a sci-fi terminal interface in the style of Banks' Culture universe.

| Element | Value |
|---|---|
| Background | `#000000` pure black |
| Primary accent | `#22d3ee` (cyan-400) |
| Secondary accent | `#67e8f9` (cyan-300) |
| Status green | `#34d399` (emerald-400) |
| Dim text (info box) | `#3f3f46` (zinc-700) |
| Title font | Orbitron 700 |
| Number input font | Roboto Mono 700 |
| Title effect | CSS glitch animation + cyan glow |

---

## Version History

| Version | Notes |
|---|---|
| v1.0 | Initial build with Tailwind CDN — not suitable for local deployment |
| v1.1 | Rewrote with vanilla CSS, fixed for Safari/iOS local server, added safe-area insets, numeric keypad |
| v1.2 | Locked viewport (no scroll/bounce), pure black background, dimmed info text, fixed subtitle line break, external icon support |
