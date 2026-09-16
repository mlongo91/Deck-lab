# Commander Deck Lab — notes for Claude Code

This file exists so a Claude Code session starting cold on this repo has the
same context that built it. Read this before making changes.

## What this is

A single-file static web app that analyzes Magic: The Gathering Commander
decks — legality, the official Bracket 1–5 rating, combo detection, win
conditions, and upgrade suggestions. No build step, no framework, no backend.
Live at **https://mlongo91.github.io/Deck-lab/**

The repo name is `Deck-lab` (capital D). GitHub Pages URLs are
case-sensitive — `deck-lab` 404s.

## File map

- `index.html` — the entire app. HTML, CSS, and JS in one file, vanilla JS,
  no dependencies. This is the only file a person edits by hand.
- `.github/workflows/combos.yml` — generates `combos.json` and
  `combos3.json` (see below). Runs on a schedule and on manual dispatch.
- `.github/workflows/edhrec.yml` — generates `edhrec/<slug>.json` files.
  Manual dispatch: leave the commander field blank to refresh a ~300
  commander cache (also scheduled weekly); fill it in to fetch one commander
  on demand.
- `combos.json`, `combos3.json`, `edhrec/*.json` — generated data.
  Never hand-edit these; edit the workflow that produces them.

## Why data comes from GitHub Actions instead of the browser

Two of the three data sources block direct cross-origin browser requests
(no CORS headers), which was diagnosed with a diagnostics panel built
directly into the app (now removed) that hit each endpoint and reported raw
HTTP status vs. "blocked before response." A GitHub Actions runner isn't a
browser, so it fetches fine there; the workflow commits the result into the
repo, and the app reads it same-origin, which no browser will ever block.

- **Scryfall** — allows CORS. Always fetched live, browser-side. Card data,
  legality, color identity, prices, and the official `game_changer` tag all
  come from here in real time.
- **Commander Spellbook** — blocks CORS entirely (confirmed: even a bare GET
  to their API root dies before any response reaches the page). Combo data
  is baked by `combos.yml` from their public bulk export
  (`json.commanderspellbook.com/variants.json.gz`) and split into:
  - `combos.json` — 2-card combos only, ~550KB, loaded on every page view
    because the bracket rating depends on it (a 2-card combo is part of
    what separates Bracket 3 from Bracket 4).
  - `combos3.json` — 3-card combos, ~6.5MB, loaded lazily only when the
    user taps into the "one card away" near-miss panel and asks for it.
    Don't merge these back into one file — the size was the reason they
    were split.
- **EDHREC** — status unconfirmed either way from a real browser. The app
  tries, in order: (1) `json.edhrec.com` directly, (2) two public CORS
  relays (corsproxy.io, api.allorigins.win) in case (1) is blocked, (3) a
  committed `edhrec/<slug>.json` as the last resort. This gives every
  synergy score used to rank upgrade suggestions and rank which existing
  card is the weakest cut. **Worth verifying route (1) or (2) actually
  fires in a real deployed browser** — it was reasoned through and unit
  tested locally, but never watched running live.

## Build-tracking convention

`index.html` has a plain-text build stamp near the top (`build N · short
description`, shown in gold under the title). Bump it on every shipped
change. This was the single most useful debugging tool during development —
several rounds of "it's not working" turned out to be stale deploys, caught
by fetching the live file and grepping for the build string rather than
trusting what was supposedly uploaded. Keep doing this.

**Known gap at time of writing:** the live site was on build 12 while
build 14 (archetype detection, EDHREC relay fallback) existed only in a
chat session and hadn't been pushed. Multiple chat sessions have edited
this repo without visibility into each other's changes — always diff
against `raw.githubusercontent.com/mlongo91/Deck-lab/main/index.html`
before assuming local state matches what's deployed.

## The one real known limitation: archetype detection is a guess

`deckArchetype()` in `index.html` is a hardcoded heuristic — regex
thresholds for seven patterns (Voltron, Tokens, Aristocrats, +1/+1
counters, Spellslinger, Reanimator, Landfall). It exists because the
original recommendation engine matched on keywords in card text alone and
recommended Avenger of Zendikar (a Plants-token payoff) to a deck running
zero Plants, because the card's text contains "+1/+1 counter." The
heuristic fixed that specific failure but has the same structural problem
one level up: any archetype not on the list gets misread the same way.

The correct fix, already discussed and deliberately deferred: one Claude
API call per analysis that reads the actual decklist and describes the
strategy and its real gaps in its own words, replacing the hardcoded
branches in `deckArchetype()` and the Voltron-specific query branch in
`buildRecommendations()`. The app already has the plumbing for this — an
optional API key field in Settings, stored in `localStorage` only, used
elsewhere for combo detection beyond the built-in database. The user
opted not to set up billing for this yet. If that changes, this is the
highest-value next change.

Until then: keep the heuristic honest in the UI about being a guess rather
than a finding, and prefer adding well-tested regex patterns over trusting
edge cases to resolve themselves.

## Testing convention

There's no test framework. What's been used throughout:

1. Extract the inline `<script>` block and run `node --check` on it for
   syntax validation before shipping.
2. Small throwaway harness scripts: stub `window.localStorage` and
   `window.location`, set `S.*` state directly with representative mock
   decks/cards, then call the pure functions (`parseDecklist`,
   `checkLegality`, `assessBracket`, `matchCombos`, `deckArchetype`,
   `offTheme`, etc.) and assert on the output. This caught real bugs —
   e.g. confirming `offTheme()` actually rejects Avenger of Zendikar
   before shipping the fix, and catching a Python test-harness namespace
   collision that looked like a workflow bug but wasn't.

Keep doing both before pushing changes to `index.html` or the workflow
Python.

## Design constraints worth preserving

- Single file, no build step — this is deliberate, not a shortcut. It's
  what lets the app be hosted on GitHub Pages with zero tooling and
  updated by uploading one file.
- Every AI-derived suggestion (recommendations, combos found by an
  optional Claude call) is re-validated against live Scryfall data
  (legality, color identity, budget) before being shown — never trust a
  model's card knowledge for something the rules actually enforce.
- Mobile-first: the primary user is on an iPhone via Safari, added to the
  Home Screen as a PWA-ish bookmark (manifest + apple-touch-icon are
  inlined as data URIs in `index.html`). Don't assume a desktop-sized
  viewport when changing layout.
- Saved decks live in the visitor's own `localStorage` — nothing about
  another person's deck ever touches this repo.

## Backlog

- [ ] Confirm EDHREC's live/relay fetch actually succeeds from a real
      browser session, not just reasoning about CORS.
- [ ] If an API key gets set up: replace `deckArchetype()`'s hardcoded
      branches with a single Claude read of the decklist.
- [ ] `combos3.json` at 6.5MB is a lot for a lazy phone load — consider
      trimming result text further or paginating by color identity.
- [ ] More heuristic archetypes as a stopgap (Stax, Mill, Group Hug,
      generic tribal) if the AI route stays deferred.
