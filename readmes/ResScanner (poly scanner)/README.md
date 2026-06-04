# Polymarket Scanner

CLI tool that fetches the full Polymarket active-market catalog via the public Gamma API and filters it down to a short list of tractable, interesting binary markets.

## Background

ResAgent was designed as an autonomous prediction market agent: a scanner discovers markets, an LLM research pipeline estimates probabilities, and an execution engine places trades via the Polymarket CLOB API. The full agent was spec'd across three modules (scanner, research, executor) with a shared SQLite data layer.

The scanner is the first stage of that pipeline. It is fully self-contained — no database, no LLM calls, no trading capability, no dependency on any other ResAgent module — and runs as a standalone CLI tool for market discovery.

## Architecture

Two Gamma API endpoints are hit in sequence: `/events` (to collect event-level tags like "politics", "sports") then `/markets` (to collect individual binary contracts with prices and volume). Events are fetched first because Polymarket attaches tags to events, not to individual markets — so the scanner builds an in-memory index keyed on `conditionId` and `slug` for O(1) tag resolution during market normalisation.

Pagination is offset-based (100 items/page, hard cap at 50 pages / 5,000 markets per endpoint). All filtering runs client-side because server-side date filtering (`end_date_min`/`end_date_max`) is unreliable in the Gamma API. Every rejected market is written to a deduplicated JSON cache keyed on `(market_id, scan_date)` with the filter stage and rejection reason, enabling full audit of what was excluded and why.

After filtering, surviving markets are categorised (event tags first, keyword fallback from `config/category_keywords.json`, default `"other"`) and displayed in the terminal grouped by topic.

## Filter pipeline (executes in this order)

1. **Active status** — `market.status == "active"`
2. **Resolution window** — resolves within N days (default 30, configurable via `--days`)
3. **Crypto price speculation** — regex patterns requiring a coin name (`bitcoin`, `btc`, `eth`, etc.) in proximity to a price term (`above`, `below`, `$XX,XXX`), plus an ecosystem-term blacklist (`memecoin`, `defi tvl`, `hashrate`, etc.). Passes through crypto policy and regulation markets.
4. **Sports** — three-layer cascade: event tag set lookup (~28 tags) → team-name regex (~120 US professional teams with word boundaries) → strict keyword match (~22 unambiguous phrases like `"super bowl"`, `"champions league"`). No broad patterns — `" vs "` and short words like `"match"` are deliberately excluded to preserve political and legal markets.
5. **Volume floor** — `volume_total >= $40,000` (configurable via `--min-volume`)
6. **Odds range** — YES price in `[low, high]` or `[1-high, 1-low]`, default 15–40% / 60–85% (configurable via `--odds-low`, `--odds-high`)

## Stack

- Python 3.11+
- `requests` — HTTP client for Gamma API pagination
- `pydantic` — `Market` model validation and normalisation

## How to run

```bash
pip install -e .

# scan with defaults (30 days, $40k volume, 15-40% odds)
python -m resagent.scripts.run_scan

# all parameters customised
python -m resagent.scripts.run_scan --days 14 --min-volume 50000 --odds-low 0.10 --odds-high 0.45 -v

# export rejection cache to CSV
python -m resagent.scripts.export_rejected

# export to custom path, then clear cache
python -m resagent.scripts.export_rejected -o my_export.csv --clear

# just show cache statistics
python -m resagent.scripts.export_rejected --stats
```

## Tests

53 tests across two test files covering the scanner module:

- **Normalisation** (7) — outcome price parsing from JSON strings and lists, basic field mapping, missing `conditionId` rejection, non-Yes/No 2-outcome acceptance, 3-outcome rejection, event tag attachment from index, source URL fallback to market slug
- **Active filter** (1)
- **Resolution window** (3) — within window, past due, too far out
- **Crypto precision** (9) — blocks BTC price targets, ETH above $X, memecoin; keeps "BTC bill" (policy), "Bitcoin legal tender" (adoption), GDP figures, stock prices
- **Sports precision** (10) — detects via event tag, team name regex, strict keywords; preserves "Trump vs. Biden", "US vs. China", "Rome summit", "Boxing Day sales"
- **Odds** (6) — YES and NO underdog acceptance, boundary values, rejection of 50/50 and extremes
- **Volume** (3) — above/below threshold, None handling
- **Category detection** (3) — tag-based, keyword fallback, default to "other"
- **Full pipeline** (3) — mixed-input integration, crypto policy passthrough, political "vs" passthrough
- **Rejection cache** (5) — save/load round-trip, dedup by `(market_id, scan_date)`, CSV export with list flattening, clear, empty cache handling

```bash
pytest resagent/tests/test_scanner.py resagent/tests/test_cache.py
```

## Status

Working. Runs against the live Gamma API.

Known limitations: 5,000-market cap per scan; no incremental scanning (full re-fetch each run); server-side date filtering unreliable so all markets are fetched then filtered client-side.
