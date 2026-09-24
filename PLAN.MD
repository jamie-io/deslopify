# restoreyt — Completion & Futureproofing Plan

## 1. Purpose

Turn prototype into portfolio-grade Chrome/Firefox extension that restores original
YouTube titles, thumbnails, descriptions, chapters, and audio when YouTube serves
auto-translated versions. Product must work reliably today and degrade gracefully when
YouTube changes.

Non-goals: circumventing regional restrictions, downloading content, subtitle editing.

## 2. Locked decisions

| Decision | Choice | Reason |
|---|---|---|
| Project name | `restoreyt` (manifest, package, docs, GitHub) | Neutral, descriptive |
| Git history | 4–6 logical conventional commits | Portfolio reviewers read history |
| Data sources | InnerTube default; optional Data API v3 key | Zero-setup default + stable official fallback |
| Chapters / channel avatars | Build only if Phase 0 proves a real original-text source | No fake features, no false README claims |
| Language | TypeScript (strict) + esbuild | Typed API shapes, real portfolio signal |
| Background | Dual `background.scripts` + `background.service_worker` | Chrome ignores `scripts` (>=121), Firefox requires it |
| Thumbnails | DOM-level only; DNR removed | DNR cannot fall back when `maxresdefault` is missing |
| Cache | In-memory LRU inside MAIN world | Avoids `sessionStorage` quota sharing and SW timing coupling |
| AI wording | Product copy may mention AI; no AI-authorship traces anywhere | AI-slop removal is the core feature |
| Audio toggle label | "Disable AI Audio" | Matches product feature wording |

## 3. Defects confirmed in current code

Fix all before feature work. File references are pre-refactor.

| # | Defect | Location | Impact |
|---|---|---|---|
| 1 | Duplicate `processVideoPage`; later declaration wins | `src/content/audio.js:93-105` | Original-track selection never runs |
| 2 | Original-audio detection relies on `track.isTranslatable` | `src/content/audio.js:41` | Field absent from known track shape; feature likely no-op |
| 3 | In-flight request dedup keyed by URL only | `src/lib/api.js:127,156` | Concurrent fetches return wrong video's data |
| 4 | Failed requests cached as `null`, read as "cached" | `src/lib/api.js:140,149` | One transient error disables feature for tab session |
| 5 | `sessionStorage.setItem` unguarded; 5MB cap shared with YouTube | `src/lib/cache.js:92,10` | Quota error breaks restore flow |
| 6 | Same cache key written raw and normalized | `src/lib/api.js:218`, `src/content/channel.js:61` | Type-confused cache reads |
| 7 | DNR redirect to `maxresdefault` with no fallback | `rules.json:8`, `src/content/thumbnail.js:17` | Broken thumbnails for videos without maxres |
| 8 | `hq720(?:_webp)?\.jpg` pattern matches no real URL | `rules.json:12` | Rules largely inert |
| 9 | Inline whitelist script injection | `src/content/inject.js:34` | YouTube CSP blocks; whitelist always empty |
| 10 | `currentVideoId` set before restore succeeds | `src/content/description.js:75` | Lazy-loaded description never restored |
| 11 | No post-await videoId re-check before DOM write | `src/content/title.js:52-72` | SPA race writes wrong title |
| 12 | One `getVideoDetails` per thumbnail image, no concurrency cap | `src/content/thumbnail.js:52` | Request storm while scrolling feeds |
| 13 | Script load errors resolve silently; parallel lib injection | `src/content/inject.js:28,38` | Whole extension can fail silently; no load order |
| 14 | `all_frames: true` on all YouTube origins | `manifest.json:31` | Injects into every iframe |
| 15 | Whitelist stored in `storage.sync` | `inject.js:7-16`, `settings.js` | 8KB/item quota; silent write failure |
| 16 | Settings defaults duplicated across 4 files, already drifted | `settings.js:10-19`, `inject.js:7-16`, `popup.js:7-15`, `options.js:7-16` | Config drift |
| 17 | README claims chapters and channel header images; neither exists | `README.md:12,14,109,182,240` | False claims; credibility risk |
| 18 | E2E uses `chromium.launch()` + `newContext()` | `tests/e2e/helpers.js:8,16` | Extension never loads; tests target `chrome-extension://undefined` |
| 19 | E2E URL-parsing test asserts its own literal | `tests/e2e/extension.spec.js:99-112` | False confidence |
| 20 | Husky hook with no `prepare` script; runs `npm ci` per commit | `.husky/pre-commit` | Dead hook, bad practice if wired |
| 21 | `.opencode/` tracked (todo, research docs) | repo root | Tooling residue; remove from history |
| 22 | `ESLint` `no-undef: off`; unused vars warn-only | `eslint.config.mjs:4-5` | Lint gate passes trivially |
| 23 | `channel.js` caches normalized data under API's raw key | `src/content/channel.js:61-67` | See #6 |
| 24 | `options.js` whitelist list built with `innerHTML` | `src/options/options.js:35-38` | Injection pattern in extension page |

