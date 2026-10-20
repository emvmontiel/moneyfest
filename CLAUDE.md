# Moneyfest

A single-file personal finance app for Philippine pesos (₱). Everything lives in `index.html` (~2,650 lines): CSS, markup, and one IIFE of vanilla JS. There is no build step, no package manager, and no tests. Open the file in a browser to run it.

## Project structure

```
index.html
  <style>          lines ~11-321    CSS variables (light/dark), component classes
  <body>           lines ~323-342   #app mount, bottom tab bar, overlays, hidden file inputs
  <script>         lines ~344-2660  one IIFE ('use strict'), organized by /* ===== section ===== */ banners
README.md                           stub
```

External dependencies (CDN only): Google Fonts (Manrope, Source Serif 4) and SheetJS `xlsx@0.18.5` (import/export). No other libraries.

### Script sections (in order)

| Section | What it holds |
|---|---|
| helpers | `$`, `$$`, `el()` DOM builder, `svgEl`, `round2`, `peso`/`pesoShort` (en-PH / PHP), date helpers (`todayISO`, `mkOf`, `addMonths`, `monthLabel`, `daysIn`), `uid`, `toast`, `normName`, `cleanTag`/`parseTags` |
| defaults | `DEFAULT_CATEGORIES`, `DEFAULT_WALLETS`, `KEYWORDS` (regexes, incl. Filipino/Cebuano terms, mapping item names to category ids) |
| state | global `state` object, `mergeSettings()`, lookup helpers (`cat`, `wallet`, `catName`, `walletName`), `rebuild()` |
| storage | `store` object: `init`, `saveSettings`, `changeMonth`, `commit`, write queue + retry |
| money math | `walletDelta`, `spendOf`, `incomeOf`, `balances`, `summarize`, `debts`, `netWorth`, goal helpers |
| parsing & rules | `splitItems`, `parseItem`, `ruleMatches`, `findRule`, `guessCategory` |
| shared UI | `categorySelect`, `walletSelect`, `describe`, `entryList`, `barList`, `progress`, `offerRule` |
| views | `viewAdd`, `viewEntries`, `viewPlan`, `viewReports`, `viewMore` + their sub-panels, sheets, and `openRecap` |
| import & export | `planImport` (3 formats), `buildWorkbook`, `exportXlsx` |
| render & nav | `render()`, tab click wiring, global Escape handler, bootstrap at the bottom |

## Architecture

- **No framework.** `render()` clears `#app` and rebuilds the current view from `state` using `el(tag, attrs, ...kids)`. It preserves scroll position and refocuses the previously focused input (matched by `id`, `aria-label`, or `placeholder`, so keep those stable on inputs that should survive a redraw).
- **Any data change triggers a full re-render** via `store.emit()` → `render()`. Views must be pure functions of `state`.
- **Modals** are a single `#sheet` inside `#scrim`; open by filling `#sheet` and un-hiding `#scrim`, close with `closeSheet()`. The month recap is a separate full-screen `#recap` overlay. Other overlays: `#bulkbar` (multi-select actions), `#rulebar` (the "always put X in Y?" prompt), `#toast`.
- **Delete buttons use tap-twice-to-confirm** (`armed` flag), not `confirm()`.
- Theme: CSS variables on `:root`, dark via `prefers-color-scheme` or `data-theme`. Semantic colors: `--earn`, `--spend`, `--debt`, `--move`, `--warn`.
- Money is rounded with `round2()` everywhere; never do float math without it.

## Storage

`store.init()` picks a mode:

- **`db`**: when `window.claude.use('db')` and `use('user')` exist (page opened from a Claude account). Data lives at `data/users/{userId}/`:
  - `settings` doc: the whole settings object, rewritten on each `saveSettings`.
  - `ledger/months/{YYYY-MM}` docs: `{ entries: [...] }`, one doc per month. `changeMonth` reads-modifies-writes a month doc through a per-key promise queue with retry on `unavailable`/`resource_exhausted`; an empty month doc is deleted.
  - Both are subscribed with `onSnapshot`, which feeds `state` and re-renders.
  - `migrateDays()` folds the legacy `ledger/days/*` docs into months once.
- **`local`**: fallback; everything is JSON in `localStorage['moneyfest-local-v2']` (`{ settings, months }`), with a one-time upgrade from `moneyfest-local`. A banner warns that data is browser-only.

Optional host capabilities (all feature-detected): `downloads` (save the xlsx), `sample` (Claude call used for receipt scanning, only offered if it supports images).

