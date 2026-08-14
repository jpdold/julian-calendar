# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

There is no build system, package manager, test runner, or linter — and no dependencies. The entire app is one self-contained file, `index.html` (~630 lines, inline CSS + JS). Don't look for `package.json` or a dev server.

- **Run it:** open the file directly — `Start-Process index.html` (Windows) or any browser. `file://` is fine; no HTTP server is required.
- **Verify a change:** reload the page. There are no automated tests; changes are checked by hand in the browser.
- **Voice Entry requires Chrome or Edge plus an internet connection** (the Web Speech API relays audio to the browser vendor's speech service) and mic permission. It silently does nothing useful in Firefox — `listen()` alerts and bails when `SpeechRecognition` is absent. Voice cannot be tested from `file://` in some Chrome builds; serve over `http://localhost` if the mic prompt never appears.

## "Julian date" means day-of-year

Throughout this codebase, *Julian date* is the **ordinal day of the year, 001–365/366** (DoD/logistics convention), not the astronomical Julian Day Number and not the historical Julian calendar. `jdnOf(y,m,d)` just sums the preceding months' lengths. The page title says "JDN" but the math is day-of-year — don't "fix" one to match the other without asking.

The underlying grid is proleptic Gregorian: `dowOf()` delegates to `new Date(y, m-1, d).getDay()`, and every day box shows the Gregorian date as a secondary badge.

## Architecture

**State** is three module-level globals in the `<script>`: `store`, `nextId`, and `view`.

- `store` maps a **Gregorian day key** `"y-m-d"` (built by `key()`, parsed by `parseKey()`) to an array of entries.
- An entry is `{id, jewel, what, type, time, desc, prio, prioC}`. **The date is not stored on the entry** — it is the store key. Moving an entry to another day means splicing it out of one array and pushing it into another (see the `editing` branch of the entry-save handler), not mutating a field.
- `nextId` is a monotonic counter, and it is persisted alongside `store` so that reloaded data doesn't collide with newly created entries.

**Rendering is full teardown, no diffing.** `draw()` dispatches to either `render()` (month grid) or `renderChart()` (year table) based on `chartMode`; both wipe their container with `innerHTML=""` and rebuild every node plus every listener. Consequence: **any mutation of `store` must end with a `draw()` call** or the screen goes stale, and nothing may hold a DOM reference across a redraw.

**Three independent view flags** — don't conflate them:
- `chartMode` (JS boolean) — month grid vs. year chart. Also repurposes the ◀/▶ buttons to step by *year* instead of month.
- `body.chart` (CSS class) — the styling half of the above, toggled in `setMode()`.
- `body.bej` (CSS class) — Bejeweled mode, a pure-CSS overlay that hides card text and renders animated gems. It changes behavior in two places: hover tooltips only fire when it's on (`showTip` early-returns otherwise), and a card click opens the details modal instead of doing nothing.

**Day-box packing** is the trickiest part. `layout(n)` returns `[{x,y,s}]` positions inside a 144 px box using fixed tiers: 1 entry → one 144 px card; 2–4 → 72 px quadrants; 5–9 → 48 px ninths; 10–17 → eight 48 px cells plus the ninth cell subdivided into 16 px squares. `MAX_PER_DAY = 17` is the hard cap and is enforced separately at each of the four insertion points (day-box click, entry save, voice entry, and the edit-moves-to-a-full-day case).

The sizes are physical: `--cell:144px` is 1.5 in at 96 dpi, so 72/48/16 are 0.75 in / 0.5 in / 0.1667 in. **`layout()`'s `s` values are coupled to CSS class names** — each card gets `class="card s{s}"`, and `.s144/.s72/.s48/.s16` carry the matching font sizes. Adding a new size tier requires edits in both places, and `.s16` additionally hides all text and renders a dot.

**`WHAT_TYPES`** is the single taxonomy driving the What/Type dropdown pair *and* the voice parser, which iterates its keys. Adding a category makes it speakable automatically.

**`parseVoice(raw)`** is a destructive-consume pipeline: it matches what → type → time → date in that order, deleting each match from the string, and whatever survives (minus a filler-word blocklist) becomes the description. **Order is load-bearing** — reordering the stages changes what lands in the description. Two deliberate quirks: a bare hour of 1–6 with no am/pm is assumed to be PM, and a matched *type* word is only stripped when a *what* word was also spoken, so "physical therapy" survives intact as a description.

## Persistence

`{nextId, store}` is mirrored into `localStorage` under `LS_KEY` (`"julian-calendar.v1"`) by `persist()`, called from `draw()` — the one choke point every mutation already passes through. `restore()` runs once in the init IIFE, before the first `render()`. **A new mutation path therefore needs no persistence code of its own, as long as it ends in `draw()`.**

Save JSON / Load JSON remain the portable path and write the identical `{nextId, store}` shape, so a file and a storage payload are interchangeable.

Both entry points go through `adopt(d)`, which validates the payload and then **reseeds `nextId` past the highest live `id`**. This matters because ids are the only handle `findEntry`/edit/delete have; a stale or hand-edited `nextId` would otherwise reissue a live id and make two entries indistinguishable.

Storage failures (private windows, blocked site data, quota) are caught, and `lsFailed()` replaces the header hint with a warning exactly once rather than alerting repeatedly — the app keeps working in memory. Bump `LS_KEY` if the entry shape ever changes incompatibly; `restore()` treats an unparseable payload as "start empty" rather than throwing.

## Conventions

Match the existing style rather than modernizing it: `"use strict"`, ES5-ish `function` declarations, semicolon-dense compact lines, section banners like `/* ====== name ====== */`, and DOM built via `document.createElement` + `textContent` for anything user-supplied (`innerHTML` is used only for fixed markup). Keep the file self-contained — no external scripts, stylesheets, fonts, or network calls, since the app is meant to run offline from `file://`.

There is a `prefers-reduced-motion` block near the end of the CSS that neutralizes the priority blink and the Bejeweled shimmer; new animations should be disabled there too.
