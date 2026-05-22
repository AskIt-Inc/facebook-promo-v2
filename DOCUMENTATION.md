# Facebook Promo Card v2 — Technical Documentation

**File:** `index.html`  
**Location:** `drupal-oav/plans/figma-make/facebook-promo-v2/`  
**Purpose:** Single-file in-browser tool for generating, previewing, and exporting session promo cards for STTT (Somebody To Talk To). Cards are exported as PNG or PDF for use on Facebook and other social platforms, or sent directly to a printer.

---

## 1. Architecture Overview

The entire tool is a single self-contained HTML file with no build step and no server-side rendering. It works entirely in a browser tab.

```
index.html
├── <head>
│   ├── CDN: dom-to-image-more@3.4.5   (DOM → PNG rasteriser)
│   ├── CDN: jsPDF@2.5.2               (PNG → PDF wrapper)
│   └── <style>                        (all CSS, ~400 lines)
└── <body>
    ├── .search-panel                  (step 1: search for a session)
    ├── .controls (hidden initially)   (size toggle + export buttons)
    └── .preview-wrapper (hidden)      (the actual print card)
        └── .print-surface             (the 4-zone card)
            ├── Zone 1: .logo-bar
            ├── Zone 2: .session-band
            ├── Zone 3: .presenter-section
            └── Zone 4: .fine-print
    └── <script>                       (all JS, ~340 lines)
```

**Key constraint:** Both partner logos are embedded as base64 inside the HTML. This is why the file is ~279 KB despite having no external assets at runtime. The card can be used fully offline after initial load.

**Initial state:** `.controls` and `.preview-wrapper` are hidden via `style="display:none"`. They are only revealed on the first successful call to `populateCard()`, ensuring the user always searches before seeing a card.

---

## 2. Design System (CSS Custom Properties)

Defined on `:root`. All components reference these variables; never use raw colour values in new rules.

| Variable        | Value       | Usage                                      |
|-----------------|-------------|--------------------------------------------|
| `--card-width`  | `6in`        | Print surface default width                |
| `--card-height` | `4in`        | Print surface default height               |
| `--maroon`      | `#8B1F2D`   | Primary brand colour; band bg, titles, CTAs |
| `--maroon-dark` | `#6E1A24`   | Reserved (not currently applied)            |
| `--gold`        | `#FFE8A3`   | Series label text on maroon band            |
| `--border`      | `#E8E8E8`   | Logo bar bottom border                      |
| `--border-rose` | `#DDD0D2`   | Headshot border, date pill border, QR border |
| `--bg-rose`     | `#f5eaec`   | Headshot placeholder bg, date pill bg       |

**Typography:** `'Helvetica Neue', Helvetica, Arial, sans-serif` throughout. No web font loading — deliberate, ensures print fidelity without network dependency.

---

## 3. Card Layout — Four Zones

The `.print-surface` is a vertical flex column. Zones 1, 2, and 4 are `flex-shrink: 0` (fixed height). Zone 3 is `flex: 1` (fills remaining space).

```
┌──────────────────────────────────────────────────────┐
│  ZONE 1 · Logo Bar (72px)                            │
│  [STTT logo] | [date · time] | [partner logo]        │
├──────────────────────────────────────────────────────┤
│  ZONE 2 · Session Band (maroon)                      │
│  [series label (optional)] [session title] [pill]    │
├──────────────────────────────────────────────────────┤
│  ZONE 3 · Presenter Section (flex: 1)                │
│  [headshot] │ [date pill + desc + CTA] │ [QR code]   │
├──────────────────────────────────────────────────────┤
│  ZONE 4 · Fine Print (~30px)                         │
└──────────────────────────────────────────────────────┘
```

### Zone 1 — Logo Bar (`.logo-bar`)

`display: flex; justify-content: space-between; height: 72px`

| Element            | Class / ID            | Notes                                        |
|--------------------|-----------------------|----------------------------------------------|
| STTT logo          | `.logo-left`          | base64 PNG, 48px tall, max-width 160px       |
| Vertical divider   | `.logo-divider`       | 1px × 46px, `--border-rose`                  |
| Date block         | `#header-date`        | Populated by JS: "Thursday, May 21st"        |
| Time block         | `#header-time`        | Populated by JS: "@ 6pm ET"                  |
| Vertical divider   | `.logo-divider`       | Second instance                              |
| Partner logo       | `.logo-right`         | base64 PNG, 48px tall, max-width 180px       |

The center date/time block uses `flex: 1` to expand between the two dividers.

