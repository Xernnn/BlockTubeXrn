# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

BlockTube: a Manifest V3 browser extension (Chrome/Edge/Brave + Firefox,
including Firefox for Android) that permanently removes blocked YouTube
channels and videos from feeds, search, and recommendations — deleting the
DOM element outright rather than showing a "blocked" placeholder. No
package.json, no bundler, no build step: it's a plain unpacked extension
loaded directly from source. All scripts are classic (non-module) so they
can share globals via `self` without an import graph.

## Commands

The extension still has no build step. `package.json` holds only dev
tooling (Playwright for the e2e suite, `web-ext` for Firefox), symlinked in
from wherever `node_modules` already exists — `npm install` if it's missing.

```bash
# Syntax-check every script + validate manifest.json
npm run check

# End-to-end suite: real Chromium + this extension + live youtube.com.
# Needs outbound network (youtube.com, api.github.com). ~10-15 min for all.
npm test                        # every test/*-test.mjs, prints a PASS/FAIL summary
node test/run.mjs settings age  # only files whose name contains these substrings
node test/allowlist-test.mjs    # one file directly

# Firefox (needs a local Firefox binary — can't run in most sandboxes)
npm run lint:firefox            # web-ext lint
npm run run:firefox             # launch Firefox with the extension loaded
npm run build:firefox           # package a .zip for AMO
```

Plain `node --check <file>` still works for a single file; `npm run check`
just loops it over all of them and parses `manifest.json`.

### Live-test coverage (`test/*-test.mjs`)

The suite is **in the repo now** (Phase 1 of the "implement everything"
pass moved it out of `/tmp`). Each file is standalone — launches its own
persistent Chromium context, prints `ok:` / `FAIL:` lines, exits non-zero
on any failure. `test/README.md` has the contract and the one known flaky
assertion (`settings-test` "removeShorts on"). Uses **synthetic data only**
— never the user's real `blocktube_backup*.json`.

| file | what it checks |
|---|---|
| `settings-test`, `features-test`, `ui-hide-test` | 12 toggles render/persist to `bt_settings`; DNR home rule tracks `redirectHomepage`; `removeShorts` on/off live; masthead / video-actions / voice / account / search-suggestions each remove+restore |
| `import-test`, `improve-test` | 4.7k-channel synthetic import in <0.5s, one bulk write, sync-chunk/overflow split, idempotent re-import |
| `oldformat-debug-test` | `convertOldBlockTube()` parses the original BlockTube export shape; `?bt-debug` logs the banner + every removal's reason to the page console |
| `uc-handle-test` | UC-keyed block → 0 tiles (repro), enrich → `@handle` cached → 18→0 on a fresh page **and** a live open tab via `REBROADCAST_BLOCKLIST` |
| `enrich-test` | 8 channels (subs+@handle+name) in ~1.5s, 8-wide, one write |
| `age-test`, `locale-age-test` | `age=3d` removes week/month/year-old MKBHD tiles; `parseAgeDays()` (pure Node, extracted from content.js) parses "…ago" in 13 languages and rejects non-dates |
| `allowlist-test` | "Never-block" beats a full block + the pre-paint CSS + the DNR redirect; removing it re-blocks |
| `ownpage-test` | a video-only channel's own `/videos` grid 30→0; whitelisted video survives |
| `posts-playlists-test` | a blocked channel's playlists gone from its own Playlists tab + playlist search (both modes); its community posts gone from the Posts tab |
| `members-test` | Join button, members-only tiles, Membership tab all gone; toggle off restores |
| `endscreen-test`, `layout2-test`, `player-fit-test` | modern `.ytp-fullscreen-grid` neutralised; no horizontal overflow 1200–2560px; player + title above the fold |
| `extras-test`, `ux-test` | always-on scrubs; 5 page tabs, page memory, mass-clear + tombstones |

Not covered (no signed-in test account, Chromium only): masthead
Create/Notifications, real Subscriptions feed, Firefox, `m.youtube.com`.

To actually run/verify behavior, load it in a browser:
- **Chrome/Edge/Brave**: `chrome://extensions` → enable Developer mode → "Load unpacked" → select this folder.
- **Firefox**: `about:debugging#/runtime/this-firefox` → "Load Temporary Add-on…" → select `manifest.json`.

After changing background/content scripts, reload the extension from the
browser's extensions page (content scripts also require refreshing any open
YouTube tabs) for changes to take effect.

### Testing against real, live YouTube (no manual browser needed)

There's no *product* test harness, but a real Chromium instance with this
extension loaded and pointed at the live site can be driven from the shell
via Playwright — this has been done successfully in this environment and is
how the `yt-lockup-view-model` finding below was actually caught, not
guessed. It needs outbound network access, which is not guaranteed in every
sandbox this repo might be worked in — try it, and fall back to
asking the user for a live DOM sample if it's unavailable.

```bash
cd /tmp && mkdir pw-test && cd pw-test && npm init -y
npm install playwright@latest
npx playwright install chromium   # ~180MB download, cached under ~/.cache/ms-playwright after
```

Then, loading the unpacked extension requires `headless: "new"` (old
headless mode can't load extensions at all) plus `--no-sandbox` (there's
usually no permission to run `playwright install-deps`, i.e. no real Chrome
sandbox available, in these environments):

```js
import { chromium } from "playwright";
const EXT_PATH = "/home/sonn/Documents/Projects/BlockTubeXrn";
const ctx = await chromium.launchPersistentContext("/tmp/pw-test/profile", {
  headless: false, // overridden by --headless=new below; extensions require this combo
  args: [
    `--disable-extensions-except=${EXT_PATH}`,
    `--load-extension=${EXT_PATH}`,
    "--headless=new",
    "--no-sandbox",
  ],
});
const page = ctx.pages()[0] ?? await ctx.newPage();
await page.goto("https://www.youtube.com/...");
// ctx.serviceWorkers() finds background.js's service worker (confirms it started
// without errors); page.on("pageerror", ...) surfaces content-script exceptions.
```

To exercise the blocklist itself (rather than just watching the DOM get
scrubbed), open the extension's own popup/options page as a normal
Playwright page — those origins get full `chrome.runtime` access, unlike a
page-world `page.evaluate()` on youtube.com itself (that's the content
script's *isolated* world, which Playwright can't target directly):

