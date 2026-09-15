# Ledger

A single-user expense tracker that lives entirely on your phone. No account, no server, no
network calls. Open it, type an amount, pick a category — done in a few seconds.

It's a web app you install to the home screen, so there's no build step and no tooling. You can
edit it from the phone if you want to.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app — markup, styles, and logic in one file |
| `sw.js` | Service worker. Caches the app so it opens offline |
| `manifest.json` | Name, icons, and colours for the installed app |
| `icon-192.png`, `icon-512.png` | Launcher icons |
| `icon-maskable-512.png` | Icon for Android's adaptive launcher shapes |
| `tests.html` | Test page. Open it in a browser; no tooling needed |

## Setting it up

The app needs to be served over HTTPS, because a service worker won't run otherwise. Opening
`index.html` straight from the file manager will not work — you get no offline support, no
install prompt, and unreliable storage.

### Host it on GitHub Pages

1. Create a public repository and put all seven files in the root.
2. Settings → Pages → Source: *Deploy from a branch*, branch `main`, folder `/ (root)`.
3. Wait a minute, then open `https://yourname.github.io/your-repo/` in Chrome on the phone.
4. Three-dot menu → **Install app**. If you only see *Add to Home screen*, something in
   `manifest.json` or an icon path is wrong.
5. Open the installed app → Settings → **Keep data safe → Turn on**. This asks the browser not
   to clear your expenses when the device runs low on space.

All paths are relative, so it works at any repo name or subpath without editing anything.

### Or test on a computer first

```
python3 -m http.server 8000
```

Then open `http://localhost:8000`. Localhost counts as a secure context, so the service worker
runs and you get DevTools — the Application tab shows your IndexedDB contents and any manifest
errors.

**Data does not move between hosts.** IndexedDB is scoped to the origin, so entries made at
`localhost:8000` won't appear at `github.io`, and entries at `github.io` won't follow you to a
custom domain. Pick the final URL before you start logging real expenses.

## When you change something

The service worker caches aggressively — that's what makes it work offline, and it also means
your edits won't show up. Bump the version string at the top of `sw.js`:

```js
const VERSION = "ledger-v2";   // was v1
```

That throws away the old cache on next load. During development on desktop, tick
**Update on reload** in DevTools → Application → Service Workers instead.

## How the data is stored

One IndexedDB database, `ledger`, with two object stores.

**`expenses`** — key `id`, auto-incrementing, with an index `byDate` on `dateEpochDay`.

| Field | Type | Notes |
|---|---|---|
| `id` | number | assigned by the database |
| `amountMinor` | number | whole paise or cents. Never a decimal |
| `categoryId` | string | matches an id in `CATEGORIES` |
| `note` | string | may be empty |
| `dateEpochDay` | number | days since 1 Jan 1970, UTC. Indexed |
| `createdAt` | number | epoch millis, orders entries within a day |

**`settings`** — key/value pairs: `currencyCode`, `monthlyBudgetMinor`, `themeMode`,
`lastCategory`, `lastExport`.

Two rules worth keeping:

Money is always whole minor units. `19.99` is stored as `1999`. Amounts are parsed with string
arithmetic in `parseMinor`, never `parseFloat`, so 500 entries of ₹19.99 total exactly ₹9,995.00
with no drift. Only the final display step divides.

Dates are epoch day numbers computed in UTC, so daylight saving can never shift an expense into
the wrong day or month. A month query is a range scan: `IDBKeyRange.bound(firstDay, lastDay)` on
the `byDate` index.

## Adding a category

Open `index.html`, find `CATEGORIES` near the top of the script, and add a line:

```js
const CATEGORIES = [
  { id:"food",   name:"Food",   color:"#2E6F4F" },
  ...
  { id:"travel", name:"Travel", color:"#4A6B7C" }   // new
];
```

That's the whole change — chips, legend, chart segments, and CSV import all read from this list.

Two things to be careful about. **Never reuse or rename an existing `id`**, because saved
expenses point at it by string; change `name` freely, but leave `id` alone. And if you delete a
category that already has expenses against it, those rows still render — they fall back to
showing the raw id in grey. Better to re-file them first.

`categoryId` is a string rather than a number precisely so this stays a one-line change, and so
user-editable categories can be added later without rewriting stored rows.

## CSV

Export writes `date,amount,category,note`, sorted oldest first — for example
`2026-09-14,19.99,food,Coffee`. Amounts are in major units, notes are quoted when they contain a
comma.

Import appends rather than replacing, and tells you how many rows landed and how many were
skipped. A row is skipped if the date isn't `YYYY-MM-DD` or the amount doesn't parse above zero.
An unrecognised category becomes `other` rather than failing the row. A header line is detected
and ignored.

Export is your only backup. Clearing this site's browser data erases everything, with no undo.

## Tests

Open `tests.html` in a browser — on the phone or on localhost. It pulls the real functions out of
`index.html` and checks the money arithmetic, the month boundary cases (the 1st and the 31st each
belong to their own month), leap years, and the currency formatting.

## Known limits

Swipe-to-delete is touch-only. On a desktop browser, tap a row and use **Delete** in the sheet.

The list renders every row in the month. A thousand entries is fine; if you ever get to tens of
thousands in a single month, it would need windowing.

iOS support is weaker than Android. Safari can install to the home screen, but storage eviction
rules are stricter, so export more often if that's your phone.
