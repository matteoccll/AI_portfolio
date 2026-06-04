# BiasMiti Extension

A Manifest V3 Chrome/Edge extension that mitigates **base-rate neglect** and **anchoring bias** at the moment of reading — entirely client-side, with no LLM and no network requests at runtime. Reference datasets and pattern libraries are static JSON baked into the bundle; nothing is fetched and nothing phones home.

## Demo

All four mechanisms firing on **live Wikipedia articles**, in a real browser. ▶ **[Watch the full walkthrough (MP4)](media/biasmiti-showcase.mp4)** — or see each mechanism below.

<!-- For an inline-playing video on GitHub: open a new issue (or the repo's release notes) in the
     browser, drag media/biasmiti-showcase.mp4 into the text box, and GitHub returns a
     https://github.com/<user>/<repo>/assets/... URL that renders as a player. Paste that URL here
     in place of the link above. Relative paths render as a download link, not an inline player. -->

**1 · Frequency Reframe** — a screening claim recast as a natural-frequency tree with the corrected posterior, on the *Breast cancer screening* article (Gigerenzer & Hoffrage, 1995):

![Frequency Reframe — Bayesian frequency tree on Wikipedia](media/screenshots/1-frequency-reframe.png)

**2 · Counter-anchor** — hovering a salient figure surfaces a reference value (with sparkline and source) before the article's number can anchor you, on *United States*:

![Counter-anchor — reference tooltip with sparkline on Wikipedia](media/screenshots/2-counter-anchor.png)

**3 · Where's the Denominator?** — a count stated without its reference population is flagged inline, on *Western African Ebola epidemic*:

![Where's the Denominator — missing-denominator flag on Wikipedia](media/screenshots/3-wheres-the-denominator.png)

**4 · Suppress & Estimate** — article numbers are masked (▒▒▒) so you commit your own estimate before seeing the original, on *Economy of the United States*:

![Suppress & Estimate — masked economic figures on Wikipedia](media/screenshots/4-suppress-and-estimate.png)

> The walkthrough was produced by `tests/e2e/record-showcase-os.mjs`, which drives the built extension across live pages in a real Chromium window and screen-records it with ffmpeg.

## Architecture

Four independent features, each targeting a specific cognitive bias with a mechanism grounded in the academic literature. A single content script (`src/content/boot.ts`) reads user settings and registers each enabled feature; every feature exposes a `register(ctx) → teardown` contract, so features can be toggled on and off live (via the popup) without a page reload and without leaking side effects.

Boot order is fixed and load-bearing. Counter-anchor registers first and wraps matched numbers in marker spans; the shared DOM walker excludes any text already inside one of those spans, so Where's the Denominator? (which registers next) never double-wraps them. Frequency Reframe follows as a selection/context-menu listener only — it mutates no DOM on load — and Suppress & Estimate registers last and is opt-in per page. The walker itself uses a `TreeWalker` over text nodes with an exclusion list (scripts, inputs, code blocks, contenteditable, already-injected nodes) and resolves overlapping matches deterministically, so re-runs are idempotent.

Injected UI is rendered with Preact into a **Shadow DOM** root, isolating it from host-page styles. The Suppress & Estimate calibration journal persists to **IndexedDB**, owned by the extension's service worker (not the page origin, so host pages can't read it); it stores **hostname only — never full URLs** (a deliberate privacy constraint) and is capped at **500 entries** with oldest-first eviction. One notable revision: `web_accessible_resources` was removed and the calibration dashboard wired up as an explicit Vite entry instead, to shrink the extension's fingerprinting surface.

## Features

- **Frequency Reframe** — translates percentage statistics into natural frequencies at read time; when a selection contains prevalence + sensitivity + specificity, it renders a full Bayesian frequency tree (Gigerenzer & Hoffrage, 1995).
- **Where's the Denominator?** — flags bare counts and relative-risk claims that omit the reference population, prompting the reader to supply the base rate before trusting the figure.
- **Counter-anchor** — on hover, injects a counter-anchor drawn from baked-in reference tables (population, GDP, deaths by cause, CPI, etc.) before the original figure can anchor the reader (Mussweiler, Strack & Pfeiffer, 2000).
- **Suppress & Estimate** — masks article numbers, elicits the reader's own independent estimate, then reveals the original and logs the delta, tracking calibration across sessions in IndexedDB.

## Stack

- **TypeScript 5.x** — strict mode, including `noUncheckedIndexedAccess`; no unjustified `any`.
- **Vite 5** + **`@crxjs/vite-plugin`** — MV3 build pipeline (rewrites source `.ts` manifest paths at build time).
- **Preact (~10.25)** — JSX runtime for injected UI, chosen over React to stay within the bundle budget.
- **Shadow DOM** — style isolation for all injected UI.
- **`@mozilla/readability`** — article detection for the long-form Suppress & Estimate feature.
- **`idb`** — thin IndexedDB wrapper for the calibration journal.
- **Vitest** (jsdom) for unit tests, **Playwright** (headed, for MV3 service-worker support) for end-to-end tests.

Bundle: **≈80 KB JS / ≈27 KB gzipped** — well under the project's 500 KB budget.

## How to install (developer mode)

The built `dist/` folder is **committed**, so you can sideload without building:

1. Open `chrome://extensions` → enable **Developer mode**.
2. Click **Load unpacked** → select the `dist/` folder.

To build from source instead (requires [pnpm](https://pnpm.io)):

```sh
pnpm install
pnpm build        # produces dist/
```

This project uses pnpm (a `pnpm-lock.yaml` is committed; there is no npm lockfile). No environment variables are required.

## Tests

```sh
pnpm test         # vitest unit tests
pnpm e2e          # builds the e2e bundle, then runs Playwright against dist/
```

86 tests total, **all passing**: 60 Vitest unit tests (number parsing, Bayesian math, denominator pattern matching, context-window checks, DOM walking, article detection, the selection watcher) and 26 Playwright end-to-end tests exercising each feature against committed HTML fixtures. The suite does not depend on any live URL.

### The closed-vs-open shadow build switch (why `pnpm e2e` differs from `pnpm build`)

All injected UI lives in a **closed** Shadow DOM root, so a hostile host page cannot reach into it via `document.querySelector('#bias-ext-root').shadowRoot`. That same closed boundary, however, also blocks Playwright's automated driver from confirming that a popup rendered — to the test robot, a closed shadow is indistinguishable from a snooping page. As a result the e2e suite could never verify the in-shadow UI, and that gap masked a real regression (see below).

The fix is a single compile-time switch, `__BIAS_EXT_E2E__`, that already existed for other test bridges:

- **`pnpm build` (production — the only version that ships):** `__BIAS_EXT_E2E__` is `false`. The shadow root is `mode: 'closed'`; the e2e test bridges are dead-code-eliminated from the bundle. This is the version in the committed `dist/` and the only one you would ever install or publish.
- **`pnpm e2e` (test-only, never shipped):** builds with `--mode=e2e`, so `__BIAS_EXT_E2E__` is `true` and the shadow root is `mode: 'open'`, letting Playwright auto-pierce it and verify all 26 interactions. This bundle is a throwaway used by the test runner and is never committed or published.

There is exactly **one** product. The compiler — not a manual edit you have to remember to undo — guarantees that any production build is closed: `attachShadow({ mode: 'closed' })` is the literal that survives in the shipped bundle. **To verify both yourself:** run `pnpm e2e` (expect 26/26), then run `pnpm build` and grep the emitted `dist/assets/boot.ts-*.js` for `attachShadow` — it reads `mode:"closed"`.

> While first running the e2e suite against the in-shadow UI we surfaced a regression introduced by the security hardening: the leaky-event guard on the shadow host stopped `input`/key events in the **capture** phase, which fired before the event reached the extension's own input field and so prevented the Suppress & Estimate prompt from ever registering a typed estimate. It is now fixed (the guard stops events in the bubble phase, on the way *out* to the page, after the extension's own handler runs). See `SECURITY_AUDIT.md` and `DECISIONS.md`.

## Status

Working. Build passes with zero errors; the full unit **and e2e** suites pass (60 + 26, verified against the built `dist/`); the extension loads as an unpacked extension and runs in both Chrome and Edge. **Not yet published to the Chrome Web Store.**

Honest limitations (see `DECISIONS.md` for the full record): Counter-anchor's reference-table selection is keyword-based and misses non-US or uncommon topics, and Where's the Denominator?'s pattern set misses some forms (e.g. ratios and word-form numbers). These are documented tradeoffs, not unknown bugs.