```js
const extPage = await ctx.newPage();
await extPage.goto(`chrome-extension://${extensionId}/popup/popup.html`);
await extPage.evaluate(() => chrome.runtime.sendMessage({ type: "BLOCK_CHANNEL", id: "@handle", name: "..." }));
```

Caveats hit in practice: no logged-in test account exists or should be
improvised here (don't attempt to automate a real Google login), so
anything gated on being signed in — the masthead's Create/Notifications
buttons, a real subscriptions feed, personalized recommendations — can't be
verified this way. Cross-browser (Firefox) and mobile (`m.youtube.com`,
`ytm-*` selectors) are also unverified by this method; it only launches
Chromium.

**Pitfall when unit-testing a CSS selector in isolation**: don't
`document.createElement()` a real YouTube tag name (`ytd-video-renderer`,
etc.) on a live youtube.com page to build a synthetic test fixture — it gets
silently upgraded by YouTube's own registered custom element definition the
instant it's connected, and that component's own logic can rewrite or wipe
the children you just appended before you get to assert anything, making
the test fail for reasons that have nothing to do with the selector being
tested. Use a fake, unregistered tag name (`bt-test-tile`, plain `div`,
whatever) to test a selector's *logic* in isolation, then separately verify
the real thing end-to-end (block something for real, load a real page,
check what's left) — that's how the `:has()` instant-hide layer below was
actually confirmed.

## Architecture

Four contexts talk to each other exclusively through `chrome.runtime`
messages defined in `shared/constants.js` (`BlockTube.MSG`) — there is no
shared module system, so this file is the map of what messages exist and
must be loaded via `<script>`/`importScripts` before any file that uses
`self.BlockTube`.

The background context is split across two files. On Chrome, `background.js`
is the service worker and pulls the other two in with
`importScripts("../shared/constants.js", "./gist-sync.js")`. On Firefox
(where MV3 background service workers don't run and `importScripts` doesn't
exist), `manifest.json`'s `background.scripts` array loads all three as
classic scripts in order — `shared/constants.js`, `background/gist-sync.js`,
`background/background.js`. The manifest carries **both** `background.service_worker`
and `background.scripts`; each browser reads the one it supports (Chrome logs
a harmless "Unrecognized manifest key 'background.scripts'" warning). The
`typeof importScripts === "function"` guard at the top of `background.js` is
what keeps the two paths from double-loading.

- **`background/background.js`** (service worker) — the *only* writer to
  storage. Owns the chunked `chrome.storage.sync` engine (blocklist entries
  are spread across multiple small keys to stay under the 8KB-per-item /
  512-item sync quota; once that's exhausted, new entries silently fall back
  to `chrome.storage.local` as "local-only"). Also rebuilds
  `declarativeNetRequest` dynamic rules whenever the blocklist changes, and
  broadcasts `BLOCKLIST_UPDATED` to every open YouTube tab after each write.
  `rebuildDnrRules()` assigns rule IDs from fixed bases —
  `DNR_RULE_ID_BASE_VIDEO = 100000` and `DNR_RULE_ID_BASE_CHANNEL = 500000`
  are *bases*, one rule at `base + index` per blocklist entry, while
  `DNR_RULE_ID_HOME = 900001` / `DNR_RULE_ID_SHORTS = 900002` are singletons
  always present regardless of the blocklist. Every call clears all dynamic
  rules and re-adds from scratch, so the ID ranges only need to not collide.
  Rules are **capped** (`DNR_MAX_VIDEO_RULES = 800` — regex-filtered, under
  Chrome's 1000 regex-rule limit; `DNR_MAX_CHANNEL_RULES = 3000` — url-filtered)
  and go to the *most recently blocked* entries (`ts` desc); DNR is only the
  no-flash optimisation for direct navigation, and `content.js`'s
  `checkCurrentPageAndRedirect()` / `yt-navigate-start` guards still cover
  everything else. `updateDynamicRules()` is wrapped in try/catch so a DNR
  failure can never abort a blocklist write. The `DNR_RULE_ID_HOME` /
  `DNR_RULE_ID_SHORTS` singletons are added only when their feature toggle is
  on (`getSettings()` reads `bt_settings`), and a `chrome.storage.onChanged`
  listener rebuilds the rule set when `bt_settings` changes. All other contexts read/write the
  blocklist by sending it messages (`GET_BLOCKLIST`, `BLOCK_CHANNEL`,
  `BLOCK_VIDEO`, `UNBLOCK_CHANNEL`, `UNBLOCK_VIDEO`, `IMPORT_BLOCKLIST`) rather
  than touching storage directly (also `SET_ENTRY_HIDDEN` /
  `SET_CHANNEL_AGE_RULE` — one-entry edits via the generalized
  `mutateEntry(kind, id, mutate)`, of which `mutateChannelEntry` is now just
  an alias — `FETCH_CHANNEL_SUBS` / `BULK_FETCH_CHANNEL_INFO`, which use
  `applyChannelInfo()` for a batched single write, and `ALLOW_CHANNEL` /
  `DISALLOW_CHANNEL` for the `bt_allowlist` "Never-block" override — see
  "Channel allow-list"). `IMPORT_BLOCKLIST` merges in memory and does one
  `writeFullState()` (never `addEntry`-per-entry — a multi-thousand-entry
  import would blow `chrome.storage.sync`'s write-rate quota and take minutes)
  and carries across the optional per-entry extras (`hidden`,
  `blockOlderThanDays`, `subs`).
  Every blocked entry now also carries `updated_at` (bumped on every
  mutation, stamped from `syncedNow()` — `Date.now()` plus a GitHub-derived
  `clockOffset`), and `removeEntry()` writes a **tombstone** to
  `chrome.storage.local` (`bt_tombstones`, `{ "channel:<key>"|"video:<id>":
  deletedAtMs }`, pruned after `TOMBSTONE_TTL_MS`); both exist purely so the
  gist merge below can do last-write-wins without a stale device
  resurrecting a delete. `writeFullState()` re-chunks the whole blocklist
  from a merged in-memory state (what the merge and `IMPORT_BLOCKLIST`
  produce; `addEntry`/`removeEntry` only touch one entry). Its bulk
  `chrome.storage.sync.set` is all-or-nothing, so on failure (one 50-entry
  chunk over the 8KB/item cap — long multi-byte names) it retries in order and
  spills from the first chunk that won't fit onward to the local overflow
  bucket, keeping `bt_ch_0..N` contiguous.

- **`background/gist-sync.js`** (loaded into the background context, before
  `background.js`) — the cross-vendor sync engine. `chrome.storage.sync`
  can't bridge Chrome↔Firefox; this mirrors the blocklist to a single
  **private GitHub Gist** instead (no server), written with a fine-grained
  PAT the user pastes into the options page (`Gists: Read and write` scope,
  stored in `chrome.storage.local` under `bt_sync`, never synced). It
  exposes `self.BlockTube.gistSync` with `init(host)` / `onLocalChange()`
  (debounced push, called from `broadcastUpdate()`) / `onPoll()` (the
  `bt-sync-poll` `chrome.alarms` tick, every `SYNC.POLL_PERIOD_MIN`) /
  `configure(token)` / `syncNow()` / `status()`. The one `cycle()` does
  pull → per-entry LWW merge (on `updated_at`, with tombstones) → apply
  locally via `host.applyMergedState()` (which calls `writeFullState()` then
  `broadcastUpdate(state, { fromSync: true })` so it doesn't loop back into
  another push) → push if the merged result differs from the remote. It
  never touches blocklist storage directly — only through the `host` object
  `background.js` hands it. No realtime channel: changes land on the next
  poll or the next debounced push. Timestamps come from GitHub's response
  `Date` header, not the device clock, so skew can't decide a merge.

- **`content/content.js`** (injected into `youtube.com`/`m.youtube.com` at
  `document_start`) — does four jobs:
  0. **Instant, pre-paint hiding via CSS**: the very first thing this script
     does is create a `<style id="bt-instant-hide">` tag. It holds two
     parts: static rules (Shorts, sidebar Home/Shorts/Report-history,
     masthead Create/Notifications, `#secondary`) set once at startup, and a
     dynamic rule rebuilt by `updateInstantHideBlocklistCSS()` every time
     the blocklist loads/changes, using `:has()` to match any tile
     containing a blocked channel/video link — but only for the
     `INSTANT_HIDE_MAX_CHANNELS` (1500) / `INSTANT_HIDE_MAX_VIDEOS` (500)
     *most recently blocked* (`newestFirst()`), since one `:has()` with a few
     thousand `a[href]` selectors is re-evaluated on every style recalc; the
     rest rely on the one-frame-later JS scrub. Because `:has()` is
     matched continuously by the browser's style engine as part of normal
     layout — not by a JS callback reacting after the fact — this hides a
     tile before the `MutationObserver` below would ever get a turn to run,
     which matters during a fast burst of insertions (e.g. search results
     loading). It only *hides* (`display: none`); the scrub below is what
     actually deletes the node, and is the only thing that works at all in
     a browser without `:has()` support. `manifest.json` sets Firefox
     `strict_min_version` to `115.0`, roughly the `:has()` baseline in
     Gecko — older Firefox falls back to the JS scrub (job 1) only, with a
     one-frame flash the CSS layer would otherwise prevent.
  1. **DOM scrubbing**: a `MutationObserver` on `document.documentElement`
     catches every inserted node, matches it against `RENDERER_SELECTOR`
     (the list of YouTube's custom element tag names for video/playlist/
     channel-card tiles across every surface — home grid, search, sidebar,
     shorts, playlists), and removes it outright if it matches the
     blocklist. Mutation batching happens via `requestAnimationFrame` to
     avoid re-scanning on every one of YouTube's frequent DOM writes. After
     a removal, `scheduleShelfPrune()` checks the tile's shelf ancestor
     (`SHELF_SELECTOR` — "Latest from [channel]", a Shorts shelf, etc.) and
     removes the whole shelf ~1s later if it's now empty, so blocking
     doesn't leave a headless shelf frame behind. The same observer also
     watches for the `is-active` attribute flipping on
     `ytd-reel-video-renderer` — YouTube's vertical Shorts feed keeps
     several mounted at once and just re-flags which is current, so a plain
     childList watch would miss it.
  2. **Hover-to-block UI**: injects a 🚫 button into unblocked tiles with a
     menu listing "Block this video" plus, per distinct channel found in
     that tile (a tile can list more than one via YouTube's
     channel-collaboration feature — see `collectChannels()`), *two* entries:
     a full "Block channel: X" and a softer "Block all videos from X (allow
     some later)" — see "Channel block modes" below for what the difference
     actually does.
  3. **Navigation interception**: `declarativeNetRequest` (background.js)
     only catches exact blocked video IDs or exact channel *page* URLs
     belonging to a FULL-mode channel at the network layer — it cannot know
     "this video's channel/collaborator is blocked" or "this playlist
     belongs to a blocked channel" from the URL alone, and deliberately
     doesn't redirect an EXCEPT_WHITELIST channel's own page at all.
     `checkCurrentPageAndRedirect()` closes the remaining gap by
     re-checking the page's actual rendered channel(s) — via
     `findScopeForCurrentPage()` (scoped to e.g.
     `ytd-watch-metadata`/`ytd-playlist-header-renderer` so it never picks
     up unrelated channels from elsewhere on the page) and `channelBlocks()`
     (which knows about whitelist exceptions) — on load and after every
     `yt-navigate-finish` (YouTube's SPA navigation), redirecting to
     `SAFE_LANDING_URL` (Subscriptions) if blocked. `yt-navigate-start` is
     also intercepted for an earlier bounce on exact video-ID matches and on
     navigating straight to a FULL-blocked channel's page.

  Separately, `content/content.js` also enforces things that are **not**
  blocklist-driven. Each is gated on a **feature toggle** in
  `settings` (`shared/constants.js` `DEFAULT_SETTINGS`, stored as one object
  in `chrome.storage.sync` under `SETTINGS_KEY` / `bt_settings`, all default
  `true`). `applySettings()` merges the stored object over the defaults,
  rebuilds the static instant-hide stylesheet (`buildStaticCSS()` joins only
  the `STATIC_CSS_PARTS` whose toggle is on) plus the blocklist half, and —
  on a *live* change — re-runs `runExtras()`. The startup `runExtras()` is
  deferred until `chrome.storage.sync.get` resolves (`settingsLoaded`),
  because a `remove*` a user has turned off must not fire its irreversible
  `.remove()` once before their settings arrive. `chrome.storage.onChanged`
  keeps it live. Blocklist enforcement itself has no toggle — it's the point
  of the extension. The toggles and what they gate:
  - `removeShorts` → `scrubShorts()` deletes every Shorts shelf, shelf tile,
    and the full-screen player (plus the `/shorts/*` and `reelWatchEndpoint`
    redirects in the nav guards, and the `DNR_RULE_ID_SHORTS` network rule).
  - `cleanSidebar` → `scrubGuide()` removes Home/Shorts from the sidebar by
    matching their link (`a[href="/"]` / `a[href="/shorts"]`, scoped to the
    guide containers only — *not* document-wide, to avoid the masthead logo's
    own `href="/"`), and "Explore"/"More from YouTube"/"Report history" by
    visible label text. Also the sidebar's small-print footer.
  - `cleanMasthead` → `scrubMasthead()` removes the Create and Notifications
    buttons, matched by aria-label/title text.
  - `removeRelated` → `scrubSideRecommendations()` deletes the whole
    `#secondary` column beside the player; the reflow CSS in the toggle-driven
    `<style>` caps `#primary` at
    `min(100%, max(426px, calc((100vh - 144px) * 16/9)))` — wide enough to use
    the freed space, but bounded so a 16:9 player's height stays under
    `viewport − masthead − title`, i.e. the player *and* the title still fit
    above the fold (going full-width made the player taller than the viewport
    on wide screens). The inner chain (`#player-container` etc.) is forced to
    fill that capped column with zero right padding (YouTube's JS had sized it
    for the 2-column layout, leaving a right gap), and `overflow-x: clip` on
    `ytd-app` stops the page ever scrolling sideways — without that, at some
    widths a scrollbar appears and arrow keys pan the page instead of seeking.
    `[theater]` is left alone.
  - `hideSearchSuggestions` → CSS only: hides the search autocomplete dropdown
    (`.ytSearchboxComponentSuggestionsContainer` new / `ytd-search-suggestions-section`
    + `tp-yt-paper-listbox#suggestions` legacy).
  - `removeEndScreen` → `scrubEndScreen()` + CSS kill the end-of-video
    suggestion grid and in-player cards. YouTube renamed these once already
    (`.html5-endscreen`/`.ytp-videowall-still` → `.ytp-fullscreen-grid`/
    `.ytp-modern-videowall-still`); both name sets are handled — see "Known
    fragility" for the details and how to re-probe.
  - `logoToSubscriptions` → `retargetLogo()` rewrites the masthead
    YouTube-logo anchor's `href` to `SAFE_LANDING_URL` (scoped to the
    masthead so it never hits the guide's own Home link).
  - `redirectHomepage` → the `/` → Subscriptions redirect: the
    `DNR_RULE_ID_HOME` network rule (background.js, added only when the
    toggle is on) plus `checkCurrentPageAndRedirect()` / `yt-navigate-start`
    (which also bounce `browseId === "FEwhat_to_watch"` so an SPA logo click
    has no home-feed flash — gated on `redirectHomepage || logoToSubscriptions`).
  - `hideVoiceSearch` → CSS only: `#voice-search-button` + the
    `[aria-label="Search with your voice"]` mic button.
  - `accountButtonOnHover` → CSS only: `ytd-masthead …:has(#avatar-btn)` gets
    `opacity:0`, back to `1` on `ytd-masthead #end:hover`. Nothing to hide
    when signed out (no `#avatar-btn`), so unverifiable without a test
    account, like `cleanMasthead`.
  - `hideVideoActions` → `scrubVideoActions()` (+ a CSS mirror) removes
    Share / Save / Download / Clip / Thanks / "More actions" from
    `ytd-watch-metadata #actions`, matched by `aria-label` via `LABELS_BY_LANG`
    (see "UI-language labels"; English + 5 more, English fallback). Removing
    "More actions" is what takes Report out of reach. Never touches the
    shared `ytd-menu-renderer` wrapper or the Like/Dislike buttons.
  - `hideMemberships` → CSS hides the "Join" button (`#sponsor-button`,
    `ytd-sponsor-button-renderer`, `yt-sponsor-button-view-model`) and legacy
    `.badge-style-type-members-only` tiles; `isMembersOnlyTile()` (a tile
    with a badge whose text is exactly "Members only" — text-matched because
    the new `badge-shape` class `ytBadgeShapeCommerce` is shared with other
    paid badges) is checked in `processRenderer()` for instant removal and in
    the throttled `scrubMembersOnly()` (which also removes a "Membership" /
    "Members-only content" shelf and the channel Membership tab).

  All of the above `scrub*` / `retargetLogo` / `scrubOwnChannelPage`
  functions (there are ~10), plus `recheckHydratingTiles()` (playlist tiles
  with late-hydrating bylines, and keyword tiles with late titles), run
  through the `scheduleExtrasScrub()` throttle — at most every ~400ms rather
  than on every mutation, because several full-document `querySelectorAll`
  passes per DOM write made the first page load noticeably slower. A
  `setInterval` heartbeat re-triggers the same throttle every 2s as a
  fallback, and `yt-navigate-finish` runs the batch on SPA navigation. Only
  the blocklist-driven removals in `processRenderer()` — and `scrubPosts()`,
  cheap because posts are rare — run in the hot per-mutation path
  (`scheduleFlush` / the `MutationObserver` callback via `sweep()`) —
  don't add a new document-wide scan there without routing it through the
  throttle.

  `SAFE_LANDING_URL` (`https://www.youtube.com/feed/subscriptions`) is
  defined independently in both `content/content.js` and
  `background/background.js` (no shared module system to hold one copy) —
  change both if it ever needs to move.

- **`popup/`** — quick-block UI for the tab you're currently on. Asks the
  content script for `GET_PAGE_TARGET` (current video + all channels found
  in scope) and renders both block-mode buttons per channel (see "Channel
  block modes" below), same as the hover menu.

- **`options/`** — full blocklist manager (search, unblock, JSON
  export/import) opened via `chrome.runtime.openOptionsPage()`. Listens to
  `chrome.storage.onChanged` to live-refresh (debounced ~300ms;
  `suppressStorageRefresh` mutes it during a bulk sub-count load). Four
  **top-level page tabs** (`.page-tab` → `showPage()` toggles the `.page`
  divs `#page-blocklist` / `#page-keywords` / `#page-settings` / `#page-sync`;
  last one remembered in `localStorage`), so the keyword list, toggle card
  and gist card aren't in the way of the list. The content column is ~1120px wide. Within
  Blocklist, **filter tabs** (`.filter-tab`, `tab` state) narrow to All /
  full channels / video-only channels / videos / **Hidden**; `RENDER_CAP`
  (300) bounds rows per section. Each row has a **Hide** button
  (`buildHideBtn` → `SET_ENTRY_HIDDEN`); the video section has a
  **"Clear all N blocked videos"** button (`#clear-videos-btn` →
  `CLEAR_BLOCKED_VIDEOS`, a single bulk write + a tombstone per video) with an
  undo entry in the **Recently unblocked** `<details>` below it; a
  video-only channel's whitelist/age panel is a `<details>` collapsed by
  default so a page of them stays short. Channel rows show a subscriber-count
  chip (`buildSubsChip` → `FETCH_CHANNEL_SUBS` for one, cached on the entry,
  stale after 30d) and — once known — the channel's **`@handle`** in place of
  the raw `UC…` id (a channel keyed by `@handle` shows no separate id line at
  all; the `UC…` becomes a tooltip). `bulkFetchSubs()` batches ids (40 each)
  into `BULK_FETCH_CHANNEL_INFO` messages, two in flight; each message
  fetches its batch 8-wide in the worker and does **one** storage write —
  the old one-message-and-one-write-per-channel path is what made a
  few-thousand-channel sweep crawl. It also captures `entry.handle` +
  a real `entry.name`. Button doubles as Stop, mutes the storage-refresh
  meanwhile, fires one `SYNC_NOW` at the end. "Load sub counts"
  (`#load-subs-btn`) does the rendered rows, "Fetch all sub counts"
  (`#fetch-all-subs-btn`) the whole blocklist (resumable — skips fresh
  counts); a successful **import** offers to run the whole-blocklist sweep. A **Sort** select
  (`sortBy`: recent / subs↓ / subs↑ / name — `subsToNumber()` parses
  "1.2M"-style strings, unknowns sort last) and a **"Hide channels under N
  subs"** view filter (`hideSmall` / `smallThreshold`, unknown counts kept)
  sit above the channel list. The **"What this extension changes"** card
  (`#settings-card`, `SETTING_DEFS` → `renderSettings()`) is the 12 feature
  toggles as switches, writing `bt_settings` to `chrome.storage.sync` on
  every flip, plus Reset-to-defaults; it live-updates from
  `chrome.storage.onChanged`. Also hosts the **Sync** card (`#sync-card`,
  `initSync()`) — paste a GitHub PAT to connect, "Sync now" / "Disconnect this
  device", and a status block from `GET_SYNC_STATUS` showing the synced
  channel/video counts, a link to the gist, and a "paste the same token on
  your other devices" hint. (`ensureGistId()` sets `lastPushedAt` when it
  *creates* the gist, since creating it with the snapshot is the first push —
  otherwise the line read "last pushed: never" right after a successful
  connect.) The token round-trips through `SET_SYNC_CONFIG` to
  `background/gist-sync.js`; the options page never talks to GitHub itself. Each EXCEPT_WHITELIST channel row expands into a panel
  (`buildWhitelistSection()`) with the whitelist editor (`extractVideoId()`
  parses a URL or bare ID), the "only block videos older than N days" control
  (`buildAgeRule()` → `SET_CHANNEL_AGE_RULE`), and a flip-to-FULL button.

- **`shared/constants.js`** — message-type enum, storage chunking config
  (`CHUNK_SIZE`, `MAX_SYNC_ITEMS`, key prefixes, plus `TOMBSTONE_KEY` /
  `TOMBSTONE_TTL_MS` / `SYNC_KEY`), the gist-sync config block (`SYNC.*` —
  gist filename, poll period, push debounce), and the feature-toggle
  contract (`SETTINGS_KEY` = `bt_settings`, `DEFAULT_SETTINGS` = 12 booleans
  all `true`). Loaded as a plain script
  everywhere (`importScripts` in the service worker, `<script src>` in HTML
  pages, first entry in `content_scripts.js`) and attaches to `self`.

### Channel block modes

A blocked-channel entry is `{ name, ts, updated_at, localOnly, mode, whitelist,
blockOlderThanDays?, hidden?, subs?, subsAt?, handle?, ucid? }` (a video entry
is `{ title, ts, updated_at, hidden? }`). `mode` is from
`BlockTube.CHANNEL_MODE` (`shared/constants.js`):

- **`FULL`** (the default, and what every entry predating this field
  implicitly is — `channelBlocks()`/DNR treat a missing `mode` as `FULL`):
  the channel's own page is gone (network-level DNR redirect) and every
  tile/video/playlist of theirs is removed everywhere, no exceptions.
- **`EXCEPT_WHITELIST`**: the channel's own page and its non-video tiles
  (its own channel card, "Channels" tab entries, etc.) are deliberately left
  alone — no DNR rule is generated for the channel URL at all (see
  `rebuildDnrRules()` in background.js) — but its *videos* are blocked
  except the IDs in `entry.whitelist` (`{ [videoId]: title }`, managed via
  `MSG.WHITELIST_CHANNEL_VIDEO`/`UNWHITELIST_CHANNEL_VIDEO`, or flipped
  wholesale to/from `FULL` via `MSG.SET_CHANNEL_MODE`). If
  `entry.blockOlderThanDays > 0` (set via `MSG.SET_CHANNEL_AGE_RULE`), only
  videos older than that are blocked — recent uploads pass, whitelist still
  wins. In feeds/search/recs the age rule needs a tile with a channel byline
  **and** a parseable relative publish date; the channel's own `/videos` tab
  tiles have neither byline nor (for age) always a relative date, so
  **`scrubOwnChannelPage()`** handles that surface separately: when
  `normalizeChannelKey(location.pathname)` resolves to a blocked entry it
  treats *every* video tile on the page as that channel's and runs
  `channelBlocks()` on it (whitelist + age honoured), and removes *every*
  playlist tile outright (`info.isPlaylist` — a playlist has no publish
  date, so it goes for any mode incl. age-ruled). A companion
  `<style id="bt-channel-page-hide">` (`updateChannelPageHideCSS()`, scoped
  `ytd-browse[page-subtype="channels"]`) has two clauses — the video/short
  grid (`:not(:has())` the whitelisted IDs, skipped for age-ruled channels
  since CSS can't do dates) and a playlist clause (`:has(PLAYLIST_LINK_SEL)`,
  always). Wired into `scheduleExtrasScrub()`, `yt-navigate-finish`, and both
  blocklist-load paths; the style clears itself when you navigate off the
  channel page. Date parsing (`parseAgeDays()`) is
  language-independent (`AGE_UNITS`, ~15 languages — see "UI-language
  labels"); only `currentPageAgeDays()`'s absolute-date fallback is still
  English. `extractInfo()` only reads the date when `anyAgeRule` is true
  (some blocked channel has the field), so the common case pays nothing.

Other per-entry fields (channels *and* videos): `hidden` — a pure
options-UI flag (`MSG.SET_ENTRY_HIDDEN`); the entry stays fully blocked,
it's just filtered out of the blocklist manager unless the "Hidden" tab is
active. `subs` / `subsAt` / `handle` (channels only) — the scraped
subscriber-count string, when it was fetched, and the channel's `@handle`.
`scrapeChannelInfo()` in the background fetches the channel page (no API key)
and regexes out the sub count, `@handle` (`vanityChannelUrl` /
`canonicalBaseUrl` / `rel="canonical"`) and a real name (`og:title`); it
**streams the response and stops early** (those bits are near the top of
`ytInitialData`, so it doesn't pull the whole ~1MB page) and reports a 429
so the caller can back off. `applyChannelInfo(map)` writes a whole batch of
results with one storage write. `MSG.FETCH_CHANNEL_SUBS` (one channel) and
`MSG.BULK_FETCH_CHANNEL_INFO` (a batch, fanned out 8-wide) both go through
it — **neither calls `broadcastUpdate`** (this metadata is cosmetic: content
scripts and DNR don't use it), so a big sweep isn't N × DNR-rebuilds; the
options page repaints from `storage.onChanged` and pushes to the gist once
at the end via `SYNC_NOW`. All of these fields ride the gist sync
automatically as extra keys on the entry.

### Title-keyword and duration filters

`chrome.storage.sync` key `bt_keywords` =
`{ list: [{ p, re }], ts, durMinSec, durMaxSec }`. A video is scrubbed if
its title matches any pattern **or** (when either bound is > 0) its duration
badge is shorter than `durMinSec` / longer than `durMaxSec`. `content.js`
compiles the patterns in `applyKeywords()` (loaded alongside `bt_settings`,
live via `chrome.storage.onChanged`) into `keywordMatchers`; `blockReason()`
(the single function `isBlocked()` delegates to — it returns a *string* so
the debug log and the real decision can't drift) and the nav guard's
`matchesFilter()` check both. `anyFilter()` gates the work: `extractInfo()`
only reads the duration badge (`readDurationSec()` — scans the
time-status/duration badges specifically, ignoring "4K"/"New") when a bound
is set, only reads the title from `#video-title` when a keyword exists.
**JS-scrub only** — CSS can't match text or parse a badge — so there's a
one-frame window vs the `:has()` layer; `processRenderer()` leaves a
keyword-relevant tile *unchecked* while its title hasn't hydrated and
`recheckHydratingTiles()` re-processes those. The options **Keywords** page
tab edits both (title list + a "Block videos by length" block). The gist
merge carries the whole `keywords` object as last-write-wins on `ts`;
`kwFingerprint()` (list + duration bounds, ts excluded) in `sameBlocklist()`
is what makes a duration-only edit still push.

### Channel allow-list ("Never-block")

`chrome.storage.sync` key `bt_allowlist` =
`{ list: { [channelKey]: { note?, ts } }, ts }` — a hard override that beats
*every* block path: a direct block, a collaborator block, a keyword/duration
filter, an age rule, and the DNR channel-page redirect. `@handle` keys are
stored lowercased. Background owns the writes (`MSG.ALLOW_CHANNEL` /
`DISALLOW_CHANNEL` → `setAllowed()`); it's carried in the `GET_BLOCKLIST` /
`BLOCKLIST_UPDATED` payload and also read live by `content.js` via
`chrome.storage.onChanged`. `content.js` builds `allowSet` and short-circuits
in `blockedEntryFor()` (returns `undefined` for an allow-listed key — covers
the nav guards and `scrubOwnChannelPage`), at the top of `blockReason()`
(covers keyword/duration), in `checkCurrentPageAndRedirect()`, and skips the
channel's clause in `updateInstantHideBlocklistCSS()`. `background.js`
excludes allow-listed keys from `rebuildDnrRules()`. `gist-sync.js` syncs it
exactly like `keywords` (LWW on `ts`, `allowFingerprint()` in
`sameBlocklist()`). Options **Never-block** page tab + a "Never-block"
button in the bulk-select bar manage it. A DOM node already deleted before a
channel was allow-listed only reappears on YouTube's next re-render (same as
unblocking) — the allow-list is not retroactive within a paint.

### UI-language labels (`LABELS_BY_LANG` / `L` in content.js)

Most of YouTube's chrome is matched by tag name / href / stable id — all
locale-proof. The handful matched by visible text — the watch-page action
buttons, masthead Create / voice search, the "Join" / "Members only"
wording, the guide's Explore / "More from YouTube" / "Report history" — read
from `LABELS_BY_LANG[UI_LANG]` (detected from `<html lang>` then
`navigator.language`), with the English strings always merged in and English
the fallback for an unlisted language. Six languages ship (en, vi, es, pt,
fr, de); adding one is a data edit. `ariaSel()` interpolates the label lists
into the static CSS (`cleanMasthead` / `hideVoiceSearch` / `hideVideoActions`
selectors); the JS scrubs (`scrubVideoActions`, `scrubGuide`,
`scrubMasthead`, `scrubMembersOnly`) build `Set`s / regexes from them.
`parseAgeDays()` is **separate and fully language-independent**: `AGE_UNITS`
is a `[unit-spellings-across-~15-languages, days]` table, anchored by the
preceding `<number>` so a short token can't match inside a word
(`locale-age-test.mjs` guards it).

### Debug mode

`?bt-debug` in the URL or `localStorage.bt_debug === "1"` turns on `DEBUG` in
`content.js`. `dbg()` then logs every removal with its `blockReason()` string
to the page console (visible under the content script's context), and
`window.__blockTube` (isolated world — reach it via the console's JS-context
dropdown) exposes `state()`, `why('<selector>' | element)` (runs
`extractInfo` + reports each channel key's index resolution + the
`blockReason` verdict), and `enable()` / `disable()`.

### Recently-unblocked undo log

`chrome.storage.local` key `bt_recent_unblocks` (not synced) — newest-first,
capped at `RECENT_UNBLOCK_MAX` (40). `removeEntry()` pushes a
`{ t: "one", kind, id, entry }` record (full entry, so a restore keeps
mode/whitelist/handle/age rule); `CLEAR_BLOCKED_VIDEOS` pushes one
`{ t: "bulk", kind: "video", entries, count }`. `RESTORE_UNBLOCK { index }`
re-adds via `addEntry` and drops the record; `GET_RECENT_UNBLOCKS` /
`CLEAR_RECENT_UNBLOCKS` round it out. Shown as a `<details>` under the video
section in the options Blocklist tab.

`content/content.js`'s `channelBlocks(entry, videoId, ageDays)` is the
single source of truth both the DOM scrub (`isBlocked()`) and the navigation
guards (`checkCurrentPageAndRedirect()` — which computes `ageDays` via
`currentPageAgeDays()` — and `yt-navigate-start`) call through — pass
`null` for videoId to get "does this mode block the channel itself", a real
video ID (+ optional `ageDays`) to get "does this mode block this specific
video". The instant-hide CSS layer
mirrors this exactly in `updateInstantHideBlocklistCSS()`: an
`EXCEPT_WHITELIST` channel gets its own `:has(channelLink):has(a[href*="v="],
a[href*="/shorts/"]):not(:has(whitelistedLink...))` rule (one clause per such
channel, since each has a different whitelist) rather than joining the
single shared "hide if it has any of these links" rule used for `FULL`
channels and individually-blocked videos (that shared rule's `:is()` also
includes `POST_SELECTOR`, so a FULL channel's community posts fall to it for
free). The `:has(a[href*="v="], ...)` clause is deliberate — it keeps a
soft-blocked channel's own card (no video link) out of the *video* rule. A
soft channel additionally gets, regardless of any age rule, one clause each
for its playlists (`PLAYLIST_TILE_TAGS`, or a lockup `:has(PLAYLIST_LINK_SEL)`)
and its posts (`:is(POST_SELECTOR):has(channelLink)`) — soft channels are
few, so the extra clauses are cheap; FULL channels get playlists/posts from
the shared rule and cost nothing extra. The age-ruled soft channel still
gets **no** *video* clause (CSS can't do date math) — the JS scrub handles
that alone.

### Playlists and community posts

`extractInfo()` sets `info.isPlaylist` (the tile matches `PLAYLIST_TILE_TAGS`
or carries a `PLAYLIST_LINK_SEL` link — `list=PL…/UU…/OL…/FL…`, not
auto-`RD…` mixes). `blockReason()` then returns "blocked" for a playlist
tile as soon as *any* channel on it resolves via `blockedEntryFor()` —
regardless of block mode or age rule, since a playlist has no single date.

Community posts are **not** tiles (`RENDERER_SELECTOR` doesn't list them) —
`POST_SELECTOR` (`ytd-backstage-post-thread-renderer` + friends) gets its
own `processPost()` / `scrubPosts()` path: any `/@` or `/channel/UC` link in
the post that resolves to a blocked entry (and none allow-listed) removes
it, marking `data-bt-post="1"` when clear (cleared on `BLOCKLIST_UPDATED`
and an allow-list change, like `data-bt-checked`). `scrubPosts()` is in the
`sweep()` hot path (home-feed posts) and the `scheduleExtrasScrub()` /
`runExtras()` / `yt-navigate-finish` backups. A `/post/…` permalink is
covered by `findScopeForCurrentPage()` picking up the post's author for
`checkCurrentPageAndRedirect()` (FULL bounces; soft strips in place).

### Channel identity

A channel is keyed either by its canonical ID (`UCxxxxxxxxxxxxxxxxxxxxxx`,
stored without the `channel/` prefix) or by its `@handle`, whichever format
appears in the DOM link being inspected (`normalizeChannelKey()` in
`content/content.js`). The storage key stays whatever was captured, but the
scrub **does** cross-reference now: `applyBlocklist()` builds two side
indexes — `blockedByHandle` (from each entry's own `@handle` key *and* its
scraped `entry.handle`) and `blockedByUcid` (from each entry's own `UC…` key
*and* its scraped `entry.ucid`) — and every lookup goes through
`blockedEntryFor(key)`, which tries the raw key, then the lowercased handle,
then the UC id. This matters because **modern YouTube feed/search tiles link
the channel byline via `/@handle`, not `/channel/UC…`** — a `UC…`-keyed entry
(e.g. everything in an imported old BlockTube list) blocks *nothing* in feeds
until its `entry.handle` is populated by the "Fetch all sub counts" sweep
(`scrapeChannelInfo` grabs `subs` + `handle` + `name` + `ucid`). The
`updateInstantHideBlocklistCSS()` `:has()` rules likewise emit
`a[href="/@handle" i]` / `a[href="/channel/<ucid>"]` when known. So: **an
imported `UC…` list must be enriched to actually block** — the post-import prompt says so, and a
bulk enrich ends with `REBROADCAST_BLOCKLIST` so open tabs pick up the new
handles without a reload. Blocking directly from a channel's own page (via
the popup) still captures the format the page shows.

### Sync: two layers

1. **`chrome.storage.sync`** (the chunked engine in `background.js`) — free,
   automatic, but only bridges browsers signed into the *same vendor's*
   account sync (Chrome↔Chrome via Google, Firefox↔Firefox via a Firefox
   Account). It does **not** bridge Chrome desktop to Firefox for Android.
2. **GitHub Gist** (`background/gist-sync.js`, opt-in per device via a PAT in
   the options page) — closes the cross-vendor gap. Layered *on top of*
   layer 1, not replacing it: the chunked local store is still the fast
   read/write path and the offline cache, and `gist-sync.js` reconciles it
   with the gist on a timer. `localOnly` / the 480-item overflow still apply
   as a local concept; entries past the cap that also need to sync ride the
   gist but not `chrome.storage.sync`.

Merge is per-entry last-write-wins on `updated_at` with tombstones (see the
`background/gist-sync.js` bullet above). It is intentionally simple —
conflicts are rare for one person with a few devices, and the semantics are
forgiving (re-adding an already-blocked entry is a no-op; the worst case is
a resurrected unblock, which the tombstone TTL bounds). If a genuinely
robust multi-writer story is ever needed, that's a CRDT/vector-clock
rewrite, not a tweak here.

## Known fragility

- `manifest.json` declares the `contextMenus` permission, but nothing calls
  `chrome.contextMenus.*` anywhere — it's a leftover. Safe to drop, or to
  build the right-click "block this" entry it was presumably reserved for.
- `background/gist-sync.js` assumes the GitHub REST shape (`/gists`,
  `/gists/:id`, `history[0].version`) and that the account's BlockTube gist
  is on page 1 of `/gists?per_page=100` (true right after creation; a user
  with >100 gists *and* a pre-existing BlockTube gist could make a second
  one, leaving two to reconcile — rare, and the merge still converges). The
  push debounce is a plain `setTimeout` that a suspended worker can drop;
  the `bt-sync-poll` alarm (`SYNC.POLL_PERIOD_MIN`, min ~1 in production
  Chrome) is the backstop. Live-tested only against a 401 (bad token) — a
  real multi-device gist round-trip wasn't exercised in this environment (no
  test PAT should be improvised).
- YouTube is mid-migration to a new tile component family, `yt-lockup-view-model`,
  that shares nothing with the legacy `ytd-*`/`ytm-*` custom elements. Confirmed
  live: a channel's own "Playlists" tab and playlist results in search render
  *only* `yt-lockup-view-model` (zero `ytd-playlist-renderer`); a channel's
  "Videos" tab wraps one *inside* the legacy `ytd-rich-item-renderer` as a
  transitional shim. `LOCKUP_SELECTOR` + `queryTiles()` in `content/content.js`
  handle both cases (a bare lockup is treated as its own tile; a nested one is
  skipped since its wrapper already covers it) — if more surfaces migrate,
  or the legacy tags disappear entirely, that's the pairing to revisit. This
  was the real cause of "a blocked channel's playlist still shows up", not
  the hydration-timing theory that shipped first.
- `RENDERER_SELECTOR`, `SHELF_SELECTOR`, and `findScopeForCurrentPage()` in
  `content/content.js` hard-code YouTube's current custom-element tag names
  (`ytd-*` desktop, `ytm-*` mobile). YouTube changes these periodically —
  when tiles stop being removed or the popup stops finding a
  channel/video, these are the first places to check. The `ytm-*` mobile
  selectors are unverified against a live device (see README).
- `scrubGuide()` matches Home/Shorts by `href` (very stable — confirmed live
  that both mini-guide and drawer link to exactly `/` and `/shorts/`, note
  the trailing slash on the latter) but everything else — `HIDDEN_NAV_LABELS`
  ("Shorts" as a backup, "Explore", "More from YouTube", "Report history")
  and `scrubMasthead()`'s Create match — by visible label/aria-label text.
  These now come from `LABELS_BY_LANG` (6 languages + English fallback — see
  "UI-language labels"), so a listed non-English UI is handled, but an
  unlisted one still falls back to the English strings and misses. More
  resilient to a redesign than tag-name matching, but not immune.
  `navLabelOf()` reads the label off the entry's *direct child*
  `<a>` (`:scope > a`, deliberately not a deeper search) — confirmed live
  that a plain descendant search grabs an icon's empty wrapper `<span>`
  before it reaches the real label text, and an unscoped `querySelector("a")`
  on a *section* wrapper wrongly finds its first child entry's anchor
  instead of reporting "no anchor here". If a wanted entry (Subscriptions,
  Library, the account menu) starts vanishing, or Explore/Report
  history/masthead buttons stop being removed, those are the mechanisms to
  re-check first — this exact class of bug shipped once already.
  `scrubMasthead()`'s Create/Notifications removal could not be verified at
  all in this environment (both only render for a signed-in account, and no
  test account exists or should be improvised here). `GUIDE_FOOTER_SIGNAL_RE`
  (matching on footer link text like "How YouTube works") is the equivalent
  fragility point for the sidebar's small-print footer removal.
- Everything except `processRenderer()`'s blocklist removals runs on the
  ~400ms `scheduleExtrasScrub()` throttle (see Architecture) — a small
  latency window traded for page-load performance. Anything that must
  disappear *instantly* belongs in the per-mutation path, not here.
- `scrubEndScreen()` / its instant-hide CSS match player-chrome class names,
  not custom-element tags. YouTube **already redesigned this once** (Sept
  2026): the end-of-video suggestions moved from `.html5-endscreen` +
  `.ytp-videowall-still` to **`.ytp-fullscreen-grid`** full of
  **`.ytp-modern-videowall-still`** tiles (the "ytp-delhi" player revision) —
  the old names stopped matching and recommendations came back. Both name
  sets are handled now: the CSS `display:none`s the `.ytp-fullscreen-grid`
  container (player-owned, so CSS only — not `.remove()`d), the JS pass
  deletes the content pieces (legacy overlay, `.ytp-ce-element` cards,
  `.ytp-cards-teaser`, the individual `<a>` still-tiles). If suggested
  videos survive the end screen again, the videowall/grid class names are
  the thing to re-probe with a near-end seek. (Verified live to the extent
  headless allows — a seeked-near-end video never truly fires `ended`, so
  the elements and the toggle gating are confirmed, not a frame-by-frame
  appear-then-vanish.)
- `scrubShorts()`'s catch-all pass relies on finding an `a[href*="/shorts/"]`
  somewhere inside a tile/shelf when the tag name alone doesn't identify it
  as Shorts (e.g. `ytd-rich-shelf-renderer`, reused for many non-Shorts
  shelf types too). If a Shorts tile ever renders with no such link visible
  in its markup, it can survive — live-DOM confirmation would tighten this.
