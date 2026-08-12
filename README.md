# Haz Labs — Waste Authorization Ledger

Single-file static web app (`index.html` — HTML/CSS/vanilla JS, no build step, no
framework) that lets Haz Labs staff browse waste-authorization records stored in
Supabase: which companies are authorized to handle which waste categories, in what
quantities, under which application numbers, and whether those authorizations are
still active.

Everything — markup, styles, and script — lives in `index.html`. There is no backend
of its own; Supabase (Postgres + PostgREST + Auth) is the entire backend, called
directly from the browser with the **publishable/anon** key.

## Data model (Supabase)

- **`v_waste_status`** — the main view the app reads from. One row per
  (company, application, waste-category line item). Columns used:
  `company_name`, `address`, `district`, `application_no`, `granted_date`,
  `expiry_date_raw`, `status` (`active` / `expired` / `unknown`), `category`
  (raw string like `"5.1 - Spent solvents"`), `quantity`, `unit_raw`,
  `unit_group` (`mass` / `volume` / `onetime` / `unspecified`),
  `quantity_normalized`, `id`.
- **`district_options`** — lookup table of valid districts, column `district`.
- Auth: Supabase Auth (email/password). RLS on `v_waste_status` presumably
  restricts reads to signed-in users — the app's own auth gate is
  belt-and-suspenders on top of that.

The app never writes to Supabase; it's read-only.

## Config

Near the top of the `<script>` block:

```js
const CONFIG = {
  SUPABASE_URL: "...",
  SUPABASE_PUBLISHABLE_KEY: "sb_publishable_...",
};
```

If either is blank, the page shows a `#configError` banner and throws before doing
anything else. Only ever put the **publishable/anon** key here — this file is static
and its source is visible to anyone with the URL.

## The MAX_ROWS ceiling (the concept that shapes everything else)

Supabase/PostgREST enforces a server-side "Max Rows" cap (here, 1000 —
`MAX_ROWS` constant). Any single request is silently truncated to that cap no
matter what range you ask for — requesting `MAX_ROWS + 1` rows to "peek" at
whether a next page exists does **not** work, the extra row just never arrives.

Consequences baked into the code:
- All paginated fetches request **exactly** `MAX_ROWS` rows per page
  (`.range(offset, offset + MAX_ROWS - 1)`).
- A full page (`rows.length === MAX_ROWS`) is treated as a *signal* that a next
  page might exist (`state.hasNextPage`), not as certainty.
- If the user clicks "Next" past the real last page, the fetch comes back empty
  and the app self-corrects: steps back a page, marks `hasNextPage = false`,
  refetches (see `fetchAndRender`).
- Any query that needs *every* matching row site-wide (category dropdown
  population, company-name search) loops in `PAGE_SIZE`-sized chunks and
  concatenates, rather than trusting one big request.

## App flow / state machine

1. **Auth gate** — `checkSession()` on load; `loginWrap` vs `appWrap` toggle.
   Signing in calls `bootstrapApp()` once (`bootstrapped` flag guards re-entry).
2. **Bootstrap** — loads `district_options` into the district `<select>`, wires
   up all the filter controls, loads the initial category pool for `ALL`
   districts.
3. **Filters are "staged", not live** — changing district or waste-category
   selections only updates in-memory `state`; nothing is fetched until the
   user clicks **Search** (`setupSearchButton` → `fetchAndRender`). Exception:
   changing district *does* immediately reload the category dropdown's option
   pool (`loadCategories`), so the picker reflects categories that actually
   exist in that district — it just doesn't fetch ledger rows yet.
4. **`fetchAndRender()`** — the main query: applies district (`.eq`) and
   category filters (`.in`, skipped entirely if every available category is
   selected — see below), fetches one `MAX_ROWS` page at offset
   `state.page * MAX_ROWS`, groups rows into companies (`groupRows`), renders.
5. **`groupRows(rows)`** — reshapes flat rows into
   `company → applications[] → items[]` for the collapsible ledger UI.
   Company key is `company_name + "||" + district` (so same-named companies in
   different districts don't merge).
6. **Render** — `render()` sorts companies by total normalized quantity
   (`state.sortDir`), draws the ledger head/body/pagination bar, and
   `attachHandlers()` wires up the expand/collapse click handlers for company
   rows and application rows (items table is always expanded once its parent
   app row is open).

## Category filter (searchable multi-select combo)

Custom combobox (`setupCombo`), not a native `<select multiple>`. `CATEGORY_POOL`
holds the categories valid for the *currently selected district* (drives the
dropdown list); `CATEGORY_INFO` is a site-wide `raw → {code, name, raw}` map so a
category the user already picked keeps showing as a tag even after switching to a
district that doesn't have it (rendered with a "not present in this district"
style) — the subsequent ledger query then correctly comes back empty instead of the
filter silently vanishing.

