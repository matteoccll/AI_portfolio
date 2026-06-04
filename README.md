# Portfolio — Matteo Cacialli

MSc Psychology candidate. These projects span prompt engineering, software
architecture, and full-stack development. They are AI-assisted; the design,
specification, and engineering decisions are mine.

This hub repository holds the **design specs, system prompts, and data work**.
The software projects each live in their own repository — that's where the code,
tests, and failure write-ups are. Both are linked together below, project by project.

---

## Software

Each row is one project. **Repo** is the standalone codebase; **Spec** is the
design document in this repository; **Live** is a running deployment.

| Project | What it is | Links |
|---|---|---|
| **Indulj** | Static marketing site for a mobile massage therapy service. Design-token CSS, fluid `clamp()` typography, fully accessibility-compliant, no framework, no build step. Work in progress (~3 hours in, no prior web-development experience); content and copy are placeholders pending sign-off from the client. Published with the subject's knowledge and permission. | [live](https://matteoccll.github.io/indulj_website/) · [repo](https://github.com/matteoccll/indulj_website) |
| **BiasMiti** | Manifest V3 browser extension implementing four cognitive-debiasing mechanisms (base-rate neglect, anchoring) entirely client-side — no LLM, no network. Grounded in the cognitive-psychology literature. 86 passing tests; demo recorded on live Wikipedia. | [repo](https://github.com/matteoccll/BiasMiti_Extension) · [spec](software/bias_browser_extension/bias_extension_reference_manual.md) |
| **Polymarket Scanner** | Python CLI that fetches the full Polymarket catalog via the Gamma API and filters it to tractable markets through a six-stage pipeline with a deduplicated rejection-audit cache. 53 tests. Pure data pipeline — no trading. | [repo](https://github.com/matteoccll/Polymarket_Scanner) · [spec](software/polymarket-scanner/spec%20scanner.md) |
| **Armed Intelbot** | Three-mode systems project: topic-filtered on-chain WebSocket monitoring, a paper-trading engine with dual-regime P&L, and a live CLOB execution layer behind a nine-gate risk manager and circuit breaker. Its `DECISIONS.md` records three diagnosed failures. | [repo](https://github.com/matteoccll/Armed_Intelbot) · [spec](software/polymarket-trading-bot/spec.md) |
| **AP Outreach Agent** | Claude Code-orchestrated six-component pipeline for autonomous job-application outreach across NHS Trusts, private practices, and third-sector organisations. Design spec only — not implemented. | [spec](software/ap_outreach_agent_spec/ap_outreach_agent_spec.md) |

> For the scanner and Intelbot, the engineering is the point — async event
> processing, on-chain log decoding, risk-gate state machines, API-key rotation.
> The prediction market is just the data source.

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