Write entries through `store.commit({ put: [...], del: [ids] })`. It handles an entry moving between months when its date changes. Write settings through `store.saveSettings(patch)` (shallow merge).

## Data model

### In-memory `state`

`settings`, `months` (Map `YYYY-MM` → entries[]), `entries` (all entries flattened and sorted by `date` then `ts`), `index` (Map `id` → entry), plus UI state: `tab`, `mode`, `planView`, `planMonth`, `reportView`, `reportMonth`, `more`, `filters`, `selected`, `quickDraft`, etc. `entries`/`index` are derived; only call `rebuild()` after changing `months`.

### Entry (stored in month docs)

```js
{
  id,            // 16-char uid
  ts,            // creation timestamp, tiebreaker for ordering
  date,          // 'YYYY-MM-DD' (determines which month doc it lives in)
  kind,          // see below
  item,          // free-text name / label
  amount,        // pesos, > 0 always (sign comes from kind)
  category,      // category id (expense / income / forgiven)
  wallet,        // wallet id ('' when unpaid)
  toWallet, fee, // transfers only
  person,        // debt kinds only
  pcs,           // optional quantity (>1)
  tags: [],      // lowercase slugs, no '#'
  note,
  reviewed,      // false => sits in the "To review" queue
  hidden,        // excluded from spending/income/reports (still moves wallet balance)
  unpaid,        // expense not paid yet: affects spending but no wallet delta
  billId,        // set when logged from a Bill
  src            // 'csv' | 'sheet' for imported rows
}
```

`kind` values and their effects (`walletDelta` / `spendOf` / `debts`):

| kind | Wallet | Counts as | Debt ledger |
|---|---|---|---|
| `expense` | −amount | spending in `category` | – |
| `income` | +amount | income in `category` | – |
| `transfer` | −(amount+fee) from `wallet`, +amount to `toWallet` | only `fee` counts, as spending in `fees` | – |
| `lent` | −amount | nothing | person owes me + |
| `borrowed` | +amount | nothing | I owe + |
| `got_back` | +amount | nothing | person owes me − |
| `paid_back` | −amount | nothing | I owe − |
| `forgiven` | none | spending in `category` (default `presents`) | person owes me − |

Debts are grouped per person by case-insensitive trimmed name (`debts()`); there is no separate person record. The UI calls debt "utang" in places (internal ids `utang`).

Changing an entry's category also flips its `kind` between `expense` and `income` based on the category's `type`.

### Settings (`mergeSettings` defines the shape and defaults)

```js
{
  categories: [{ id, name, type: 'expense'|'income', hidden? }],  // 12 spending + 4 income defaults
  wallets:    [{ id, name, opening, hidden?, limit? }],           // cash, gcash, maya, bank, creditcard (always ensured)
  openingDate,   // opening balances are "as of" this date; balances() can compute before/after it
  lastWallet,    // default wallet for new entries
  rules: [{ id, text, match: 'contains'|'exact', category, tags[], hide, skipReview }],  // first match wins, order matters
  tags: [],      // extra known tags (entries' tags are merged in by allTags())
  budget: { set, income, buckets: { [catId]: 'fixed'|'nonmonthly'|'flex' }, planned: { [catId]: amount }, start },
  bills: [{ id, name, amount, day, category, wallet, match }],
  goals: [{ id, name, target, date, wallet, monthly, created, allocs: [{ d, a }] }],
  recapSeen      // last month key whose recap was viewed
}
```

Category and wallet ids are stable; renaming only changes `name`. Hiding removes them from pickers but keeps history. Default ids are readable (`food`, `gcash`); user-created ones are `c`/`w` + 8 random chars.

### Derived calculations

- `balances(asOf)`: wallet balances at the start of `asOf` (exclusive), from opening balances plus entry deltas.
- `summarize(entries)`: `{ income, spent, net, cats, incCats }` using `spendOf`/`incomeOf`.
- `netWorth(asOf)`: wallets + owed to me − I owe.
- Budget (`budgetNumbers`): flexible plan = `budget.income` − fixed plans − non-monthly plans − monthly goal contributions; non-monthly buckets roll over from `budget.start` (`rolloverBefore`).
- Goals: money is "set aside" via `allocs`, it never moves between wallets. `setAsideIn(wallet)` is subtracted to show what is free.
- Bills: `billStatus` is paid when a non-hidden expense in that month has the bill's `billId`, or its item matches the bill's `match` words (and category, if set).

### Categorization pipeline (`guessCategory`)