## 4. Futureproofing strategy

YouTube changes. Goal: detect breakage before users, contain fixes in one place, degrade
instead of breaking the page.

1. **Platform isolation layer** — `src/platform/` owns every YouTube-specific assumption:
   DOM selectors, InnerTube response shapes, `ytcfg` config discovery, client
   identification, thumbnail URL patterns. No YouTube knowledge anywhere else. A shape
   change is a one-file fix.
2. **Runtime config discovery** — read `INNERTUBE_API_KEY`, `INNERTUBE_CLIENT_VERSION`,
   `VISITOR_DATA`, client name from the page's own `ytcfg` at runtime. Never rely on a
   hardcoded client version (`api.js:35` is the most likely future break).
3. **Layered data sources** — per feature, in order: (a) embedded page bootstrap data,
   (b) runtime-configured InnerTube, (c) optional user-supplied Data API v3 key. Each
   layer independently disables on failure or shape mismatch.
4. **Runtime shape validation** — TypeScript types are compile-time only. Platform layer
   parses `unknown` responses, narrows with guards, returns typed or `null`. Contract
   tests pin recorded shapes so drift fails CI.
5. **Feature isolation and circuit breaker** — every feature wraps execution in error
   boundaries and counts consecutive failures. After a threshold, the feature
   auto-disables and the badge warns. The extension must never degrade YouTube.
6. **Contract fixtures** — sanitized, recorded InnerTube/Data API responses in
   `tests/fixtures/`. Extractor tests assert required fields exist. Roles kept separate:
   contracts catch our breakage; the canary catches YouTube's.
7. **Live canary** — scheduled CI job (weekly) loads the built extension against a fixed
   matrix of known translated videos and asserts restored end state. Uploads HAR and
   diagnostics JSON on failure. Non-blocking, so transient YouTube noise does not redden
   required checks.
8. **Diagnostics** — options-page panel: version, background type, loaded features,
   request/error counters per layer, last failure, self-test against fixtures, export
   diagnostics JSON for bug reports. Local only; no telemetry.
9. **Selector resilience** — each DOM query has ordered fallbacks; a selector miss is
   counted and surfaced in diagnostics, not silently ignored.
10. **Version discipline** — manifest and package versions checked in CI; release
    workflow tags and publishes the zip.

## 5. Phases

### Phase 0 — Evidence spike (gate; no product code)

Goal: replace assumptions with evidence. Blocks every later phase.

Tooling: `tools/spike/` Playwright script using a persistent Chromium context.

- Force YouTube UI language different from video language (German UI, English videos)
  via `PREF` cookie. Validate translations actually appear before trusting any result.
- Matrix: 5 English videos with auto-translated titles, 2 with translated thumbnails,
  2 with dubbed audio tracks, 2 with chapters, 2 large channels. One logged-out and one
  logged-in variant.
- Capture per page: HAR, `ytInitialPlayerResponse`, `ytcfg` keys, `next` response,
  `getAvailableAudioTracks()` output, chapter panel DOM, thumbnail request URLs,
  `document.cookie` SAPISID presence, console errors.
- Verify bootstrap mechanisms: `<script type="application/json">` data block readable
  under YouTube CSP; classic sequential script injection executes in MAIN world;
  module injection outcome recorded for the record.
- Record DNR/host-permission necessity: confirm DOM-only thumbnail restore works.

