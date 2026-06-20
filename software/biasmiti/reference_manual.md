# BiasMiti — Reference Manual
**Version:** 1.0 (living document)
**Last updated:** 2026-05-30
**Scope:** Bias 2 (Base-rate Neglect) + Bias 3 (Anchoring) — four shipped features, fully deterministic, zero LLM, zero network at runtime.
**Stack:** Manifest V3 (Chromium) · TypeScript 5.x strict · Vite 5 · CRXJS v2.4.0 · Preact · Shadow DOM · IndexedDB

> **What this document is.** A single-source reference manual for the extension as it actually exists. It is not the build spec (`building_spec_bias2_bias3.md`), not the operating contract (`CLAUDE.md`), and not the user-facing README. Read this to understand the *whole* system — what each feature does, why it works, how the pieces fit, where the data lives, what the security model guarantees — without reading a line of code. Where the implementation diverged from the spec, this manual documents the implementation, and flags the divergence.

---

## 0. Orientation

### 0.1 One-paragraph summary

This is a Chrome/Edge browser extension that intervenes against two cognitive biases **at the moment of reading**. It runs entirely on-device: no network calls, no API keys, no telemetry, no accounts. Four features (two per bias) annotate, reframe, or mask numbers on the page using deterministic rules, hand-rolled SVG, and a small set of bundled reference datasets. Each intervention is grounded in a specific, replicated finding from the cognitive-psychology literature — natural-frequency reframing (Gigerenzer & Hoffrage 1995), denominator-neglect auditing (Reyna & Brainerd), counter-anchoring (Mussweiler, Strack & Pfeiffer 2000), and anchor suppression with calibration tracking.

### 0.2 The problem each feature attacks

| # | Feature | Bias targeted | The reading-moment failure it interrupts |
|---|---------|---------------|------------------------------------------|
| 1 | **Frequency Reframe** | Base-rate neglect / Bayesian misjudgment | Reader sees "95% accurate test, positive result" and concludes "95% chance I'm sick", ignoring prevalence. |
| 2 | **Where's the Denominator?** | Denominator neglect | Reader accepts "1,200 people died" or "rose by 15%" as meaningful without the population base or prior value. |
| 3 | **Counter-anchor** | Anchoring | Every quoted number silently anchors the reader's later estimates of related quantities. |
| 4 | **Suppress & Estimate** | Anchoring (prevention, not mitigation) | The only way to *prevent* anchoring is to not see the anchor until you've committed your own estimate. |

### 0.3 Design commitments (inviolable)

These are constraints, not preferences. They define what the extension is.

- **Deterministic only.** No LLM, no inference, no model weights. Every output is a pure function of page text + bundled data.
- **Zero network at runtime.** `host_permissions` is empty. Reference tables are baked into the bundle as static JSON imports. Nothing phones home.
- **Privacy by construction.** The calibration journal stores `urlHost` only (never full URLs, query params, or page titles), lives in the extension's own storage partition (not the page's), and is capped at 500 entries.
- **No page-readable secrets.** "Hidden" numbers and user keystrokes are held in the content-script isolated world (a `WeakMap` + a closed shadow root), never in page-readable DOM.
- **Minimal footprint.** ~81 KB JS (≈28 KB gzipped); ~116 KB total unpacked. Budget is 500 KB.
- **Non-intrusive by default.** Three features default on and annotate unobtrusively; the one intrusive feature (Suppress & Estimate) defaults off and is opt-in per page.

---

## 1. Literature foundation

Each feature implements a specific, peer-reviewed intervention. This section is the evidentiary backbone — the "why this works" that the UI quietly rests on. Sources are drawn from the project's literature documentation (`docs/Biases_Descriptions_Mitigation.docx`, `docs/biases_examples.docx`).

### 1.1 Bias 2 — Base-rate neglect (with denominator neglect)

**The bias.** The tendency to underweight prior probabilities (base rates) when individuating evidence is available (Kahneman & Tversky 1973; Bar-Hillel 1980). In reading, it manifests as accepting "X% test positive" without integrating disease prevalence, or "N people died from Z" without the denominator. **Denominator neglect** (Reyna & Brainerd) is the specific sub-failure: numerators are processed, but the population base is not.

**The intervention with the largest, most replicated effect: natural frequencies** (Gigerenzer & Hoffrage 1995, 1999; Cosmides & Tooby 1996). Re-expressing "10% of women have X, sensitivity 80%, specificity 90.4%" as a concrete population —

> "Of 1,000 women, 100 have X; 80 of those test positive; of the 900 healthy, 86 also test positive — so 80 in 166 positives actually have X."

— moves physician accuracy on Bayesian inference problems from **~10% to ~46%** with no other training. **Frequency trees** and **icon arrays** (10×10 grids) extend the gain by making the nested-set structure visually explicit (Sloman et al. 2003 — the *nested-sets hypothesis* explaining *why* the reframe works). A 2026 Frontiers replication confirmed the effect holds for complex multi-stage inferences.

