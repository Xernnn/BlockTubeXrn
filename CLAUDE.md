# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

BlockTube: a Manifest V3 browser extension (Chrome/Edge/Brave + Firefox,
including Firefox for Android) that permanently removes blocked YouTube
channels and videos from feeds, search, and recommendations — deleting the
DOM element outright rather than showing a "blocked" placeholder. No
bundler, no build step: it's a plain unpacked extension loaded directly
from source. All scripts are classic (non-module) so they can share globals
via `self` without an import graph.

## Commands

`package.json` holds dev tooling only (Playwright for the e2e suite,
`web-ext` for Firefox) — the extension itself ships from source.

```bash
npm run check                   # node --check every script + test, parse manifest.json
npm test                        # every test/*-test.mjs, PASS/FAIL summary (~10-15 min)
node test/run.mjs settings age  # only files whose name contains these substrings
node test/allowlist-test.mjs    # one file directly
npm run lint:firefox            # web-ext lint
npm run run:firefox             # launch Firefox with the extension loaded
npm run build:firefox           # package a .zip for AMO
```

Environment facts that bite:

- **A fresh clone has no `node_modules`** — `npm install` before `npm test`.
  On this machine it is a *symlink* to `/tmp/pw-test/node_modules` (untracked,
  as it should be), and `/tmp` is wiped on reboot, so a sudden
  `Cannot find module 'playwright'` is a dead symlink, not a code problem.
- **`web-ext` is not currently installed** (that symlinked tree holds only
  `playwright` + `playwright-core`), so all three Firefox scripts fail until
  `npm install`. They also need a local Firefox binary, which most sandboxes
  lack.
- **`npm test` needs outbound network** (youtube.com, api.github.com) — the
  suite drives live YouTube. `locale-age-test` and the other pure-Node tests
  are the only ones that run offline.

To verify behavior by hand, load it in a browser:

- **Chrome/Edge/Brave**: `chrome://extensions` → Developer mode → "Load
  unpacked" → this folder.
- **Firefox**: `about:debugging#/runtime/this-firefox` → "Load Temporary
  Add-on…" → `manifest.json`.

After changing background/content scripts, reload the extension from the
browser's extensions page; content scripts also need any open YouTube tab
refreshed.

[`test/README.md`](test/README.md) has the test contract, the per-file
coverage matrix, the one known flaky assertion, and how to drive live
YouTube from the shell with Playwright (including the headless flags that
are required for an extension to load at all).

**Tests use synthetic data only.** `blocktube_backup (2).json` and
`blocktube-import.json` in the repo root are the user's real exports,
gitignored — never read them from a test or a script.

## Layout

| path | what's in it |
|---|---|
| `manifest.json` | MV3; carries **both** `background.service_worker` (Chrome) and `background.scripts` (Firefox) |
| `shared/constants.js` | `BlockTube.MSG` message enum, storage config, the `SETTING_GROUPS` toggle tree. Loaded first everywhere; attaches to `self` |
| `background/background.js` | service worker; the **only** writer to storage; owns DNR rules |
| `background/gist-sync.js` | cross-vendor sync via a private GitHub Gist |
| `content/content.js` | injected at `document_start`; CSS pre-paint hide, DOM scrub, hover UI, nav guards |
| `content/content.css` | the hover 🚫 button only — everything toggle-driven lives in content.js's injected `<style>` |
| `popup/`, `options/` | quick-block UI for the current tab; full blocklist manager |
| `test/*-test.mjs` | standalone live-YouTube e2e files |

## Architecture in one screen

Four contexts, no shared module system: they talk **exclusively** through
`chrome.runtime` messages defined in `shared/constants.js`. That file is the
map of what messages exist, and must be loaded (`<script src>` /
`importScripts` / first entry in `content_scripts.js`) before anything that
reads `self.BlockTube`.

Enforcement is layered, fastest first, and each layer exists because the one
above it can't cover everything:

1. **`declarativeNetRequest`** (background) — network-level redirect for an
   exact blocked video ID or a FULL-mode channel's own page. No flash, but it
   only knows URLs.
2. **Instant-hide CSS** (`<style id="bt-instant-hide">`, content) — `:has()`
   rules the style engine evaluates before paint, so a tile never appears.
   Capped at the 400 most recently blocked channels / 250 videos; **that cap
   is a measured performance budget** (0.21s of style recalc at 400 vs 0.86s
   at 1500).
3. **DOM scrub** (`MutationObserver` → `processRenderer()`, content) — the
   only layer that actually deletes nodes, the only one that can read a title
   or a date, and the only one that works without `:has()`.
4. **Navigation guards** (`checkCurrentPageAndRedirect()` +
   `yt-navigate-start`, content) — catches what a URL can't express: this
   video's *channel* is blocked, this playlist's owner is blocked.

Read [`docs/architecture.md`](docs/architecture.md) before changing any of
it.

## Invariants

Each of these is a bug that already shipped once. Breaking one looks like
working code.

- **`runExtras()` is the single list.** `scheduleExtrasScrub()`, the 2s
  heartbeat and the `yt-navigate-finish` handler all *call* it; they must
  never inline a copy. Three copies is how the page-redirect check ended up
  dead on fresh loads while looking wired up.
- **Only `processRenderer()` and `scrubPosts()` run per-mutation.**
  Everything document-wide goes through the ~400ms `scheduleExtrasScrub()`
  throttle, or the first page load gets visibly slower.
- **Never guess a channel.** `/c/…` and `/user/…` name neither identity
  format, and a related-video tile names none at all — report unresolvable
  rather than matching on a display name. Blocking the wrong channel is worse
  than blocking nothing.
- **A channel has two identity formats** (`UC…` and `@handle`) and a tile
  links exactly one, varying by surface. An entry that knows only its own
  format is invisible on half of YouTube — see
  [`docs/blocking-rules.md`](docs/blocking-rules.md) → Channel identity.
- **Never read a date or a title from `document.body`** — it includes
  `<script>` JSON (~807KB of it), which always matches "N units ago". That's
  how an age rule came to bounce every video. Narrow scopes only, everything
  through `plausibleAgeDays()`.
- **`bounce()` replaces, nav guards push.** Standing on the blocked page
  means `location.replace()`, or Back returns to it; guarding *before*
  navigation means the blocked URL must never enter history.
- **`SAFE_LANDING_URL` is defined twice** (content.js and background.js).
  Change both.
- **Adding a setting is four edits**: a leaf in `SETTING_GROUPS`, the gating
  code in content.js, and the toggle-count assertions in
  `settings`/`ui-hide`/`members`/`ux`/`layout2`. Tests must select switches by
  `data-key`, never by index. There are 33 leaves in 7 groups today, 32 on by
  default (`blockInEmbeds` is the opt-out).
- **Don't build test fixtures from real YouTube tag names.** A
  `document.createElement("ytd-video-renderer")` gets upgraded by YouTube's
  own component and wipes your children. Use `bt-test-tile`.

## Deeper docs

| doc | when to read it |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | changing any context: storage chunking, DNR, the scrub pipeline, the settings tree, sync layers |
| [`docs/blocking-rules.md`](docs/blocking-rules.md) | block modes, whitelist/age rules, keyword+duration filters, the allow-list, playlists/posts/comments, channel identity |
| [`docs/subscriber-counts.md`](docs/subscriber-counts.md) | the sub-count scrape and why it runs exactly once per entry |
| [`docs/ui-and-tools.md`](docs/ui-and-tools.md) | locale labels, right-click blocking, `?bt-debug`, the undo log |
| [`docs/fragility.md`](docs/fragility.md) | something stopped being blocked, or you're about to touch a YouTube selector |
| [`test/README.md`](test/README.md) | running, writing, or debugging a test |
| [`README.md`](README.md) | the user-facing manual (install, sync setup, feature walkthrough) |