### Zone 2 — Session Band (`.session-band`)

`background: var(--maroon); display: flex; justify-content: space-between`

| Element           | Class / ID              | Visibility                             |
|-------------------|-------------------------|----------------------------------------|
| Series label row  | `#band-series-row`      | `display: none` → `.visible` when `series_label` present |
| Series dot (SVG)  | `.band-series-dot`      | Gold star SVG icon, always inside row  |
| Series text       | `#band-series-label`    | `--gold` colour, italic, 14px          |
| Session title     | `#band-title`           | White, 800 weight, 20px                |
| Partner pill      | `#uoc-pill`             | `display: none` → `.visible` when `partner` present; white bg, rounded |
| Pill logo         | `img` inside `#uoc-pill`| 9rem width, base64 embedded            |

### Zone 3 — Presenter Section (`.presenter-section`)

`display: grid; grid-template-columns: 110px 1fr 96px; column-gap: 16px; padding: 12px 22px 0`

Three fixed-width columns:

**Column 1 — Headshot (`.headshot-col`)**

- 94×94px circular image with `--border-rose` ring
- `#headshot-img`: shown when `headshot_base64` available, hidden otherwise
- `#headshot-fallback`: shown when no headshot; text "No photo"
- `#presenter-name`: maroon, 11.5px, 800 weight — assembled from title + first/last + suffix
- `#presenter-cred`: always empty (field exists, unused by current API)
- `#presenter-role`: populated with `row.partner` — note naming mismatch (see §7)

**Column 2 — Content (`.content-col`)**

- `.date-pill`: rose bg pill with calendar SVG icon
  - `#date-pill-date`: "May 18, 2026"
  - `#tz-row`: "6:00 PM ET · 5:00 PM CT · 4:00 PM MT · 3:00 PM PT"
- `#session-desc`: 11.5px body text, HTML-stripped
- `.register-row`: "Register free:" label + `#register-url` monospace chip

**Column 3 — QR Code (`.qr-col`)**

- 82×82px bordered box
- `#qr-img`: 78×78px image, shown when `qr_base64` available
- `#qr-placeholder`: fallback text when no QR
- `.qr-label`: "Scan to Register", uppercase, 8px, maroon

### Zone 4 — Fine Print (`.fine-print`)

Static italic text. 10.5px, `#666`, `border-top: 1px solid #F0F0F0`. `margin-top: auto` pushes it to the bottom of the flex column.

---

## 4. Size Variants

Two sizes available via the size toggle buttons.

| Size  | Button     | CSS class added       | Dimensions        | DPI target | Pixel output  |
|-------|------------|-----------------------|-------------------|------------|---------------|
| 4×6   | `#btn-4x6` | _(none, default)_     | 6in × 4in         | 300        | 1800 × 1200   |
| 3×5   | `#btn-3x5` | `.size-3x5` on surface| 5in × 3in         | 300        | 1500 × 900    |

The 3×5 variant uses a cascade of `.print-surface.size-3x5 .{component}` overrides to scale down every font, spacing, and image dimension proportionally. There are 20 override rules covering all zones.

---

## 5. JavaScript Modules

### 5.1 Size Toggle — `setSize(s)`

Adds/removes `.size-3x5` from `#print-surface`. Toggles `.active` class on the two size buttons. Updates the `#size-label` text.

### 5.2 Print — `handlePrint()`

1. Detects current size.
2. Dynamically injects a `<style id="__print-page-style">` with `@page { size: 6in 4in landscape; margin: 0 }` (or 5in 3in for 3×5).
3. Calls `window.print()`.
4. Removes the injected style after 1 second via `setTimeout`.

The `@media print` rule hides `.controls`, `.card-label`, and `.search-panel` automatically.

### 5.3 Capture Pipeline — `captureCard(scale, pxW, pxH)`

Core rasterisation function used by both PNG and PDF exports.

**Extension stylesheet bug workaround:** Browser extensions inject `@font-face` rules referencing `chrome-extension://` URLs. `dom-to-image-more` scans all `document.styleSheets` and tries to fetch those URLs, getting `ERR_FILE_NOT_FOUND`, which aborts the export with a Blob error.

Fix implemented:
1. Iterate `document.styleSheets`.
2. For each sheet, attempt to read `cssRules`. Cross-origin sheets throw — catch and disable those too.
3. If any rule contains `chrome-extension:`, `moz-extension:`, or `safari-extension:` — disable the sheet.
4. Call `domtoimage.toPng()` with a `transform: scale(N)` applied in the style override to reach target pixel dimensions.
5. In `finally`: re-enable all disabled sheets.