`allCategoriesSelected` (every pool category checked, e.g. via "Select all
matches") is treated as equivalent to *no* category filter — the `.in()` clause
is skipped. This isn't just an optimization: sending ~300+ category strings as
URL query params, especially combined with "ALL districts", can build a request
URL long enough for PostgREST to reject outright.

### Default categories only (`state.defaultOnly`)

A switch (`setupDefaultOnlyToggle`) next to the category combo. `v_waste_status`
carries an `is_default` boolean per row/category; when the switch is on:

- `loadCategories()` adds `.eq("is_default", true)`, so `CATEGORY_POOL` (and
  therefore the dropdown) only ever offers default categories.
- `fetchAndRender()` and `searchCompaniesAcrossDB()` **also** add
  `.eq("is_default", true)` directly, independent of `state.categories` — this
  matters when no specific category is selected (the "all categories" case),
  since otherwise that would silently mean "all categories" rather than "all
  *default* categories".

Toggling it re-loads the category pool (same as changing district) but doesn't
re-run an already-displayed search on its own — same "click Search to apply"
convention as every other filter here. `setupClearFilters()` resets it to off.

## Company name search — searches every page of the *filtered* results, not just the loaded page

This is the feature most likely to be revisited, so the concept is worth being
explicit about: **it is a separate, direct Supabase query**, independent of the
paginated main ledger query — but it is intentionally scoped to run *on top of*
an already-run district/category search, not as an alternative to one.

- The input starts `disabled` and stays that way until a main **Search** has
  run at least once (`hasSearched`, toggled via `enableCompanySearch()`) — the
  workflow is: pick district/category → click Search → *then* company search
  becomes usable. It re-disables if the user clears filters back to the
  pre-search state.
- Debounced (350 ms, `debounce()` helper), requires ≥2 characters before it
  fires, to avoid hammering the DB on every keystroke.
- `searchCompaniesAcrossDB(query)` runs `ilike company_name ilike '%query%'`
  (with `%`/`_` escaped so literal wildcards in a company name don't act as
  patterns) against `v_waste_status`, **re-applying whatever district/category
  filters the last main Search used** (`state.district`, `state.categories`) —
  it does not search outside those filters. It ignores pagination, though:
  it loops in `PAGE_SIZE` (1000) chunks up to `MAX_PAGES` (5, i.e. 5000 rows)
  and unions the results, so it finds a match on page 4 of the filtered set
  even while page 1 is what's on screen.
- A stale-response guard checks the input box's current value still matches
  the query that was in flight before applying results (a slower older
  request finishing after a newer one shouldn't clobber it).
- While active, `state.companySearchMode = true`: the pagination bar is hidden
  (results aren't paged — they're the full filtered match set already), and
  the `#sortNote` / `#companySearchNote` text switches to search-specific
  wording.
- Clearing the box reverts to whatever the main paginated search last showed
  (`fetchAndRender()`, since company search can only be active when
  `hasSearched` is already true).
- A brand-new main **Search** click always drops any active company search
  (`state.companySearchMode = false`, input cleared, then re-enabled once the
  new search's results are in) since it replaces the underlying filtered set
  the company search was scoped to.

## Sorting

Only one sort dimension: total normalized quantity (`quantity_normalized`,
summed per company/app via `companyQtyTotal` / `appQtyTotal`), ascending or
descending (`state.sortDir`, toggled by `.dir-btn` elements). Applies at every
level — companies, their applications, and items within an application
(`sortByDir`). Items with a null/undefined normalized quantity sort as
`-Infinity` so they always land at the "smallest" end.

## Pagination

Simple offset-based paging in `MAX_ROWS`-sized pages, driven entirely by
`state.page`. Prev/Next buttons re-run `fetchAndRender()` at the new offset.
Hidden entirely during company search mode (see above), since that mode
already fetched everything matching in one go.

## Known constraints worth remembering

- Server row cap is 1000 (`MAX_ROWS`) — never assume a single request returns
  more.
- Company search caps at 5000 total matching rows (`MAX_PAGES * PAGE_SIZE`);
  beyond that it shows a "refine your search" note rather than fetching
  indefinitely.
- No write path exists in this app at all — purely read/browse.
- No server-side full-text index assumed for the `ilike` company search —
  fine at current data volume, but would need a Postgres trigram/GIN index if
  the table grows large and searches get slow.
- Company/category names are interpolated into `innerHTML` without escaping
  (pre-existing pattern throughout the render functions) — acceptable only
  because the data source is internal/trusted, not user-submitted HTML.

## File map (all inside `index.html`)

- `<style>` — design tokens (`:root` custom properties), layout for login
  screen, filter bar, combo dropdown, ledger (company/app/item rows), flat
  view leftovers, pagination.
- `CONFIG` / `MAX_ROWS` — top of `<script>`.
- Auth: `showAppFor`, `checkSession`, login/sign-out handlers.
- Bootstrap: `bootstrapApp`, `loadCategories`.
- Filter UI: `setupCombo` (category combobox), `setupSortToggle`,
  `setupSearchButton`, `setupPagination`, `setupClearFilters`.
- Company search: `setupCompanySearch`, `searchCompaniesAcrossDB`, `debounce`.
- Data fetch/shape: `fetchAndRender`, `groupRows`, `splitCategory`.
- Render: `render`, `renderPagination`, `renderCompany`, `renderApp`,
  `attachHandlers`, `filteredCompanies`, sort helpers (`companyQtyTotal`,
  `appQtyTotal`, `itemQty`, `sortByDir`).
