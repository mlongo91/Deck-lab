# Commander Deck Lab

A deck analyzer for Magic: The Gathering Commander, built as a single
static page — no install, no account.

**Live:** https://mlongo91.github.io/Deck-lab/

## What it does

- Paste a decklist (plain text or a Moxfield/Archidekt export)
- Checks Commander legality — singleton, color identity, banned list
- Rates the deck against the official Bracket 1–5 system
- Finds two- and three-card combos and shows what one card away looks like
- Reads what the deck is actually trying to do and suggests upgrades that
  fit it, with a specific card to cut for each addition
- Stage swaps, re-analyze, and save the result — or discard and start over

All card data is live from Scryfall. Combo and synergy data are refreshed
by scheduled GitHub Actions in this repo (see `CLAUDE.md` for how and why).

## Updating

`index.html` is the whole app — edit it, commit, push. GitHub Pages
redeploys automatically within a minute or two. There's a build number
near the top of the page; bump it on every change so it's easy to tell
whether the live site actually picked up an edit.

## Developing this further

See `CLAUDE.md` for the architecture, the current known limitations, and
the backlog — it's written for whoever (or whatever) picks this project
up next.