Scale is computed as: `cfg.pxW / surface.offsetWidth` — i.e., how much to multiply the rendered CSS inches to reach target raster pixels.

### 5.4 PNG Export — `downloadPNG()`

1. Determines size config (px dimensions + filename tag).
2. Calls `captureCard()`.
3. Creates a temporary `<a>` element, sets `href` to the data URL, triggers `.click()`, then removes it.
4. Filename: `facebook-promo-4x6.png` or `facebook-promo-3x5.png`.

### 5.5 PDF Export — `downloadPDF()`

1. Same size detection as PNG.
2. Calls `captureCard()` to get a PNG data URL.
3. Creates a `jsPDF` instance: `{ orientation: 'landscape', unit: 'in', format: [wIn, hIn] }`.
4. Calls `pdf.addImage(dataUrl, 'PNG', 0, 0, wIn, hIn)` — zero margins, fills page exactly.
5. Calls `pdf.save(filename)`.

### 5.6 Search — `runSearch()`

**API endpoint:** `https://www.somebodytotalkto.com/api/spotlight/microsite/session/search?q={query}`

**Trigger conditions:**
- Input `keydown` Enter → immediate
- Button click → immediate (clears debounce to prevent duplicate)
- Input `input` event → 400ms debounce, triggers only when `length >= 3`
- Minimum query length: 2 characters

**Race condition protection:** A `searchToken` integer is incremented on every new search call. The token value is captured in closure at call time. Before rendering results (or errors), the response checks if `token === searchToken`. If not, the response is silently discarded. This prevents out-of-order responses from stomping a newer result.

**Result handling:**
- 0 results → status message only
- 1 result → auto-calls `populateCard(rows[0])`
- 2+ results → renders clickable `.search-result-item` list; each click calls `populateCard(row)` and clears the list

### 5.7 Populate Card — `populateCard(row)`

Maps API response fields onto DOM elements. Detailed field-by-field mapping:

| API Field              | DOM Element           | Transform applied                                              |
|------------------------|-----------------------|----------------------------------------------------------------|
| `row.date`             | `#header-date`        | Parsed → "Thursday, May 21st" (UTC parse, ordinal suffix)      |
| `row.times_by_zone.ET` | `#header-time`        | Parsed → "@ 6pm ET" (strips minutes if :00)                    |
| `row.series_label`     | `#band-series-label`  | Sets text; adds `.visible` to `#band-series-row` if non-empty  |
| `row.title`            | `#band-title`         | Strips `session_type:` prefix if `row.session_type` present    |
| `row.partner`          | `#uoc-pill`           | Adds `.visible` to pill if non-empty                           |
| `row.presenters[0]`    | `#presenter-name`     | Assembled: `field_taxo_title + first + last + name_suffix`     |
| _(cleared)_            | `#presenter-cred`     | Always set to `''` — field unused by current API               |
| `row.partner`          | `#presenter-role`     | Displays partner name in role slot (see §7, Known Quirks)      |
| `row.date`             | `#date-pill-date`     | Raw string, no transform                                       |
| `row.times_by_zone`    | `#tz-row`             | ET/CT/MT/PT joined with ` · `                                  |
| `row.description`      | `#session-desc`       | HTML-stripped via temp `<div>.textContent`                     |
| `row.headshot_base64`  | `#headshot-img`       | Set as `src`; toggles img/fallback visibility                  |
| `row.qr_base64`        | `#qr-img`             | Set as `src`; toggles img/placeholder visibility               |
| `row.short_url`        | `#register-url`       | Prefers short URL; falls back to `row.reg_link.url`; strips `https://`; truncates at 36 chars if no short URL |

**Card reveal:** On the first `populateCard()` call, `#controls-bar` and `#preview-wrapper` have their `display:none` removed, revealing the card and export controls.

---

## 6. API Response Contract

Expected shape from `GET /api/spotlight/microsite/session/search?q=`:

```json
{
  "data": [
    {
      "title":          "How to Build a Healthcare Team You Trust",
      "session_type":   "Webinar",
      "series_label":   "Spotlight Series",
      "partner":        "University of Calgary",
      "date":           "May 21, 2026",
      "time":           "6:00 PM ET",
      "times_by_zone": {
        "ET": "6:00 PM",
        "CT": "5:00 PM",
        "MT": "4:00 PM",
        "PT": "3:00 PM"
      },
      "description":    "<p>Building a strong healthcare team…</p>",
      "presenters": [
        {
          "first_name":    "Jane",
          "last_name":     "Smith",
          "display_name":  "Jane Smith",
          "title":         "Dr.",
          "name_suffix":   "MD, FRCPC"
        }
      ],
      "headshot_base64": "data:image/jpeg;base64,…",
      "qr_base64":       "data:image/png;base64,…",
      "short_url":       "bit.ly/sttt-xyz",
      "reg_link": {
        "url": "https://zoom.us/webinar/register/…"
      }
    }
  ]
}
```

**Notes:**
- `headshot_base64` and `qr_base64` include the `data:image/…;base64,` prefix — set directly as `src`.
- `description` may contain HTML markup; the card strips it.
- `short_url` is optional; fallback chain is `short_url → reg_link.url → static default`.
- Only `presenters[0]` is used; multi-presenter sessions show only the first presenter.

---

## 7. Known Quirks and Debt

### `#presenter-role` displays `row.partner`

The DOM element ID suggests it holds the presenter's role or job title. In practice it is populated with `row.partner` (the institutional partner name). The `presenter-cred` element is cleared to empty on every render. This is a naming mismatch that should be addressed if a presenter credential field is ever needed.

### `presenter-cred` is always cleared

`document.getElementById('presenter-cred').textContent = ''` is hardcoded. There is no API field currently mapped to it. The element and CSS rule remain in place as structural debt.

### No credential display in current API

The presenter object has no `credentials` or `role` field in the current API response. If this is needed in future, it would need to be added to both the API endpoint and `populateCard()`.

### Single presenter only

`(row.presenters || [])[0]` — only the first presenter is rendered. No layout exists for co-presenters.

### Partner logo is hardcoded base64

The right-side logo (`logo-right`) is embedded as base64. It does not change based on `row.partner`. The `uoc-pill` (which shows the partner logo inside the maroon band) is similarly a single hardcoded image. If multi-partner support is needed, both would need to become dynamic.

### File size

At ~279 KB, the file is large for an HTML document. Approximately 270 KB is base64-encoded image data (two logos). This is intentional for portability but makes the file impractical to diff or code-review in a standard tool.

---

## 8. Export Specifications

| Format | Size  | Pixel dimensions | Colour profile | Filename                 |
|--------|-------|------------------|----------------|--------------------------|
| PNG    | 4×6   | 1800 × 1200      | sRGB           | `facebook-promo-4x6.png` |
| PNG    | 3×5   | 1500 × 900       | sRGB           | `facebook-promo-3x5.png` |
| PDF    | 4×6   | 1800 × 1200 img  | landscape, 6×4in | `facebook-promo-4x6.pdf` |
| PDF    | 3×5   | 1500 × 900 img   | landscape, 5×3in | `facebook-promo-3x5.pdf` |
| Print  | both  | @page native      | exact colour   | _(browser print dialog)_ |

The PDF is image-only (rasterised PNG embedded in a PDF wrapper). It is not a vector PDF. Text is not selectable in the output PDF.

---

## 9. Dependencies

| Library             | Version  | Source                                   | Purpose                          |
|---------------------|----------|------------------------------------------|----------------------------------|
| dom-to-image-more   | 3.4.5    | `cdn.jsdelivr.net`                       | DOM → PNG rasterisation          |
| jsPDF               | 2.5.2    | `cdn.jsdelivr.net`                       | PNG → PDF wrapping               |

Both loaded via `<script src>` in `<head>`. No npm, no build step. If CDN is unavailable, exports will silently fail (try/catch surfaces error via `alert()`). Print still works without network.

---

## 10. Adding a New Field to the Card

1. Add the DOM element inside the appropriate zone in `<body>`.
2. Add CSS rules for it (follow existing naming conventions).
3. If the field needs to be smaller in 3×5 mode, add a `.print-surface.size-3x5 .your-class` override rule.
4. Add the field to the `populateCard()` mapping block with a comment in the header comment (lines ~706–719).
5. Confirm the API returns the field — check `SEARCH_API` response shape.
6. If the field is conditionally visible (like `.uoc-pill` or `.band-series`), use the `.visible` class toggle pattern rather than direct `display` manipulation.

---

## 11. File Structure

```
facebook-promo-v2/
├── index.html          Single-file tool (~279 KB, includes base64 logos)
└── DOCUMENTATION.md    This file
```

No other files. The tool has no dependencies on the surrounding Drupal codebase at runtime — it communicates only with the live STTT search API.
