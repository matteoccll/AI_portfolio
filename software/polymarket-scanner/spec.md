# Polymarket Scanner

## Standalone Market Discovery & Filtering

---

## 1. Purpose

The scanner connects to Polymarket's public Gamma API, fetches the full catalog of active binary markets, filters them through a 6-stage pipeline, categorises survivors, and displays results in the terminal grouped by topic. Every market that gets filtered out is recorded to a local JSON cache for later inspection or CSV export. No database, no LLM, no trading — pure API + filtering + display.

---

## 2. Data Model

All markets are normalised into a single Pydantic model (`resagent/data/models.py`):

```
Market
  market_id          str              conditionId (or id fallback)
  question           str              market question text
  description        Optional[str]    resolution criteria
  category           Optional[str]    raw API category (rarely populated)
  resolution_date    Optional[str]    YYYY-MM-DD
  source_url         Optional[str]    polymarket.com link (event or market)
  first_seen_at      Optional[str]    ISO timestamp
  last_scanned_at    Optional[str]    ISO timestamp
  current_yes_price  Optional[float]  0.0–1.0
  current_no_price   Optional[float]  0.0–1.0
  volume_24h         Optional[float]  USD
  volume_total       Optional[float]  USD lifetime volume
  total_liquidity    Optional[float]  USD
  status             MarketStatus     active | resolved_yes | resolved_no | excluded
  event_tags         List[str]        from parent event (e.g. ["politics", "elections"])
  event_slug         Optional[str]    parent event slug
```

All Polymarket markets are binary (2 outcomes). Multi-outcome events are collections of separate 2-outcome contracts. The scanner accepts any 2-outcome market regardless of outcome labels (not just "Yes"/"No").

---

## 3. Polymarket Gamma API

**Base URL:** `https://gamma-api.polymarket.com`
**Auth:** None (public read-only)

### 3.1 Endpoints Used

| Endpoint | Purpose |
|---|---|
| `GET /events` | Fetch event groups — these carry **tags** (sports, politics, etc.) that individual markets lack |
| `GET /markets` | Fetch individual binary contracts with prices, volume, dates |

### 3.2 Pagination

Both endpoints use offset-based pagination:
- `limit` (default 100) + `offset` query params
- A generic `_paginate()` function handles both endpoints identically
- Progress printed every 10 pages
- Hard cap: `MAX_PAGES = 50` (5,000 items max per endpoint)

### 3.3 Query Parameters

Both endpoints accept server-side date filters (though the API may not always respect them):
- `active=true&closed=false` — active markets only
- `end_date_min` — resolution date lower bound (ISO 8601)
- `end_date_max` — resolution date upper bound (ISO 8601)

### 3.4 Rate Limiting & Error Handling

| Scenario | Behaviour |
|---|---|
| Success | 1-second delay between pages (`RATE_LIMIT_DELAY`) |
| HTTP 429 | Back off 30 seconds, retry. Max 3 retries total. |
| HTTP 5xx / other | Log error, stop pagination for that endpoint. |
| Network error | Exponential backoff (2^n seconds), max 3 retries. |
| Malformed JSON | Skip page, advance offset, continue. |
| Request timeout | 30 seconds per request. |

### 3.5 Event Tag Index

Events are fetched first. An index is built for O(1) tag lookup during market normalisation:

```
Dict[str, Dict]
  "cid:{conditionId}" -> { tags: [...], title: "...", slug: "..." }
  "slug:{eventSlug}"  -> { tags: [...], title: "...", slug: "..." }
```

Tags are extracted from the API's `tags` field, which can be a list of strings or a list of `{label, slug}` objects. All are lowercased.

---

## 4. Normalisation

`normalise_market()` converts a raw API dict into a `Market` model. A market is accepted if:

1. It has a `conditionId`, `condition_id`, or `id` field (tried in that order)
2. It has a non-empty `question`
3. It has exactly 2 outcomes (any labels, not just Yes/No)
4. It has parseable `outcomePrices` (JSON string or list of 2 floats)

Normalisation steps:
- `outcomePrices`: parse from JSON string `"[0.62, 0.38]"` or list → `(yes_price, no_price)`
- `volume`, `liquidity`, `volume24hr`: safe-cast to float (None on failure)
- `endDate`: ISO datetime → `YYYY-MM-DD` date string
- `groupSlug` → `source_url` as `https://polymarket.com/event/{slug}` (falls back to `https://polymarket.com/market/{marketSlug}`)
- Event tags resolved from the pre-built event index by `conditionId` then by `groupSlug`
- Closed markets get `RESOLVED_YES` (if yes_price >= 0.95) or `RESOLVED_NO` status

Unparseable markets are skipped with a warning. First 5 skipped markets are logged with diagnostic fields for debugging.

---