| Question | Pass criterion | If false |
|---|---|---|
| Embedded bootstrap data untranslated? | Rendered title differs from embedded title | Use InnerTube or Data API path |
| SAPISID readable from page? | Cookie visible to `document.cookie` | Remove auth code; correct README |
| Thumbnail translation signature? | Translated asset identifiable by URL or DOM diff | Detect via image element diff, not URL pattern |
| Chapter originals obtainable? | Untranslated chapter titles recoverable | Cut chapters feature |
| Channel branding translated? | Translated channel name/avatar/banner observed | Cut avatars; title-only if name differs |
| Audio original track detectable? | Reliable original-vs-dubbed signal in player state | Cut or reduce to passive indicator |
| MAIN-world injection works? | Feature bundle executes | Switch loader strategy |

Artifacts: `docs/spike-report.md` (feeds README "How it works"),
`tests/fixtures/*.json` with automated redaction (visitor data, tokens, account IDs,
cookies) plus a human review pass before commit.

Exit gate: matrix validation passed; every question answered with evidence; no open
"probably" items.

### Phase 1 — Foundation

- TypeScript strict; esbuild bundles: MAIN-world content, background, popup, options.
  Scripts: `build`, `watch`, `package` (zip), `verify` (lint + typecheck + test + build).
- `src/platform/`: selectors, InnerTube clients, `ytcfg` discovery, shape guards,
  thumbnail URL builders, chapter/audio/player accessors.
- `src/core/`: settings schema (single source of truth), in-memory LRU with TTL and
  short negative TTL, request queue with concurrency 4, abort on navigation epoch,
  8s request timeout, per-feature error counters.
- Background script (classic, no ESM): `runtime.onMessage`, `storage.onChanged`
  fan-out to content, badge state, diagnostics buffer, circuit-breaker persistence.
  Content scripts degrade gracefully if background is asleep.
- Settings handoff: CSP-safe JSON data block + `CustomEvent` handshake, no inline script.
- Injection: single classic script, sequential load, hard-fail reporting to badge and
  diagnostics on load error.
- Manifest: `include_globs` for watch/shorts/embed; `all_frames` only where embed
  requires; host permissions limited to YouTube origins; `declarativeNetRequest`
  removed; WAR lists the single bundle.
- Whitelist in `storage.local`; handles resolved once via InnerTube, cached; quota
  handling and user-visible error on write failure.
- Per-defect fixes: #1, #3, #4, #5, #6, #9, #10, #11, #12, #13, #14, #15, #16, #20, #22, #23, #24.

Exit gate: `npm run verify` clean; bundle loads unpacked; diagnostics reports every
feature active; no console errors on a local mock page.

### Phase 2 — Features on evidence

Titles, descriptions, thumbnails, audio, channel titles (each only if Phase 0 supports it).

- Titles: diff-before-write; restore only when different; batch by videoId.
- Thumbnails: DOM-first with `onerror` fallback chain (maxres to hqdefault to original
  element state). Existence probe deduped by videoId with negative TTL.
- Descriptions: retry on missing container; restore chapters and links safely.
- Audio: set original track when detection is proven; handle `audiotrackchange` without
  wrapper stacking; respect user track choice within session.
- Data API v3 layer: options "Advanced" section, key validated via `videos.list`,
  stored in sync, quota and 403 trigger automatic fallback plus a notice. Key-only; no
  OAuth.
- Performance budget: no scroll jank on a 200-item feed; restoration completes under 2s
  per page; requests per visible page bounded (target under 20 uncached).

Exit gate: quantitative acceptance per feature (>=95% visible titles restored on matrix;
zero DOM writes on untranslated pages; zero console errors; chapters byte-identical on
at least 4 of 5 fixtures if built).

### Phase 3 — Conditional features

Chapters, channel avatars/banners. Implement only with a passing Phase 0 criterion and
fixture tests. Otherwise remove from code, toggles, README, and store copy. No fallback
built on guessed timestamp parsing.

### Phase 4 — Tests and CI

- Unit: extractors, shape guards, settings schema, LRU TTL, queue, whitelist, chapter
  parser, Data API and InnerTube parsers.
- DOM: jsdom fixtures per feature (translated and original markup).
- Contract: recorded-shape assertions; drift fails CI.
- E2E hermetic: `tests/e2e/mock/` local page mimicking watch/search/shorts DOM with
  `ytcfg` and `page.route` stubs for `/youtubei/*`. Extension loaded via
  `launchPersistentContext` + `channel: 'chromium'` + `waitForEvent('serviceworker')`.
  Assert popup, options, toggles, restoration against stubs. No live YouTube in blocking CI.