- → **Feature 1 (Frequency Reframe)** implements frequency reformatting + base-rate injection + the nested-sets visualization (icon array + Gigerenzer frequency tree with visible posterior).
- → **Feature 2 (Where's the Denominator?)** implements comparison-set forcing + externalized memory, targeting denominator neglect directly (Reyna & Brainerd; replicated through 2024).

### 1.2 Bias 3 — Anchoring

**The bias.** Numerical estimates are pulled toward any salient prior number — even one known to be arbitrary or random (Tversky & Kahneman 1974). The effect is automatic, robust to warning, and operates via **selective accessibility** (Mussweiler & Strack 1999): the anchor activates anchor-consistent information in semantic memory. In reading, every quoted number ("experts estimate", "the average is", "up to 90%") anchors the reader's subsequent estimates of related quantities. **Pure awareness does not work.**

**The most reliable mitigation: consider-the-opposite / counter-anchoring** (Mussweiler, Strack & Pfeiffer 2000). The mechanism is forced retrieval of anchor-*inconsistent* information, neutralizing selective accessibility. Multiple / counter-anchor presentation is comparably effective (Lahtinen et al. 2020 "rotating the reference point"; Bixter & Luhmann; Adame 2016). 2024 replications (Information Sciences AI-recommendation study; supplier-evaluation work) confirm the effect, with an asymmetric caveat: CTO works well against *high* anchors, less reliably against *low* ones.

The **strongest** form of all is **suppression** — preventing the anchor from being encoded at all. This is the only intervention that *prevents* anchoring rather than mitigating it post-hoc.

- → **Feature 3 (Counter-anchor)** implements counter-anchor injection + distributional display at the exact moment of anchor exposure.
- → **Feature 4 (Suppress & Estimate)** implements the strongest version — anchor suppression — plus calibration tracking as a measurable side-effect.

### 1.3 Why "at the moment of reading"

One-shot debiasing interventions produce persistent effects 2+ months later (Morewedge et al. 2015, *Policy Insights*) — the intervention does not require repeated training. But the *timing* matters: a counter-anchor shown after the anchor is encoded is weaker than one shown at exposure, and suppression must happen *before* the number is seen. The extension's entire value proposition is that it fires at the read, not in a separate reflection step.

### 1.4 Scope boundary

The broader bias taxonomy (`docs/Biases_Descriptions_Mitigation.docx`) also describes **Bias 1 (Confirmation bias)** and **Bias 4 (Availability heuristic)**, each with their own solutions (disconfirm-selection, reverse-search interception; availability requires LLM/network features). **These are explicitly out of scope for this build.** The shared utilities (§3) were built to be reused by those later slices, but no Bias 1 or Bias 4 code ships here.

---

## 2. System architecture

### 2.1 The big picture

```
┌─────────────────────────────────────────────────────────────────────┐
│  PAGE (any https URL, top frame only)                               │
│                                                                     │
│   content script: src/content/boot.ts  (isolated world)            │
│     │                                                               │
│     ├─ getSettings()         ← chrome.storage.local                │
│     ├─ detectArticle()       ← Readability + heuristics            │
│     └─ register() each enabled feature:                            │
│          counter-anchor → wheres-the-denominator →                 │
│          frequency-reframe → suppress-and-estimate                 │
│                                                                     │
│   ALL injected UI mounts into ONE closed shadow root               │
│     <div id="bias-ext-root">  (position:fixed; inset:0;            │
│                                pointer-events:none; z-index max)   │
└───────────────┬─────────────────────────────────────────────────────┘
                │ chrome.runtime messaging (sender-validated)
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SERVICE WORKER  src/background/service-worker.ts  (extension origin)│
│     ├─ context menu: "Reframe as natural frequency"                │
│     ├─ journal:add / journal:list / journal:clear                  │
│     │     → owns IndexedDB 'bias-ext' (extension-origin partition)  │
│     └─ retention cap: 500 most-recent estimates                    │
└─────────────────────────────────────────────────────────────────────┘
                ▲
                │ chrome.storage.local + chrome.runtime
┌───────────────┴────────────────┐   ┌────────────────────────────────┐
│  POPUP  src/popup/popup.tsx    │   │  OPTIONS  src/options/options  │
│  4 feature toggles +           │   │  thresholds, flag density,     │
│  per-page SE activation +      │   │  privacy (export/clear journal)│
│  "open dashboard"              │   │  reference-table list, About   │
└────────────────────────────────┘   └────────────────────────────────┘
                                       │
                                       ▼
                          DASHBOARD  src/popup/dashboard.html
                          calibration scatter + log-ratio stats
```

### 2.2 The four runtime surfaces

| Surface | Entry | Runs in | Job |
|---------|-------|---------|-----|
| **Content script** | `src/content/boot.ts` | Page (isolated world) | Detect article, register features, inject UI into shadow root |
| **Service worker** | `src/background/service-worker.ts` | Extension origin (event-driven) | Context menu, journal IndexedDB owner, message router |
| **Popup** | `src/popup/popup.{html,tsx}` | Extension origin | Feature toggles, per-page Suppress-&-Estimate activation, dashboard link |
| **Options** | `src/options/options.{html,tsx}` | Extension origin | Thresholds, flag density, privacy controls, reference-data list, citations |
| **Dashboard** | `src/popup/dashboard.html` + `calibration-dashboard.tsx` | Extension origin (new tab) | Read-only calibration visualization |

### 2.3 The feature contract

Every feature is a folder under `src/content/features/<name>/` exporting one function:

```ts
export function register(ctx: BootContext): () => void
```

`register(ctx)` activates the feature and returns a **teardown** function that unwraps spans, removes UI, and disconnects observers. `boot.ts` collects teardowns; when settings change (`chrome.storage.onChanged`), it tears everything down and re-registers the now-enabled set. Adding a future feature is one `import` + one `if (settings.features[x]) teardowns.push(register(ctx))` line.

`BootContext`:
```ts
type BootContext = {
  settings: Settings                      // resolved, defaults filled
  article: ArticleInfo | null             // detectArticle() result, computed once
}
```

### 2.4 Boot order (and why it matters)

Within one page load (`src/content/boot.ts`):

1. `getSettings()` — read `chrome.storage.local['bias-ext:settings']`, merge defaults.
2. `detectArticle()` — run **once**, cached in `ctx` for all features.
3. Register, in this fixed order:
   1. **counter-anchor** — wraps numbers in the article body first.
   2. **wheres-the-denominator** — runs after, so numbers already wrapped by counter-anchor (now inside a `bias-ext-injected` span) are skipped. Annotate-at-most-once: if both want the same text, counter-anchor wins.
   3. **frequency-reframe** — registers a context-menu message listener + selection watcher; no DOM mutation at boot.
   4. **suppress-and-estimate** — only if pre-activated and article is long-form.

The two DOM-mutating features (1 and 2) cooperate via the shared `bias-ext-injected` class: `dom-walker` skips any text node whose ancestor chain contains it, so neither feature re-annotates the other's spans, and re-runs are idempotent.

### 2.5 Build pipeline

- **Vite 5** + **`@crxjs/vite-plugin` v2.4.0** parse `manifest.json` as the entrypoint. Source manifest references `.ts`/`.tsx` paths; CRXJS rewrites them to bundled JS in `dist/manifest.json`. **Never hand-edit `dist/manifest.json`.**
- **Preact** is the JSX runtime (`jsxImportSource: 'preact'`, ~4 KB) — chosen over React (~45 KB) and a custom factory for the right balance of bundle weight and proper component lifecycle (see `DECISIONS.md` D-002).
- **Static JSON imports** load reference tables and pattern libraries at build time (`import patterns from '../data/...'`) — no runtime `fetch`, no async orchestration.
- A compile-time constant **`__BIAS_EXT_E2E__`** (Vite `define`) gates all test-only bridges. Production builds (`vite build`) dead-code-eliminate them; e2e builds (`vite build --mode=e2e`) keep them.

---

## 3. Shared utilities (`src/content/shared/`)

All four features depend on these. They were built and unit-tested in isolation before any feature code.

### 3.1 `dom-walker.ts` — safe text-node walking + span wrapping

The single most load-bearing primitive: it finds regex matches in page text and wraps them in spans **without breaking the page**.

```ts
type TextMatch = { node: Text; index: number; length: number; text: string }
function findMatches(root: Node, pattern: RegExp): TextMatch[]
function wrapMatches(matches: TextMatch[], spanFactory: (m: TextMatch) => HTMLSpanElement): void
```

**`findMatches`** uses a `TreeWalker(SHOW_TEXT)` whose filter **rejects** text nodes under:
- `SCRIPT, STYLE, NOSCRIPT, IFRAME, OBJECT, EMBED`
- `INPUT, TEXTAREA, SELECT, OPTION, BUTTON`
- `CODE, PRE, KBD, SAMP, VAR`
- any `contenteditable="true"` or `aria-hidden="true"`
- our own UI: `id="bias-ext-root"` or class `bias-ext-injected`

It **never mutates during the walk** (the walker destabilizes if the tree changes) — matches are collected first, mutation happens in a second pass.

**`wrapMatches`** algorithm (must be exact):
1. Group matches by Text node.
2. Sort ascending by `index` within each group.
3. Resolve overlaps left-to-right: discard any match starting before the previous accepted match's end (earlier-start wins).
4. Process each group **in reverse** (largest index first): `splitText(index+length)`, then `splitText(index)`, replace the middle node with a `<span class="bias-ext-injected …">`. Reverse order keeps earlier offsets valid as later splits happen.

Each wrapped span carries `bias-ext-injected` so subsequent walks skip it → idempotent re-runs.

**Edge cases (tested):** matches spanning two text nodes are skipped (v1 does not wrap across nodes); empty (`length === 0`) matches rejected; whitespace-only nodes skipped; match at offset 0 and match consuming the whole node both handled.

### 3.2 `number-detector.ts` — numeric detection + normalization

```ts
type NumberUnit = 'percent' | 'currency' | 'plain'
type ParsedNumber = {
  raw: string; value: number; unit: NumberUnit
  currency?: 'USD'|'EUR'|'GBP'|'JPY'|null
  hasDecimal: boolean; digitCount: number; isLikelyYear: boolean
}
export const NUMBER_PATTERN: RegExp        // compiled once, case-insensitive, no `g`
function parseAll(text: string): ParsedNumber[]
function parseMatch(raw: string): ParsedNumber | null
```

The master pattern is an alternation, **most-specific first** so longer matches win:

1. Currency + suffix multiplier — `$1.5B`, `€3 million`
2. Currency plain — `$1,234.56`
3. Percentage with `%` — `5%`, `12.5 %`
4. Percentage with word — `5 percent`, `5 per cent`
5. Plain number + multiplier word — `1.5 million`
6. Comma-grouped plain — `1,234`
7. 4+ digit plain — `2024`, `334900000`

**Parsing rules:** `1,234.56 → 1234.56`; `1.5 million / $1.5M → 1_500_000`; `5% → value 5` (human-facing, not 0.05; downstream divides by 100); currency magnitude in `value`, symbol in `currency`, no FX conversion.

**`isLikelyYear`** is `true` only for a 4-digit plain integer in **[1900, 2199]** with no commas/prefix/multiplier/`%`. (The spec says 1900–2099; the range was extended to 2199 in security Finding 13 so it doesn't expire.) Suppress-&-Estimate uses this to avoid masking "2024" in a 2024-dated article. Other features ignore it.

**Deliberately NOT detected:** standalone 2–3 digit numbers (too noisy — `DECISIONS.md` D-005), ratios ("1 in 5"), scientific notation, word-form numbers ("five hundred"), Roman numerals. Years that look like plain 4-digit numbers *do* match the regex but harmlessly fail every reference-table lookup.

### 3.3 `article-detector.ts` — find the main article body

```ts
type ArticleInfo = { root: HTMLElement; textContent: string; wordCount: number;
                     isLongForm: boolean; title: string|null; byline: string|null }
function detectArticle(): ArticleInfo | null
```

Three strategies, first success wins:
1. **Semantic selectors** — `article[itemtype*="Article"]`, `article`, `main article`, `[role="main"] article`, `main`, `[role="main"]`.
2. **Content-host class names** — `.post-content`, `.entry-content`, `.article-body`, `.story-body`, `#article`, `#main-content`; pick the largest by text length.
3. **Readability fallback** — clone the document (Readability mutates its input, so always clone), gate with `isProbablyReaderable`, run `new Readability(clone).parse()`, then locate the corresponding live-DOM element by normalized-text prefix match and climb to the nearest block ancestor.

`isLongForm = wordCount >= settings.thresholds.suppressEstimate.minArticleWords` (default 500). Returns `null` if all three fail → **Suppress-&-Estimate becomes a no-op, and (post security Finding 9) Counter-anchor and Where's-the-Denominator also early-return rather than scanning `document.body`.**

### 3.4 `selection-watcher.ts` — track user text selection

```ts
type SelectionEvent = { text: string; range: Range; rect: DOMRect }
function onSelection(handler: (e: SelectionEvent) => void): () => void
```

Listens to `mouseup` + `keyup`, debounced 150 ms. Skips empty selections and selections inside `bias-ext-injected`. Returns an unsubscribe.

### 3.5 `shadow-host.ts` — the single isolated UI root

All injected UI mounts into **one closed shadow root** on a `<div id="bias-ext-root">` appended to `document.body`:

```
all:initial; position:fixed; inset:0; z-index:2147483647; pointer-events:none
```

`pointer-events:none` on the host means it never blocks the page; individual components opt back in with `pointer-events:auto`. The root is **`mode: 'closed'`** so page scripts cannot reach it via `getElementById('bias-ext-root').shadowRoot`.

```ts
function getShadowRoot(): ShadowRoot
function mountInShadow(node: HTMLElement): void
function isInShadowUI(target: EventTarget | null): boolean   // for click-outside
function eventInShadowUI(e: Event): boolean                  // composed-path aware
```

**Keystroke-leak defense:** composed events (`input, change, keydown, keyup, keypress, beforeinput`) cross the shadow boundary retargeted to the host. A `{ capture: true }` listener on the host calls `stopImmediatePropagation()` so they never reach page scripts (security Finding 2). The host itself never exposes the host element — `getShadowHost()` was removed in favor of the predicate helpers.

### 3.6 `feature-toggles.ts` — settings (single source of truth)

```ts
export const STORAGE_KEY = 'bias-ext:settings'
function getSettings(): Promise<Settings>          // merges DEFAULTS
function saveSettings(s: Settings): Promise<void>  // popup + options MUST use this
function watchSettings(cb: (s: Settings) => void): () => void
```

`STORAGE_KEY` is exported so popup, options, and content scripts all read/write the same key (security Finding 4 fixed a bug where the popup wrote `settings` while content scripts read `bias-ext:settings`, making every toggle a silent no-op). See §6 for the `Settings` shape.

---

## 4. The four features

### 4.1 Feature 1 — Frequency Reframe (Bias 2, Solution 1)

**Folder:** `src/content/features/frequency-reframe/` · **Default:** ON

**Trigger.** Right-click a text selection → context-menu item **"Reframe as natural frequency"** (registered in the service worker, `contexts: ['selection']`). The click messages the active tab; `index.ts` reads the *live* selection (for the `Range`, which the flattened context-menu string lacks) and runs the reframe. (An auto-suggest floating button on selection is specced for v1.1 but not shipped.)

**Logic** (`index.ts` → `health-test-detector.ts` → `bayes.ts`):
1. `parseAll(selectionText)` → percentages. Zero percentages → "no probabilities detected" toast, stop.
2. **Health-test detection.** Gate: selection must contain **≥ 2 percentages AND ≥ 2 distinct test-keywords** (keywords loaded from `data/patterns/health-test-keywords.json`: *test, sensitivity, specificity, prevalence, incidence, screening, diagnostic, positive, negative, disease, infection, condition, disorder, risk of*). Each percentage is labelled by its nearest keyword within ±8 tokens; tie-break priority `sensitivity > specificity > prevalence > false-positive > incidence > false-negative`.
3. **All three of {prevalence, sensitivity, specificity} must be present** to build the Bayes tree. Two-of-three is not enough → fall back to plain icon arrays with a note naming the missing parameter.
4. Render a floating panel anchored to the selection.

**Math** (`bayes.ts`):
```ts
function singleToFrequency(pct, base=1000): { numerator; denominator }
function bayesTree({ prevalence, sensitivity, specificity, cohort=1000 }): BayesTree
```
`BayesTree` splits a cohort into withDisease/withoutDisease → true/false positives/negatives, and computes `posteriorPositive = TP / (TP + FP)` on **unrounded** reals. Display leaves are rounded to integers, then the largest leaf is adjusted ±1 so they sum to the cohort exactly. Edge cases: prevalence 0 → posterior undefined (skip, explain); `cohort·prevalence/100 < 1` → auto-scale cohort up to 10⁴/10⁵/10⁶ so the disease-positive leaf is ≥ 1.

Canonical check (unit test): mammography **prev 1%, sens 80%, spec 90.4%, cohort 1000 → posterior ≈ 7.8%**.

**UI:**
- `icon-array.tsx` — SVG grid of `denominator` circles, first `numerator` filled. Three size classes: dots (small), grouped-by-100 (≥1000), single proportional bar + inset 10×10 (≥10000). `aria-label` describes the array.
- `frequency-tree.tsx` — horizontal SVG tree (cohort → sick/well → test ±), with the posterior as a prominent callout: *"Given a positive test: 8 / (8 + 95) = 7.8% chance of actually being sick."* The posterior is the punch line — it is visually emphasized.
- `panel.tsx` — floating, viewport-clamped, mounted in the shadow root. Closes on ESC / click-outside / × button.

**Acceptance (all green):** Bayes selection → tree + posterior; "5%" alone → 5-of-100 array; no percentages → toast; one keyword only → plain percentage (not Bayes); ESC closes; context-menu item present (manually verified — OS menus aren't WebDriver-inspectable, `DECISIONS.md` D-006).

> **Posterior note (D-007):** the spec quotes "≈ 7.8%" while also saying "90% specificity"; 90% spec actually yields ≈ 7.5%. The 7.8% figure is the canonical 90.4%-specificity mammography case. The unit test asserts 90.4% → 7.8% exactly; the e2e test asserts structural presence, not the exact value.

### 4.2 Feature 2 — Where's the Denominator? (Bias 2, Solution 2)

**Folder:** `src/content/features/wheres-the-denominator/` · **Default:** ON

**Trigger.** Automatic on `document_idle`, scanning the **article body only** (early-returns if no article detected — security Finding 9). Re-scans on a 500 ms-debounced `MutationObserver` for SPA/infinite-scroll content; capped at 1 scan-pass per 2 s.

**Pattern library** (`data/patterns/denominator-missing.json` — **8 patterns**, conservative by design to keep false positives low):

| id | Catches | Missing |
|----|---------|---------|
| `raw-count-people-verb` | "12,000 people died/were/reported…" | population_base |
| `rose-by-percent` | "rose/fell by 15%" | base_value |
| `highest-in-N-years` | "highest level in 20 years" | prior_series |
| `N-people-died-of` | "1,200 people died of X" | population_at_risk |
| `N-bare-demographic` | bare count + demographic noun | population_base |
| `doubled-tripled` | "doubled / tripled / quadrupled" | base_value |
| `studies-show-percent` | "studies show 40%…" | sample_size |
| `N-times-more-likely` | "3× more likely" | base_rate |

Each pattern is a capture-grouped regex + structured metadata (`missing`, `explanation`, optional `lookup_hint`). Regexes use the **`i` flag, not `gi`** — the `g` flag caused `lastIndex` drift between `.test()` calls (a bug caught and fixed; see `PROGRESS.md`).

**Context-window suppression** (`context-check.ts`): for each match, extract ±`denominatorContextWindow` words (default 20) and test against the union of denominator tokens from `data/patterns/denominator-context-tokens.json` (**24 tokens**: `out of \d`, `per capita`, `per 100,000`, `per N people`, `compared to/with \d`, `up/down from \d`, `population/sample/cohort/study of \d`, `n = \d`, `previous year/month/quarter`, `year earlier/ago/prior`, …). If a denominator is present, **suppress the flag**.

> **Deliberate exclusion:** pure time-rate tokens ("per year", "per month") are **not** in the suppression list — a rate over time without a population base is exactly the kind of claim that *should* still flag. Only true population denominators suppress.

**Reference enrichment.** If a bundled reference table matches the topic, the flag tooltip surfaces the actual denominator ("US population was 334.9M in 2024 — Census"). Otherwise it just names what's missing.

**Flag UI** (`flag.tsx`): a small inline amber pill (`?`) with a thin underline after the matched span (class `bias-ext-injected`). Hover/click → tooltip with the explanation, the `Missing:` field, the optional reference context, and a × to suppress this flag for the page. **Density cap:** ≤ 1 flag per 200 words (`normal`) or per 500 (`minimal`); excess matches silently skipped.

**Acceptance (green):** "1,200 people died from X last year" → flag; "1,200 of the 50,000 people in the study died" → suppressed; "GDP rose by 5%" → flag; "GDP rose by 5% from $20T to $21T" → suppressed; toggle-off clears flags within ~1 s; re-scan annotates new content without re-annotating old.

### 4.3 Feature 3 — Counter-anchor (Bias 3, Solution 1)

**Folder:** `src/content/features/counter-anchor/` · **Default:** ON

**Trigger.** Automatic on `document_idle` (early-returns if no article, or if a visible `<input type="password">` is present — security Finding 9). Pipeline: `detectArticle()` → `findMatches(root, NUMBER_PATTERN)` → `parseMatch` each → **`lookupReference(parsed, ±50 words)`**. Only numbers that *match a reference table* get wrapped + a hover handler. **Lookup before wrap** — this is the salience filter; most numbers don't match, which is correct.

**Lookup** (`lookup.ts`):
- **11 bundled tables** registered statically (see §5.1). Each has a keyword list (`TABLE_KEYWORDS`); the surrounding ±50 words are lowercased and scored by keyword overlap. Highest-scoring tables are tried first.
- `bestSeriesEntry` picks the series row: for time-series tables, the entry matching the article's context year (else newest); for non-time tables (e.g. by-state), the row whose `period` (state name) appears in context.
- **Plausibility gate (critical for false-positive control):** reject if `parsed.value <= 0` or `refValue <= 0` *before* `log10` (a `NaN > 1` comparison is `false` and would otherwise leak bad matches). Then reject if `|log10(value / refValue)| > 1` — more than one order of magnitude apart means probably not the same quantity.
- Comparison: the next-older series entry → `{ label: "vs. <period>", value, delta }`.

**UI:**
- `sparkline.tsx` — 200×40 SVG: time on X, value on Y (log-scaled if range > 2 orders), thin polyline, a distinct vertical marker at the article's claimed value; optional p10–p90 shaded band.
- `tooltip.tsx` — on `mouseenter` (200 ms debounce in, 100 ms out so the cursor can travel into it): reference label, sparkline, up to two comparisons (year-over-year, 10-year median), source + `fetched_at` line, × close. Mounted in the shadow root.
- **Visual indicator:** a thin **1px dotted underline** on the annotated number; solid on hover. No bold, no background — must not compete with the page.

**Acceptance (green):** "US population is 334 million" → underline + Census sparkline; "the company has 334 employees" → **not** annotated (population table fails the plausibility/keyword gate, no employee table exists); "GDP grew to $27 trillion" with a year → annotated GDP sparkline; toggle-off clears underlines within ~1 s.

### 4.4 Feature 4 — Suppress & Estimate (Bias 3, Solution 2)

**Folder:** `src/content/features/suppress-and-estimate/` · **Default:** OFF (opt-in; intrusive)

**Activation.** Via popup ("Activate on this page", per-tab) or a global long-form setting in options. Requires `detectArticle()` → `isLongForm: true` (≥ 500 words); otherwise a "too short" toast and no-op.

**Masking** (`masking.ts`):
1. `findMatches(articleRoot, NUMBER_PATTERN)`.
2. Keep numbers where `(unit === 'percent' OR digitCount >= digitsThreshold[=4]) AND !isLikelyYear`. The year filter prevents masking "2024" in a 2024-dated article.
3. Wrap each in a span that **preserves layout exactly**:
   ```html
   <span class="bias-ext-masked bias-ext-injected" style="position:relative;display:inline-block">
     <span class="bias-ext-actual" style="visibility:hidden">$ORIGINAL</span>     <!-- in flow, sets box size -->
     <span class="bias-ext-mask-overlay" style="position:absolute;inset:0;…">▒▒▒</span>
   </span>
   ```
   **Critical:** the actual text uses `visibility:hidden` (kept in flow so the box keeps the original width/height), never `display:none` or the `hidden` attribute — those collapse layout and cause a visible reflow on reveal. On reveal, the overlay is removed and visibility restored; **no layout shift**.

> **Security (Finding 1):** the original value is **not** stored in `data-*` attributes (page-readable). It lives in a content-script `WeakMap<HTMLElement, MaskedRecord>`. `prompt.tsx` reads it via `getMaskedRecord(span)`; the actual text is materialized only at reveal, then the WeakMap entry is deleted.

**Estimate prompt** (`prompt.tsx`): click a masked number → inline modal in the shadow root. Title "Estimate this number"; ±15-word context (mask still in place); a `type="text"` input parsed through `number-detector.parseMatch` (so "5%", "1.5M", "$3.2 billion" normalize) with a live "interpreting as: 1,500,000" preview; "I don't know" (skip); "Reveal" (disabled until a parseable estimate or skip). `handleInput` also calls `stopPropagation()` as defense-in-depth (Finding 2). On submit: compute signed + absolute proportional delta, reveal the original with an inline annotation (`5% (you guessed 12%, ×2.4 too high)`), and write a journal entry.

**Journal** (`journal.ts` client → service worker):
```ts
type EstimateEntry = { id?; timestamp; urlHost; maskedSnippet;
  actualValue; actualUnit; estimateValue|null; skipped; delta|null; absDelta|null }
recordEstimate(e) · listEstimates(opts?) · exportEstimatesAsJSON() · clearEstimates()
```
> **Security (Finding 3):** the journal client holds **no `openDB`**. All I/O is `chrome.runtime.sendMessage({ type: 'journal:add' | 'journal:list' | 'journal:clear' })`. The **service worker** owns the IndexedDB (`bias-ext`, store `calibration-estimates`, indexes `by-timestamp` + `by-url-host`) in the **extension origin** — so page scripts can't read it and the dashboard (extension origin) can. Retention cap **500** (oldest pruned via `by-timestamp`). **Privacy (Finding 7):** `urlHost` only — never full URL, query params, or page title (the `articleTitle` field was removed entirely).

**Calibration dashboard** (`calibration-dashboard.tsx`, opened in a new tab from the popup): read-only. Summary stats over entries where `estimateValue > 0 AND actualValue > 0` (zeros/negatives excluded from log math but counted in the total) — **mean absolute log-ratio** `mean(|log10(estimate/actual)|)` (the single calibration score), **bias direction** `mean(log10(estimate/actual))` (positive = over-estimating), and skip rate. A scatter plot (`x=log10(actual)`, `y=log10(estimate)`, y=x is perfect) plus the 10 worst-calibrated entries. "Export JSON" / "Clear data".

**Acceptance (green):** 1000-word article → 4+ digit numbers and all percentages masked; 300-word article → "too short" toast; click → modal with ±15-word context; estimate → reveal with delta; "I don't know" → skip recorded, revealed without delta; dashboard shows all entries; clear → empty; **revealing causes no reflow**.

---

## 5. Data layer

### 5.1 Reference tables (`src/data/reference-tables/` + `reference-manifest.json`)

**11 bundled tables**, indexed by `reference-manifest.json` (version `2024-q4`). Static JSON imports — no runtime fetch.

| id | Source | Tags |
|----|--------|------|
| `us-population-total` | US Census Bureau | population, us |
| `us-population-by-state` | US Census Bureau | population, us, state |
| `us-deaths-total-annual` | CDC | mortality, us, annual |
| `us-deaths-by-cause-cdc` | CDC | mortality, us, cause |
| `us-gdp-total` | BEA | economics, us, gdp |
| `world-gdp-per-capita` | World Bank | economics, world, gdp |
| `world-life-expectancy` | OWID | demographics, world |
| `us-cpi-monthly` | BLS | economics, us, monthly |
| `us-unemployment-monthly` | BLS | economics, us, labor |
| `us-violent-crime-rate-fbi` | FBI | crime, us, annual |
| `world-co2-emissions-owid` | OWID | climate, world |

**Table schema:**
```jsonc
{
  "id": "us-population-total",
  "source": "US Census Bureau, 2024 estimate",
  "source_url": "https://www.census.gov/",
  "fetched_at": "2025-01-01",
  "unit": "people",
  "series": [ { "period": "2024", "value": 334900000 }, … ],  // newest-first
  "distribution": null,
  "percentiles": null | { "p10": …, "p50": …, "p90": … }
}
```
`series` is ordered **newest-first** (the lookup relies on `series[0]` being the most recent). Adding a table is a **content task**, not engineering: drop a JSON file, add a manifest row, add it to `ALL_TABLES` + `TABLE_KEYWORDS` in `lookup.ts`. Provenance: baked in at build time; refresh = rebuild (no runtime refresh in v1).

### 5.2 Pattern libraries (`src/data/patterns/`)

- `denominator-missing.json` — 8 capture-grouped regex patterns (§4.2).
- `denominator-context-tokens.json` — 24 suppression tokens (§4.2).
- `health-test-keywords.json` — 14 keywords for the Bayes gate (§4.1).

### 5.3 Calibration journal (IndexedDB `bias-ext`)

Owned by the service worker (extension origin). One store `calibration-estimates`, auto-increment key, indexes `by-timestamp` + `by-url-host`, 500-entry retention. Schema = `EstimateEntry` (§4.4). The **only** persistent user data the extension writes, and it never leaves the device.

---

## 6. Settings & configuration

Stored under `chrome.storage.local['bias-ext:settings']`. `getSettings()` merges these defaults:

```ts
type Settings = {
  features: {
    'frequency-reframe': boolean       // default true
    'wheres-the-denominator': boolean  // default true
    'counter-anchor': boolean          // default true
    'suppress-and-estimate': boolean   // default false  (opt-in)
  }
  thresholds: {
    suppressEstimate: {
      digitsThreshold: number          // 4  — mask numbers with ≥4 integer digits
      maskPercentages: boolean         // true
      minArticleWords: number          // 500 — long-form gate
    }
    denominatorContextWindow: number   // 20 — ±words for denominator suppression
  }
  ui: { flagDensity: 'minimal' | 'normal' }  // 'normal' — WTD flag cap
}
```

**Live update:** `watchSettings` listens to `chrome.storage.onChanged`; `boot.ts` tears down all features and re-registers the enabled set, so toggles take effect within ~1 s with clean unwrapping.

**Surfaces:**
- **Popup** — 4 toggles, per-page Suppress-&-Estimate activation (enabled only on long-form), "open dashboard" (enabled only when the journal has entries).
- **Options** — toggles with longer copy; thresholds (digits, min words, context window); flag density; privacy (the journal stores hostname + snippet + values only, 500-cap, no title — export / clear); reference-table list with `fetched_at`; About section with literature citations.

---

## 7. Security & privacy model

A full audit (`SECURITY_AUDIT.md`, 13 findings, all CLOSED) hardened the extension against a hostile page. The five load-bearing fixes:

| # | Severity | Was | Now |
|---|----------|-----|-----|
| 1 | Critical | Masked values in page-readable `data-*` attrs | Held in content-script `WeakMap`; materialized only at reveal |
| 2 | Critical | Open shadow root + composed events leaked keystrokes | `mode:'closed'` + capture-phase `stopImmediatePropagation` on leaky events |
| 3 | Critical | Journal in **page-origin** IndexedDB (page-readable, dashboard-invisible) | Service-worker-owned IndexedDB in **extension origin** |
| 4 | High | Popup wrote `settings`; content read `bias-ext:settings` (toggles were no-ops) | Single exported `STORAGE_KEY` + `saveSettings()` |
| 5/6 | High | `data-bias-ext-test-features` attribute let any page force-enable features | Gated behind compile-time `__BIAS_EXT_E2E__`; stripped from production |

**Hygiene fixes:** features no longer run when no article is detected or when a visible password field is present (Finding 9); explicit CSP on extension pages (Finding 10); message listeners validate `sender.id === chrome.runtime.id` (Finding 11); ESLint `no-restricted-syntax` bans `innerHTML`/`insertAdjacentHTML`/`document.write`/`new Function` in `src/` (Finding 12); `web_accessible_resources` removed from the source manifest so reference data + dashboard aren't fingerprintable (Finding 8 — CRXJS re-injects only JS loader chunks, a structural MV3 requirement).

**Net privacy guarantee:** no network, no telemetry, no accounts; the only persisted data is the calibration journal (hostname + snippet + numeric estimate/actual), capped at 500 entries, in the extension's own partition, exportable and clearable by the user.

---

## 8. Build, test, and verification

### 8.1 Commands

```sh
pnpm install
pnpm build        # production dist/ (~116 KB total, <500 KB budget)
pnpm build:e2e    # build with test bridges (vite --mode=e2e)
pnpm test         # vitest unit tests (60)
pnpm e2e          # build:e2e then playwright (26 e2e)
pnpm lint         # eslint (0 errors; innerHTML guardrail active)
npx tsc --noEmit  # strict typecheck, clean
```

### 8.2 Test inventory

**Unit (vitest, 60):** `number-detector`, `bayes`, `patterns`, `context-check`, `dom-walker`, `article-detector`, `selection-watcher`. Notable cases: mammography posterior ≈ 0.078; `2024` → `isLikelyYear true`; `2,024` (comma) → false; `5GB` → no match; "1,200 of 50,000 patients died" → context-suppressed.

**E2e (Playwright, 26):** headed Chromium with the built `dist/` loaded (`--load-extension`). One spec per feature against committed fixtures in `tests/fixtures/`:
- `frequency-reframe.html`, `counter-anchor.html`, `denominator-missing.html`, `suppress-estimate.html`.

Playwright requires `headless:false` for MV3 extensions; the service worker is grabbed via `context.serviceWorkers()` / `waitForEvent('serviceworker')` because it may be idle.

### 8.3 Manual QA — live-site first run

`tests/manual-qa/walk-sites.mjs` drives 10 live sites and records counts + console errors (`report.json`, screenshots committed). First-run results:

| Site | Type | Counter-anchors | WTD pills | SE masked | Console errors |
|------|------|----------------:|----------:|----------:|----------------|
| Wikipedia (US economy) | reference, number-heavy | 143 | 9 | 894 | 0 |
| Wikipedia (COVID-19) | health stats | 68 | 12 | 1044 | 0 |
| BBC News | homepage | 0 | 0 | 0 | 0 (page warnings only) |
| Guardian | homepage | 1 | 0 | 2 | 0 |
| Reuters | homepage | 0 | 0 | 2 | 0 |
| CNN | homepage | 0 | 2 | 3 | page's own (CNN ad/player scripts) |
| AP News | homepage | 3 | 0 | 5 | page's own (BC debug) |
| NPR | homepage | 0 | 0 | 0 | 0 |
| arXiv (CS paper) | math-heavy | 0 | 0 | 9 | 0 |
| old.reddit /r/news | UGC | 0 | 0 | 0 | page 403 only |

**Reading the table:** the extension fires richly on number-dense article bodies (Wikipedia) and stays quiet on homepages and UGC — exactly the intended salience profile. All console errors are the *pages'* own scripts, not the extension. arXiv shows the year-masking guard working (no spurious counter-anchors on a math page).

---

## 9. File map

```
bias2bias3/                          # repo root IS the extension root (DECISIONS D-001)
├─ manifest.json                     # MV3 source manifest (CRXJS entrypoint; .ts paths)
├─ vite.config.ts                    # CRXJS + Preact JSX + __BIAS_EXT_E2E__ define
├─ package.json                      # deps: @mozilla/readability, idb, preact
├─ CLAUDE.md                         # how-to-work operating contract
├─ building_spec_bias2_bias3.md      # authoritative build spec (§0–§10)
├─ DECISIONS.md / PROGRESS.md        # decision log / build log
├─ SECURITY_AUDIT.md                 # 13 findings, all closed
├─ README.md                         # user/dev quickstart
├─ docs/
│  ├─ bias_extension_reference_manual.md   # ← THIS FILE
│  ├─ ap_outreach_agent_spec.md            # sibling project (reference for format)
│  ├─ Biases_Descriptions_Mitigation.docx  # literature: bias descriptions + mitigations
│  └─ biases_examples.docx                  # literature: worked examples per bias
├─ src/
│  ├─ background/service-worker.ts    # context menu + journal IDB owner + msg router
│  ├─ content/
│  │  ├─ boot.ts                      # detect article, register features, watch settings
│  │  ├─ shared/                      # dom-walker, number-detector, article-detector,
│  │  │                               #   selection-watcher, shadow-host, feature-toggles
│  │  └─ features/
│  │     ├─ frequency-reframe/        # bayes, health-test-detector, icon-array,
│  │     │                            #   frequency-tree, panel, index
│  │     ├─ wheres-the-denominator/   # patterns, context-check, flag, index
│  │     ├─ counter-anchor/           # lookup, sparkline, tooltip, index
│  │     └─ suppress-and-estimate/    # masking, prompt, journal, calibration-dashboard, index
│  ├─ data/
│  │  ├─ reference-manifest.json
│  │  ├─ reference-tables/*.json      # 11 datasets
│  │  └─ patterns/*.json              # denominator-missing, context-tokens, health-keywords
│  ├─ ui/styles/tokens.css            # design tokens
│  ├─ popup/   (popup.{html,tsx}, dashboard.html)
│  ├─ options/ (options.{html,tsx})
│  └─ global.d.ts                     # __BIAS_EXT_E2E__ ambient declaration
└─ tests/
   ├─ unit/        (7 vitest specs, 60 tests)
   ├─ e2e/         (4 playwright specs, 26 tests, + static-server.mjs)
   ├─ fixtures/    (4 committed HTML fixtures + sample-articles/)
   └─ manual-qa/   (walk-sites.mjs, report.json, screenshots/)
```

---

## 10. Known limits & deferred work

**Per-feature limits (real, documented):**
- **Counter-anchor** — table selection is keyword-heuristic; foreign currencies, non-US economies, and uncommon topics match nothing and produce no annotation. Only 11 datasets ship (the spec flags 25–30 as desirable for real-world coverage — open question 5).
- **Where's the Denominator?** — 8 patterns miss scientific notation, word-form numbers, and ratios ("1 in 5"); the ±20-word window can miss a denominator stated a sentence further away (false positive) or, rarely, suppress on an unrelated nearby number.
- **Suppress & Estimate** — long-form only (≥ 500 words); years (1900–2199) never masked; journal is local-only, never synced.
- **Frequency Reframe** — the Bayes branch needs all three of prevalence + sensitivity + specificity; two-of-three falls back to a plain icon array.

**Deferred / out of scope for this slice:**
- Bias 1 (Confirmation) and Bias 4 (Availability) features — they will reuse the shared shadow host, dom-walker, number-detector, and reference tables, but require the LLM/network layer this build deliberately omits.
- API-key handling, encryption-at-rest, BYOK (no secrets are stored here, so N/A).
- Local embedding models (transformers.js) — only needed for later biases.
- Cross-frame support, mobile, sync, auto-refresh of reference tables.
- Frequency-Reframe auto-suggest floating button (context menu only in v1).
- Word-form / ratio / scientific-notation number detection.

**Open questions carried from the spec (§9 of the build spec)** — surfaced, not unilaterally redefined:
1. Counter-anchor plausibility threshold (currently 1 order of magnitude) — tune against real articles.
2. Suppress-&-Estimate full-article vs. progressive-on-scroll masking (v1 = full article).
3. Multi-percentage non-Bayes selections render as side-by-side icon arrays (v1 = yes).
4. Flagging inside direct quotes (v1 = yes — the missing denominator still misleads the reader).
5. Reference-table count for visible value (ship more than 11 before a public v1).

---

## 11. Quick reference card

| You want to… | Look at |
|--------------|---------|
| Understand why a feature works | §1 (literature), then the feature's §4.x |
| Add a reference dataset | §5.1 — JSON + manifest row + `lookup.ts` registry/keywords |
| Add a denominator pattern | `data/patterns/denominator-missing.json` (§4.2) + a `patterns.test.ts` case |
| Change a default threshold | `feature-toggles.ts` DEFAULTS (§6) |
| Trace where masked values live | `WeakMap` in `masking.ts`; never in DOM (§7, Finding 1) |
| Find the journal storage owner | service worker; client in `journal.ts` (§4.4, §5.3) |
| Confirm the security posture | `SECURITY_AUDIT.md` + §7 |
| Run the whole verification | §8.1 |
| See real-world behavior | `tests/manual-qa/report.json` + §8.3 |

*End of reference manual. This is a living document — update it when behavior changes, not just when the spec does.*
