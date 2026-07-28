# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A static, dependency-free trip-planning site for a Seoul trip (2026/08/23–08/28). There is no build system, package manager, or server-side code — the entire project is two self-contained HTML files (inline `<style>` and `<script>`, no external JS/CSS files, no CDN dependencies).

- `首爾行程_0823-0828.html` — the day-by-day itinerary (6 days: sights, meals, shopping, transit notes, fan/K-pop stops).
- `旅遊記帳.html` — a trip expense tracker (multi-currency conversion, category/day breakdowns, budget tracking, CSV/JSON export) that links back to the itinerary page.

## Commands

There is no build/lint/test tooling. To work on either page, just open the file directly in a browser (double-click, or `file://` URL), or serve the directory with any static file server if you need `fetch`/relative-path behavior — neither page currently needs one. Changes are visible on browser refresh.

## Architecture

### Shared conventions across both pages

- **Design tokens**: all colors/fonts are CSS custom properties defined in `:root`, with a parallel `:root[data-theme="dark"]` block and a `@media (prefers-color-scheme: dark)` block that mirrors it. When adding UI, style with `var(--token)`, never hard-coded colors, so both pages stay in sync visually and in dark mode.
- **Dark mode toggle**: both pages share the same localStorage key `seoul2026_theme` (`'dark'`/`'light'`), so the preference persists across the two pages. The toggle button sets `data-theme` on `<html>`.
- **No frameworks**: plain DOM APIs, template strings for rendering, `insertAdjacentHTML`/`innerHTML` for lists. Keep new features consistent with this style rather than introducing a framework.
- **Font stack**: `--font-display` (Noto Serif TC, serif headings) and `--font-body` (Noto Sans TC, sans body text).

### 首爾行程_0823-0828.html (itinerary)

- All content is data-driven from a single `const days = [...]` array (one object per day: `n`, `iso`, `date`, `theme`, `stops[]`, optional `arrivalNote`/`backupActivities`/`backupMeal`/`endNote`). To edit itinerary content, edit this array — the DOM is rendered from it, not hand-written HTML.
- Each `stop` has a `kind` (`meal`/`sight`/`shop`/`transport`/`fan`, etc.) which drives the category accent color (`--cat-*` tokens) and icon; and a `transit` object describing how to get there from the previous stop.
- Rendering happens in a few passes over `days` (~line 880+): building the sticky day-nav pills (`#daynavInner`), the main day sections (`#days`), and (later) computing footer stats straight from the data (explicitly avoids inventing a single "total cost" or "total distance" number — see the in-page disclaimer — cost tracking was intentionally split out into the expense tracker instead).
- Search (`#searchInput`/`#searchResults`) does a client-side scan over `days`/`stops`.
- `IntersectionObserver`-based scroll-spy highlights the current day in the sticky day-nav.
- Export features: `window.print()` for PDF, and `downloadTxt()` for a simplified plain-text export.
- Map links are generated via `naverLink(query)` — Naver/Kakao Map are used instead of Google Maps because Google Maps walking directions don't work in Korea (noted in the UI copy).

### 旅遊記帳.html (expense tracker)

- All state lives in `localStorage`, two keys:
  - `seoul2026_expenses` — array of expense records (`day`, `cat`, `amount`, `currency`, `payment`, `note`, `split`).
  - `seoul2026_expense_settings` — `{ budget, travelers, enabled: [currency codes...], rates: { CODE: rateToTWD } }`.
- Config lives in top-of-script constants: `DAYS` (the 6 trip days), `CATEGORIES` (spend categories + accent colors, aligned with the itinerary's `--cat-*` tokens), `QUICK_TEMPLATES` (one-tap common expenses), `CURRENCIES` (the full list of currencies selectable in Settings, with default TWD conversion rates).
- **Currency model**: TWD is the fixed base currency (rate always 1, cannot be removed). Users add/remove other currencies from `CURRENCIES` into `settings.enabled` via a dropdown-based picker in the Settings section; each enabled non-TWD currency gets an editable "1 X = ? TWD" rate input. `toTWD(amount, currency)` is the single conversion function everything else (summary stats, charts, list, converter) is built on — so all money math funnels through `settings.rates`.
- **Rendering**: one `renderAll()` fans out to `renderSummary()`, `renderDonut()` (SVG category pie via `stroke-dasharray` on stacked `<circle>`s), `renderDayBars()`, and `renderList()`. Call `renderAll()` after any mutation to `expenses` or `settings`.
- **Section anchors + tab nav**: each `<section>` has a stable `id` (`sec-overview`, `sec-settings`, `sec-converter`, `sec-add`, `sec-charts`, `sec-list`). The sticky `.tabnav` links to these ids and an `IntersectionObserver`-based scroll-spy toggles `.tab-pill.active`. If you add/rename/remove a section, update both the `<section id="...">` and the corresponding `<a data-tab="...">` in `#tabnav`, plus `scroll-margin-top` accounting for the sticky nav height.
- Backup/export: JSON export/import round-trips `{ expenses, settings }` directly; CSV export flattens expenses to TWD-converted rows for spreadsheet use. There is no server sync — data is per-browser only, which the UI explicitly warns about.