- Canary: scheduled live job, non-blocking, HAR + diagnostics upload.
- CI: Node 20 and 22; `permissions: contents: read`; concurrency cancel; lint,
  typecheck, unit, e2e, `web-ext lint`, package job, coverage >=80% on core and platform.
- Release: tag-triggered zip plus GitHub release; manifest/package version sync check.
- Hooks: slim pre-commit (lint + typecheck + test) or none; CI remains the gate.
- Dependabot for dev dependencies and GitHub Actions.

Exit gate: full suite green on Chromium and Firefox; canary runs green once manually
triggered; packaged zip loads in a clean profile.

### Phase 5 — Firefox and store package

- `browser_specific_settings.gecko.id`, dual background keys, `minimum_chrome_version: 121`.
- Firefox e2e project; MAIN-world and storage parity verified.
- Store pack in `docs/store/`: listing copy, 1280x800 screenshots, 440x280 tile,
  permission justifications, reviewer repro steps (logged-out, matrix video IDs),
  data-usage answers, single-purpose statement.
- `PRIVACY_POLICY.md`: no third-party requests, no telemetry, optional API key handling,
  permission rationale.
- Keep existing ToS and InnerTube-fragility disclaimer in README.

Exit gate: clean-browser install verified for Chrome and Firefox; store pack complete.

### Phase 6 — Docs, hygiene, release

- README rewrite: shipped features only, architecture, permissions table, spike report,
  demo GIF, badges, install for Chrome and Firefox, honest limitations.
- CHANGELOG, CONTRIBUTING, CODE_OF_CONDUCT aligned; SECURITY.md added.
- `docs/adr/`: data layers, no-DNR, TypeScript plus esbuild, cache location,
  background duality.
- Remove `.opencode/` from tracking, add to `.gitignore` along with `.graphene/`.
- Verify no AI-authorship traces in tree, history, or GitHub UI: no assistant trailers,
  no "generated with" markers, no tool badges, no tool topics.
- Name availability check (npm, Chrome Web Store, GitHub) before rename commits.
- Confirm `git remote -v` is empty before any history work.
- Commit series: `chore: scaffold`, `feat: platform layer`, `feat: restore original
  content`, `test: contract and e2e coverage`, `feat: firefox and packaging`,
  `docs: portfolio and store assets`.
- First push to new `restoreyt` repository.

## 6. Definition of done

- Every Phase 0 question answered with evidence in `docs/spike-report.md`
- All 24 listed defects fixed or removed
- Titles, descriptions, thumbnails working on watch, search, recommendations, Shorts,
  embeds; zero writes on untranslated pages
- Chapters and avatars shipped only with proof, or documented as cut
- Unit, DOM, contract, and hermetic e2e green on Chromium and Firefox
- Lint and typecheck clean; coverage >=80% on core and platform
- Zero console errors across a 10-video manual matrix
- Live canary scheduled and passing
- Chrome and Firefox clean-profile installs verified
- Store pack, privacy policy, and reviewer notes complete
- No AI-authorship traces in repository, history, or GitHub surface
- README claims traceable to spike evidence or a passing test

## 7. Risks and mitigations

| Risk | Mitigation |
|---|---|
| InnerTube shape drift | Platform layer, runtime guards, contract fixtures, canary |
| Hardcoded client version rejected | Runtime `ytcfg` discovery with fallback chain |
| YouTube CSP blocks injection | JSON data block bootstrap; verified in Phase 0 |
| Live e2e flakiness | Hermetic mock harness; canary non-blocking |
| Firefox background/event-page divergence | Dual keys; Firefox e2e project |
| DNR removed but thumbnails need network-level fix | Phase 0 verifies DOM-only sufficiency; canary watches |
| Cache quota collisions | In-memory LRU only |
| API key abuse concerns | Key-only validation, sync storage, documented in privacy policy |
| Scope creep from conditional features | Phase 3 gate; documented cut instead of workaround |
| Recorded fixtures leak session data | Automated redaction plus human review before commit |

## 8. Open items

1. **Dark mode / i18n** — README roadmap lists dark mode. In scope, or cut as stretch?
2. **Scope guard** — implementation proceeds one phase at a time with a review
   checkpoint after each phase.
```