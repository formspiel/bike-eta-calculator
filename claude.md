# claude.md — ETA Calculator

## What this project is
A single-file, offline-capable mobile-first web app for calculating ETA while driving. The user enters their current average speed and remaining distance; the app shows the arrival time, a 7-row comparison table at ±0.3 km/h steps, and a ready-to-send text message summary shared via the native OS share sheet.

## Single file
Everything lives in `eta.html` — HTML, CSS, and JS. No build step, no dependencies, no framework. Deployable by dropping onto any static host (GitHub Pages etc.) or opening locally via `file://`.

---

## Hard constraints — never change these

- **No framework.** Vanilla JS (ES5-compatible), plain HTML, plain CSS only.
- **No network requests.** The file must work fully offline and with bad connectivity.
- **Minimal code.** Only add what is necessary. Prefer simple over clever.
- **WCAG 2.2 AA accessibility.** `aria-*` attributes, `role="region"`, `scope` on `<th>`, focus-visible outlines, and screen-reader-only text (`.sr-only`) must be maintained on every change.
- **No `localStorage`, no cookies, no server.** Zero data persistence by design — there is no security surface and no login is needed.

---

## Tech decisions and why

### `type="text"` inputs, not `type="number"`
`type="number"` silently strips commas before any JS event fires, making German decimal input (e.g. `23,5`) impossible. Both inputs use `type="text" inputmode="decimal"` so the numeric keyboard appears on mobile and commas are accepted. `parseVal()` does `.replace(',', '.')` before `parseFloat`.

### Floating card labels (`.card` / `.card-label`)
All sections — ETA, ETA by speed, Text message summary — use `.card` + `.card-label` (absolutely positioned, background matches surface) to visually match the native `<fieldset>` + `<legend>` look. The table card must **not** have `overflow:hidden` or the label gets clipped.

### i18n via `S` object
All user-facing strings live in `S.en` and `S.de`. The `applyLang()` function iterates `[data-i18n]` elements and sets `textContent` for plain strings, `innerHTML` for strings containing markup (e.g. the `<span class="empty-text">` wrapper for italic text). `summary` keys are functions, not strings.

### Empty state styling
`.empty` sets `font-size: var(--fs-body)` and `color: var(--text-sec)`. The ℹ️ emoji sits outside any italic styling; the text is wrapped in `<span class="empty-text">` which carries `font-style: italic`. `td.empty` explicitly overrides `text-align: left` to prevent the `td:last-child` rule from right-aligning colspan cells.

### Share button
Uses `navigator.share({ text: … })`. Only rendered when `!!navigator.share` is true (mobile browsers). No copy button — the share sheet covers all use cases on mobile. No fallback needed; if share is unavailable the button simply stays hidden.

### Dynamic Type / font sizing
`-apple-system` font stack picks up iOS Dynamic Type. `font-size: 100%` on `<html>` respects the user's browser font size setting on Android. All sizes use `var(--fs-body)`, `var(--fs-label)`, `var(--fs-caption)`, `var(--fs-title)`.

### Dark mode
Handled entirely via `@media (prefers-color-scheme: dark)` CSS custom properties. Zero JS involved.

### Safe area
`env(safe-area-inset-*)` applied to `body` padding for notch and home-bar clearance on iOS.

---

## Key CSS classes

| Class | Purpose |
|---|---|
| `.card` | Bordered rounded container with `position: relative` for floating label |
| `.card-label` | Floating label above card border, matches `<legend>` visually |
| `.field` | Flex row for label + input + unit |
| `.unit` | Fixed-width (`2.75rem`) unit label so inputs align across rows |
| `.empty` | Placeholder state: body font size, secondary colour |
| `.empty-text` | Italic span inside `.empty`, excludes emoji from italic |
| `.use-btn` | Ghost button in table rows to adopt that row's speed |
| `.btn-primary` | Filled accent button (Share) |
| `.sr-only` | Visually hidden, accessible to screen readers |

---

## Key JS functions

| Function | Purpose |
|---|---|
| `applyLang()` | Applies current `lang` to all `[data-i18n]` elements and aria-labels |
| `update()` | Main recalculation: reads inputs, updates ETA, table, summary |
| `setSummary(spd, km)` | Writes active summary text, shows share button |
| `arrival(spd, km)` | Returns `Date` of arrival |
| `dur(hours)` | Formats decimal hours as `1h 23m` or `45 min` |
| `parseVal(el)` | `parseFloat` with comma→period normalisation |
| `tick()` | Updates clock every second |

---

## Deployment

@github-ftp-deploy.md

---

## Open todos

1. ~~**Verify last visual fixes in a real browser**~~ — confirmed by user: card label not clipped, empty message left-aligned in secondary colour, emoji not italic.

2. ~~**Share button hidden on desktop**~~ — resolved: a **Copy** button (`#copy-btn`) now shows on desktop when `navigator.share` is unavailable but `navigator.clipboard.writeText` is available. Shows "Copied!" for 2 s then resets. Fully i18n-aware (`copy`/`copied` keys in `S.en` and `S.de`).

3. **Font scaling / Dynamic Type — needs browser testing** — `@supports (font: -apple-system-body)` was added to hook into iOS Dynamic Type. At large text sizes (WCAG requires support up to 200% zoom) not all elements scale correctly. Needs a real-device test at max Dynamic Type setting and at 200% browser zoom; fix any layout breaks (overflow, clipping, misaligned card labels, input widths).
