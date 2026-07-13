# Ledger — personal finance tracker

A single-file, no-backend expense/income/savings tracker. Everything — markup, styles,
and logic — lives in `money-tracker.html`. No build step, no dependencies, no server.

## Running it

Just open `money-tracker.html` in a browser. To use it like an app:

- **GitHub Pages**: push this repo, enable Pages, open the hosted URL on your phone,
  then "Add to Home Screen" from the browser share menu.
- **Local file**: opening the file directly (`file://...`) also works, but each
  browser/profile that opens it will have its own separate data (see below).

## Where your data lives

Everything is stored in the browser's `localStorage` under the key `ledger_data_v1`.
That means:

- Data is tied to **one browser on one device**. Same site in a different browser,
  or in an incognito window, starts empty.
- Clearing browser data/cache can wipe it.
- There's no sync between devices.

**Settings → Export backup (JSON)** downloads the entire data object as a `.json`
file. **Import backup** reads one back in and *replaces* everything currently
stored — it asks for confirmation first. There's no merge logic; import is all-or-nothing.

Back up periodically, especially before browser updates, OS updates, or clearing site data.

## Data model

Everything lives in one JS object, `DATA`, persisted as a whole on every change:

```js
DATA = {
  categories: [
    { id, name, kind: 'expense' | 'income', color, budget: number | null }
  ],
  entries: [
    { id, type: 'expense' | 'income', amount, categoryId, date: 'YYYY-MM-DD', note, recurringId? }
  ],
  recurring: [
    { id, name, amount, categoryId, type, day: 1-31, startDate, active }
  ],
  goals: [
    { id, name, target, deadline: 'YYYY-MM-DD' | null,
      contributions: [{ id, date, amount }] }
  ],
  reports: [
    { id, type: 'monthly', year, month, generatedAt, data: {...snapshot} }
  ]
}
```

`entries` is the single source of truth for money movement. Recurring expenses don't
store their own running totals — they generate real `entries` rows (tagged with
`recurringId`) when their due date arrives.

## How recurring expenses work

`processRecurring()` runs once on every page load. For each active recurring item it
walks month by month from `startDate` up to today, and for each occurrence where
`entries` doesn't already have a matching row (`recurringId` + that exact date), it
inserts one. This means:

- If you don't open the app for a while, it "catches up" and backfills every missed
  occurrence the next time you open it — it does not skip missed months.
- Editing a recurring item's amount only affects entries created *after* the edit;
  past auto-created entries are untouched (so historical reports stay accurate).
- Deleting a recurring item does not delete the entries it already created.

## How reports work

Reports are **snapshots**, not live views. `buildMonthlySnapshot(year, month)` reads
current `entries`/`goals` at the moment you click "Generate report" and saves the
computed result into `DATA.reports`. Viewing a saved report later shows that frozen
snapshot, even if you've since edited or deleted the underlying entries.

- If a report already exists for a month, opening Reports shows it plus a
  "Regenerate with latest data" button (overwrites that month's saved snapshot).
- Weekly view (the Monthly/Weekly toggle) is **not** saved — it's computed live from
  `entries` every time, using simple 7-day buckets starting on day 1 of the month
  (not calendar weeks/Mondays).
- Month-over-month comparison in a report compares against whatever `entries` exist
  for the previous month at generation time — if you regenerate a report much later,
  that comparison can shift if you've since edited last month's data too.

## Screens (all in one HTML file, toggled via CSS classes, no routing library)

`showScreen(name)` swaps `.active` on `.screen` elements and calls `renderScreen(name)`,
which re-renders that screen's DOM from `DATA` + `state`. There's no virtual DOM —
each render function does a full `innerHTML` rewrite of its section. Simple, but means
large data sets (thousands of entries) would eventually get slow; fine for personal use.

`state` (in-memory, not persisted) tracks: current screen, viewed year/month, selected
calendar day, in-progress entry type/category, and monthly/weekly report toggle.

## Extending it

Common changes and where to make them:

- **New category field** (e.g. an icon): extend the category object shape in
  `defaultData()`, then update `openCategoryModal()` and anywhere categories render.
- **New screen**: add a `<section class="screen" id="screen-X">`, a nav button with
  `data-screen="X"`, an entry in the `titles` object, and a `renderX()` function
  wired into `renderScreen()`.
- **Change budget warning thresholds**: see the `pct >= 100` / `pct >= 80` checks in
  `renderCategories()`.
- **Change how far ahead "upcoming scheduled" looks**: the `in14` date math in
  `renderDashboard()`.

## Known limitations (by design, given no backend)

- Single device/browser only, no accounts, no sync.
- No undo — deletes are immediate after a confirm dialog.
- No multi-currency support; all amounts are formatted with `toLocaleString` in the
  browser's default locale currency style (symbol is hardcoded to `$`).