## 5. Filter Pipeline

Six sequential filters. Each filter receives the surviving list and returns a subset. The order matters — cheaper/broader filters run first.

```
Raw markets
    │
    ▼
┌─────────────────────┐
│ 1. Active           │  status == "active"
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 2. Resolution window│  0 < days until resolution <= max_days (default 30)
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 3. Crypto price     │  Blocks price speculation, keeps policy/regulation
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 4. Sports           │  3-layer detection: tags → team names → strict keywords
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 5. Volume           │  volume_total >= min_volume (default $40,000)
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 6. Odds             │  YES price in [low, high] or [1-high, 1-low]
└─────────────────────┘
    │
    ▼
  Survivors → categorise → display
```

### 5.1 Active Filter

Simple: keeps only markets where `status == "active"`.

### 5.2 Resolution Window Filter

Keeps markets resolving within `max_days` days from now (default: 30). Markets with no resolution date or already past due are dropped.

Condition: `0 < days_until_resolution <= max_days`

### 5.3 Crypto Price Filter

**Goal:** Block crypto price speculation. Keep crypto policy, regulation, and adoption markets.

**Detection strategy:**

1. **Regex patterns** (4 patterns) — require a crypto coin name AND a price-related term in proximity:
   - `{coin} ... {price_word}` — e.g. "Bitcoin reach $100k"
   - `{price_word} ... {coin}` — e.g. "price of Ethereum"
   - `{coin} ... ${digits}` — e.g. "BTC $100,000"
   - `${digits} ... {coin}` — e.g. "$5,000 ETH"

   Coin names: `bitcoin, btc, eth, ethereum, solana, sol, xrp, doge, dogecoin, cardano, litecoin, polkadot, avalanche, bnb`

   Price words: `price, above, below, reach, hit, drop, fall, rise, pump, dump, value`

2. **Exact ecosystem terms** — always blocked regardless of context:
   `memecoin, meme coin, defi tvl, gas fee, hashrate, halving, stablecoin depeg, usdt peg, usdc peg, tether depeg`

**Passes through:**
- "Will Venezuela declare a BTC bill?" (no price word)
- "Will El Salvador keep Bitcoin as legal tender?" (no price word)
- "Will US GDP go above $30 trillion?" (no crypto coin name)

**Blocks:**
- "Will BTC reach $100k by June?" (coin + price target)
- "ETH above $5,000 by December?" (coin + price word + dollar amount)
- "Will this memecoin 10x?" (ecosystem term)

### 5.4 Sports Filter

**Goal:** Block sports markets. Keep political "vs" matchups, geographic references to sports-city names, and general uses of sports-adjacent words.

**3-layer detection (in priority order):**

1. **Event tags** — most reliable signal. Checks `market.event_tags` against a set of ~28 known sports tags:
   `sports, sport, nba, nfl, mlb, nhl, mls, soccer, football, basketball, baseball, hockey, tennis, golf, mma, ufc, boxing, cricket, rugby, f1, nascar, olympics, ncaa, college, esports, racing, motorsport`

2. **Team name regex** — catches markets mentioning ~120 US professional team names:
   `eagles, chiefs, lakers, celtics, yankees, dodgers, warriors, rams, cowboys, ...`
   All matched with word boundaries (`\b`) to avoid substring collisions.

3. **Strict keyword phrases** — only long, unambiguous sports phrases (~22 entries):
   `super bowl, nba finals, nfl draft, world series, stanley cup, march madness, champions league, premier league, bundesliga, la liga, wimbledon, grand prix, ballon d'or, ...`

**Deliberately NOT used:**
- No `" vs "` pattern (kills "Trump vs. Biden", "US vs. China", "Apple vs. Samsung")
- No short words like "match", "game", "win", "score" (too many non-sports meanings)
- No city names (kills geographic references like "Rome summit")

**Passes through:**
- "Trump vs. Biden debate winner?"
- "Will the Rome summit produce a climate deal?"
- "Will Boxing Day sales exceed $10B?"

### 5.5 Volume Filter

Keeps markets with `volume_total >= min_volume` (default: $40,000). Markets with `None` volume are dropped.

### 5.6 Odds Filter

Keeps markets where the YES price falls in an "interesting" range — not too certain, not too unlikely. Since all markets are binary, the filter checks both sides symmetrically:

```
Keep if: (low <= yes_price <= high) OR ((1-high) <= yes_price <= (1-low))
```

Default: `low=0.15, high=0.40` → keeps markets where either outcome is priced 15%–40%.

This means a market at YES 25% passes (underdog YES side), and a market at YES 75% also passes (underdog NO side at 25%).

**Most permissive setting:** `--odds-low 0.01 --odds-high 0.50` captures everything except markets priced at exactly 0% or 100%.

---

