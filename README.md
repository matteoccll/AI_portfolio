# Portfolio — Matteo Cacialli

MSc Psychology candidate. These projects span prompt engineering, software
architecture, and full-stack development. They are AI-assisted; the design,
specification, and engineering decisions are mine.

This hub repository holds the **design specs, system prompts, web builds, and
data work**. Each software project lives in its own repository — that's where
the code, tests, and failure write-ups are — and the spec here links out to it.
The websites likewise have their own repos and live deployments, with a snapshot
of each built site kept here so the hub stays self-contained. Everything is
linked below, project by project.

---

## Software

Each row is one project. **Repo** is the standalone codebase; **Spec** is the
design document in this repository.

| Project | What it is | Links |
|---|---|---|
| **BiasMiti** | Manifest V3 browser extension with four reading-moment debiasing features across two cognitive biases — base-rate neglect (Frequency Reframe, Where's the Denominator?) and anchoring (Counter-anchor, Suppress & Estimate). Fully deterministic and entirely on-device — no LLM, no network. Grounded in the cognitive-psychology literature. 86 passing tests; demo recorded on live Wikipedia. | [repo](https://github.com/matteoccll/BiasMiti_Extension) · [manual](software/biasmiti/reference_manual.md) |
| **Polymarket Scanner** | Data pipeline. Python CLI that fetches the full Polymarket catalog via the Gamma API and filters it to tractable markets through a six-stage pipeline with a deduplicated rejection-audit cache. 53 tests. | [repo](https://github.com/matteoccll/Polymarket_Scanner) · [spec](software/polymarket-scanner/spec.md) |
| **Armed Intelbot** | Three-mode systems project: topic-filtered on-chain WebSocket monitoring, a paper-simulator engine with dual-regime P&L, and a live CLOB execution layer behind a nine-gate risk manager and circuit breaker. Its `DECISIONS.md` records three diagnosed failures. | [repo](https://github.com/matteoccll/Armed_Intelbot) · [spec](software/armed-intelbot/spec.md) |
| **AP Outreach Agent** | Claude Code-orchestrated six-component pipeline for autonomous job-application outreach across NHS Trusts, private practices, and third-sector organisations. Fully built and in active use, running live outreach across London. The repository stays private to protect the people and correspondence it touches — not because the code is unfinished. | [spec](software/ap-outreach-agent/spec.md) · code on request |

---

## Websites

These are real client partnerships, not self-directed demos — each site is built
with and for a working professional. That is the thing to understand here: their
pace and polish are set by the client, not by me. Copy, photography, and the
back-end wiring all wait on the partner's availability and sign-off, so both sites
are deliberately unfinished rather than abandoned.

**Indulj** is an early stub — a first pass. **Two Little Giraffes** has been through
many more rounds of iteration and is essentially there; it is waiting only on
genuine photographs from the shop and the final wiring to take it live online.

Each site still has its own repo and live deployment, with a snapshot kept here
(under [`websites/`](websites/)) so the hub stays self-contained — **Live** is the
deployment, **Repo** the standalone codebase, **Source** the in-repo snapshot.

| Project | What it is | Links |
|---|---|---|
| **Two Little Giraffes** | Static multi-page site for an artisan gelato & café in Battersea — six pages served from one self-contained HTML file via hash-based vanilla-JS routing. Design-token CSS, three variable fonts, fluid `clamp()` type, and a data-driven flavour board rendered from inline JSON. No framework, no build step; accessibility via semantic landmarks, ARIA navigation state, and `prefers-reduced-motion`. Many iterations in and essentially complete — awaiting genuine photographs from the shop and the final go-live wiring. Published with permission. | [live](https://matteoccll.github.io/twolittlegiraffes_website/) · [repo](https://github.com/matteoccll/twolittlegiraffes_website) · [source](websites/twolittlegiraffes/) |
| **Indulj** | Static marketing site for a mobile massage therapy service. Design-token CSS, fluid `clamp()` typography, fully accessibility-compliant, no framework, no build step. Work in progress (~3 hours in, no prior web-development experience); content and copy are placeholders pending sign-off from the client. Published with the subject's knowledge and permission. | [live](https://matteoccll.github.io/indulj_website/) · [repo](https://github.com/matteoccll/indulj_website) · [source](websites/indulj/) |

---

## System prompts

Standalone prompts in [`system_prompts/`](system_prompts/), each engineering a
specific reasoning workflow.

| Prompt | What it does |
|---|---|
| **[Cover Letter Tailor](system_prompts/cover_letter_tailor.md)** | A 7-step argumentative writing workflow: criterion crosswalk, pre-write gates, named voice moves, and hard fabrication boundaries. |
| **[Personal Diagnostician](system_prompts/personal_diagnostician.md)** | Clinical-reasoning protocol: five-phase diagnostic flow, dual evidence/confidence tagging, self-examination maneuvers as first-class outputs. |
| **[Polymarket Research Analyst](system_prompts/polymarket_research_analyst.md)** | Nine-step calibration-first research protocol; market price withheld until step 7; built from a documented taxonomy of seven live-testing failure modes. |
| **[De-AI-ify Text](system_prompts/de-ai-fier.md)** | Editing protocol classifying LLM prose patterns into hard cuts and judgment calls, with an overcorrection guard. |

---

## Data

| Project | What it is |
|---|---|
| **[Excel Dashboard](excel%20dashboard/)** | Before/after data transformation: raw rows to pivot tables, formulas, and a performance-gap dashboard. Native Excel only, built in three days via AI-assisted upskilling. |