Order of precedence: user's pick this session (`state.overrides`) → saved **rule** → **history** (last reviewed entry with the same normalized name) → **KEYWORDS** regex guess → `otherincome` (incoming) or none. `reviewed` is `true` only when a rule decided it (or the user confirmed); everything else lands in the review queue. After a manual category change, `offerRule()` offers to save a rule and re-categorize past matches.

## Screens

Bottom tab bar with five tabs (`state.tab`): the review count shows as a badge on Transactions.

1. **Add** (`viewAdd`, title "Transactions"). Mode chips: *Expense*, *Income*, *Move Funds*.
   - Expense/Income panel (`itemsPanel`): notes field with autocomplete from past entries (selecting a known name prefills amount/category/wallet), amount, pcs, date, category, wallet, "Not paid yet", and a "This is a debt" toggle (lent / paid back / forgiven for expenses; borrowed / got back for income) with a person field. Category is auto-guessed on blur. Optional receipt scan if the host supports it. Ctrl/Cmd+Enter saves.
   - Move Funds (`movePanel`): from/to wallet, amount, fee, date, note.
   - `utangPanel` exists but is not reachable from the current UI; debt entry goes through the toggle above and the Debt screen's quick buttons.
2. **Transactions** (`viewEntries`, tab id `entries`):
   - "Money on hand" total (and how much is set aside for goals), or a first-run banner prompting starting balances.
   - Wallet dashboard (flows in/out per wallet over 30/90 days/all time) and per-wallet "Streams" cards.
   - "To review" queue (`reviewBlock`): per-row category select + confirm, "Confirm all".
   - All transactions: search, filters (type, category, wallet, tag, status, min/max, date range), grouped by day, 150-row paging, tap to edit, "Select" mode with bulk bar (change category, wallet, add tag, etc.).
   - Edit sheet (`openEdit`): fields vary by kind; hide, needs-review, not-paid, delete, and **Split** (`openSplit`) into multiple categories.
3. **Plan** (`viewPlan`): shows bills due within 3 days (or missed) at the top, then a segmented control:
   - **Budget**: setup/edit flow (`budgetEditor`, seeded from `historyAverages`), then "Flexible spending left" with per-day allowance, fixed and non-monthly sections, income vs expected. Month navigation.
   - **Bills**: list with status badges, "Log payment" (`logBill`), add/edit sheet.
   - **Goals**: progress, on-track/behind status, "Set aside money"/"Take out", add/edit sheet, free-for-goals pills per wallet.
4. **Reports** (`viewReports`): banner for last month's recap, then Month / Year / Net worth:
   - Month: stats (came in, spent, kept + savings rate, ended with), money-flow Sankey-style SVG (`flowChart`), category bars (tap → filtered Transactions), tag totals.
   - Year (`reportYear`) and Net worth (`reportWorth`, `lineChart` over time).
   - Recap (`openRecap`): swipeable full-screen cards (glance, budget, top categories, bills, net worth). Marks `recapSeen`.
5. **More** (`viewMore`, `state.more` selects the sub-screen):
   - **Debt** (`moreUtang`): who owes you / you owe / settled, with per-person history and quick actions that prefill the Add screen.
   - **Rules** (`moreRules`): edit, reorder, delete rules; apply to past entries.
   - **Tags** (`moreTags`): rename, remove, jump to filtered transactions.
   - **Wallets and categories** (`moreSetup`): opening date, starting balances, credit limit (card-like names only), rename/hide/add.
   - **Import and export** (`moreData`): export to `.xlsx` (one sheet per year in the old monthly layout, plus "All transactions"). Import accepts that round-trip sheet (matches by `ID`, updates or adds), the legacy monthly sheet (`planOldSheet`, reconstructs entries from per-day category totals plus a Notes column of `item (amount)`), or a generic CSV with Date and Amount columns. Imports preview a plan (`showImportPlan`) before applying.

## Conventions when editing

- Keep it one file, no build tooling, no new dependencies unless unavoidable (external scripts only from cdnjs).
- Build DOM with `el()`; set text via `text:` or text nodes, never `innerHTML` with user data.
- Persist through `store.commit` / `store.saveSettings` only, and catch errors with `saveError`.
- Use `peso()` for display, `round2()` for arithmetic, and ISO strings (`YYYY-MM-DD`, `YYYY-MM`) for dates; avoid `Date` parsing across time zones except where the code already does.
- User-facing copy is plain, friendly English; Filipino terms (utang, sweldo, balato) appear in keywords and a few labels.
- Respect mobile layout (max-width 760px column, fixed tab bar, safe-area insets) and both color schemes.