## 6. Rejection Tracking

Every market eliminated by the filter pipeline is recorded with:

```
{
  market_id        str        unique market identifier
  question         str        market question
  rejected_by      str        filter name: "active" | "resolution" | "crypto" | "sports" | "volume" | "odds"
  rejection_reason str        human-readable explanation
  yes_price        float      price at time of scan
  volume_total     float      total volume at time of scan
  resolution_date  str        YYYY-MM-DD
  source_url       str        polymarket.com link
  event_tags       list[str]  tags from parent event
  scan_date        str        YYYY-MM-DD of this scan
}
```

The `_filter_step()` wrapper in `scanner.py` handles this transparently. Each filter function is unchanged — the wrapper diffs the input/output sets and records what was dropped.

---

## 7. Rejection Cache

**Location:** `resagent/scanner/cache/rejected_markets.json` (gitignored)

A flat JSON array that accumulates across scans. No external database.

### Operations

| Function | Behaviour |
|---|---|
| `load_cache()` | Read JSON file, return list. Empty list if file missing or corrupted. |
| `save_cache(entries)` | Overwrite file with provided entries. Creates directory if needed. |
| `append_rejections(rejections)` | Merge new rejections into existing cache. **Deduplicates by `(market_id, scan_date)` tuple.** Returns count of newly added entries. |
| `export_csv(output_path)` | Write all cached rejections to CSV. List fields (event_tags) are joined with `"; "`. Returns row count. |
| `clear_cache()` | Delete the cache file. Returns count of entries removed. |

### CSV Columns

`market_id, question, rejected_by, rejection_reason, yes_price, volume_total, resolution_date, source_url, event_tags, scan_date`

---

## 8. Category Detection

After filtering, surviving markets are categorised for display grouping.

**Strategy:** Event tags first, keyword fallback second.

### 8.1 Tag-Based Detection

If a market has event tags, they're mapped to categories via a hardcoded lookup:

| Tags | Category |
|---|---|
| `politics, elections, us politics, world politics` | politics |
| `geopolitics, war` | geopolitics |
| `economics, finance, crypto, business` | economy |
| `science, health, climate` | science |
| `technology, ai, tech` | technology |
| `entertainment` | entertainment |
| `culture` | culture |
| `sports, sport` | sports |

### 8.2 Keyword Fallback

If no tag matches, the market question is checked against `config/category_keywords.json` — a JSON file mapping 9 category names to keyword lists. First match wins.

Categories: `politics, geopolitics, regulation, economy, sports, technology, entertainment, science, culture`

### 8.3 Default

Markets matching no tag or keyword get category `"other"`.

---

## 9. CLI Interface

### 9.1 Scanner (`python -m resagent.scripts.run_scan`)

Runs one full scan cycle and displays results in the terminal.

| Flag | Default | Description |
|---|---|---|
| `--days` | 30 | Maximum days until resolution |
| `--odds-low` | 0.15 | Lower bound for interesting odds |
| `--odds-high` | 0.40 | Upper bound for interesting odds |
| `--min-volume` | 40000 | Minimum total market volume (USD) |
| `-v, --verbose` | off | Show debug logging |

**Terminal output:**
- Header with scan timestamp
- Filter pipeline diagnostics (count at each stage)
- Markets grouped by category, sorted by days to resolution within each group
- Each market shows: question, odds (colour-coded), days to resolution (colour-coded), Polymarket link

**Display conventions:**
- Categories shown in preferred order: politics, geopolitics, regulation, economy, sports, technology, entertainment, science, culture, other
- Odds: green highlight on the underdog side (15–40%)
- Days to resolution: red (<=3d), yellow (<=7d), dim (>7d)

### 9.2 Export (`python -m resagent.scripts.export_rejected`)

Export or inspect the rejection cache.

| Flag | Default | Description |
|---|---|---|
| `-o, --output` | `rejected_markets.csv` | Output CSV file path |
| `--stats` | off | Show cache statistics only (no export) |
| `--clear` | off | Clear cache after exporting |

**Stats display:** Total entries, scan dates covered, breakdown by rejection reason.

---

## 10. Orchestration Flow

`run_scan()` in `scanner.py` is the single entry point. Returns `(Dict[str, List[Market]], Dict[str, int])` — categorised markets and diagnostic counters.

```
1. Compute date window: end_date_min = now, end_date_max = now + max_days

2. Fetch events from /events (for tags)
   → Build event tag index (cid + slug lookups)

3. Fetch markets from /markets
   → Normalise all to Market models (attach event tags from index)

4. Run 6-stage filter pipeline:
   active → resolution → crypto → sports → volume → odds
   (each step records rejections via _filter_step wrapper)

5. Cache all rejections to disk (append with dedup)

6. Categorise survivors (tag-first, keyword-fallback)
   → Sort each category by days to resolution (ascending)

7. Return (markets_by_category, diagnostics)
```

