# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file, offline-capable, mobile-first web app for calculating cycling/driving ETA. The user enters speed and remaining distance; the app shows arrival time, a 7-row comparison table at ±0.3 km/h steps, and a shareable text summary. Everything lives in `index.html` — HTML, CSS, and JS in one file.

## Development

No build step, no dependencies, no package manager. Open `index.html` directly in a browser (`file://` works) or serve with any static host. There are no tests, no lint commands, and no compile step.

**CI pipeline (on push to `main`):** stamps `__VERSION__` placeholder with `v1.{commit-count} ({short-hash})`, minifies `index.html` with `html-minifier-terser` (~27% size reduction), then deploys via FTPS. The source file in git is never minified; minification only happens during deployment.

## Hard constraints

- **No framework.** Vanilla JS (ES5-compatible), plain HTML, plain CSS only.
- **No network requests.** Must work fully offline.
- **Minimal code.** Only add what is necessary. Prefer simple over clever.
- **WCAG 2.2 AA.** Maintain `aria-*`, `role="region"`, `scope` on `<th>`, focus-visible outlines, and `.sr-only` text on every change.
- **No `localStorage`, cookies, or server.** Zero data persistence by design — **exception:** user-modified settings (climbing penalty) are saved to `localStorage` under the key `eta_penalty`. Ride data (speed, distance, ascent) is never persisted.

## Architecture

The entire app is `index.html` (~450 lines): CSS custom properties → HTML structure → vanilla JS. No module system, no bundler.

### Key tech decisions

**`type="text"` inputs, not `type="number"`** — `type="number"` silently strips commas before JS fires, breaking German decimal input (e.g. `23,5`). Both inputs use `type="text" inputmode="decimal"`. `parseVal()` does `.replace(',', '.')` before `parseFloat`.

**Floating card labels** — All sections use `.card` + `.card-label` (absolutely positioned, background matches surface) to mimic `<fieldset>` + `<legend>`. The table card must **not** have `overflow:hidden` or the label gets clipped.

**i18n via `S` object** — All strings live in `S.en` and `S.de`. `applyLang()` iterates `[data-i18n]` elements, sets `textContent` for plain strings and `innerHTML` for strings with markup. `summary` keys are functions, not strings.

**Ascent penalty** — Optional "Ascent to go" input (meters). Adds a fixed time penalty: `extraHours = (asc_m / 100) * penaltyMin / 60`. Default penalty is 9 min per 100 m. The same `extraHours` is added to every row in the comparison table (penalty is speed-independent). The penalty value is configurable in the Settings card and persisted to `localStorage`. Speed input is the rider's moving average (Wahoo-style auto-pause), so stops are already excluded — no stop adjustment needed.

**Share/Copy buttons** — Share uses `navigator.share({ text: … })` (mobile). Copy (`#copy-btn`) is shown on desktop when `navigator.share` is unavailable but `navigator.clipboard.writeText` is available. Both are i18n-aware.

**Dark mode** — Entirely CSS via `@media (prefers-color-scheme: dark)`. No JS.

**Safe area** — `env(safe-area-inset-*)` on `body` padding for iOS notch/home bar.

### Key CSS classes

| Class | Purpose |
|---|---|
| `.card` | Bordered rounded container with `position: relative` for floating label |
| `.card-label` | Floating label above card border |
| `.field` | Flex row: label + input + unit |
| `.unit` | Fixed-width (`2.75rem`) unit label so inputs align |
| `.empty` / `.empty-text` | Placeholder state; `.empty-text` is italic, emoji is excluded |
| `.use-btn` | Ghost button in table rows to adopt that row's speed |
| `.btn-primary` | Filled accent button (Share) |
| `.sr-only` | Visually hidden, screen-reader accessible |

### Key JS functions

| Function | Purpose |
|---|---|
| `applyLang()` | Applies current `lang` to all `[data-i18n]` elements and aria-labels |
| `update()` | Main recalculation: reads inputs, updates ETA, table, summary |
| `setSummary(spd, km)` | Writes summary text, shows share/copy buttons |
| `arrival(spd, km)` | Returns `Date` of arrival |
| `dur(hours)` | Formats decimal hours as `1h 23m` or `45 min` |
| `parseVal(el)` | `parseFloat` with comma→period normalisation |
| `tick()` | Updates clock every second |
| `penaltyMin` | Module-level var; default 9, loaded from `localStorage` on init |

## Open todos

- **Font scaling / Dynamic Type** — `@supports (font: -apple-system-body)` hooks into iOS Dynamic Type, but layout at max Dynamic Type setting and 200% browser zoom needs real-device testing; fix any overflow, clipping, or misaligned card labels.
