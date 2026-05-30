# AP Volunteer Outreach Agent — Build Specification
**Version:** 2.3
**Stack:** Claude Code (orchestrator) · TypeScript · Node 22.13+ · `node:sqlite` (built-in) · MCP (Outlook/Microsoft Graph, Tavily Search, Firecrawl) · Playwright (optional, puppet mode)
**Owner:** MSc Psychology candidate seeking voluntary Assistant Psychologist positions

**Changes from v2.2 (2026-05-29):** Composer rebuilt to a **6-slot static/dynamic assembly** (`src/compose.ts --assemble`). The model now writes only two dynamic fragments (opener hook + relevance bridge); the verbatim STATIC blocks live in `config/email_blocks.yaml` and are spliced in mechanically. Cover letters retired as corpus in favour of `corpus/SNIPPETS.md` (crosswalk + canonical facts + banned list). Follow-ups are fully static and re-attach the CV. Initial-email word band is 150–185 (target). See §4.5.

**Changes from v2.1:** Brave Search MCP replaced by Tavily Search MCP — Brave killed its free tier in early 2026; Tavily offers 1,000 free credits/month with no card required and is purpose-built for AI agent web search.

**Changes from v2.0:** geography simplified; Contact Resolver added between Scout and Score (org → person → email); CV and cover letters relocated to `corpus/` with corpus-as-context composer model; puppet mode (Playwright + user's real browser profile) added for anti-bot scraping and gated-site lookups.

---

## 1. Geographic Scope

Within ~2 hours of Hounslow TW3 by public transport. Claude judges per target during scoring using common knowledge of London-area travel; no journey-time API needed.

---

## 2. Target Taxonomy

### Tier 1 — NHS Mental Health Trusts (Primary)

**Trust list current as of May 2026.** Reflects the November 2024 north-London merger and the April 2026 Tavistock & Portman acquisition.

- West London NHS Trust *(local — highest priority)*
- Hounslow and Richmond Community Healthcare NHS Trust (HRCH)
- Central and North West London NHS Foundation Trust (CNWL)
- South West London and St George's Mental Health NHS Trust (SWLSTG)
- South London and Maudsley NHS Foundation Trust (SLaM)
- North London NHS Foundation Trust (NLFT) — formed 1 Nov 2024 from Camden & Islington + Barnet/Enfield/Haringey MH Trust; absorbed Tavistock and Portman NHS FT on 1 Apr 2026. Hosts the former Tavistock CAMHS/psychotherapy services and the Portman forensic clinic.
- East London NHS Foundation Trust (ELFT)
- Oxleas NHS Foundation Trust
- Surrey and Borders Partnership NHS Foundation Trust
- Berkshire Healthcare NHS Foundation Trust
- Oxford Health NHS Foundation Trust
- Hertfordshire Partnership University NHS Foundation Trust
- Hampshire and Isle of Wight Healthcare NHS Foundation Trust *(formed 1 Oct 2024 from Solent + Southern Health + IoW community/MH)*

The list lives in `config/tier1_trusts.yaml` (editable without code changes). Generic mailboxes (`info@`, `comms@`, `enquiries@`) are dropped by the Contact Resolver — see §4.3.

### Tier 2 — Clinical Psychologist Private Practices

Sources, in priority order:
1. **Psychology Today UK** — psychologytoday.com/gb/counselling (anti-bot — see puppet mode in §4.3)
2. **Counselling Directory** — counselling-directory.org.uk
3. **BPS Find a Psychologist** — bps.org.uk/find-psychologist
4. Practice websites discovered via search.

Not used: HCPC register (publishes registration status only, no contact data).

### Tier 3 — Mental Health Charities & Third Sector

Mind (local branches), Rethink Mental Illness, YoungMinds, Place2Be, Samaritans (local branches), SANE, OCD-UK, Cruse Bereavement Support, Anna Freud, Richmond Fellowship.

### Tier 4 — Specialist & Niche Settings

CAMHS services (NHS and third sector), NHS Talking Therapies providers, neuropsychology departments at general hospitals within scope, forensic services (HMP/YOI Feltham — local, Bedfont Road TW13 4ND — and HMP Wormwood Scrubs), older-adult psychology services, university psychology clinics (Brunel, St Mary's, Kingston, UCL, KCL, Royal Holloway), Primary Care Network mental-health workers.

---

## 3. System Architecture

The orchestrator is **Claude Code itself**, running locally in the project directory with full filesystem and tool access. Deterministic work (DB, JSON validation, file moves, pacing) is TypeScript invoked as `npm` scripts; all network IO — MCP search/scrape and Outlook sends — is done by Claude Code through MCP tools. LLM-needing work (scoring, composing) happens in the same Claude Code session — no separate Anthropic API key.

**Puppet mode (optional):** for sites that block headless browsers (Psychology Today, sometimes LinkedIn), Playwright launches Chrome with the user's *real* persistent profile (`~/Library/Application Support/Google/Chrome/Default` on macOS, equivalent on Linux/Windows). Single-shot, low-volume use of the user's own logged-in sessions — not bulk scraping. Default scout uses Firecrawl; puppet mode is opt-in per target via `--puppet`.

```
┌──────────────────────────────────────────────────────────────────┐
│                Claude Code (interactive session)                 │
│   reads CLAUDE.md → calls npm scripts and MCP tools as needed    │
└──┬───────────────────────────────────────────────────────────────┘
   │
   │  npm run scout              (Tavily Search MCP → org candidates)
   │  npm run resolve            (Firecrawl/Playwright → person + email)
   │  npm run review             (CLI review)
   │  npm run send               (Outlook MCP)
   │  npm run check-replies      (Outlook MCP)
   │  npm run followup:scan      (DB + Outlook MCP)
   │
   ▼
┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐
│ SCOUT  │──▶│ CONTACT  │──▶│  SCORE   │──▶│ COMPOSE  │──▶│ REVIEW QUEUE │
│ (orgs) │   │ RESOLVER │   │ (Claude) │   │ (Claude) │   │   (outbox/)  │
└────────┘   │ (person  │   └──────────┘   └──────────┘   └──────┬───────┘
             │  + email)│                                        │
             └──────────┘                              (human approves)
                                                                 │
                                                                 ▼
                                                          ┌─────────────┐
                                                          │   SENDER    │
                                                          │(Outlook MCP)│
                                                          └──────┬──────┘
                                                                 │
                                                                 ▼
                                              ┌──────────────────────────┐
                                              │ REPLY DETECTOR +         │
                                              │ FOLLOW-UP SCHEDULER      │
                                              └──────────────────────────┘
```

**Two run modes, same codebase:**

| Mode | Trigger | LLM billing route |
|---|---|---|
| Interactive (primary) | `claude` opened in project dir | Subscription interactive bucket — no extra charge under plan limits |
| Headless (cron) | `claude -p "..."` from a cron wrapper | Pre-15-Jun-2026: subscription. Post-15-Jun-2026: monthly Agent SDK credit (Pro $20 / Max 5x $100 / Max 20x $200) — still subscription-bundled, not Console-billed |

`ANTHROPIC_API_KEY` must be **unset** in the shell. `package.json`'s `prestart` script fails loudly if it's present — protects against accidental Console spend.

---

## 4. Component Specifications

### 4.1 Orchestrator — Claude Code + `CLAUDE.md`

No `main.ts`. The orchestrator is a Claude Code session reading the project's `CLAUDE.md`, which tells it the pipeline order, the prompt locations, the run modes, and the inviolable rules (never send without approval; never write to `agent.db` outside the migration scripts; never commit `corpus/cv.pdf` or `corpus/SNIPPETS.md` if the repo goes public).

### 4.2 Scout (`src/scout.ts`) — Org Discovery

**Tools:** Tavily Search MCP (`tavily-mcp`, the official Tavily-maintained server) + Firecrawl MCP (`firecrawl-mcp`). Both configured in `.mcp.json` so Claude Code picks them up on session start.

**Free tiers:** Tavily 1,000 credits/month (no card). Firecrawl 10 scrapes/min, 5 searches/min.

**Job:** discover candidate organisations only. Does *not* try to find the right person or email — that's the Resolver. Outputs to `tmp/orgs.json`:

```typescript
interface OrgCandidate {
  id: string;
  org_name: string;
  tier: 1 | 2 | 3 | 4;
  location: string;
  source_url: string;
  source_fetched_at: string;
  raw_notes: string;   // what the scrape revealed about the org
}
```

Deduplicates against `contacts.org_name` already in the DB.

### 4.3 Contact Resolver (`src/resolve.ts`) — Person + Email

This is the component most outreach pipelines skip and most reply rates die on. Splits into 4 sub-steps per org.

**Step 1 — Role hunt.** Search for the right named role at the org:
- NHS trusts: `"Head of Psychology" OR "Principal Clinical Psychologist" OR "Lead Psychologist" "[trust name]"`
- Tier 3 charities: `"clinical lead" OR "service manager" OR "head of services" "[charity name]" [borough]`
- Tier 4 services: service-specific (e.g., `"clinical lead" CAMHS "[area]"`)

Then scrape the org's "Meet the team" / "Our people" / service pages via Firecrawl. If gated or blocked, retry via Playwright in puppet mode using the user's real Chrome profile.

**Step 2 — Person extracted.** From the scraped markdown, identify a named person with role and service area (`Dr Jane Smith, Head of Psychological Services, Adult Mental Health`). Multiple candidates per org are fine — Resolver records them all.

**Step 3 — Email resolution.** In order of preference:
1. **Published.** Email visible on the org page (sometimes partial: `j.smith@swlstg.nhs.uk` — record verbatim).
2. **Constructed from convention.** Per-trust email conventions live in `config/nhs_email_conventions.yaml`. Example:
   ```yaml
   swlstg.nhs.uk:    "firstname.lastname"
   slam.nhs.uk:      "firstname.lastname"
   westlondon.nhs.uk: "firstname.lastname"
   nhs.net:          "firstname.lastname"   # generic NHS Mail
   ```
   Charities: derive convention by sampling any one visible email on their site (`contact@mindinhounslow.org.uk` → `firstname.lastname@mindinhounslow.org.uk`).
3. **LinkedIn lookup** via puppet mode using the user's logged-in session. Single-shot per target, never bulk. Confirms role + service; doesn't typically yield the email but supplements step 2.

**Step 4 — Confidence flag.** Each resolved contact gets `email_confidence: "published" | "constructed" | "unverified"`. Constructed emails are flagged in the review queue so the human can sanity-check.

**Output schema (`tmp/resolved.json`, becomes the input to Score):**

```typescript
interface ResolvedContact {
  id: string;
  org_id: string;
  org_name: string;
  tier: 1 | 2 | 3 | 4;
  contact_name: string;
  contact_role: string;
  contact_service_area?: string;
  contact_email: string;
  email_confidence: "published" | "constructed" | "unverified";
  source_urls: string[];
  raw_notes: string;
  resolved_at: string;
}
```

**Generic-mailbox filter:** any candidate where the resolved email is `info@`, `admin@`, `enquiries@`, `comms@`, `contact@`, `hello@`, `office@`, or `reception@` is dropped here. The point of the Resolver is precisely to avoid those.

### 4.4 Scorer

Performed by Claude Code reading `prompts/score.md` and `tmp/resolved.json`, writing `tmp/scored.json`. No standalone LLM service.

**Scoring criteria:**

| Criterion | Weight |
|---|---|
| Named person with role and service area | High |
| Psychology dept / provision confirmed | High |
| AP / trainee track record visible | High |
| Email confidence (`published` > `constructed` > `unverified`) | High |
| Active service signals (recent news, current ads, publications) | Medium |
| Travel feasibility (~2hrs from TW3 by judgment) | Medium |
| Supervision / CPD signals | Medium |

0–100 score per target. ≥ 60 → `scored`, eligible to compose. 40–59 → `deferred`. < 40 → `rejected`.

### 4.5 Composer (6-slot assembly)

The email is **assembled mechanically from six slots** — four verbatim STATIC blocks plus two DYNAMIC fragments. Claude Code writes only the fragments; `src/compose.ts --assemble` splices in the static blocks and writes the draft. This shrinks the model's surface to the one load-bearing sentence and makes the credentials un-driftable (the failure mode of the first guinea-pig run).

Slots, in order: (1) greeting, (2) OPENER_STATIC + dynamic opener hook, (3) CREDIBILITY_STATIC, (4) dynamic relevance bridge, (5) LOGISTICS_STATIC, (6) SIGNOFF_STATIC. The static blocks live verbatim in `config/email_blocks.yaml`.

Performed by Claude Code (on **Opus, max effort** — the relevance bridge is the hardest step in the pipeline), reading:
- `prompts/compose_initial.md` — procedure + rules
- `corpus/SNIPPETS.md` — content reservoir: CRITERION CROSSWALK (population → strongest evidence), CANONICAL FACTS (ground truth), BANNED PHRASINGS
- `corpus/cv.pdf` — context; attached to every email
- the `ScoredContact` — `contact_service_area` / `raw_notes` drive the bridge

The model writes a `ComposeInput` to `tmp/compose_input.json` (opener hook + relevance bridge + subject), then runs `npm run compose -- --assemble tmp/compose_input.json`.

**Relevance bridge (the load-bearing sentence):** read the target's specialty/population as specifically as possible → look up the SNIPPETS crosswalk → introduce a **fresh, concrete instance** (a named resident, a specific practice, a real episode the credibility block did not mention) and **assert** it carries over (never hedge; disclaimer-then-claim for partial matches). Do **not** restate or reword a skill the credibility block already lists — that restatement (the "put it into words" echo) is the worst failure mode, now hard-errored by the validator.

**Email rules (enforced by `src/compose.ts --validate`):**
- Word band **150–200** target (hard-fail outside 150–220). Static blocks + greeting are ~130 words, so the old ≤250-word target no longer applies. (Relaxed 2026-05-30 from 150–185/200: the first four sends clustered at the ceiling, and a binding budget rewarded abstraction/restatement over concrete specifics.)
- No em dashes or en-dash connectors anywhere (subject or body).
- No BANNED PHRASINGS; no hedged skill transfers; no lecturing the recipient about their own field; no echo of the credibility block's signature phrasings (`static_echo_phrases`).
- CV attached (`corpus/cv.pdf`). SNIPPETS/CV never named or attached as documents.
- Verbatim presence of all four static blocks; the opener hook continues the opener sentence (no full stop after "…position").

**Draft schema (`outbox/{target_id}.json`):**

```typescript
interface DraftEmail {
  target_id: string;
  to: string;
  cc?: string;
  subject: string;
  body: string;
  attachments: string[];        // ["corpus/cv.pdf"]
  composed_at: string;
  status: "pending_review" | "approved" | "rejected" | "edited";
  email_confidence: "published" | "constructed" | "unverified";  // surfaced in review
  edit_history?: { at: string; reason?: string }[];
}
```

### 4.6 Review Queue (Human-in-the-Loop)

```
> npm run review

[1/4] North London NHS Foundation Trust — Dr Sarah Chen, Principal Clinical Psychologist
Email: sarah.chen@northlondonmentalhealth.nhs.uk  (constructed — verify)
Subject: Voluntary AP inquiry — MSc Psychology, available from September
Attachments: corpus/cv.pdf
---
[Body shown]
---
(a) approve  (r) reject  (e) edit  (s) skip  (q) quit
```

Editing reopens the body in `$EDITOR`. Only `status: approved` drafts are visible to the Sender. **No email is ever sent without explicit human approval, including follow-ups.**

### 4.7 Sender (`src/sender.ts`)

Outlook MCP (`@softeria/ms-365-mcp-server` + `outlook-file-attach-mcp`; see CLAUDE.md "Outlook setup"). Initial emails are sent draft → attach CV by file path → send-draft; follow-ups (no attachment) via direct send-mail. Pre-send checks in order:
1. Suppression list — skip if email is suppressed.
2. Daily rate-limit count (`config/runtime.yaml`, default 10 / UTC day).
3. Outside `send_window_hours`? Warn, require `--yes` to override.
4. Confirmation prompt unless `--yes` was passed.

**Send-time pacing.** Within a session, the sender inserts a randomised 4–12 minute gap between consecutive sends — the dominant "looks bot-like" signal in cold outreach is 10 emails sent in the same second, not the transport. The pacing is configurable.

**On success:**
- `contacts.status` → `sent`, `sent_at` → now.
- Insert `send_log` row with `provider_message_id` and `provider_conversation_id`.
- Insert two `followups` rows (`+10d`, `+20d`, both `pending`).
- Move `outbox/{id}.json` → `sent/{id}.json`.

### 4.8 Reply Detector + Follow-up Scheduler (`src/scheduler.ts`)

**Reply Detector** runs before any follow-up is composed. For each `sent` contact with pending follow-ups, queries the Outlook MCP for messages from the recipient (same conversation / received after `sent_at`):
- Any inbound reply from the original recipient → `contacts.status` = `replied`, cancel pending follow-ups (`status: skipped`, reason `reply_received`).
- Scan reply body for opt-out tokens (`unsubscribe`, `remove me`, `do not contact`, `please stop`, `no thank you`, `not interested`). On match → write to `suppressions`.

**Follow-up scheduler** then processes `followups` rows where `scheduled_for <= now` AND parent contact is still `sent`. Composition is **mechanical** (`prompts/compose_followup.md`): the assembler emits the fixed `FOLLOWUP_N` template (only the first name varies) with the CV re-attached, dropping it into the review queue.

**Cron (compose-only, never auto-send):**
```
0 9 * * * cd /path/to/agent && claude -p "Run reply-detection then compose any due follow-ups into the review queue."
```

---

## 5. Data Layer (SQLite via node:sqlite)

`agent.db` (gitignored). Daily backup to `agent.db.bak` via `npm run db:backup`.

```sql
CREATE TABLE contacts (
  id TEXT PRIMARY KEY,
  org_name TEXT NOT NULL,
  tier INTEGER NOT NULL,
  contact_name TEXT,
  contact_role TEXT,
  contact_service_area TEXT,
  contact_email TEXT NOT NULL,
  email_confidence TEXT,        -- 'published' | 'constructed' | 'unverified'
  source_urls TEXT,             -- JSON array
  location TEXT,
  score INTEGER,
  status TEXT NOT NULL DEFAULT 'discovered',
  -- discovered | resolved | scored | deferred | rejected
  -- composed | sent | replied | closed | aborted_suppressed
  discovered_at TEXT,
  resolved_at TEXT,
  sent_at TEXT,
  closed_at TEXT,
  notes TEXT,
  UNIQUE (org_name, contact_email)
);

CREATE TABLE send_log (
  id TEXT PRIMARY KEY,
  contact_id TEXT NOT NULL REFERENCES contacts(id),
  email_type TEXT NOT NULL,     -- 'initial' | 'followup_1' | 'followup_2'
  sent_at TEXT NOT NULL,
  subject TEXT,
  provider_message_id TEXT,
  provider_conversation_id TEXT
);

CREATE TABLE followups (
  id TEXT PRIMARY KEY,
  contact_id TEXT NOT NULL REFERENCES contacts(id),
  followup_number INTEGER NOT NULL,
  scheduled_for TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  skip_reason TEXT,
  sent_at TEXT
);

CREATE TABLE suppressions (
  email TEXT PRIMARY KEY,
  org_name TEXT,
  added_at TEXT NOT NULL,
  reason TEXT NOT NULL          -- 'opt_out_reply' | 'manual' | 'bounce'
);

CREATE INDEX idx_contacts_status ON contacts(status);
CREATE INDEX idx_followups_due ON followups(status, scheduled_for);
```

---

## 6. Configuration

### `config/candidate.yaml`
```yaml
name: "[Your Name]"
degree: "MSc Psychology, [University], [Year]"
skills: ["psychometric assessment", "CBT-informed approaches", "research and audit", "risk assessment"]
clinical_interests: ["[e.g. adult mental health]", "[e.g. CAMHS]"]
availability: "[e.g. 2–3 days per week from September 2026]"
signature: |
  [Your Name]
  MSc Psychology | [University]
  [Phone] | [Email]
  [LinkedIn URL]
from_name: "[Your Name]"
```

### `config/runtime.yaml`
```yaml
targeting:
  tiers_enabled: [1, 2, 3, 4]
  min_score: 60
  defer_score: 40
sending:
  daily_send_limit: 10
  send_window_hours: [9, 17]   # UTC
  pacing_minutes: [4, 12]      # randomised gap between sends
  attach_cv: true
followups:
  enabled: true
  schedule_days: [10, 20]
review:
  outbox_dir: "./outbox"
  sent_dir: "./sent"
```

### `config/tier1_trusts.yaml`, `config/nhs_email_conventions.yaml`
Editable lists for the Scout and Resolver.

### `.mcp.json`
```jsonc
{
  "mcpServers": {
    "tavily": {
      "command": "npx",
      "args": ["-y", "tavily-mcp"],
      "env": { "TAVILY_API_KEY": "${TAVILY_API_KEY}" }
    },
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": { "FIRECRAWL_API_KEY": "${FIRECRAWL_API_KEY}" }
    }
    // Plus "outlook" (@softeria/ms-365-mcp-server) and "outlook-attach"
    // (outlook-file-attach-mcp) — see CLAUDE.md "Outlook setup (one-time)".
  }
}
```

---

## 7. Project File Structure

```
ap-outreach-agent/
├── CLAUDE.md                 # Operating instructions for the Claude Code session
├── README.md
├── .mcp.json
├── .env.example              # TAVILY_API_KEY, FIRECRAWL_API_KEY
├── .gitignore                # agent.db, agent.db.bak, corpus/, outbox/, sent/, tmp/
├── package.json
├── tsconfig.json
├── agent.db                  # gitignored
├── outbox/                   # drafts pending review
├── sent/                     # sent archive
├── tmp/                      # orgs.json, resolved.json, scored.json (gitignored)
├── corpus/                   # gitignored — personal material
│   ├── cv.pdf                # attached to every email
│   └── SNIPPETS.md           # content reservoir: crosswalk, canonical facts, banned list
├── config/
│   ├── candidate.yaml
│   ├── email_blocks.yaml     # verbatim static blocks + word band + banned/hedge/echo lists
│   ├── runtime.yaml
│   ├── tier1_trusts.yaml
│   └── nhs_email_conventions.yaml
├── prompts/
│   ├── score.md
│   ├── compose_initial.md
│   └── compose_followup.md
├── scripts/
│   └── check-no-api-key.js   # prestart guard
└── src/
    ├── db.ts                 # node:sqlite wrapper, migrations
    ├── scout.ts              # Tavily Search MCP, Firecrawl MCP
    ├── resolve.ts            # role hunt + email construction + LinkedIn (puppet)
    ├── puppet.ts             # Playwright with real Chrome profile (opt-in)
    ├── compose.ts            # --assemble (6-slot assembler) + --validate
    ├── review.ts             # CLI review interface
    ├── sender.ts             # Outlook MCP dispatch, pacing, pre-send checks
    ├── scheduler.ts          # reply detector + follow-up queueing
    └── types.ts
```

---

## 8. Dependencies

| Component | Choice | Notes |
|---|---|---|
| Orchestrator | Claude Code interactive + `claude -p` for cron | Subscription billing — no API key |
| Database | `node:sqlite` (built-in) | Synchronous, single-user; requires Node 22.13+ (no native toolchain needed) |
| Search | Tavily Search MCP (`tavily-mcp`) | Free: 1,000 credits/month, no card required |
| Headless scraping | Firecrawl MCP (`firecrawl-mcp`) | Free: 10 scrapes/min, 5 searches/min |
| Puppet scraping | `playwright` + user's real Chrome profile | Opt-in. For Psychology Today, LinkedIn, anything gated |
| Email send + reply read | Outlook MCP (`@softeria/ms-365-mcp-server`) | Microsoft Graph via delegated OAuth — recipient-side this looks identical to a human send |
| CV attachment by file path | `outlook-file-attach-mcp` | Attaches a local file to a draft without inlining base64 through the model. Reads its own flat token cache (`token-cache.attach.json`) — see CLAUDE.md "Outlook setup" |
| YAML | `yaml` | |
| UUIDs | `crypto.randomUUID()` | Node 20+ built-in |
| CLI prompts | `@inquirer/prompts` | |
| PDF reading (for CV ingestion) | `pdf-parse` | Composer reads `corpus/cv.pdf` as context |

---

## 9. LLM Billing — Operational Notes

1. `ANTHROPIC_API_KEY` stays **unset**. `prestart` script fails the run if it leaks in.
2. Primary mode: open `claude` in the project, drive the cycle interactively. Subscription bucket, no extra charge.
3. Cron mode: `claude -p`. Pre-15-Jun-2026 same bucket; post-15-Jun-2026 draws from the Agent SDK monthly credit (Pro $20 / Max 5x $100 / Max 20x $200), still subscription-bundled. If that runs out and overflow billing isn't on, headless pauses; interactive continues.

---

## 10. GDPR / Compliance Notes

- Cold outreach to staff at work addresses is generally permissible on a legitimate-interests basis under UK GDPR, subject to the recipient's right to object.
- Reply scanner watches for opt-out tokens; matches go to `suppressions` and the Scout/Resolver drop them silently in future cycles.
- `corpus/` is gitignored. The CV is attached only to the candidate's own outbound emails. `SNIPPETS.md` is reference material that never leaves the local machine.
- Manual suppression: `npm run suppress -- <email> "reason"`.

---

## 11. Build Order

1. `db.ts` + migrations. Verify with `sqlite3 agent.db ".schema"`.
2. `config/*.yaml` (incl. `email_blocks.yaml`), `corpus/cv.pdf`, `corpus/SNIPPETS.md`, prestart guard, `.mcp.json`.
3. `scout.ts` with `--dry-run --tier=1 --limit=10` → check `tmp/orgs.json` looks right.
4. `resolve.ts` on those orgs → check `tmp/resolved.json`. Validate constructed emails against actual NHS conventions for those trusts.
5. Score those 10 manually in Claude Code (`prompts/score.md`). Calibrate threshold.
6. Compose 3 emails via the assembler. Critically review the two dynamic fragments — especially the relevance bridge (no hedge, no lecture, adds a fresh specific rather than restating the credibility block). Iterate `prompts/compose_initial.md`.
7. `review.ts`.
8. `sender.ts` — send one test to the candidate's own address first. Confirm pacing works.
9. `scheduler.ts`.
10. `CLAUDE.md`.
11. First live run: 3 emails. Review carefully. Send. Monitor for 24h before going to full daily volume.

---

## 12. Bootstrap Prompt for Claude Code

```
Open the ap-outreach-agent project. Read CLAUDE.md.

Run one cycle of 5 Tier 1 targets, stopping before review:
  1. npm run scout -- --tier=1 --limit=5  →  tmp/orgs.json
  2. npm run resolve -- --input=tmp/orgs.json  →  tmp/resolved.json
       (use puppet mode if Firecrawl gets blocked on any source)
  3. Score the resolved contacts using prompts/score.md
       →  tmp/scored.json
  4. For any target scoring >= 60, compose an initial email
       using prompts/compose_initial.md and corpus/SNIPPETS.md:
       write the two dynamic fragments (opener hook + relevance
       bridge) plus a subject into tmp/compose_input.json, then
       npm run compose -- --assemble tmp/compose_input.json.
       The assembler splices the verbatim static blocks, attaches
       corpus/cv.pdf, and writes outbox/{id}.json
       (status=pending_review, ~150-200 words). Run on Opus, max effort.
  5. Stop. Report what's in the outbox; do not run review or send.
```