### Diagnostics Dict

Returned alongside results for pipeline transparency:

| Key | Value |
|---|---|
| `events_fetched` | Number of events retrieved from API |
| `markets_fetched` | Number of raw markets retrieved from API |
| `normalised` | Markets after normalisation (2-outcome, parseable) |
| `with_tags` | How many normalised markets have event tags |
| `after_active` | Count after active filter |
| `after_resolution` | Count after resolution window filter |
| `after_crypto` | Count after crypto price filter |
| `after_sports` | Count after sports filter |
| `after_volume` | Count after volume filter |
| `after_odds` | Count after odds filter |
| `rejections_cached` | New rejection entries written to cache |

---

## 11. File Layout

```
resagent/
  data/
    models.py                    Market model (+ other unused legacy models)
  scanner/
    client.py                    Gamma API client: fetch, paginate, normalise
    filters.py                   All filter logic + category detection
    scanner.py                   Orchestration: run_scan()
    cache.py                     Rejection cache: load/save/append/export/clear
    cache/
      rejected_markets.json      Runtime cache file (gitignored)
  config/
    category_keywords.json       Keyword → category mapping (9 categories)
    filter_keywords.json         Legacy file, NOT used by current filters
  scripts/
    run_scan.py                  CLI entry point for scanning
    export_rejected.py           CLI entry point for cache export
  tests/
    test_scanner.py              48 tests: normalisation, all filters, categories, pipeline
    test_cache.py                5 tests: save/load, dedup, CSV export, clear, empty
```

### Configuration Files

**`config/category_keywords.json`** — Active. Used by `detect_category()` as keyword fallback when event tags don't match. 9 categories with ~20 keywords each.

**`config/filter_keywords.json`** — **Stale. Not used by current code.** The crypto and sports filter logic is implemented directly in `filters.py` using regex patterns and constants, not loaded from this JSON file. Kept for reference only.

---

## 12. Test Coverage

### Normalisation Tests (7 tests)
- Outcome price parsing: JSON string, list, None, 3-value rejection
- Basic market normalisation with all fields
- Missing conditionId rejection
- Non-Yes/No 2-outcome acceptance
- 3-outcome rejection
- Event tag attachment from index
- Source URL fallback to market slug

### Filter Tests (27 tests)

**Active (1):** Keeps only active status, drops excluded/resolved.

**Resolution (3):** Within window passes, past-due drops, too-far-out drops.

**Crypto (9):** Blocks BTC price target, ETH above $X, Bitcoin price keyword, memecoin. Keeps crypto policy ("BTC bill"), crypto adoption ("Bitcoin legal tender"), non-crypto, GDP with "above $", stock prices.

**Sports (10):** Detects via event tag, NBA tag, team name (Lakers), strict keyword (super bowl). Keeps "Trump vs. Biden", "US vs. China", "Apple vs. Samsung", "Rome summit", "Boxing Day sales", "striking workers".

**Odds (6):** Keeps YES underdog (0.25), NO underdog (0.75). Rejects 50/50 and extreme (0.05). Boundary tests at 0.15 and 0.40.

**Volume (3):** Keeps high volume, rejects low volume, rejects None volume.

### Category Detection Tests (3 tests)
- Tag-based detection (politics tag → politics)
- Keyword fallback (inflation → economy)
- No match → "other"

### Full Pipeline Tests (3 tests)
- Mixed batch: only the "good" market survives all 6 filters
- Crypto policy market passes through entire pipeline
- Political "vs" market passes through entire pipeline

### Cache Tests (5 tests)
- Save and load round-trip
- Append with dedup by (market_id, scan_date)
- CSV export with list field flattening
- Clear cache
- Empty/missing cache file handling

---

## 13. Known Limitations

1. **Server-side date filtering unreliable.** The Gamma API may ignore `end_date_min`/`end_date_max` params, returning all active markets regardless. The client-side resolution filter handles this, but it means fetching up to 5,000 markets when fewer are needed.

2. **5,000 market cap.** `MAX_PAGES = 50` at 100 per page. If Polymarket ever has more than 5,000 active markets, some will be missed.

3. **No incremental scanning.** Each scan fetches the full catalog. There's no "fetch only new since last scan" mechanism.

4. **Sports detection gaps.** The team name regex covers ~120 US professional teams. International clubs, college teams, and individual athletes are only caught via event tags or strict keywords.

5. **Category detection is best-effort.** Keyword matching can miscategorise. Event tags from the API are more reliable but not always present.

6. **Stale config file.** `config/filter_keywords.json` exists but is not used. The actual filter logic is hardcoded in `filters.py`. This avoids runtime config issues but means filter changes require code changes.
