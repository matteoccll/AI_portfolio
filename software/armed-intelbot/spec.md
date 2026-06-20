# Armed Intelbot — Technical Reference

**Audience**: Claude Code (or any developer continuing this project).

---

## Purpose

Armed Intelbot monitors a Polymarket trading bot's wallet on Polygon. When the target bot trades, IntelBot:

1. Detects the trade on-chain via Alchemy WebSocket
2. Resolves `token_id` → market metadata via Polymarket CLOB API
3. Enriches with real-time sportsbook odds (TheOddsAPI) or esports match info (PandaScore)
4. Stores everything in PostgreSQL

The goal is **data collection for reverse-engineering** — capture what odds/game state existed at the exact moment the bot traded, then analyze its edge.

A **Dry Run Engine** (paper trading simulator) sits on top of IntelBot and simulates a market-making strategy on soccer markets (moneylines, over/under, and handicap spreads) using multi-bookmaker consensus odds vs Polymarket prices.

A **Live Execution Layer** sits on top of the Dry Run Engine and routes its trading decisions to Polymarket's CLOB as real GTC limit orders (no server-side expiry — the bot's own mechanisms cancel). It shares the same regime logic (spread vs directional) and enrichment pipeline — the dry run path still runs alongside for shadow comparison.

Environment variables (including `POLY_PRIVATE_KEY` etc.) are loaded from `.env` via `python-dotenv` (`load_dotenv()` in `config/settings.py`).

---

## Target Bot

```
Wallet: 0x2005d16a84ceefa912d4e380cd32e7ff827875ea
Chain:  Polygon
```

Known trading patterns:

| Sport | Volume Share | Strategy |
|-------|-------------|----------|
| Esports (LoL, CS2) | 40-85% | Large orders at 0.40-0.47, high-frequency, trades both sides |
| Soccer | 15-45% | Smaller orders, "No"-heavy, odds-based |
| Other (Tennis, Basketball) | 5-15% | Opportunistic |

---

## Architecture

```
main.py
  └── IntelCopyBot (src/orchestrator.py)
        ├── WalletMonitor (src/core/wallet_monitor.py)
        │     └── Alchemy WebSocket with topic filtering + key rotation
        │
        ├── TradeParser (src/core/trade_parser.py)
        │     └── Decodes CTF Exchange ABI (OrderFilled events)
        │
        ├── PolymarketClient (src/enrichment/polymarket.py)
        │     └── token_id → market title/outcomes/dates + order book snapshot
        │
        ├── UniversalOddsService (src/enrichment/universal_odds.py)
        │     └── Search-first odds enrichment for any sport
        │
        ├── SoccerOddsClient (src/enrichment/soccer_odds.py)
        │     └── TheOddsAPI client with key rotation
        │
        ├── EsportsClient (src/enrichment/esports.py)
        │     └── PandaScore match info (free tier: no live game state)
        │
        ├── MarketClassifier (src/enrichment/market_classifier.py)
        │     └── Market title parsing, sport classification, team name normalization
        │
        ├── OutcomeResolver (src/enrichment/outcome_resolver.py)
        │     └── Polls Polymarket for market resolution, records win/loss
        │
        ├── MatchAuditRecorder (src/match_audit/)
        │     └── Diagnostic table `match_audit` — one row per rn1 trade at find_match resolution
        │
        ├── RefreshLogRecorder (src/refresh_log/)
        │     └── Diagnostic table `theodds_refresh_log` — one row per sport_key per refresh_events() cycle
        │
        ├── ConsensusOutlierRecorder (src/consensus_outlier_log/)
        │     └── Diagnostic table `consensus_outlier_log` — one row per consensus extraction that fires an outlier trigger
        │
        ├── OddsRawLogRecorder (src/odds_raw_log/)
        │     └── Diagnostic table `theodds_raw_log` — one row per league poll, full TheOddsAPI payload
        │
        └── Database (src/storage/database.py)
              └── PostgreSQL via SQLAlchemy async + asyncpg

  Dry Run Engine (src/dryrun/)
        ├── engine.py           # Orchestrator — real-time order creation + background tasks
        ├── scanner.py          # BACKUP: catches markets the real-time path missed + live dispatch via _dispatch_to_live()
        ├── consensus_odds.py   # Multi-bookmaker consensus: vig removal, staleness detection
        ├── bid_calculator.py   # v4 dual-regime: spread capture (equal shares) + directional (full budget heavy)
        ├── fill_simulator.py   # 3-tier fill simulation (optimistic/realistic/conservative) + self-fill on creation
        ├── position_tracker.py # Aggregates fills into positions, tracks portfolio (spread/directional regime)
        ├── live_odds_poller.py # Per-league consensus polling, frozen odds detection, contrarian flip (cancel-replace on direction change)
        ├── price_poller.py     # Dual-source Polymarket price feed (IntelBot trades + API backup with token inversion)
        ├── resolution_watcher.py # Settles positions, dual-regime P&L (spread pairs vs directional bets)
        ├── reporter.py         # Dual-regime reports: pair rate (spread) + win rate (directional)
        └── models.py           # ORM models (dryrun_orders, dryrun_orders_no_logic, dryrun_positions, dryrun_snapshots)

  Live Execution Layer (src/execution/)
        ├── executor.py         # Async wrapper around py-clob-client-v2 (sync SDK → ThreadPoolExecutor)
        ├── risk_manager.py     # 9-gate pre-trade checks + circuit breaker (HALT)
        ├── order_tracker.py    # Order state machine + fill tracking + reconciliation + V2 fee accounting
        ├── user_ws.py          # Polymarket User Channel WebSocket (real-time fills, forwards V2 fee field)
        ├── manager.py          # Central coordinator: regime dispatch, safe cancel-replace, shutdown
        ├── observability.py    # Process-wide counter registry, heartbeat loop, order-id log tag helper
        └── models.py           # ORM models (live_orders, live_positions, live_audit_log)

  Live Storage Adapter (src/storage/)
        └── live_store.py       # DB adapter implementing OrderStore + RiskDataStore protocols
```

---

## CLI Commands

Run modes (mutually exclusive — pick one):

```bash
python main.py                    # Monitor only (+ auto-resolves outcomes every 5min)
python main.py --dryrun           # Monitor + paper-trading engine
python main.py --live             # Monitor + dryrun shadow + real CLOB orders
```

Non-obvious / load-bearing commands:

```bash
python main.py --backfill 100              # Backfill last N blocks, then start monitoring
python main.py --validate [hours]          # Validate Neg Risk fix — BUY/SELL ratio + source_exchange breakdown (default 2h)
python main.py --resolve                   # One-shot outcome resolution (no bot needed)
python main.py --dryrun-resolve            # One-shot dryrun position resolution (no bot needed)
python main.py --dryrun-reset              # Wipe all dryrun tables (fresh paper-trading start)
python main.py --live-cancel-all           # Emergency: cancel all open CLOB orders (no bot needed)
python main.py --match-audit-export [F]    # Export match_audit diagnostic table to CSV
python main.py --refresh-log-export [F]    # Export theodds_refresh_log diagnostic table
python main.py --consensus-outlier-export [F]  # Export consensus_outlier_log diagnostic table
python main.py --odds-raw-export [F]       # Export theodds_raw_log (full per-fetch payloads) to CSV
python main.py --dryrun-no-logic-export [F]  # Export dryrun_orders_no_logic counterfactual table
python check_flip_residual.py              # Audit contrarian-flip rows for edge==cp-p-spread (Bug 2)
python scripts/backfill_league_labels.py   # Idempotent rewrite of soccer_odds_states.league to canonical form (--dry-run)
python scripts/migrate_wallet_to_v2.py     # Idempotent V2 wallet setup — wraps USDC.e -> pUSD, approves V2 exchanges + CTF setApprovalForAll
python scripts/diagnose_v2_receipt.py TX   # Dumps a V2 OrderFilled receipt's logs/topics/data for ABI debugging
python main.py --log-level DEBUG           # DEBUG/INFO/WARNING/ERROR
```

Routine read-only commands (`--stats`, `--api-usage`, `--export`, `--dryrun-report` / `-orders` / `-positions` / `-config` / `-export`, `--live-status` / `-report` / `-orders` / `-positions` / `-export`) — see `python main.py --help`. Standalone scripts: `debug_resolution.py` (CLOB/Gamma resolution check), `test_cache.py` (Polymarket token resolution), `scripts/round_trip_test.py` (connect → balance → place → cancel → verify against V2 SDK).

Schema migration: `migrations/v2_add_fee_columns.sql` adds `live_orders.fee_usdc`, `live_positions.total_fees_usdc`, `dryrun_orders.shadow_fee_usdc`. Apply with `psql -f migrations/v2_add_fee_columns.sql "$DATABASE_URL"`. Idempotent (`ADD COLUMN IF NOT EXISTS`). Fresh DBs created via `metadata.create_all` already have the columns; **existing databases must apply this migration before any post-V2 run** — the dryrun `shadow_fee_usdc` column is on the SQLAlchemy `DryRunOrder` model with a default value, so every dryrun INSERT references it and a missing column makes order creation fail in `--dryrun` as well as `--live`. The dryrun engine's auto-migration `_migrate_v2_columns` does not currently include `shadow_fee_usdc`; the SQL file is the operator's responsibility.

The dry run engine piggybacks on IntelBot's trade stream — it cannot run standalone. `--live` implies `--dryrun` (the dry run engine runs in shadow mode alongside live execution).

### Live-Mode Logging

`--live` activates quiet logging. Dryrun (`src.dryrun.*`), enrichment (`src.enrichment.*`), storage, and trade-parser INFO/DEBUG output is filtered from the **console** via a `logging.Filter` on the `StreamHandler` (`_ConsoleSilenceFilter` in `main.py`). WARNING and ERROR from those namespaces still reach the console; the namespaces are never silenced at the logger level, so the file handler keeps full INFO history. The orchestrator emits one concise line per detected trade with a `[PRE]`/`[LIVE]` tag based on match timing (end_date past, or match date ≤ today):

```
[LIVE] rn1 BUY $450.00 on "Arsenal vs Man Utd" (soccer) @ 0.65
   🔍 consensus match (5 bookmakers) home=0.58 away=0.42 | 380ms
```

Or when no consensus odds are available:

```
[PRE] rn1 BUY $200.00 on "Will X win?" (politics) @ 0.45
   ↳ no consensus odds (category_filtered)

[LIVE] rn1 BUY $18.15 on "Arsenal vs Man Utd: O/U 2.5" (soccer) @ 0.55
   ↳ no consensus odds (ou_disabled)

[LIVE] rn1 BUY $12.00 on "Spread: Arsenal (-1.5)" (soccer) @ 0.48
   ↳ no consensus odds (handicap_disabled)
```

A dedicated `live.engine` logger (separate from `src.dryrun.engine`) stays at INFO in live mode, providing diagnostic breadcrumbs at every decision point in the engine:

```
   ↳ edge: yes=+3.2% no=-3.2% → DIRECTIONAL (poly=0.5300, consensus_yes=0.5620)
   💰 LIVE DIRECTIONAL → submitting: Yes @ $0.5420, 36.9 shares (spread=1.0%)
```

Emoji scheme organized by pipeline stage:

**Wallet monitor** (`src.core.wallet_monitor`): 🎯 = target wallet detected via direct log match, 🎯🔥 = target wallet matched in transaction topic / full trade confirmed.

**Orchestrator** (`src.orchestrator`): 🔍 = consensus odds matched for this trade (with bookmaker count), ↳ = info/skip reason.

**Engine** (`live.engine`): 🚀 = directional regime detected (edge ≥ 2%), 💰 = live order submitted to CLOB, ↳ skip: = order creation skipped (with reason).

**Order tracker** (`src.execution.order_tracker`): 💵 = fill confirmed (partial or full, with size/price/spend).

**Direction flip group** (🔄 prefix — outcome changes Yes↔No): 🔄✅ = flip completed (CLOB cancel + new placement), 🔄🚫 = flip blocked (filled order on target slot), 🔄📏 = flip skipped (edge on new side too wide), 🔄🔑 = flip aborted (can't resolve token_id for new outcome), 🔄⚠️ = live flip failed (dryrun stays on old outcome for consistency), 🔄💥 = live flip exception, 🔄🗑️ = pre-place cleanup: stale order cancelled (side-flip or same-side replace), 🔄🏁 = pre-place cleanup: order filled before cancel completed (kept), or partially-filled same-side order preserved.

**Non-flip bid update group** (🪜 prefix — same direction, price change): 🪜✅ = bid repriced successfully, 🪜📏 = bid update skipped (edge too wide), 🪜💥 = live cancel-replace failed, 🪜⏭️ = cancel-replace skipped (price delta below minimum tick threshold), 🪜⏳ = cancel-replace skipped (per-market cooldown still active).

**Risk**: 🛡️❌ = order rejected by risk gates (any regime, any path).

**Execution lifecycle**: 📏 = sub-minimum order bumped to 5 shares, 💥📝 = initial order submission failed, 💥🗑️ = CLOB cancel API failed, 🏁 = old order already fully filled, 🕳️💀 = POSITION GAP (cancelled but replacement failed — needs manual attention), ❓ = order not found (stale reference), 🔁 = duplicate directional signal suppressed by dedup guard, ⚠️⚠️⚠️⚠️ = postOnly rejected (bid crossed spread, retrying), ⏰ = age-based re-validation triggered (order older than max age threshold).

**Cleanup** (🧹 prefix — cardinality and staleness enforcement): 🧹 = cardinality cleanup (duplicate orders cancelled, kept best-filled survivor), 🧹 = edge-gone cancel (live order cancelled because edge disappeared on re-evaluation), 🧹 = stale sweep (background sweeper cancelled order exceeding absolute max age), 🧹💥 = cleanup cancel failed (CLOB API error during any of the above).

**Shutdown/halt**: 🔌 = shutdown initiated, 🔌✅ = shutdown complete, 💥🔌 = shutdown cancel failed, 🚨🛑 = emergency halt (risk circuit breaker), 💥🛑 = halt cancel failed.

**Poll cycle health** (end of each LiveOddsPoller cycle): 💚 = all orders updated, 💛 = some updated, 🔴 = none updated.

**Safe-replace group** (🛡️ prefix — the submit-new-then-cancel-old cancel-replace path; see "Safe Cancel-Replace" in ExecutionManager): 🛡️🪜 = safe cancel-replace rejected by CLOB (old order preserved), 🛡️🗑️ = safe-flip CLOB cancel of old failed after new accepted (DB row left LIVE for reconciliation — state drift is worse than a false CANCELLED), 🛡️🔄 = safe-flip complete (old retired after new accepted) or reserved new order before submit, 🛡️❌ = risk rejected flip.

**Observability** (📊 prefix — `src.execution.observability`): 📊 = heartbeat snapshot of counters (totals + delta since last, emitted every 5 min from `heartbeat_loop`). Counter names appear in this emoji's log block.

Live execution logs (`src.execution.*`) remain at full `INFO` verbosity, including CLOB ACCEPTED/REJECTED status per order. With console filtering, poll cycle health (💚/💛/🔴) and non-flip success (🪜✅) are hidden from the terminal in `--live` mode but still captured in `logs/bot-YYYYMMDD.log`. Warning/error emojis (🔄📏, 🛡️❌, 🕳️💀, etc.) always appear on the console. Demoted logs are also recoverable on the console with `--log-level DEBUG`.

**File handler**: `main.py:setup_logging` also attaches a `logging.handlers.RotatingFileHandler` to the root logger at `logs/bot-YYYYMMDD.log`. UTF-8 encoded (so emojis render correctly in Notepad / VS Code / `findstr`, independent of Windows' cp1252 default code page that mangles them when redirecting stdout). Rotation: 50 MB per file, 5 backups (~250 MB rolling history). Captures EVERYTHING at INFO level including the dryrun/enrichment namespaces that are console-filtered — the console filter is a `logging.Filter` on the `StreamHandler` only, so the file handler is unaffected. `logs/` is gitignored.

The file handler has no console-silencing filter by design. Moving the namespace-silencing to the logger level (via `logger.setLevel(WARNING)`) would silence the file handler too — defeating the forensic-grep purpose.

---

## Contract Addresses (Polygon — Polymarket V2)

```
CTF Exchange:           0xE111180000d2663C0091e4f400237545B87B996B
Neg Risk CTF Exchange:  0xe2222d279d744050d28e00520010520000310F59
Conditional Tokens:     0x4D97DCd97eC945f40cF65F87097ACe5EA0476045   (unchanged across V1/V2)
Collateral (pUSD):      0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB
USDC.e (legacy):        0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174   (used only by the wrap script)
CollateralOnramp:       0x93070a847efEf7F70739046A929D47a521F5B8ee   (USDC.e -> pUSD wrap)
CollateralOfframp:      0x2957922Eb93258b93368531d39fAcCA3B4dC5854   (pUSD -> USDC.e unwrap, rollback path)
```

The V1 Fee Module / Neg Risk Fee Module contracts do not exist in V2 — fees are collected on-chain by the exchange itself and surfaced in the `fee` field of `OrderFilled`. Anything in the codebase reading those legacy addresses is dead.

**Guardrail — do not re-add Fee Module constants.** Two slots existed in `CONTRACTS` for `FEE_MODULE` and `NEG_RISK_FEE_MODULE`; both are deleted in V2. A future session that "restores them defensively" will route topic-filtered subscriptions to addresses with no live events, breaking the wallet monitor's CU savings.

---

## API Keys & Rotation

**Key counts are dynamic** — keys are regularly added and removed. All rotation logic uses `len(api_keys)` at runtime, never a hardcoded count. The numbers below are approximate at time of writing.

### Alchemy (Polygon RPC)
- Defined in `config/settings.py` (`ALCHEMY_API_KEYS`)
- `AlchemyKeyRotator` (singleton via `get_alchemy_rotator()`) rotates on 429/timeout with 60s cooldown

### TheOddsAPI
- Defined in `config/settings.py` (`THEODDS_API_KEYS`) as a fixed-length list of 551 `os.getenv("THEODDS_API_KEY_N", default)` slots. Slots 1–464 currently carry live keys; slots 465–551 are empty-string placeholders preserved so future key rotations can drop into slot `N` without editing code, and the `THEODDS_API_KEY_N` env-var override remains available at every position.
- Each key: 500 requests/month. Total monthly capacity = `len(live_keys) × 500`
- `APIKeyRotator` (in `soccer_odds.py`, imported by `universal_odds.py`) — `UniversalOddsService` and `SoccerOddsClient` each instantiate their own rotator over the same key list
- 401/403 → mark exhausted + rotate immediately (no retry count increment). Empty-string placeholder slots fail the first real call and are marked exhausted via the same path — no dedicated filter for blank slots
- 429 → exponential backoff (1s→2s→4s→8s), then force rotate
- `MAX_RETRIES = len(api_keys)` (total, not active count — prevents shrinking limit)
- `--api-usage` checks live quota via `/v4/sports` (free endpoint, reads `x-requests-remaining` header)
- **Preflight** (`APIKeyRotator.preflight(probe)`): on startup, probes keys 0..N-1 against `/v4/sports` (free) via `_preflight_probe()` on both `UniversalOddsService.initialize()` and `SoccerOddsClient.preflight()` (wired from `orchestrator.py`). Dead keys are marked exhausted and `current_key_index` advances to the first live key. Since slots 1–464 carry live keys the probe stops at the first slot; the 87 placeholder slots at the tail are never probed during normal operation. Without this, fresh sessions cascade through "Key N exhausted..." warnings at first-match time instead of at startup. Runs twice (once per rotator) — accepted cost since the probe endpoint is free. The probe must NOT route through `_get()` or it would itself trigger rotation and defeat the point. Note: `/v4/sports` is documented as free — an over-quota key still returns 200 here but returns 401 on the actual odds endpoints (`/events`, `/odds`), so preflight proves "key is authenticated" not "key has quota"; the latter is still discovered on first real call

### PandaScore (Esports) — 1 key
- Free tier: 1000 req/hour. Provides match info but **no live game state** (no kills/gold/towers)

### Polymarket — no key required
- CLOB API: ~3000 req/10min
- Token resolution: on startup, fetches all active markets → builds `token_id → market` cache in DB (`token_market_cache` table). Unknown tokens trigger re-fetch.

---

## Wallet Monitor

**File**: `src/core/wallet_monitor.py`

### Topic-Filtered WebSocket Subscriptions

Server-side filtering by Alchemy — only events involving the target wallet are delivered. Two subscriptions, both filtered to the V2 exchange contract addresses:

```python
topics: [None, target_padded]        # Sub A: target at topic[1]
topics: [None, None, target_padded]  # Sub B: target at topic[2]
```

Topic positions for the V2 exchange events:

| Event | topic[1] | topic[2] | topic[3] |
|---|---|---|---|
| `OrderFilled` | `orderHash` (32-byte hash) | `maker` | `taker` |
| `OrdersMatched` | `takerOrderHash` (hash) | `takerOrderMaker` | — |
| `FeeCharged` | `receiver` (Polymarket fee collector) | — | — |

**Sub B (load-bearing)** matches `topic[2] == target` — for `OrderFilled`, this is `target == maker`. rn1 places GTC limit orders so rn1 is always the maker; every rn1 trade comes through Sub B.

**Sub A (defensive)** matches `topic[1] == target` — for the events listed above, topic[1] is always a 32-byte hash, so Sub A is currently inert (a hash will not collide with a 32-byte-padded address). It is kept in case V2 ever introduces a new event with the bot wallet at topic[1].

Target-as-taker (topic[3] of OrderFilled) is intentionally not subscribed. rn1 doesn't take. The polling fallback (`_polling_monitor`) would catch any taker-side trade if rn1's behaviour ever changed.

Together these reduce CU consumption from thousands/hour to ~10-20/hour.

### Staleness Detection
WebSockets can silently die (Alchemy stops events while TCP stays alive):
```
MAX_CONSECUTIVE_TIMEOUTS = 60  # 60 x 30s = 30 minutes
# After 30 min with no events → ConnectionError → reconnect with key rotation
```
Conservative timeout because topic-filtered subscriptions naturally have long gaps.

### Producer-Consumer Architecture
Trade detection and enrichment are decoupled via a bounded `asyncio.Queue` (capacity: 500). The WebSocket listener pushes raw events into the queue (fast), and 8 concurrent enrichment workers pull and process in parallel. This prevents trade bursts (e.g., 1,700+ trades on a single match) from stalling the WebSocket listener. Queue depth warning triggers at 200+ pending items; depth is logged every 30s.

### Non-Blocking Web3 Calls
`get_block` and `get_transaction` are synchronous Web3 calls wrapped with `run_in_executor` to avoid blocking the async loop.

### Trade Parser Integration
`trade_parser` is passed as a callable `get_trade_parser` to always get the current Web3 instance after key rotation:
```python
get_trade_parser=lambda: TradeParser(self.wallet_monitor.web3)
```

---

## Trade Parser

**File**: `src/core/trade_parser.py`, `src/core/abi.py`

Decodes CTF Exchange `OrderFilled` events from V2 transaction logs. Extracts:
- `token_id`, `shares`, `price`, `side` (BUY/SELL), `usdc_amount`
- `maker_address`, `fee_amount` (V2 explicit pUSD fee — zero for maker fills, taker pays)
- `source_exchange`: "ctf" or "neg_risk" (determined by which contract emitted the event)

Unique constraint: `(tx_hash, token_id)` — a single transaction can fill multiple orders.

### V2 OrderFilled event shape

V2 emits `OrderFilled` with three indexed topics + seven non-indexed words (224 bytes data). The Solidity declaration in the V2 source:

```solidity
event OrderFilled(
    bytes32 indexed orderHash,
    address indexed maker,
    address indexed taker,
    uint8   side,                  // 0 = maker BUYing tokens, 1 = maker SELLing
    uint256 tokenId,               // outcome token (collateral leg implicit)
    uint256 makerAmountFilled,     // pUSD if side=BUY, shares if side=SELL
    uint256 takerAmountFilled,     // shares if side=BUY, pUSD if side=SELL
    uint256 fee,                   // pUSD fee charged at match time (taker only)
    bytes32 builder,               // builder code or zero (V2 liquidity-rewards attribution)
    bytes32 metadata               // hashed metadata or zero
);
// signature hash: 0xd543adfd945773f1a62f74f0ee55a5e3b9b1a28262980ba90b1a89f2ea84d8ee
```

V2 also emits two informational events from the same exchange contracts that the bot recognizes (so they don't pollute `decode_other` counts) but does not extract trades from:

```solidity
event OrdersMatched(
    bytes32 indexed takerOrderHash,
    address indexed takerOrderMaker,
    uint8   side,
    uint256 tokenId,
    uint256 makerAmountFilled,
    uint256 takerAmountFilled
);
// signature hash: 0x174b3811690657c217184f89418266767c87e4805d09680c39fc9c031c0cab7c

event FeeCharged(address indexed receiver, uint256 amount);
// signature hash: 0x55bb3cade9d43b798a4fe5ffdd05024b2d7870df53920673bfc7e68047cd0ab1
```

The V2 fee value is already in `OrderFilled.fee`, so subscribing to or extracting from `FeeCharged` would double-count. All three signature hashes are pinned in `tests/migration/test_v2_orderfilled_decode.py` against `keccak256(declaration)` — if any V2 declaration drifts, the test fails before production starts emitting "0 trades for tx ..." canary warnings.

**Guardrail — V2 OrderFilled is NOT byte-identical to V1.** The V2 migration spec claimed the event was byte-identical so the decoder could be reused unchanged. Production evidence after cutover proved otherwise: V1's `(makerAssetId, takerAssetId)` pair was replaced by an explicit `side` (uint8) plus a single `tokenId`, and V2 added trailing `builder` / `metadata` (bytes32 each). Data length grew from 5 non-indexed words (160 bytes) to 7 (224 bytes); topic[0] hash is different. The V1 ABI's `process_log` rejects every V2 log on shape mismatch. If a future migration claims event-byte-identical without runtime verification, treat that as a hypothesis to test, not a fact.

### Function-input decoder is intentionally a no-op under V2

`parse_transaction()` (which decodes calldata via the function-input ABI) short-circuits to `[]`. The V2 order struct dropped `taker` / `expiration` / `nonce` / `feeRateBps` and added `timestamp` / `metadata` / `builder`, so the V1 function selectors and struct shape are wrong for every V2 calldata. The orchestrator's two-pass design (`parse_transaction` then `parse_from_receipt`) catches this transparently — `parse_from_receipt` decodes the OrderFilled event log directly, which is the source of truth for fill amounts anyway. The `_parse_fill_order` / `_parse_fill_orders` / `_parse_match_orders` / `_extract_trade_from_order` helpers are kept in the file as a working template if V2 selectors are ever pinned and someone wants to revive the calldata-decoding fast path.

### Dual-Exchange Monitoring
Polymarket has two parallel exchanges: CTF Exchange and Neg Risk CTF Exchange. Both are monitored via topic-filtered WebSocket subscriptions. `EXCHANGE_ADDRESSES` in `trade_parser.py` is the lower-cased pair of V2 exchange addresses (read from `CONTRACTS` in `config/settings.py` so the constant tracks the source of truth).

### Double-Counting Filter
Per Paradigm research (Dec 2025), V1 emitted two sets of `OrderFilled` events per trade: a maker-focused real-trade event and a redundant taker-focused summary where the `taker` field was an exchange contract address. Whether V2 still emits the duplicate is uncertain — the exchange architecture changed — so the filter was relaxed to only fire when **both** conditions hold:

1. `taker` is one of the V2 exchange contract addresses (V1 summary fingerprint), AND
2. target is NOT the maker on this event.

A real maker-side trade always has target as maker, so the filter can only false-positive on the legacy summary event, never drop a real trade. Under V2 single-emission, target-as-maker events are kept regardless of the taker field.

### Side Detection (Wallet-Perspective)

V2's `side` (uint8, from the maker's perspective) directly answers "did the maker buy or sell tokens?" — `0` = maker BUYing tokens (sending pUSD, receiving tokens), `1` = maker SELLing tokens. The parser maps this to rn1's perspective by:

- If target is the **maker**, `target_side = side`.
- If target is the **taker**, `target_side = 1 - side` (flipped, because taker is opposite of maker).
- Events where target is neither maker nor taker are skipped.

For amount extraction:
- `side == BUY (0)`: `usdc_amount = makerAmountFilled`, `shares = takerAmountFilled`
- `side == SELL (1)`: `usdc_amount = takerAmountFilled`, `shares = makerAmountFilled`

The single `tokenId` field is the outcome token — there is no second `takerAssetId` to inspect (the collateral leg is implicit in V2).

**Guardrail — wallet-perspective is mandatory.** The original V1 implementation used taker-perspective (whoever was emitted as `taker` in the event), which inverted BUY/SELL whenever rn1 was the maker. Pre-fix: 95% SELL ratio in `bot_trades`. Post-fix: balanced. The V2 rewrite preserves the wallet-perspective invariant by checking `maker == target` first and inverting from there. Do not regress to "use the side from the event directly" — half of rn1's trades are taker-side counterpart events where that would silently invert.

### Failure-mode canary

When `parse_from_receipt` sees exchange logs but produces zero trades for a tx, it logs a single WARNING summarizing what happened: `decoded(OF=N, OM=N, FC=N, other=N) skipped(taker_is_exchange=N, target_not_involved=N)`. The `other` count is the canary — under healthy V2 operation it stays at zero because all three V2 exchange events are recognized. A non-zero `other` means a new event signature appeared on the exchange contracts that the ABI doesn't know about. Run `scripts/diagnose_v2_receipt.py <tx_hash>` to dump the receipt and identify the new signature.

---

## Enrichment Pipeline

### Category Filtering

Only soccer markets receive paid odds enrichment (`DRYRUN_CONFIG["categories"] = ["soccer"]`). Non-soccer trades (esports, basketball, tennis, etc.) still get stored in `bot_trades` and receive an order book snapshot, but skip the TheOddsAPI call to conserve API quota. The trade's `enrichment_status` is set to `"skipped"` and `data_quality` to `"category_filtered"`.

### Market-Type Toggles (O/U, Handicap)

Both toggles default to off and use the same 3-layer pattern. All three layers are necessary: filtering only at the engine layer still burns ~400ms + API quota per discarded trade; relying only on the `fetch_odds` markets filter still runs event matching and consensus extraction for trades that will be discarded.

1. **Orchestrator** (`_process_single_trade`): detect, then skip `_enrich_universal()` entirely when disabled. Sets `enrichment_status="skipped"` and `data_quality=<reason>`. Order book state is still captured.
2. **API markets-param narrowing**: prevents the relevant market type from being fetched even if layer 1 is bypassed (e.g., a non-target enrichment that happens to reference the same event).
3. **`DryRunEngine._create_orders_realtime()`**: safety net — skip order creation if the market somehow reaches the engine.

| Toggle | Env var (default) | Layer-1 detection | Layer-1 `data_quality` | Layer-2 change |
|---|---|---|---|---|
| Over/Under | `LIVE_ENABLE_OVER_UNDER` (`false`) | title regex `O/U\s+[\d.]+` in `_process_single_trade` | `ou_disabled` | `UniversalOddsService.fetch_odds` requests `["h2h", "spreads"]` instead of `["h2h", "spreads", "totals", "alternate_totals"]` |
| Handicap | `LIVE_ENABLE_HANDICAP` (`true`) | classifier output `parsed_market.market_type == "spread"` | `handicap_disabled` | `LiveOddsPoller._fetch_odds` includes/omits `"spreads"` in markets param |

Handicap detection uses the classifier's `market_type` rather than a fresh title regex because the classifier already did that work and handles soccer-vs-basketball routing — re-applying a regex in the orchestrator would duplicate logic and risk divergence.

### Market Classification (`src/enrichment/market_classifier.py`)

Regex patterns applied in specificity order:
1. Esports series/map/game — prefix-matched (`LoL:`, `Counter-Strike:`)
2. Soccer team win — `"Will X win on DATE?"`
3. Soccer/Basketball O/U — `"Team A vs. Team B: O/U 2.5"` (line ≤50 → soccer, line >50 → basketball)
4. Soccer draw — `"Will X vs Y end in a draw?"`
5. Basketball match — prefix-matched (`NBA:`, `NCAAB:`)
6. Spread — `"Spread: TEAM (±N)"` (checks soccer indicators to route soccer vs basketball)
7. Generic `vs` match — falls back to `_guess_category_from_teams()`

Markets without a date in the title (spreads, generic matches) get their date from the Polymarket `end_date` field as fallback.

The `Spread:` pattern deserves special attention: it matches any `Spread: TEAM (±N)` format regardless of sport. To prevent soccer clubs (e.g., "Wuhan San Zhen FC") from being misclassified as basketball, the spread handler checks for soccer indicators in the team name. If found, it returns `category="soccer"` with `market_type="spread"`, routing the trade into the soccer enrichment pipeline where single-team lookup finds the matching event.

`_guess_category_from_teams()` tokenises the lowercased combined team string on non-alphanumeric boundaries and tests set membership against two frozensets (`_SOCCER_TEAM_TOKENS`, `_ESPORTS_TEAM_TOKENS`). Whole-token match is required — not substring. "Oakland Athletics vs Seattle Mariners" (MLB) splits into `{oakland, athletics, seattle, mariners}` and does NOT match the soccer indicator `"athletic"`, so it resolves to `"other"` and is filtered out before `_enrich_universal` spends any API quota.

**Guardrail — token membership, not substring.** The earlier implementation used `if any(ind in combined for ind in soccer_indicators)` which caused "Oakland Athletics" to match `"athletic"` via substring and be tagged `soccer`. Under category filtering those false-soccer trades reached `find_match`, burned quota, and then failed with `vs_match_failed` — polluting the matcher audit and wasting enrichment cycles. Token-set membership is the fix; do not revert to substring `in`. A separate narrow false-positive remains: "Kansas City Royals" still has `"city"` as a full token and is tagged soccer by design — it is pinned by `test_kansas_city_still_misclassified_as_soccer` in `tests/enrichment/test_classifier_word_boundary.py` so any future tightening surfaces the regression explicitly.

### League label normalization (`market_classifier.normalize_league_label`)

`soccer_odds_states.league` receives writes from two enrichment paths that produce different shapes for the same league: the universal-odds path emits `"{sport_group}/{sport_title}"` (e.g. `"Soccer/Bundesliga"`, `"Soccer/Süper Lig"`, `"Soccer/Bundesliga - Germany"`) while the soccer_odds path emits the TheOddsAPI sport_key (e.g. `"soccer_germany_bundesliga"`). Without normalisation, league-level slices double-count because `"Bundesliga"` / `"bundesliga"` / `"soccer_germany_bundesliga"` survive as distinct rows.

`normalize_league_label(raw)` slugifies via NFKD + ASCII strip + non-alphanumeric collapse (Süper → super, `Soccer/Bundesliga - Germany` → `soccer_bundesliga_germany`), looks up `_LEAGUE_CANONICAL_MAP`, and falls back to stripping a leading sport-group token (`soccer_`, `basketball_`, `tennis_`, …) so the Group/Title and bare Title shapes collapse onto the same canonical entry. Unknown labels pass through unchanged and emit a single `unknown_league_label=<raw>` WARNING per session per slug — extend the map when warnings appear. Canonical form is lowercase snake_case ASCII (`premier_league`, `bundesliga`, `la_liga`, `ligue_1`, `champions_league`, …).

The normalisation is applied at the single chokepoint `Database.insert_soccer_odds_state` — every write of `soccer_odds_states.league` runs through `normalize_league_label` before insertion regardless of caller. `theodds_raw_log.sport_key` is intentionally NOT normalised: that table is a forensic capture of the raw API request shape.

`scripts/backfill_league_labels.py` rewrites existing rows to canonical form. `--dry-run` reports what would change without writing; idempotent — running it twice is a no-op.

**Guardrail — extend the map; do not relax the chokepoint.** The map is exhaustive enumeration, not a regex. New `<League> - <Country>` shapes that TheOddsAPI emits for top-5 leagues (e.g. `Bundesliga - Germany`, `Serie A - Italy`, `Ligue 1 - France`, `Primeira Liga - Portugal`) are listed under their canonical entry alongside the sport_key and the bare title. Adding a regex-based fallback that tries to "guess" canonical form from the slug would silently merge unrelated leagues whose names overlap. Extend the map; let the warn-once log surface unknown shapes; ship the new entries.

### Universal Odds (`src/enrichment/universal_odds.py`)
Search-first approach: instead of classifying sport then searching, it:
1. Extracts participant names from market title via regex patterns (draw, spread, BTTS, "vs", "Will X win", etc.)
2. Normalizes names: strips accents/diacritics via `_strip_accents()` (Almería → Almeria, Leganés → Leganes), strips prefixes (BV, VfB, FK, UD, CD, SD, CA, 1.) and suffixes (FC, SC, City, United), year numbers
3. Searches a pre-built participant index across ALL sports in TheOddsAPI
4. Falls back to word-part search if full name fails ("BV Borussia 09 Dortmund" → "borussia" → "dortmund")

The event index refreshes atomically in background (builds new dict, then single-assignment swap).

Accent stripping (`_strip_accents()`) is applied in `_normalize_name()`, `_has_word_overlap()`, `_word_overlap_count()`, and `_get_significant_words()` (the canonical tokenizer used by `_team_score` / `_will_win_score` / `_rebuild_word_frequency`) — without it, "Almería" ≠ "Almeria" silently fails all comparisons. Spanish club prefixes (UD, CD, SD, CA) are in the prefix-strip regex for the same reason. The function first translates atomic Nordic letters that NFKD cannot decompose (`ø → o`, `Ø → O`, `æ → ae`, `Æ → AE`), then runs `unicodedata.normalize('NFKD', ...)` to strip combining marks for the rest (ü, ö, ä, å, é, í, á). Without the atomic-letter pass, `"Brøndby"` stayed `"Brøndby"` after normalisation and failed word overlap with TheOddsAPI's `"Brondby"` — 100% failure on Danish Superliga clubs. Translation-style differences (`"København"` vs `"Copenhagen"`) are out of scope and require an alias map, which is not implemented.

**Guardrail — `_strip_accents(None)` returns `""`, `_get_significant_words(None)` returns `set()`.** TheOddsAPI sends `home_team: null` / `away_team: null` for tournament-shaped events (tennis tournament outright, MMA bouts, golf), and the deserialized `OddsEvent` carries `home_team=None` / `away_team=None`. `_rebuild_word_frequency()` walks every event in `self.events` unconditionally on every refresh cycle, so any None team would call `None.translate(...)` mid-init and crash the bot before it can take its first trade — observed in production as a fatal `'NoneType' object has no attribute 'translate'` immediately after "Initializing universal odds service…". The pre-existing matcher paths (`_has_word_overlap` from `find_match` candidate filtering) had the same latent crash for tennis/MMA/golf trades but never tripped because rn1 traded almost exclusively team-sport markets; the unconditional rebuild surfaced it. Both helpers now short-circuit on falsy input. Do not "tighten" the type signature to `text: str` (without `Optional`) and add a `if text is None: raise` guard — the runtime tolerance is load-bearing and pinned by `test_strip_accents_handles_none` / `test_get_significant_words_handles_none` / `test_rebuild_word_frequency_handles_null_team_names` in `tests/enrichment/test_match_scoring.py`.

**Strict matching**: `find_match()` uses word-overlap (4+ character words, case-insensitive, accent-insensitive) for the boolean correctness gate, then a rarity-weighted *information-fraction score* for ranking and threshold checks. No fuzzy matching, no fallbacks beyond word overlap:
- **VS markets**: Both teams must overlap with different sides of the event (boolean check via `_check_vs_match`). The matched event is then scored by `_vs_match_score(team_a, team_b, event)` — sum of per-team `_team_score` against home/away (max). Below `_MIN_VS_MATCH_SCORE = 1.00` → `parse_stage_reached="vs_match_below_threshold"`. Above → confidence 0.95.
- **Will-win markets**: Single team, candidate events filtered by word-overlap, then ranked by `_will_win_score(query_team, event)`. The top score must clear `_MIN_WILL_WIN_SCORE = 0.50` (else `parse_stage_reached="will_win_below_threshold"`) AND be a strict top under `_TIEBREAKER_EPSILON = 0.01` (else `will_win_ambiguous`). Confidence buckets off the top score and its margin over the runner-up: `≥0.85 + margin ≥0.30` → high (0.90, `strict_will_win`), `≥0.65 + margin ≥0.20` → medium (0.85, `strict_will_win_tiebreak`), else low (0.80, `strict_will_win_low`).
- **Fallback**: All extracted participants tried via word overlap, then gated on the same score thresholds. Confidence 0.80–0.85.

### Match scoring

For each significant 4+ char word `w` produced by `_get_significant_words(name)`, `_word_score(w) = 1 / log(1 + freq[w])` where `freq[w]` is how many distinct events in the current `self.events` index contain that word. The frequency table `self._word_frequency` is rebuilt by `_rebuild_word_frequency()` at the end of every `refresh_events()` cycle, immediately after the atomic swap of `self.events`. Words absent from the table default to `freq=1`, giving them maximum information value (correct: a query-only novel term is maximally rare).

Per-team match score: `_team_score(query_team, event_words) = sum(word_score(w) for w in overlap) / sum(word_score(w) for w in query_words)`, bounded `[0, 1]`. The score is the fraction of the query's information value recovered by overlap — scale-invariant, so the threshold doesn't drift as the index grows or shrinks.

`_will_win_score(team, event)` calls `_team_score(team, home_words ∪ away_words)`; `_vs_match_score(team_a, team_b, event)` sums per-team scores, taking the max over home/away for each.

The score is exposed on `MatchResult.information_score` and persisted to `match_audit.information_score` (FLOAT, NULL when no candidate was scored). Calibration playbook: pull the audit, plot `information_score` distribution by `parse_stage_reached`, expect matched ≥0.65, rejected-below-threshold ≤0.40, very few in between.

**Guardrail — keep `_check_vs_match` as a boolean correctness gate, do not collapse it into the score.** The vs-match path requires the two query teams to overlap with *opposite sides* of the event (one with home, the other with away). `_vs_match_score` only sums per-team scores — it doesn't enforce side disjointness, so a query with both teams overlapping the same side would still produce a high combined score. `_check_vs_match` runs first and rejects same-side overlap; the score gate runs second and rejects generic-only overlap. Replacing the boolean with a score-only check would silently accept "Real Madrid vs Real Betis" when the query was "Real Madrid vs Some Other Real-prefixed Team", because both query teams would overlap on `real` plus their distinctive word with the same side.

**Guardrail — count is not score, score is not count.** Generic football vocabulary (`deportivo`, `city`, `united`, `real`, `athletic`, `racing`, `club`) appears in dozens of clubs across leagues. The previous count-based ranker (`_word_overlap_count`) accepted single-generic-word overlaps as strict tops, latching queries with missing distinctive words onto unrelated events from unpolled leagues — observed in production with live trades firing on the wrong event's odds. The information-fraction score makes generic-word overlaps contribute near-zero (`1/log(1+30) ≈ 0.29`) while distinctive overlaps contribute near-1, and `_MIN_WILL_WIN_SCORE = 0.50` rejects matches recovering less than half the query's information value. Do NOT add a fallback that re-introduces count-based ranking — the soft-penalty shape is load-bearing, and a hard fallback would silently re-open the false-positive class.

### Per-event odds fetch cache

`UniversalOddsService.fetch_odds(event)` is wrapped in a 10-second TTL cache (`_fetch_odds_cache`) keyed on `(event.id, tuple(sorted(markets)))`. Populated on successful API responses with a non-empty bookmakers list; failures (`data is None`) and empty-bookmaker responses are intentionally NOT cached so retries reach the API. On cache hit, increments the `fetch_odds_cache_hit` counter and returns the stored bookmakers list unchanged.

Module-level constants pin the behaviour: `_FETCH_ODDS_CACHE_TTL_S = 10.0`, `_FETCH_ODDS_CACHE_MAX = 500`. Eviction is FIFO on insert when full — Python dict insertion order provides the ordering. The cap is a long-session safeguard; the 10-second TTL does the real bounding.

**Purpose.** rn1's large limit orders partial-fill into 15–30 `OrderFilled` events over 30–90 seconds. Each partial re-enters `_enrich_universal → enrich_trade → fetch_odds` for the same matched TheOddsAPI event, producing ~20 identical `/odds` calls per single economic rn1 trade. The TTL collapses each burst to 2–3 API calls. The `fetch_odds_cache_hit` counter appears in the 5-minute heartbeat; counting it against the burst size is how savings are measured.

**Key includes `markets`.** `fetch_odds` accepts an optional `markets` list. When `markets=None` it falls back to the module-level default that varies with `LIVE_CONFIG.enable_over_under` (`["h2h", "spreads", "totals", "alternate_totals"]` vs `["h2h", "spreads"]`). The orchestrator narrows the list per classifier output before calling `enrich_trade(..., markets=...)` (see "Orchestrator market narrowing" below). Including the sorted-markets tuple in the cache key prevents a narrow-request caller from silently reading back a response cached under a wider market set (and the reverse). Two back-to-back calls with different `markets` lists are always two API calls.

### Orchestrator market narrowing

`_enrich_universal` consults `_narrow_markets_for(parsed.market_type)` (helper at the top of `src/orchestrator.py`) and passes the result to `UniversalOddsService.enrich_trade(..., markets=...)`, which forwards it through `fetch_odds(event, markets=...)`. The narrowing table is keyed on the classifier's output string (NOT the ORM label):

| `market_type` | `markets` passed to `fetch_odds` |
|---|---|
| `match_winner` | `["h2h"]` |
| `team_win` | `["h2h"]` |
| `draw` | `["h2h"]` |
| `spread` | `["h2h", "spreads"]` |
| `over_under` | `None` (use default — preserves alternate_totals correctness) |
| `series_winner`, `map_winner`, `game_winner`, `unknown`, anything else | `None` |

Narrowing is what activates the A-cache (poller handoff) for non-O/U trades — without it, the default markets list always includes `alternate_totals`, which trips the alt-line guard in `fetch_odds` and forces a per-event API call. O/U trades intentionally fall back to the default so the alt-line guard still catches them. The table is exhaustive: anything not listed maps to `None`.

**Guardrail — keep narrowing keyed off classifier output, not the ORM label.** The classifier's `market_type` and the engine/DB's `market_type` ORM column are two different namespaces (`match_winner` vs `moneyline`, etc.). Keying narrowing off the ORM label silently maps every soccer h2h trade into the `None` bucket, which keeps the alt_totals guard tripping and re-dormants the A-cache. The naive alternative is plausible-looking and would produce a passing build with `fetch_odds_poller_hit_h2h = 0` in production — there is no test that catches it because there is no soccer-categorised path that exercises the ORM label without also exercising the classifier label. If you refactor either side of the chain, verify both `match_winner` and `team_win` still narrow to `["h2h"]`.

**Related caches on the poller side.** `LiveOddsPoller.last_odds[event_id]` is a per-event consensus snapshot with two writers. The poll loop (`/sports/{sk}/odds` once per league every 15 s live / 300 s pre-game) writes the full dict: `home_prob`, `away_prob`, `draw_prob`, `home_team`, `away_team`, `timestamp` (probs + raw age), `bookmakers_used`, `bookmakers_excluded`, `totals`, `bookmakers_raw`, `is_live`. The enrichment path (`on_enrichment_result`, called from every rn1 enrichment) writes a probs-only update and **merges** into the prior entry: fresh `home_prob` / `away_prob` / `draw_prob` + `timestamp` overwrite, while `bookmakers_raw`, `home_team`, `away_team`, and `totals` are carried over from the prior poll write. On merge, `bookmakers_raw_timestamp` is set to the prior entry's `bookmakers_raw_timestamp` (or its `timestamp` if that field doesn't exist yet) so downstream readers can tell probs age from raw-bookmaker age. The per-event D cache above is distinct — it stores responses from the per-event endpoint (`/sports/{sk}/events/{id}/odds`) — but the two are deliberately bridged by the handoff described below, which depends on the merge preserving `bookmakers_raw`.

### Poller cache handoff (A)

`UniversalOddsService` accepts an optional `LiveOddsPoller` reference via `set_poller(poller)`, wired from `orchestrator.py` after `DryRunEngine.initialize()` when the poller exists. Monitoring-only mode (no `--dryrun` / no `--live`) does not construct the poller, so `set_poller` is never called and `fetch_odds` runs with D + API only. When set, `fetch_odds` consults the poller's cache before firing a per-event API call.

**Check order in `fetch_odds`**: D (10-second burst cache) → A (poller handoff) → API. D catches within-burst duplication; A catches cross-path duplication (poller already has fresh data for any event in a league it polls); the API call is the last resort.

**A-hit criteria** (`_read_poller_cache`):
- Poller returns a cached dict for `event.id` (`get_cached_odds(event.id)` non-None)
- Parseable ISO-8601 timestamp with wall-clock age ≤ 20 seconds (`_POLLER_CACHE_FRESHNESS_S`). The freshness gate prefers `bookmakers_raw_timestamp` (set on the merged-from-enrichment shape) and falls back to `timestamp` (set by every poll write). Poll-only entries have no `bookmakers_raw_timestamp`, so the fallback catches them; merged entries would otherwise be gated on the fresher probs' timestamp instead of the raw bookmaker list's age — which is the field about to be returned
- Non-empty `bookmakers_raw` list

On hit: populate the D cache slot for `(event.id, tuple(sorted(markets)))` with a fresh `time.monotonic()` timestamp so subsequent partial-fill reads within the 10-second D window skip the poller check entirely, tick `fetch_odds_poller_hit` and the market-type discriminator (`fetch_odds_poller_hit_h2h` if `"h2h"` is in `markets`, else `fetch_odds_poller_hit_spreads` if `"spreads"` is in `markets`, else `fetch_odds_poller_hit_other`), emit a `fetch_odds poller cache hit: event=... age=...s league=...` DEBUG line, and return the bookmakers list. The discriminator follows the same subset-not-replacement convention as `adopted_unknown_order_*` — the parent `fetch_odds_poller_hit` always ticks too. When both `"h2h"` and `"spreads"` are present (the spread-narrowing case), only `_h2h` ticks; `_spreads` is reserved for spreads-only requests so the heartbeat can still attribute mixed requests cleanly.

**Skip-on-alternate_totals**: when the caller's `markets` list includes `"alternate_totals"`, the poller cache is bypassed even on a fresh hit. The league endpoint that populates `bookmakers_raw` carries only h2h/spreads/totals; it does not carry `alternate_totals`. Serving poller data for an alt-line O/U request would silently return the wrong data.

**Guardrail — do not remove the `alternate_totals` skip.** The per-event endpoint `/sports/{sk}/events/{id}/odds` carries alt-line totals when requested; the league endpoint `/sports/{sk}/odds` does not. If the alt_totals gate is removed, any O/U market whose Polymarket line differs from Pinnacle's primary line will silently serve the wrong line's probabilities from the poller cache and miscompute edge. This is a correctness gate, not an optimisation knob.

**Guardrail — negative results are not cached.** If the poller has no entry for an event, or the entry is stale, or the timestamp fails to parse, `_read_poller_cache` returns `None` and control falls through to the API call. No "poller miss" marker is stored. The poller may populate the entry on its next 15-second tick; the next enrichment must find it.

**Guardrail — `_api_request` signature compatibility.** `fetch_odds` consumes `_api_request` via `data, _ = await self._api_request(...)`. Every internal `_api_request` caller uses the same 2-tuple unpacking. Do not flatten or re-shape the return type. Diagnostic-only per-call metadata (see refresh_log's `status_out` scratch-dict below) is an opt-in kwarg that leaves the tuple shape unchanged.

**Guardrail — `on_enrichment_result` must merge, not stomp.** The enrichment path fires on every rn1 trade that enriches successfully, typically seconds after a poll cycle populated the event. The naive "replace `last_odds[event_id]` with the fresh probs-only dict" silently deletes `bookmakers_raw`, `home_team`, `away_team`, and `totals` from the prior poll write. Three paths break at once when this happens: (a) the A-cache handoff above sees no `bookmakers_raw` and every subsequent rn1 trade on the event falls through to a per-event API call (observed in production as `fetch_odds_poller_hit = 0` across a 5-min live window), (b) `MarketScanner`'s handicap branch reads `cached["bookmakers_raw"]` / `home_team` / `away_team` and silently skips every scanned handicap market after the first enrichment, (c) `_trigger_bid_update`'s spread branch reads `bookmakers_raw` off the consensus dict and skips handicap bid updates triggered from the enrichment path. The fix is a merge that preserves the poll-derived fields and tags the preserved raw list with `bookmakers_raw_timestamp`. Do not revert to a stomp even if a refactor appears to simplify the call — all three downstream readers depend on the merge.

### Consensus Odds (`src/dryrun/consensus_odds.py`)

Odds-based decisions run through a multi-bookmaker consensus pipeline whose breadth is controlled by `LIVE_CONSENSUS_BOOKMAKERS`. Default is `pinnacle` (sharp-only); set to `*` to average across every trusted book or to a comma-separated list to mix. The consensus module:

1. Filters to 14 trusted bookmaker names: pinnacle, bet365, betfair_ex_eu, betfair_ex_uk, betfair, matchbook, draftkings, fanduel, williamhill_us, williamhill, paddypower, unibet_uk, unibet_eu, unibet. The active request list (`config.settings.BOOKMAKERS`) is a 9-key subset; the extra names are defensive accept-if-returned entries — alternate spellings (`betfair`, `unibet`, `williamhill_us`) and dead keys (`bet365`, `unibet_eu`) that the API no longer carries
2. Applies the **`LIVE_CONSENSUS_BOOKMAKERS` allowlist** (default `pinnacle` — set in `.env.example`). Set to a comma-separated list (e.g. `pinnacle,betfair_ex_uk`) to widen, or `*` to use every trusted book. The per-order `bookmakers_snapshot` and `theodds_raw_log` keep recording every bookmaker regardless, so the allowlist only changes which books drive the bid — historical per-bookmaker analysis stays available
3. Converts decimal odds to raw implied probabilities (`1/price`)
4. Removes vig by normalizing probabilities to sum to 1.0 per bookmaker
5. Averages across all non-excluded bookmakers (simple arithmetic mean, equal weight)
6. Renormalizes the final consensus

**Staleness detection (live matches only)**: `BookmakerTracker` monitors per-bookmaker odds across consecutive polls. If a bookmaker's odds are unchanged for `STALE_UNCHANGED_THRESHOLD` consecutive polls (default 4, ~60s at 15s interval; env-tunable via `LIVE_STALE_UNCHANGED_THRESHOLD`), it's marked stale and excluded from consensus. The transition from fresh→stale is **edge-triggered**: a per-bookmaker `excluded` flag on the tracker entry ensures one tick on `l1_excluded_<bookmaker>` per exclusion event, not one tick per excluded poll. This handles Pinnacle suspending h2h odds around goals while other bookmakers continue updating. The threshold must be high enough to avoid false positives from routine soccer pauses (goals, set pieces) where bookmakers legitimately hold odds for 30-60s; EU books on the widened us+uk+eu request cadence can hold 30-45s during quiet match periods.

The widened bookmaker set (see "Live Odds Polling & Bid Updates" below) raised the goldilocks point from 3 to 4. Tuning rules apply only when `LIVE_CONSENSUS_BOOKMAKERS` widens past Pinnacle-only — under the default Pinnacle-only consensus, EU books cannot exclude Pinnacle from a one-book average, so EU dominance in `l1_excluded_*` doesn't change consensus availability. If you widen back and post-deploy heartbeats show EU books dominating the L1 exclusion mix while Pinnacle exclusions stay rare, bump `LIVE_STALE_UNCHANGED_THRESHOLD=5` and `LIVE_FROZEN_UNCHANGED_THRESHOLD=8` to maintain L2 headroom. If Pinnacle exclusions stay rare and `edge_threshold_skip_update` rises during goal/VAR moments, drop `LIVE_STALE_UNCHANGED_THRESHOLD=3`.

Three convenience wrappers:
- `extract_consensus_h2h()`: Returns `{home, away, draw, bookmakers_used, bookmakers_excluded}`
- `extract_consensus_totals()`: Returns `{over, under, bookmakers_used, bookmakers_excluded}`
- `extract_consensus_handicap(bookmakers_data, handicap_line, team_name, ...)`: Returns `{cover, not_cover, bookmakers_used, handicap_line, handicap_line_match}`. Searches each bookmaker's `"spreads"` (and `"alternate_spreads"` if present) market for outcomes matching the exact team name and handicap line. Removes vig per-bookmaker, then averages across all non-excluded bookmakers. Returns `None` if no bookmaker offers the matching line.

A helper function `resolve_handicap_side(event, team_name, handicap_line)` maps the Polymarket title's team name to the correct side (home/away) of the TheOddsAPI event using word-overlap matching (same `_has_word_overlap` / `_strip_accents` functions used by `find_match()`). Returns `{side, api_team_name, handicap_line}` or `None` if neither side matches. This is needed because TheOddsAPI spread outcomes are keyed by team name, not by home/away index.

Each `extract_consensus_*` wrapper increments one `consensus_books_*` breadth bucket per successful extraction (`_le_3` / `_4_5` / `_6_7` / `_ge_8`), bucketed by `len(bookmakers_used)`. With the 9-book request set the steady-state distribution is `_ge_8` dominant (~55%) + `_6_7` (~35%); persistent `_le_3` (>10%) suggests either (a) the widening didn't take effect (regions or bookmakers param being stripped — debug the actual `_fetch_odds` request), or (b) one or more requested keys is dead at the API and the consensus filter has nothing to count (probe the API per the empirical-validation guardrail above). If all bookmakers are suspended, `calculate_consensus_prob()` returns `None`, ticks `l1_consensus_unavailable`, and the engine falls back to last-known cached odds.

### Soccer Odds (`src/enrichment/soccer_odds.py`)
TheOddsAPI client with key rotation. The canonical request shape lives in `config/settings.py`: `REGIONS = "us,uk,eu"` and `BOOKMAKERS = [pinnacle, betfair_ex_eu, betfair_ex_uk, matchbook, draftkings, fanduel, williamhill, paddypower, unibet_uk]` (9 books — three sharps anchor the consensus, six retail smooth it). Both endpoints — the per-event `/sports/{sk}/events/{id}/odds` (called from `UniversalOddsService.fetch_odds`) and the per-league `/sports/{sk}/odds` (called from `LiveOddsPoller._fetch_odds`) — request these regions and bookmakers, so the league poll and per-event fetch return shape-equivalent data. The poller honors env overrides `LIVE_POLLER_REGIONS` and `LIVE_POLLER_BOOKMAKERS` for rollback (see "Live Odds Polling & Bid Updates" below). `UniversalOddsService.fetch_odds()` requests up to 4 market types (h2h, spreads, totals, alternate_totals). The consensus filter in `src/dryrun/consensus_odds.py:BOOKMAKERS` is a superset of 14 names — it carries alternate spellings the API may return (`betfair` / `unibet` / `williamhill_us`) and two retired keys (`bet365`, `unibet_eu`) kept as defensive accept-if-returned entries. Two league lists in `config/settings.py` drive the enrichment scope: `SOCCER_LEAGUES` (35 entries, covering top-tier European leagues, major second tiers like EFL Championship / La Liga 2 / 2. Bundesliga, domestic cups for the big-5 countries, UEFA club competitions, Saudi Pro League, MLS, Liga MX, and the primary South American / Asian / Oceanian leagues) and `BASKETBALL_LEAGUES` (3 entries). The cross-sport event index built by `UniversalOddsService` walks `PRIORITY_SPORTS` (76 entries in `src/enrichment/universal_odds.py`), which is a superset spanning soccer, basketball, tennis, American football, baseball, hockey, MMA/boxing, and golf. When `LIVE_ENABLE_OVER_UNDER=false`, `UniversalOddsService.fetch_odds()` requests only `h2h` and `spreads` (omitting totals/alternate_totals). The `alternate_totals` endpoint is critical when O/U is enabled — Pinnacle's primary O/U line often differs from Polymarket's (e.g., Pinnacle says 2.5, Poly says 4.5), so we look up the exact matching line for accurate implied probabilities.

**Guardrail — `BOOKMAKERS` must be empirically validated, not derived from documentation.** TheOddsAPI silently drops requested bookmaker keys it doesn't carry — invalid keys yield zero rows in the response with no warning, no error, no rejection. Production observed two dead keys (`bet365`, `unibet_eu`) shipping in the 7-book request and silently returning 5 books per event for months — the heartbeat showed widening "took effect" because `regions=us,uk,eu` was honored, but breadth stayed at 5 because two of seven requested keys never arrived. Before adding or replacing any key in `BOOKMAKERS`, probe `/sports/{sk}/odds?regions=us,uk,eu&bookmakers=K1,K2,...` against EPL / Serie A / a long-tail league (e.g., Eliteserien) and confirm every requested key appears in `bookmakers[].key` for at least one event. Cross-check against the unfiltered `/sports/{sk}/odds?regions=...` response to find rename candidates — `unibet_eu` was renamed into regional variants (`unibet_uk`/`_fr`/`_nl`/`_se`), and there is no API-level signal that the old key is obsolete.

**Guardrail — `consensus_odds.BOOKMAKERS` must be a superset of `settings.BOOKMAKERS`.** The consensus filter (`if bm_key not in BOOKMAKERS: continue` at three sites in `consensus_odds.py` and inside `_extract_totals_by_line` in `live_odds_poller.py`) silently discards any bookmaker name not in the consensus whitelist before vig removal or averaging. Adding a key to `settings.BOOKMAKERS` without also adding it to the consensus whitelist routes API responses for that book straight into the bin — the widening looks like it took effect (the request goes out wider) but the parsed bookmaker count stays narrow. Whenever you update `settings.BOOKMAKERS`, verify the new keys exist in `consensus_odds.BOOKMAKERS`. Removing a key from `settings.BOOKMAKERS` does NOT require removing it from the consensus whitelist — leaving dead keys in the whitelist is harmless (no API responses to filter) and keeps the rollback path open if the key is ever re-added.

### Esports (`src/enrichment/esports.py`)
PandaScore integration for LoL and CS2. Free tier provides match info (status, teams, series score, tournament) but **no live game state** (kills, gold, towers, rounds). Schema has columns for these fields but they are NULL.

### Market State Capture (`src/enrichment/polymarket.py`)
At trade time, captures CLOB order book snapshot within 2 seconds:
- `best_bid`, `best_ask`, `spread`, `mid_price`
- `bid_depth`, `ask_depth` (volume at best prices)

Order book 404 errors (stale/delisted tokens) are logged at DEBUG level to avoid log spam. Non-404 HTTP errors log at WARNING.

### Outcome Resolution (`src/enrichment/outcome_resolver.py`)
Background loop (every 5 min) polls Polymarket CLOB/Gamma APIs for resolved markets. Records winning outcome and whether the bot's trade was correct. Also available via `--resolve`.

Resolution detection uses `closed`/`active` fields + `tokens[].winner` boolean from the CLOB API, with Gamma API as fallback (which has an explicit `resolved` field). The CLOB API itself has no `resolved` field — an earlier implementation checked for one that doesn't exist, resulting in zero resolutions detected across 25k trades.

### Event Matching & Data Quality

**Cross-sport validation**: `find_match()` rejects matches where the TheOddsAPI `sport_key` doesn't match the market's category (via `CATEGORY_TO_SPORT_PREFIXES` mapping). Without this, "Celtic" (Scottish soccer) would match a basketball team named "Celtics".

**O/U line matching**: The Polymarket O/U line (e.g., "O/U 4.5") is extracted from the market title and compared against Pinnacle's totals line. If they differ, alternate totals are checked. The `ou_line_match` boolean flag marks whether the lines align — mismatched lines mean implied probabilities are for the wrong line.

**Odds freshness**: `pinnacle_last_update` from TheOddsAPI tracks when Pinnacle last updated their odds. `pinnacle_odds_age_seconds` measures staleness. Classified as `fresh` (<60s), `aging_live` (60-300s during live), `stale_live` (>300s during live), or `stale_pregame` (>3600s pre-game). Additional: `stale_backfill` if `capture_latency_ms > 30s`.

**Match confidence**: Each event match gets a confidence score (`high`/`medium`/`low`/`rejected`) based on:
- Match score (word overlap) and margin over second-best candidate
- Price-probability distance: `min(|price - home_prob|, |price - (1-home_prob)|, |price - away_prob|, ...)`
- High: score≥80, margin≥30, ppd<0.25 | Medium: score≥60, margin≥15, ppd<0.35 | Rejected: ppd>0.40

---

## Data Captured Per Trade

| Category | Fields |
|----------|--------|
| **Trade** | tx_hash, block, timestamp, token_id, side, shares, price, usdc_amount, source_exchange |
| **Market** | title, slug, category, outcome_name, outcome_index, end_date |
| **Order Book** | best_bid/ask (both outcomes), spread, mid_price, bid/ask_depth |
| **Odds (9 bookmakers requested; consensus filter tolerates 14 names)** | Requested: pinnacle, betfair_ex_eu, betfair_ex_uk, matchbook, draftkings, fanduel, williamhill, paddypower, unibet_uk. Consensus filter also passes: betfair, williamhill_us, unibet (alternate spellings) and bet365, unibet_eu (retired keys, kept as defensive accept-if-returned). |
| **Odds (up to 4 market types)** | h2h (1X2), spreads (handicap), totals (O/U), alternate_totals — totals/alternate_totals omitted when `LIVE_ENABLE_OVER_UNDER=false` |
| **Implied Probs** | home, draw, away (consensus: vig-adjusted average across all bookmakers, Pinnacle-only fallback) |
| **Consensus Metadata** | bookmakers_used count, bookmakers_excluded list, consensus method |
| **Esports** | match status, teams, series score, tournament (no live game state on free tier) |
| **Enrichment Status** | `success` (full odds), `skipped` (category filtered, historical, O/U disabled, or handicap disabled), `partial` (unknown esport) |
| **Data Quality** | `complete`, `backfill_snapshot` (>60s latency), `category_filtered`, `ou_disabled`, `handicap_disabled`, `historical_no_snapshot` |
| **Quality Metrics** | capture_latency_ms, market_resolve_ms, match_find_ms, odds_fetch_ms, total_process_ms |
| **is_live_trade** | Derived at export time: `block_timestamp > commence_time` or PandaScore `match_status == "running"` |

---

## Database Schema

**File**: `src/storage/models.py`

| Table | Purpose |
|-------|---------|
| `bot_trades` | Every detected trade with full tx details + quality metrics |
| `market_snapshots` | Polymarket order book state at trade time |
| `soccer_odds_states` | Per-bookmaker JSONB columns (`odds_pinnacle`, `odds_bet365`, `odds_betfair`, `odds_draftkings`, `odds_fanduel`, `odds_williamhill`, `odds_unibet`) + consensus implied probs + freshness tracking. Columns predate the widened request list and stay as-is — books not in `settings.BOOKMAKERS` (e.g., `bet365`) simply receive `NULL`; new books in the request list (`betfair_ex_uk`, `matchbook`, `paddypower`, `unibet_uk`) currently have no dedicated column and aggregate into the consensus row only. The `league` column is normalized to a canonical lowercase snake_case ASCII form by `Database.insert_soccer_odds_state` via `market_classifier.normalize_league_label` regardless of caller (see Bug 3 §"Enrichment Pipeline → League label normalization") |
| `lol_game_states` | LoL match info (mostly NULL on free tier) |
| `cs2_game_states` | CS2 match info (mostly NULL on free tier) |
| `market_outcomes` | Resolved market outcomes for P&L |
| `token_market_cache` | token_id → market metadata mapping. `get_token_by_condition_outcome(condition_id, outcome_name)` resolves the token_id for a specific outcome — used during direction flips to find the new outcome's token |
| `theodds_api_usage` | Per-key monthly usage tracking |

Dry run tables (in `src/dryrun/models.py`):

| Table | Purpose |
|-------|---------|
| `dryrun_orders` | Every simulated order — includes market_type (`moneyline`/`over_under`/`spread`/`draw`), ou_line, handicap_line, handicap_team, handicap_line_match, side_role (heavy/spread), spread_level, poly_price_at_creation, consensus_prob, consensus_bookmakers_used, consensus_fair, edge, match_confidence, pinnacle_last_update, bid_updates count, shadow allocations (sym/kelly shares+usd for directional; zero for spread), shadow P&L, `bookmakers_snapshot` (per-bookmaker odds at creation), `shadow_fee_usdc` (column declared in the model — currently always 0; no code populates it. Reserved for a future synthetic-fee write that mirrors V2's taker-only fee model so shadow vs live P&L stays apples-to-apples — see Known Limitations #32) |
| `dryrun_orders_no_logic` | Counterfactual mirror of `dryrun_orders`. Every creation in the regular table writes a parallel row here via `insert_dryrun_order_pair`. Same bid/shares/regime/snapshot at creation time. **No nursing** — `_trigger_bid_update` does not query this table, so no edge recompute, no edge-gone cancel, no contrarian flip ever fires on these rows. Fills run normally (PricePoller iterates a parallel pass on every rn1 trade and order-book snapshot). Resolution runs per-order (no `dryrun_positions` rollup): `outcome_won` = `outcome_name == winning_outcome`; `pnl` = `shares × $1 − bid_size_usd` if won, `-bid_size_usd` if lost. Exported via `--dryrun-no-logic-export`. Unique-slot index mirrors the regular table (`uq_dryrun_orders_no_logic_active_slot`) |
| `dryrun_positions` | Aggregated position per market — includes heavy_side (Yes/No for directional, NULL for spread), commence_time, market_type, P&L decomposition (spread_pnl, directional_pnl, shadow_sym_pnl, shadow_kelly_pnl) |
| `dryrun_snapshots` | Price snapshots for fill simulation. `get_latest_dryrun_snapshot(token_id)` returns the most recent snapshot (mid_price, best_bid, best_ask, last_trade_price, captured_at) — used by `_trigger_bid_update()` to source fresh poly prices for bid recalculation |

Live execution tables (in `src/execution/models.py`):

| Table | Purpose |
|-------|---------|
| `live_orders` | Every real order sent to the CLOB — state machine (CREATED→SUBMITTED→LIVE→PARTIAL_FILL→FILLED→SETTLED), fill tracking (`size_filled`, `avg_fill_price`, `usdc_spent` (gross size × price), `fee_usdc` (V2 explicit pUSD fee, accumulated across fills, zero for maker fills)), strategy metadata (regime, side_role, spread_level, consensus_prob, edge), cancel chain (replaced_by FK), error tracking, `bookmakers_snapshot` JSONB column capturing per-bookmaker odds for the order's specific market at order-open time (filtered by `ou_line` / `handicap_line` / `handicap_team` so only the traded outcome's odds are captured; populated from `LiveOddsPoller.last_odds[event_id]["bookmakers_raw"]`). The snapshot is unaffected by the `LIVE_CONSENSUS_BOOKMAKERS` allowlist — every trusted bookmaker the API returned is recorded for forensic export, even when consensus is Pinnacle-only. **Captured on initial reserve only**: cancel-replace child rows (same-side reprice, contrarian flip) currently land with `bookmakers_snapshot = NULL` because `_safe_update_bid` / `_cancel_and_place_flipped_inner` / `cancel_replace`'s `new_order_data` builder do not thread the field — see Known Limitations #26 |
| `live_positions` | Aggregated position per market built from confirmed fills — yes/no shares, avg prices, usdc spent, `total_fees_usdc` (cumulative pUSD fees paid into this market across all fills; true cost = `yes_usdc_spent + no_usdc_spent + total_fees_usdc`), P&L fields (populated on settlement), regime, heavy_side |
| `live_audit_log` | Append-only event trail (JSONB details) for every execution-layer event: order lifecycle, fills, cancellations, halts, reconciliation, shutdown |

---

## Dry Run Engine

**Files**: `src/dryrun/` | **Config**: `config/dryrun_settings.py`

Simulates a dual-regime strategy on soccer markets (moneylines, over/under, and handicap spreads) using multi-bookmaker consensus odds:

1. **Real-Time Order Creation** (PRIMARY) — when rn1 trades, the enrichment pipeline fetches fresh odds from all bookmakers and computes consensus in parallel (~400ms). If the market is eligible (soccer, moneyline/O/U/handicap, high/medium confidence), the engine computes `max_edge = max(consensus_yes - poly_yes, consensus_no - poly_no)` and branches on whether it is below or above the `edge_threshold` (2%). Orders are created immediately via `DryRunEngine.on_enriched_trade()`.
2. **Scanner** (BACKUP) — runs every 5 min to catch markets the real-time path missed. Uses the same regime branch as the RT path.
3. **Bid Calculator** has two paths:
   - `calculate_spread_bids()` — spread regime: `bid = poly_price - 0.008`, equal shares on both sides
   - `calculate_directional_bids()` — directional regime: `bid = poly_price - spread`, full budget on heavy side only
   Both accept an optional `capital_override` parameter — used by live mode to size orders with live capital ($2-10) instead of dryrun capital (default $5)
   Consensus odds decide WHAT to bid on (edge direction, regime selection, heavy side). Polymarket price decides WHERE to bid (the actual limit price). Anchoring bids to consensus instead of poly crosses the book when the two diverge — which is the entire edge signal. For example: consensus=60%, poly=52%, spread=1% → consensus-anchored bid at 59¢ (above the market, instant adverse fill) vs poly-anchored bid at 51¢ (below the market, correct).
4. **Fill Simulator** determines if bids fill using 3 tiers:
   - Optimistic: fills if price touches bid
   - Realistic: fills if price is 0.5% below bid
   - Conservative: fills only if cumulative volume at/below bid exceeds order size × queue factor (3.0)
   - Guards: ignores trades with price ≤0.02 or ≥0.98 (null data, redemptions, resolved/resolving markets)
   - **Self-fill on creation**: When orders are created from an rn1 trade, the triggering trade is immediately replayed against the new orders. Without this, the trade that triggers order creation could never fill them (on_trade_captured fires before on_enriched_trade). For the complementary outcome, price is inverted (1 - trade_price).
5. **Position Tracker** aggregates fills, tracks portfolio, stores `commence_time` (parsed from ISO string if needed). Spread-regime positions have `heavy_side=NULL`; directional positions have `heavy_side=Yes/No`.
6. **Live Odds Poller** polls all bookmakers per-league (not per-market) to reduce API calls. Computes fresh consensus and updates bids for both regimes. Spread orders have no heavy/light roles. Directional orders perform contrarian flips (cancel-replace on the new outcome) when the heavy side changes — regardless of existing fills.
7. **Resolution Watcher** settles positions when markets resolve. Uses regime-specific P&L: spread positions decompose into paired-share profit and directional residual; directional positions use one-sided bet P&L.

Key config values (see `--dryrun-config`):
- Wallet: $1,000 (bookkeeping only — capital can go negative, never halts trading)
- Capital per market: $5 (default, env `CAPITAL_PER_MARKET`) | Max concurrent: 50 markets (only hard gate)
- Edge threshold: 2% (regime split point)
- Spread regime offset: 0.8% | Directional spread levels: [0.5%, 1.0%, 1.5%, 2.0%]
- Consensus poll: 15s live, 300s pre-game | Price poll: 30s
- Market types accepted by the engine: moneyline, draw, over/under, handicap. The `market_types` config key carries `["moneyline", "over_under"]` for legacy reasons but the gate in `_create_orders_realtime` is the per-type title regex (`is_moneyline` / `is_draw` / `is_ou` / `is_handicap`); any of the four passes. O/U: `LIVE_ENABLE_OVER_UNDER=false` (default: off). Handicap: `LIVE_ENABLE_HANDICAP=true` (default: on). Both use 3-layer toggle pattern — see "O/U Market Toggle" and "Handicap Market Toggle" in Enrichment Pipeline. Draws ride the existing h2h consensus path: title regex requires the literal `vs.` (with period) so participant extraction routes through `_parse_vs_market` → `_check_vs_match` (both teams must overlap), structurally avoiding the single-team-overlap false positives the will-win path is exposed to. Yes = match ends in a draw, No = it doesn't. | Max edge: 25% live / 8% pre-game (sanity gate)
- Longshot filter: directional bets rejected when consensus prob < `min_consensus_prob_directional` (default 0 — gate disabled; set `MIN_CONSENSUS_PROB_DIRECTIONAL=0.10` to re-enable)
- Order staleness: 3 min live / 15 min pre-game (`max_order_age_live_s` / `max_order_age_pregame_s`)
- Order book backup: 60s | Expiry: 7 days

The engine requires IntelBot to be running (`--dryrun` flag) — it uses IntelBot's trade stream as both the order creation trigger and the price feed for fill simulation. The real-time path (`on_enriched_trade`) creates orders ~400ms after rn1's trade using data already fetched during enrichment. Additionally, `on_enrichment_result()` feeds fresh consensus odds from each enrichment into the `LiveOddsPoller.last_odds` cache. The write is a **merge, not a replace**: fresh `home_prob` / `away_prob` / `draw_prob` / `timestamp` overwrite, but `bookmakers_raw`, `home_team`, `away_team`, and `totals` are carried over from the prior poll write (tagged with `bookmakers_raw_timestamp` so freshness gates downstream can distinguish probs age from raw age). This keeps the poller-cache handoff, the scanner's handicap branch, and enrichment-triggered handicap bid updates all reading the bookmaker list a prior poll populated — avoiding a per-event API call on every rn1 trade in a league already being polled. The scanner and pollers serve as backups.

**Order deduplication** relies solely on the DB partial unique index `uq_dryrun_orders_active_slot` on `(market_title, outcome_name, spread_level) WHERE status IN ('open', 'filled')`. `insert_dryrun_order()` catches `IntegrityError` on constraint violation and returns `None`, which the engine handles with `if order is None: continue`. No application-level dedup filters exist — the DB index is the single safety net. This is intentional: application-level checks (in-memory sets, pre-insertion DB lookups) are fragile across process restarts because stale rows from previous sessions can block all new orders for a market indefinitely.

`update_dryrun_order_bid()` also catches `IntegrityError`/`UniqueViolationError` and returns `False` on conflict (logged at DEBUG level). This silences the repeated ERROR-level log spam that occurs when the LiveOddsPoller tries to update a dryrun order's outcome during a flip and the target slot is already occupied by another active order. The dryrun update is quietly skipped; the live layer handles the flip independently.

**Frozen odds detection** operates at two layers:

- **L1 (per-bookmaker staleness)**: `BookmakerTracker` in `consensus_odds.py` excludes individual bookmakers unchanged for `STALE_UNCHANGED_THRESHOLD` polls (default 4, ~60s; env-tunable via `LIVE_STALE_UNCHANGED_THRESHOLD`). When all bookmakers are stale, `calculate_consensus_prob()` returns `None` and ticks `l1_consensus_unavailable`.
- **L2 (consensus frozen)**: `MatchOddsTracker` in `live_odds_poller.py` tracks consensus h2h, totals, and handicap independently. If any channel is unchanged for `FROZEN_UNCHANGED_THRESHOLD` consecutive polls (default 4, ~60s — matches L1; env-tunable via `LIVE_FROZEN_UNCHANGED_THRESHOLD`), that market type is marked frozen via `LiveOddsPoller.is_frozen(event_id, market_type)` where `market_type` is `"moneyline"`, `"over_under"`, or `"handicap"`. Movement tolerance is 0.5% (`FROZEN_MOVE_TOLERANCE = 0.005`, env-tunable via `LIVE_FROZEN_MOVE_TOLERANCE`) — small real consensus shifts (one bookmaker adjusting a tick) reset the counter. Edge-triggered: when a channel transitions from non-frozen to frozen, the corresponding `l2_frozen_h2h` / `l2_frozen_totals` / `l2_frozen_handicap` counter ticks once; subsequent polls with the channel still frozen do NOT re-tick. The handicap channel is fed from `_trigger_bid_update()` via `MatchOddsTracker.update_handicap(cover, not_cover)` after computing per-order handicap consensus — unlike h2h/totals (which are computed per-event in the poll loop), handicap consensus depends on the specific line and team from each order. Do not move handicap tracking into the poll loop's `_extract_consensus()` — that method doesn't know which handicap line or team to track (there could be multiple spread orders on the same event with different lines), so it would need to query active orders, defeating the purpose of per-event consensus extraction.

Under the current `LIVE_CONSENSUS_BOOKMAKERS=pinnacle` default, L2 is largely redundant: L1 excluding Pinnacle collapses consensus to `None`, which already gates new orders and bid updates upstream. L2 is therefore set to match L1 (4 polls / ~60s) so a frozen-but-still-returning Pinnacle case fires on the same timescale rather than lagging. **If you widen `LIVE_CONSENSUS_BOOKMAKERS` to multiple books, bump L2 back to ≥ L1 + 3 polls (e.g. 7) to preserve headroom** — without it, an L1 exclusion that nudges the multi-book average resets L2's counter and the layer never fires (cascading into permanent non-detection). The calibration playbook in §"Live Odds Polling & Bid Updates" assumes the multi-book regime.

**Frozen odds enforcement** gates order creation (both RT path and scanner) and bid updates (not fills). The poll loop itself does **not** skip frozen events (it logs but continues) — this avoids redundancy with the `None`-consensus check that already skips events when all bookmakers are truly suspended. The per-gate frozen checks in the engine RT path, scanner, and bid updates protect order creation where it matters. Frozen markets skip order creation and bid recalculation, but existing orders continue to check fills against Polymarket's actual CLOB prices. Pre-game events are never marked frozen (slow movement is expected).

### Market Types

**Moneyline (ML)**: "Will X win on date?" — uses consensus h2h implied probabilities from all bookmakers. If h2h odds are suspended during a live match (detected by `BookmakerTracker`), stale bookmakers are excluded from consensus. Falls back to the LiveOddsPoller's cached last-known h2h odds if all bookmakers are suspended.

**Over/Under (OU)**: "Team A vs. Team B: O/U 2.5" — extracts the line from the market title via regex, uses consensus `totals` (or `alternate_totals` for non-standard lines) to get over/under implied probabilities. Yes = Over, No = Under.

**Handicap/Spread (HC)**: "Spread: Arsenal (-1.5)" — single-team markets classified via soccer indicators in the team name. Uses single-team lookup to find the matching event, then enriches with handicap-specific consensus odds via `resolve_handicap_side()` + `extract_consensus_handicap()`. Yes = team covers the spread, No = team doesn't cover. The consensus extraction searches bookmaker `"spreads"` markets for outcomes matching the exact team name and handicap line, removes vig, and averages across bookmakers. If no bookmaker offers the matching line, `handicap_line_match=False` and the market is skipped. Can be enabled/disabled via `LIVE_ENABLE_HANDICAP` (default: on).

Handicap consensus flows to the engine via the `h2h_implied` parameter of `on_enriched_trade()` — the orchestrator substitutes `handicap_implied` (with keys `cover`, `not_cover`, `handicap_line`, `handicap_team`, `handicap_line_match`) into `h2h_implied` when `has_handicap` is true. The engine detects handicap markets by title regex (`is_handicap`) and reads `cover`/`not_cover` from the dict instead of `home`/`away`. Do not add a separate `handicap_implied` parameter to `on_enriched_trade()` — the engine's `_create_orders_realtime()` already re-detects market type from the title, and the parameter reuse avoids changing the method signature across the chain (`on_enriched_trade` → `_create_orders_realtime` → `_create_orders_inner`).

The codebase uses "spread" in three unrelated senses. All new code uses `handicap` or `spread_market` in variable names to avoid confusion:

| Term | Meaning | Where used |
|------|---------|------------|
| **Spread regime** | Trading regime where `max_edge < edge_threshold` (2%). Equal shares on both sides. `side_role="spread"` | `engine.py`, `bid_calculator.py`, `reporter.py` |
| **Spread level** | Price offset from Polymarket price for directional bids. `[0.5%, 1.0%, 1.5%, 2.0%]` | `bid_calculator.py`, `dryrun_orders.spread_level` |
| **Spread market** | Polymarket handicap market: `"Spread: Team (±N)"`. `market_type="spread"` | `market_classifier.py`, handicap pipeline |

### Capital & Position Sizing

**Capital model**: The wallet balance is tracked for reporting but **never gates trading**. The bot will keep placing orders even if the wallet goes deeply negative. The only hard constraint is `max_concurrent_markets` — an operator-tunable cap on how many distinct markets can hold non-terminal orders simultaneously. Once a position resolves, its slot frees up regardless of P&L.

**Operator-tunable risk knobs (live trading)**: Three values are routinely changed between sessions by editing `.env`; the spec quotes them by role, not by current default, because the defaults are starting points, not architectural invariants. All three are read by `config/live_settings.py` via `_env_int` / `_env_float` and can be changed without code edits:

| Role | Config key | Env var | Enforcement site |
|---|---|---|---|
| How many distinct markets may hold non-terminal orders at once | `max_concurrent_markets` | `LIVE_MAX_CONCURRENT_MARKETS` | `risk_manager.py:144` (gate 4 in `check_order`), counted by `count_active_markets()` over `live_orders.condition_id` |
| Bet size target — the USDC budget the engine sizes each new order against | `capital_per_market` | `LIVE_CAPITAL_PER_MARKET` | `engine.py` order-creation path (`min(dryrun_budget, LIVE_CONFIG["capital_per_market"])`), plus equal-share math `N = capital_per_market / (yes_bid + no_bid)` |
| Max total USDC exposure permitted per market (effectively caps bets-per-market = `max_per_market_usd / capital_per_market`) | `max_per_market_usd` | `LIVE_MAX_PER_MARKET_USD` | `risk_manager.py:151` (gate 5), uses committed cost `price * size` across all non-terminal orders, not post-fill `usdc_spent` |

Changes to these take effect on the next process start — they are read once into `LIVE_CONFIG` at import. A running bot does not pick up `.env` edits until restarted. Any numeric defaults quoted elsewhere in this spec are for orientation only; the live values are whatever `LIVE_*` env vars resolve to at startup.

**Dual-regime sizing**: The engine splits markets into two regimes at order creation based on `max_edge = max(edge_yes, edge_no)` vs `edge_threshold` (default 2%):

**Spread regime** (max_edge < 2%):
1. Compute edge for each side: `edge = consensus_prob - poly_price`
2. Bid price for both sides: `poly_price - spread_regime_offset (0.008)`, clamped to [0.05, 0.95]
3. Equal shares: `N = capital_per_market / (yes_bid + no_bid)`, same N shares on both sides
4. Orders: 2 per market (one Yes, one No), both `side_role="spread"`, single offset
5. Profit comes from combined cost < $1.00 per pair, not from predicting direction

**Directional regime** (max_edge ≥ 2%):
1. Compute edge for each side: `edge = consensus_prob - poly_price`
2. Heavy side = outcome with larger edge
3. **Longshot filter** (disabled by default; threshold = 0): if the heavy side's consensus probability < `min_consensus_prob_directional`, the order is rejected. Set `MIN_CONSENSUS_PROB_DIRECTIONAL` to a positive fraction (e.g. 0.10) to re-enable. When active, this prevents full-size directional bets on near-certain losers where the "edge" is technically positive but the absolute probability is in the gutter — a 3% edge at 4% consensus gets the same capital as a 3% edge at 55% consensus without this filter.
4. Full budget (`capital_per_market`, default $5) on heavy side only, no light-side order
5. Bid price: `poly_price - spread`, clamped to [0.05, 0.95]
6. Orders: 1 per spread level (4 levels), all on the heavy outcome, all `side_role="heavy"`
7. Max edge sanity check, split by match state: if any edge > 25% during live matches or > 8% pre-game, orders are skipped

**Shadow allocations** (directional regime only) are computed alongside every order for counterfactual comparison (never executed, stored in DB):
- **Symmetric (50/50)**: equal allocation to both sides — baseline spread-capture strategy
- **Half-Kelly**: `kelly_fraction = 0.5 × edge / (1 - poly_price)`, heavy% = clamp(0.50 + kelly_fraction, 0.50, 0.99) — edge-scaled directional

Shadow values are set to 0 for spread-regime orders (no directional component to compare against).

### Signal Model & Order Lifecycle

A **signal** is a computed edge: the difference between the Polymarket price and the external consensus odds (TheOddsAPI multi-bookmaker consensus). One signal = one edge value on one outcome of one market. Every live order on the CLOB exists because a signal justified it at creation time.

Orders are removed from the CLOB for two fundamentally different reasons:

1. **Signal-driven cancellation** (new signal replaces old): Fresh consensus data arrives and produces a new edge computation that differs from the one backing the current order. The old order is cancelled because it reflects stale information — a new order is placed immediately to capture the updated signal. This covers:
   - **Contrarian flip** (🔄): Edge now points to the opposite outcome. Cancel the old-side order, place on the new side.
   - **Same-direction repricing** (🪜): Edge still points the same way but at a different value, computed from a different consensus and/or Polymarket price. Cancel-replace with the updated bid price.
   - In both cases, the cancellation is **reactive** — it's triggered by the arrival of new information.

2. **Time-based expiration** (signal aged out): No new signal arrived. The order still reflects signal N, but after X minutes, signal N itself may have become stale — the market may have moved, the match state may have changed, and the edge inference is degrading. The order is cancelled **proactively** before the price comes back and fills an order placed on what is now a minutes-old stale signal. Orders are submitted as `OrderType.GTC` (no server-side expiry — `ORDER_EXPIRY_S = None` in `src/execution/executor.py`), so cancellation is the bot's sole responsibility, enforced at two layers:
   - **Bot-side edge check**: If the poller re-evaluates a stale order and the edge has disappeared (exceeds `max_edge_live` / `max_edge_pregame`), the live order is explicitly cancelled with reason `edge_disappeared`.
   - **Bot-side sweeper**: Background task in ExecutionManager cancels any LIVE/PARTIAL_FILL order older than `MAX_ABSOLUTE_ORDER_AGE_S = 600` (10 minutes). Hard ceiling — nothing on the CLOB from this bot survives past that regardless of poller state. On process crash or network loss, orders sit until the next process start, at which point `reconcile_on_startup()` adopts the orphaned CLOB orders into the local DB and then cancels them.

The distinction matters for code structure: signal-driven cancellation lives in the bid-update pipeline (new data → new order), while time-based expiration is a safety mechanism that runs independently of data flow.

### Live Odds Polling & Bid Updates

**Per-league polling**: Instead of polling per-market (expensive), the poller groups active positions by league, polls each league once, and distributes odds to all positions in that league. Live leagues poll every 15s, pre-game every 300s. Each poll requests `regions=us,uk,eu` (overridable via `LIVE_POLLER_REGIONS`) and the canonical `BOOKMAKERS` list from `config/settings.py` (overridable via `LIVE_POLLER_BOOKMAKERS`), then computes fresh consensus via `extract_consensus_h2h()` / `extract_consensus_totals()`. The `_fetch_odds()` markets param is dynamically built: always `h2h`, plus `totals` when O/U is enabled, plus `spreads` when handicap is enabled. Setting `LIVE_POLLER_REGIONS=""` (empty string) drops the `regions` query parameter entirely — TheOddsAPI then returns its US-only default. Setting `LIVE_POLLER_BOOKMAKERS=""` falls through to `settings.BOOKMAKERS` (identical to leaving the env var unset). Real rollback to a narrower book set is to set `LIVE_POLLER_BOOKMAKERS` to an explicit CSV. The recommended rollback target is `LIVE_POLLER_BOOKMAKERS="pinnacle,betfair_ex_eu,draftkings,fanduel,williamhill"` — these are the five books that the pre-widening 7-book request empirically returned (the seven-book set silently dropped `bet365` and `unibet_eu` per the BOOKMAKERS empirical-validation guardrail above, leaving these five as the known-good narrow set). Picking any other narrow subset is fine if you have a probe-validated reason; pulling random keys out of `settings.BOOKMAKERS` is not — without re-probing, you might land on another dead key.

End-of-cycle health: `poll()` ticks `poller_cycle_completed` once per invocation. If wall-clock cycle duration (`time.monotonic()` start→end) exceeds `pinnacle_poll_live_seconds` (15s), it also ticks `poller_cycle_overrun` — a leading indicator that wider responses are stretching parse time. Sustained overruns suggest dropping `spreads` from the league poll for events without active handicap orders.

**Staleness exclusion**: `BookmakerTracker` monitors each bookmaker's odds across consecutive polls. If unchanged for `STALE_UNCHANGED_THRESHOLD` polls (default 4, ~60s; env `LIVE_STALE_UNCHANGED_THRESHOLD`), the bookmaker is marked stale and excluded from consensus, edge-triggering one increment of the corresponding `l1_excluded_<bookmaker>` counter. Named counters cover the eleven whitelisted books currently mapped: `pinnacle`, `bet365`, `betfair_ex_eu`, `betfair_ex_uk`, `matchbook`, `draftkings`, `fanduel`, `williamhill`, `paddypower`, `unibet_eu`, `unibet_uk`. Anything else in the consensus 14-book whitelist (`betfair`, `unibet`) routes to `l1_excluded_other`. This handles Pinnacle (or any single bookmaker) suspending h2h odds around goals while other books continue updating.

**Calibration playbook**: L1/L2 defaults assume the widened request set. Under the current `LIVE_CONSENSUS_BOOKMAKERS=pinnacle` default the breadth distribution and L1 attribution still surface the same shape — the allowlist gates which books drive the bid, not which books are tracked for staleness. After deploy, watch the heartbeat:
- *Too east* (thresholds too aggressive) — `consensus_books_*` distribution skews to `_le_3` / `_4_5`; `l1_excluded_betfair_ex_eu` / `_betfair_ex_uk` / `_paddypower` / `_unibet_uk` dominate (the EU/UK retail and exchange books, which legitimately hold 30-45s during quiet match periods); `l1_consensus_unavailable` ticks during live windows. Bump `LIVE_STALE_UNCHANGED_THRESHOLD=5`. If you have widened `LIVE_CONSENSUS_BOOKMAKERS` past Pinnacle-only, also bump `LIVE_FROZEN_UNCHANGED_THRESHOLD` to ≥ L1 + 3 to preserve L2 headroom.
- *Too west* (thresholds too conservative) — `consensus_books_*` skews to `_6_7` / `_ge_8`; `l1_excluded_pinnacle` rarely ticks but `edge_threshold_skip_update` rises during goal/VAR moments. Drop `LIVE_STALE_UNCHANGED_THRESHOLD=3`. Under Pinnacle-only consensus, L2=L1 is fine; under a widened allowlist, keep the L2 headroom rule above.
- *Just right* — `_ge_8` ~55%, `_6_7` ~35%, `_le_3` ≤8% (residual long-tail leagues that genuinely don't carry 6+ books), `_4_5` near zero; `l1_excluded_pinnacle` several times per live match, EU/UK books rarely except during true suspensions; `l2_frozen_h2h` occasionally; `edge_threshold_skip_update` baseline matches pre-deploy.

**Poly price sourcing for bid updates**: `_trigger_bid_update()` fetches the latest Polymarket price from `dryrun_snapshots` via `db.get_latest_dryrun_snapshot(token_id)` (mid_price or last_trade_price). Falls back to `poly_price_at_creation` stored on the order. If neither is available, the bid update is skipped entirely — no fallback to consensus-anchored pricing. During flips, `poly_price_at_creation` is updated on the dryrun order to the current poly price of the new outcome.

**Age-gated re-validation**: When consensus hasn't moved enough to trigger a bid update (< 2%), orders are checked for staleness via `_check_stale_orders()`. If an order's age exceeds the threshold — 3 minutes for live matches (`max_order_age_live_s`), 15 minutes for pre-game (`max_order_age_pregame_s`) — it forces a full re-evaluation through `_trigger_bid_update()`. Match state is determined by `_is_match_live()`: `commence_time <= now <= commence_time + 3 hours`. Missing `commence_time` defaults to pre-game (conservative, longer TTL). The replacement order gets a fresh `created_at`, resetting the age counter. This creates a periodic heartbeat: stale orders are re-validated, cancel-replaced if the edge still holds, or cancelled if it doesn't. Logs with `⏰` prefix.

`created_at` is reset on ALL successful bid updates (both consensus-movement-triggered and age-triggered) via `created_at=datetime.now(timezone.utc)` in the `update_dryrun_order_bid()` call. Without this, age-triggered re-validation would fire every poll cycle for the same order since the original `created_at` never changes.

**Bid updates** (when consensus moves ≥2%, `min_pinnacle_move_to_update`):

Consensus-to-Yes/No mapping in `_trigger_bid_update()` is market-type-aware:
- **Moneyline** (`market_type="moneyline"`): Maps h2h consensus home/away to Yes/No using proximity to the order's stored `consensus_prob`.
- **O/U** (`market_type="over_under"`): Extracts the line from the title, looks up the matching line in `consensus["totals"]`, maps over→Yes, under→No.
- **Handicap** (`market_type="spread"`): Parses team and line from the order's `handicap_team`/`handicap_line` fields (falls back to title regex), calls `resolve_handicap_side()` + `extract_consensus_handicap()` against the raw bookmaker data on the consensus dict, maps cover→Yes, not_cover→No. Works from both the poll path (fresh `bookmakers_raw` written by the league refresh) and the enrichment path (carried-over `bookmakers_raw` from the prior poll, preserved by `on_enrichment_result`'s merge). Skipped only when no prior poll has populated `bookmakers_raw` for the event at all. Feeds the result to `MatchOddsTracker.update_handicap()` for frozen detection.

Then per side_role:
- **Spread orders** (`side_role="spread"`): Update both bids' prices (`poly_price - spread_regime_offset`) and recompute equal shares (`N = capital / (new_yes_bid + new_no_bid)`). Recomputed `live_size` is passed to the live cancel-replace so the CLOB order uses fresh sizing. No role changes.
- **Directional orders** (`side_role="heavy"`): Update the heavy bid's price (`poly_price - spread`). If the heavy outcome changes (edge flips), the order is **cancelled and replaced on the new outcome** regardless of fills. Partial fills are kept by the CLOB; the unfilled remainder is cancelled and a new order placed on the flipped outcome with recalculated bid/shares.

Both the flip and bid-update live-order queries use status filter `["LIVE", "PARTIAL_FILL"]` — without PARTIAL_FILL, partially-filled orders with unfilled remainder on the book are silently skipped. The pre-place cleanup in `execute_directional_orders()` uses the wider `get_pending_or_active_orders_by_condition()` which also includes CREATED/SUBMITTED to catch in-flight replacements. Under `SAFE_REPLACE_ENABLED` the pre-place loop only COLLECTS these matches into `deferred_cancels`; the actual CLOB cancel fires via `_run_deferred_cancels()` after `_place_order()` confirms acceptance.

**Cardinality enforcement**: Before the per-order loop in `_trigger_bid_update()`, the poller calls `ExecutionManager.enforce_single_order_per_outcome()` once per `condition_id`. This ensures at most 1 LIVE/PARTIAL_FILL order exists per `(condition_id, outcome_name)` at any time. If duplicates are found (from overlapping dispatch paths), the order with the highest `size_filled` survives; extras are cancelled with reason `cardinality_cleanup`. This prevents order multiplication through cancel-replace (N orders → N replacements) and contrarian flips (N orders → N flipped orders). Logs with `🧹` prefix. The enforcement must run BEFORE the per-order iteration, not inside it — if placed inside the loop, the first order's cancel-replace creates a replacement, then the enforcement runs for the second order and cancels the just-created replacement, leaving zero orders on the book.

**Cancel-replace rate limiting**: Two gates in `_trigger_bid_update()` prevent runaway cancel-replace chains on the live CLOB:
- **Per-market cooldown** (`CANCEL_REPLACE_COOLDOWN_S = 15`): a single `_cancel_replace_cooldowns[market_title]` timestamp gates BOTH paths — same-direction bid updates AND direction flips. Checked at the top of each path before any expensive work. The flip path stamps the timestamp at attempt time (when `live_orders` is non-empty), not at success, because rapid CLOB post-only rejections on the flip side would otherwise let the retry loop blow past the cooldown indefinitely. Tracked via `time.monotonic()`. Cooldown is intentionally matched to `pinnacle_poll_live_seconds` (15s) — consensus odds don't refresh faster than that, so a tighter floor would burn signed-order round-trips on unchanged signal. Suppressed attempts tick `cancel_replace_cooldown_skip`. Production observed pre-gate failure mode — the LiveOddsPoller fired four consecutive `cancel_and_place_flipped` attempts on the same market in <1s when all four were being post-only rejected.
- **Price delta threshold** (`MIN_CANCEL_REPLACE_PRICE_DELTA = $0.01`): compares the new bid price against the **live order's actual CLOB price** (not the dryrun order's `bid_price`, which updates every poll cycle even when the live replacement was gated). If the delta is below one tick, the cancel-replace is skipped. Eliminates identical-price churn. Suppressed attempts tick `cancel_replace_delta_gate_skip` — before this counter, heartbeat totals of "cooldown skip >> replace_cancelled" were ambiguous about whether the cooldown or the delta gate was the binding constraint.

Without these gates, N concurrent orders × M poly price ticks produces N×M total orders — a multiplicative blowup where each tick cancel-replaces all concurrent orders. Cardinality enforcement reduces N to 1, the cooldown caps frequency to once per 15s, and the price delta check eliminates no-ops. The dryrun bid update (DB record) still proceeds regardless — only the live CLOB cancel-replace is gated.

**Live cancel kill switch** (`disable_cancellations`, env `LIVE_DISABLE_CANCELLATIONS`, default `True`): blanket block on all three live cancel + cancel-replace paths in `_trigger_bid_update()` — edge-gone cancel, same-direction bid update, and contrarian flip. When set, the live CLOB action is suppressed before the cooldown / delta gates run; the dryrun in-place DB write still proceeds for bid updates so reporting stays honest, but the contrarian flip path skips both the live and dryrun mutations to keep the layers consistent (no half-flip). Resting orders survive until they fill or the absolute-age stale sweep retires them. Suppressed attempts tick `flip_blocked_disable_cancellations` / `edge_gone_blocked_disable_cancellations` / `bid_update_blocked_disable_cancellations`. Set `LIVE_DISABLE_CANCELLATIONS=false` to restore the original signal-driven cancel behaviour.

**Side-flip kill switch** (`disable_side_flip_cancellations`, env `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS`, default `True`): blocks the opposite-side branch of the pre-place cleanup loop in `ExecutionManager.execute_directional_orders()`. When a new directional order is placed and a stale opposite-outcome order already exists on the same `condition_id`, the cleanup is skipped and the stale opposite order is left resting until it fills or the absolute-age stale sweep retires it. The new heavy-side order still submits, and same-side replace orphan cleanup is unaffected (still ticks `replace_cancelled` as before). Suppressed attempts tick `side_flip_blocked_disable_cancellations`. Set `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS=false` to restore the pre-place opposite-side cleanup.

**Edge-gone cancellation**: When the edge sanity check in `_trigger_bid_update()` rejects a bid update (edge exceeds `max_edge_live` / `max_edge_pregame`), the corresponding live CLOB order is explicitly cancelled with reason `edge_disappeared`. Without this, stale orders with vanished edge would survive indefinitely — the bid update skips them, but never cancels them. Logs with `🧹` prefix. Gated by the `disable_cancellations` kill switch above.

**Direction flip pipeline** (`_trigger_bid_update()` flip path):
1. **DB constraint guard**: checks the partial unique index — if a filled order already occupies `(market_title, new_outcome, spread_level)`, the flip is blocked (DB would reject the update). The `spread_level` must be included in this check to match the actual index columns — without it, the guard false-positives and blocks flips into slots that the DB would accept (different spread levels are different slots)
2. **Recalculate on new side**: builds a synthetic `flipped_order` with `outcome_name` set to the new outcome, then runs `recalculate()` to get fresh edge/bid/consensus. `recalculate()` takes `(order, new_pinnacle_yes, new_pinnacle_no, current_poly_yes, current_poly_no)` and returns `{bid_price, consensus_prob, edge, side_flipped, new_heavy_side, new_role}` — it does NOT return `outcome_name`, `token_id`, or `bid_shares`. The caller must resolve those separately
3. **Edge sanity check**: the recalculated edge on the NEW side must pass the same threshold (8% pregame, 25% live). Without this, a flip could land on a side with an implausible edge
4. **Token resolution**: resolves `token_id` for the new outcome via `get_token_by_condition_outcome(condition_id, new_outcome)` (DB lookup), then sibling orders, then `_resolve_token_via_clob()` which hits `/markets/{condition_id}` on the CLOB and reads `tokens[].token_id` by outcome name. The CLOB fallback maintains a positive cache keyed by `(condition_id, outcome.lower())` (unbounded but tiny) and a 5-min negative cache per condition_id to avoid hammering the endpoint on persistently misconfigured markets. Every resolved token from the fetch response is cached, so a single fetch can satisfy subsequent flips on the same market. Ticks `token_resolved_via_clob` on success. Token resolution is required for ALL flips (including dryrun-only) because `price_poller.py` uses `token_id` to fetch the order book — a wrong token means wrong prices and wrong fill simulation
5. **Live-first ordering**: if `execution_manager` is set, the CLOB flip runs first via `cancel_and_place_flipped(..., new_consensus_prob=flipped_vals["consensus_prob"], new_edge=flipped_vals["edge"])`. The poller MUST pass the recomputed cp/edge so the new-side row is persisted with values aligned to the new outcome. If it fails, dryrun stays on the old outcome so both layers remain consistent — no half-flips
6. **Dryrun update**: only after live succeeds, updates the dryrun order in-place with new outcome, token_id, bid_price, consensus_prob, edge, and shares

**Guardrail — flip writes MUST carry new-side cp/edge.** The pre-Bug 2 flow inherited `consensus_prob` and `edge` from the old (pre-flip) row when writing the new-side order — so the new row's `edge != consensus_prob − price − spread_level` audit invariant failed by 4–6× the residual fingerprint. The fix threads `new_consensus_prob` / `new_edge` through `update_bid` → `cancel_and_place_flipped` → `_cancel_and_place_flipped_inner`. Callers MUST supply both; if either is omitted, `_cancel_and_place_flipped_inner` falls back to the old row's values, logs a loud warning, and ticks `flip_stale_cp_edge_fallback` so the regression is observable rather than silent. The poller's `bid_calculator.recalculate()` already produces these; do not "simplify" by dropping the kwargs at the call site. `check_flip_residual.py` is a post-deploy acceptance script that audits contrarian-flip rows for the invariant — run it after deploys that touch any flip path.

This replaces the old "block flip if filled" design. The naive approach of blocking flips when fills exist locks the bot into a stale position — if consensus shifts hard enough to flip the heavy side, the old side is likely losing value and the bot should follow the signal.

**H2H suspension fallback**: During live matches, if all bookmakers suspend h2h odds simultaneously, the engine falls back to the poller's last-known cached h2h odds rather than skipping the market entirely.

**Orderbook token inversion**: The backup order book poller uses the token_id from the rn1 trade, which always corresponds to the outcome rn1 traded. For the complementary outcome (e.g., "No" when rn1 traded "Yes"), the fetched order book shows prices for the wrong side. The poller detects this by comparing the fetched mid-price against `poly_price_at_creation`; if the difference exceeds 0.30, prices are inverted: `best_ask = 1.0 - best_bid_raw`. This ensures fill checks for complementary-side orders use correct price levels.

### Multi-Spread Simulation (Queue Position Analysis)

Applies to the **directional regime** only. The core economics question: rn1 is already sitting at the best bid with sub-second adjustments. To get filled ahead of rn1, you'd bid higher (tighter spread) — but that eats into your edge. To understand the tradeoff, the engine runs multiple spread levels in parallel:

**Spread levels** (configurable via `DRYRUN_CONFIG["spread_levels"]`):
- 0.5% (aggressive — tighter than rn1, more fills, thinner margin)
- 1.0% (baseline — default strategy spread)
- 1.5% (conservative — wider than rn1, fewer fills, fatter margin)
- 2.0% (passive — only fills on large moves, rarely filled)

Each spread level creates one heavy-side order. The reporter shows fill rate AND P&L per spread level, answering: "at what spread does fill rate × margin maximize expected value?"

The 3 fill tiers (optimistic/realistic/conservative) still apply within each spread level, giving a 4×3 matrix of (spread × fill-assumption) scenarios.

Spread-regime orders use a single fixed offset (0.8%) instead of multiple spread levels.

### P&L Decomposition

When a position resolves, the Resolution Watcher uses regime-specific P&L calculation:

**Spread regime** (`_calculate_spread_pnl`):
```
if both sides filled:
    paired = min(yes_shares, no_shares)
    excess = abs(yes_shares - no_shares)
    spread_pnl = paired × (1.00 - avg_yes_price - avg_no_price)
    directional_pnl = excess × (payout - excess_price)  # residual from unequal fills

if one side only (adverse selection):
    spread_pnl = 0
    directional_pnl = filled_shares × (payout - filled_price)
```

**Directional regime** (existing decomposition):
```
heavy_shares from filled heavy-side orders
spread_pnl = 0  (no paired component in pure directional)
directional_pnl = heavy_shares × (payout - avg_heavy_price)
  where payout = 1.00 if heavy won, else 0.00
```

Legacy pre-v4 orders with `side_role="light"` are still handled for backward compatibility.

The reporter splits results by regime and by market type. Key metrics per regime:
- **Spread**: pair rate (target ≥90%), average spread earned per pair, one-sided loss rate
- **Directional**: heavy win rate (target ≥50%), average P&L per bet, spread-level breakdown

When multiple market types are present, the report includes a `BY MARKET TYPE` breakdown showing position count and P&L per market type (moneyline, over_under, spread).

Shadow P&L (symmetric, Kelly) is reported for directional positions only.

### No-Logic Counterfactual (`dryrun_orders_no_logic`)

A parallel mirror of `dryrun_orders` that captures what would have happened if every nursing decision were skipped. At creation time, every regular dryrun order is mirror-inserted into `dryrun_orders_no_logic` via `Database.insert_dryrun_order_pair(**kwargs)` — same bid, shares, regime, `bookmakers_snapshot`. The two tables share an identical unique-slot partial index (`uq_dryrun_orders_no_logic_active_slot` on `(market_title, outcome_name, spread_level)` for `status IN ('open', 'filled')`), so dedup semantics match.

**No nursing fires on these rows.** `LiveOddsPoller._trigger_bid_update()` does not query `dryrun_orders_no_logic`, so no edge recompute, no edge-gone cancel, and no contrarian flip ever runs. Once written, the row is immutable on bid/cp/edge until either the fill simulator marks it filled or the resolution watcher settles it.

**Fills run normally.** `FillSimulator` keeps a parallel `_cumulative_volume_no_logic` ledger, and `PricePoller` invokes `check_fill_from_trade_no_logic` / `check_fill_from_book_no_logic` on every rn1 trade and order-book snapshot. Fills are recorded directly on the row via `update_dryrun_no_logic_order_fill`; there is no `dryrun_positions` rollup for this table.

**Per-row resolution.** `ResolutionWatcher._resolve_no_logic_orders()` runs alongside the regular settlement loop. For each filled row whose market resolved: `outcome_won = (outcome_name == winning_outcome)`; `pnl = shares × $1 − bid_size_usd` if won, `−bid_size_usd` if lost.

Export via `--dryrun-no-logic-export` (`main.py:dryrun_no_logic_export_csv`). The counterfactual lets P&L analysis isolate the contribution of the bid-update / flip / edge-gone-cancel layer from the underlying signal quality.

---

## Match Audit (`src/match_audit/`)

**Files**: `src/match_audit/` | **Config**: `MATCH_AUDIT_ENABLED` env var (default `true`) | **Full design**: `match_audit_spec.md`

Diagnostic table `match_audit` recording one row per rn1 trade at `UniversalOddsService.find_match` resolution. Captures: parsed category, extracted participants, parse stage reached, candidate count, expected sport_keys (league search space), matched event metadata (on success), and the rarity-weighted `information_score` (FLOAT, NULL when no candidate was scored). Used to diagnose matcher regressions by SQL aggregation across all leagues — answers "which league's markets are failing most, and at what stage?" and "what does the score distribution look like across matched vs. rejected rows?" without requiring anyone to eyeball the log stream.

Best-effort writes only: recorder swallows all exceptions and has a 2-second timeout. Never propagates to enrichment. Enable/disable via `MATCH_AUDIT_ENABLED` env var (default on).

Own `MatchAuditBase` / `metadata.create_all` to stay decoupled from main storage — mirrors the pattern already used by `LiveBase` in `src/execution/models.py` and the dry run base. Integration: one call from `_enrich_universal` in the orchestrator (`self.match_audit.record(ctx, result.match_result)`) and one `create_all` during `start()`, immediately followed by an idempotent `ALTER TABLE match_audit ADD COLUMN IF NOT EXISTS information_score FLOAT` so deployments predating the score column pick it up without a manual migration. Removal is a three-file delete plus ~5 lines in the orchestrator; see `match_audit_spec.md` §"Removal procedure".

The `expected_sport_keys` JSONB column is deliberately denormalised and stores the **full post-category-gate search space** — every `sport_key` in the events index whose `_validate_sport_match(sport_key, market_category)` returns True, computed once up front before the event loop. The list is identical whether iteration short-circuits on an early match or runs to exhaustion. On failure, `matched_sport_key` is null, so without this array the diagnostic SQL cannot group failed trades by league. Storing the full gate-passing set lets a single unnest-and-group query cover both success and failure rows.

**Consequence for query authors.** Because the list is the full candidate search space, a trade that matched successfully still has every candidate league in its array — unnesting `expected_sport_keys` across all rows will count each matched trade once per candidate league, inflating per-league `matched` counts. The correct pattern (and the rewritten Q1 in `match_audit_spec.md`) groups matched rows by `matched_sport_key` directly and only unnests `expected_sport_keys` for failure rows. The alternative (keying successes off the single winner by appending as iteration progresses) over-represented early-ordered leagues because `self.events` insertion order biased which keys made it into the list before the loop broke — worse failure mode than the inflated-matched one.

### Extracting the audit

`--match-audit-export` dumps every `match_audit` row (newest first) to CSV. JSONB columns (`expected_sport_keys`, `parsed_teams`) are serialised as compact JSON strings so the file opens cleanly in Excel and pandas. Default filename `match_audit_export.csv`; override with `--match-audit-export some_name.csv`. The exporter runs `MatchAuditBase.metadata.create_all` on entry so it's safe to invoke on a fresh database even if the bot hasn't started yet.

### Known coverage gaps (not bugs)

Polymarket lists markets for leagues that TheOddsAPI doesn't cover. Trades on these markets reach `find_match` and resolve as failures (no event in the index), then get recorded to `match_audit`. Listing them here so future diagnostic passes don't chase them as matcher bugs:

- **Moroccan Botola Pro** (AS FAR, RS Berkane, RCA Zemamra, OC Safi, etc.) — no `soccer_morocco_*` sport_key exists.

Any trade in `match_audit` with `parsed_category='soccer'` and an `expected_sport_keys` array that excludes the relevant regional key falls into this bucket. The fix for these is coverage — either a new TheOddsAPI sport_key if/when one is added, or a separate data source — not matcher tuning.

---

## Refresh Log (`src/refresh_log/`)

**Files**: `src/refresh_log/` | **Config**: `REFRESH_LOG_ENABLED` env var (default `true`)

Diagnostic table `theodds_refresh_log` recording one row per `fetch_sport_events(sport_key)` call during every `refresh_events()` sweep. Distinguishes the three outcomes that collapsed to the same "sport missing from index" observable before instrumentation: **thin fixtures** (normal attempts, empty responses), **under-polling** (attempts below peers), and **skipped entirely** (zero attempts). Rows sharing a single `refresh_cycle_id` UUID belong to the same sweep, so "which sports ran in the cycle that produced the bad coverage" is a single-column filter.

Same best-effort posture as `match_audit`: 2-second timeout, swallows all exceptions, own `RefreshLogBase` / `metadata.create_all`, own `async_sessionmaker` on the shared engine. A cycle that fails to log must not fail to refresh. Enable/disable via `REFRESH_LOG_ENABLED` env var.

### Table schema (`theodds_refresh_log`)

| Column | Type | Meaning |
|---|---|---|
| `sport_key` | `VARCHAR(64)` | TheOddsAPI sport key (e.g. `soccer_germany_bundesliga`) |
| `refresh_cycle_id` | `UUID` | one per `refresh_events()` invocation — groups all sports in a sweep |
| `priority_only` | `BOOLEAN` | whether the caller passed `priority_only=True` (priority list) or `False` (all active sports) |
| `attempted_at` | `TIMESTAMPTZ` | wall-clock start of the attempt (i.e. `started_at`, named for SQL greppability) |
| `duration_ms` | `INTEGER` | monotonic-clock elapsed over the `_api_request` + parse loop |
| `event_count` | `INTEGER` | events parsed and indexed; null on failure, zero is a legitimate `empty` reading |
| `http_status` | `SMALLINT` | last HTTP status seen (null on `network_error` / `exception`) |
| `api_key_index` | `SMALLINT` | rotator index of the key that served (or tried to serve) the request — distinguishes "never polled" from "polled but every key was exhausted" |
| `outcome` | `VARCHAR(24)` | `ok` / `empty` / `http_error` / `network_error` / `exception` |
| `error_message` | `TEXT` | truncated to 500 chars; null on `ok` / `empty` |
| `created_at` | `TIMESTAMPTZ` | DB-set insert timestamp |

Outcome values:
- `ok` — HTTP 2xx, `event_count ≥ 1`
- `empty` — HTTP 2xx, `event_count = 0` (thin-fixtures day)
- `http_error` — HTTP non-2xx after retries (429 / 5xx / 401/403 after all rotations)
- `network_error` — `httpx.ConnectError` / `httpx.TimeoutException` after retries
- `exception` — anything raised out of `fetch_sport_events`; caught by `gather(return_exceptions=True)` in `refresh_events`

### Wiring

`UniversalOddsService.__init__` accepts an optional `refresh_log_recorder=` kwarg. The orchestrator constructs a `RefreshLogRecorder` (alongside `MatchAuditRecorder`) and passes it in at service construction. `refresh_events()` generates one `cycle_id = uuid.uuid4()` per invocation. Inside the inner `fetch_sport_events`, a try/finally builds a `RefreshAttempt` on every exit path (ok/empty/error/exception) and writes it through the recorder.

**Per-call outcome capture from `_api_request`**: `_api_request` accepts an optional `status_out: Optional[dict] = None` scratch-dict kwarg. When non-None, every terminal return point populates `status_out["outcome"]`, `status_out["http_status"]`, and `status_out["api_key_index"]`. The kwarg is opt-in — every other `_api_request` caller in the codebase is unchanged because the default `None` is a no-op. Only `fetch_sport_events` passes a fresh `{}`.

**Guardrail — do not reshape `_api_request`'s return tuple to carry the outcome.** Every `_api_request` caller uses `data, _ = await self._api_request(...)` or `data, remaining = ...`. A third return slot would require touching every call site; the scratch-dict avoids that blast radius at no behavioural cost. Do not "simplify" by collapsing `status_out` into the tuple.

**Guardrail — re-raise exceptions after the audit write.** `fetch_sport_events` is invoked via `asyncio.gather(return_exceptions=True)`. Its body is wrapped in `try/except BaseException` with the recorder write in the `finally`; the captured exception is then re-raised immediately after the `try/except/finally` closes so `gather` still receives it. Swallowing the exception silently here would convert failures into successes from `gather`'s perspective — the refresh_log would record the failure, but every other observer (log line, future error-attribution code) would see a clean sweep. The re-raise is load-bearing, not boilerplate.

### Extracting the log

`--refresh-log-export` dumps every `theodds_refresh_log` row newest-first to CSV. Default filename `refresh_log_export.csv`; override with `--refresh-log-export some_name.csv`. The exporter runs `RefreshLogBase.metadata.create_all` on entry so it's safe to invoke on a fresh database before the bot has started. The `refresh_cycle_id` UUID is stringified; all timestamps are ISO-8601.

### Diagnostic shape

Group `theodds_refresh_log` by `sport_key` and `outcome` across a time window, then read off the pattern:
- normal attempts, `ok` low, `empty` high → thin-fixtures day, no fix
- attempts substantially below peers in the same cycle → under-polling, fix in the refresh loop
- zero attempts for a sport that should have fired → skipped, fix in the priority list or enumeration
- `http_error` dominates with a narrow `api_key_index` range → key exhaustion on that tier
- `duration_ms` spikes on a specific sport → rate-limit backoff is starving it

Rows can be joined against `match_audit` on `(sport_key, DATE(attempted_at))` to correlate refresh coverage with downstream matcher success on the same day.

### Row volume and retention

~25 priority sports × one cycle every ~2 minutes ≈ 18k rows/day, matching `match_audit`'s order of magnitude. No retention logic — manual `DELETE FROM theodds_refresh_log WHERE attempted_at < NOW() - INTERVAL '30 days'` when needed.

---

## Consensus Outlier Log (`src/consensus_outlier_log/`)

**Files**: `src/consensus_outlier_log/` | **Config**: `LIVE_CONSENSUS_DIAGNOSTICS_DB` env var (default `false`)

Diagnostic table `consensus_outlier_log` recording one row per consensus extraction whose breadth or freshness crosses a documented outlier trigger. Used to ground-truth bookmaker-breadth claims that the heartbeat counters can only summarize.

Same best-effort posture as `match_audit` and `theodds_refresh_log`: 2-second timeout, swallows all exceptions, own `ConsensusOutlierBase` / `metadata.create_all`, own `async_sessionmaker` on the shared engine. The recorder is a **no-op when `LIVE_CONSENSUS_DIAGNOSTICS_DB=false` (the default)**, so the gate is opt-in. The DB table is created unconditionally on startup so flipping the env on later does not require a schema migration.

### Triggers

A single `ConsensusOutlierEvent` can fire any subset of:

- `consensus_none` — `calculate_consensus_prob()` returned `None` (all books stale)
- `books_lt_4` — `bookmakers_used < 4` (suppressed when `consensus_none` fires; the all-stale case already implies zero books)
- `l1_excluded` — at least one bookmaker was L1-excluded on this extraction
- `l2_frozen` — the event×market_type was L2-marked frozen at the time of extraction

The recorder writes **one row per fired trigger** so filtering by `outlier_reason` stays a simple equality test. An extraction that fires `books_lt_4 + l1_excluded + l2_frozen` produces three rows with the same `(event_id, recorded_at)` pair.

### Table schema (`consensus_outlier_log`)

| Column | Type | Meaning |
|---|---|---|
| `id` | `BIGSERIAL PK` | |
| `recorded_at` | `TIMESTAMPTZ` default `now()` | |
| `event_id` | `VARCHAR(64)` | TheOddsAPI event id |
| `sport_key` | `VARCHAR(64)` | |
| `market_type` | `VARCHAR(16)` | `h2h` / `totals` / `handicap` (column accepts all three; only `h2h` is written today — see Wiring) |
| `is_live` | `BOOLEAN` | derived from `commence_time` |
| `bookmakers_seen` | `JSONB` | array of names actually returned by API |
| `bookmakers_used` | `JSONB` | array post-L1 exclusion |
| `bookmakers_excluded_l1` | `JSONB` | array, names only |
| `consensus_raw` | `JSONB` | nullable; `{home, away, draw}` for h2h, etc. |
| `l2_frozen` | `BOOLEAN` | |
| `outlier_reason` | `VARCHAR(64)` | which trigger fired (`books_lt_4` / `l1_excluded` / `l2_frozen` / `consensus_none`) |

### Wiring

The orchestrator constructs a `ConsensusOutlierRecorder(engine, enabled=LIVE_CONSENSUS_DIAGNOSTICS_DB)` alongside the match-audit and refresh-log recorders, runs `ConsensusOutlierBase.metadata.create_all` once, and assigns `self.consensus_outlier_log` to `dryrun_engine.live_odds_poller.consensus_outlier_log` after `DryRunEngine.initialize()`. The assignment runs inside the `_dryrun_enabled` branch only, so monitoring-only mode (no `--dryrun` / no `--live`) leaves the poller unbuilt and the recorder unwired — the table will exist but stay empty even with `LIVE_CONSENSUS_DIAGNOSTICS_DB=true`. The poller's `_record_consensus_outlier(...)` helper is invoked from `poll()` after each **h2h** consensus extraction (both the success path and the all-suspended `None` short-circuit). Totals and handicap extractions do not currently emit rows — extending coverage there is a follow-up. The helper synthesizes a `ConsensusOutlierEvent` with the bookmaker lists and current MatchOddsTracker frozen state, then calls `recorder.record(event)`. The recorder iterates the triggered reasons and writes one row per reason inside a single committed transaction.

### Extracting the log

`--consensus-outlier-export` dumps every `consensus_outlier_log` row newest-first to CSV. Default filename `consensus_outlier_export.csv`; override with `--consensus-outlier-export some_name.csv`. The exporter runs `ConsensusOutlierBase.metadata.create_all` on entry so it's safe to invoke on a fresh database before the bot has started. JSONB columns are serialized as compact JSON strings.

### Row volume and retention

Best-effort with no retention logic. When enabled during live windows on the widened poller, expected volume is ~one row per outlier-triggered extraction per league poll cycle — order of magnitude similar to `theodds_refresh_log` for active periods. Manual `DELETE FROM consensus_outlier_log WHERE recorded_at < NOW() - INTERVAL '7 days'` when needed.

**Guardrail — table creation is unconditional, recorder is gated.** The `metadata.create_all` runs every startup regardless of `LIVE_CONSENSUS_DIAGNOSTICS_DB`. This is intentional: flipping the env on at 3am to investigate a live anomaly should not require a schema migration step. The recorder's `enabled` flag is the only behavioural switch — when off, `record()` short-circuits before opening a session.

---

## Odds Raw Log (`src/odds_raw_log/`)

**Files**: `src/odds_raw_log/` | **Config**: `LIVE_ODDS_RAW_LOG_DB` env var (default `false`)

Diagnostic table `theodds_raw_log` capturing the **full TheOddsAPI response** for every successful league poll. One row per `/v4/sports/{sport_key}/odds` fetch. The payload column is JSONB and holds the raw event list as returned (each event with its `bookmakers`, `markets`, `outcomes`), letting forensic queries reconstruct any consensus calculation downstream — including books that were excluded from consensus by the `LIVE_CONSENSUS_BOOKMAKERS` allowlist. Filterable by `sport_key`, `endpoint`, or `recorded_at`; the heavy column is `payload`, so callers should project explicitly when scanning a window.

### Table schema (`theodds_raw_log`)

| Column | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `recorded_at` | TIMESTAMPTZ | server default `now()` |
| `sport_key` | VARCHAR(64) | e.g. `soccer_epl` |
| `endpoint` | VARCHAR(128) | e.g. `/sports/soccer_epl/odds` |
| `regions` | VARCHAR(64) | request param at fetch time |
| `markets` | VARCHAR(64) | request param at fetch time (`h2h,totals,spreads`) |
| `bookmakers` | VARCHAR(512) | request param at fetch time (csv) |
| `event_count` | INTEGER | number of events in `payload` |
| `payload` | JSONB | full response, list of event dicts |

Indexed on `(sport_key, recorded_at)` and `recorded_at`.

### Recorder

Same posture as `consensus_outlier_log`: best-effort, 2-second timeout, swallows every exception, own `OddsRawLogBase` / `metadata.create_all`, own `async_sessionmaker` on the shared engine. The recorder is a **no-op when `LIVE_ODDS_RAW_LOG_DB=false` (the default)**. The DB table is created unconditionally on startup so flipping the env on later does not require a schema migration. The orchestrator wires `self.odds_raw_log = OddsRawLogRecorder(...)` and assigns it onto `dryrun_engine.live_odds_poller.odds_raw_log` inside the `_dryrun_enabled` branch — the poller's `_fetch_odds()` calls `recorder.record(OddsRawLogEvent(...))` after each successful fetch.

### CLI export

`--odds-raw-export [FILE]` dumps every `theodds_raw_log` row newest-first to CSV. Default filename `odds_raw_log_export.csv`. The `payload` column is serialized as a compact JSON string per row. For long-running deployments the table can grow large; export workflows should usually project a time window in SQL first rather than dumping the whole table. Same `metadata.create_all`-on-entry idempotency as the other diagnostic exports.

### Volume expectations

When enabled, expect roughly **one row per league poll** — typical live window: O(1)/league/15s, so a 5-league live cluster generates ~20 rows/min. Each `payload` is the full `/odds` response (kilobytes per event, dozens of events per league), so disk usage compounds quickly. Manual `DELETE FROM theodds_raw_log WHERE recorded_at < NOW() - INTERVAL '7 days'` when needed.

**Guardrail — gated by default.** This recorder writes the heaviest payload of any of the diagnostic tables. Leaving `LIVE_ODDS_RAW_LOG_DB=true` indefinitely will fill the database. Treat as a forensic capture mode: turn on for a window, dump, turn off.

---

## Live Execution Layer

**Files**: `src/execution/`, `src/storage/live_store.py` | **Config**: `config/live_settings.py`

Routes the Dry Run Engine's trading decisions to Polymarket's CLOB as real GTC limit orders (no server-side expiry — the bot is sole cancellation authority). The execution layer is **additive** — the dry run path continues to write simulated orders alongside live ones (shadow mode). Both share the same enrichment pipeline, consensus odds, and regime branching logic.

### Integration Point

After the regime branch in `engine.py:_create_orders_inner()`, if `execution_manager` is set (live mode), the engine dispatches to the CLOB. Live dispatch is **decoupled from dryrun order creation** — it fires on every valid edge detection, even when dryrun dedup blocks the dryrun order. The risk manager's `max_market_exposure` check prevents over-exposure on the live side. Without this decoupling, the first edge detection creates a dryrun order and attempts live dispatch; every subsequent edge detection for the same market hits the dryrun dedup (DB unique index returns `None`) and silently skips live execution — making the bot one-shot-per-market.

Two dispatch paths:

- **Spread regime** (`max_edge < 2%`): calls `ExecutionManager.execute_spread_orders()` with both Yes and No token IDs (extracted from `market_info.raw_data["tokens"]` in the orchestrator), equal shares, and consensus probs. Both token IDs and condition_id must be present or the dispatch is skipped. Can be disabled via `LIVE_ENABLE_SPREAD_REGIME=false` — dryrun still simulates spread orders in shadow, but live dispatch is skipped with a log line.
- **Directional regime** (`max_edge ≥ 2%`): computes fresh bids directly using `LIVE_CONFIG["directional_spread_level"]` (default 1.0%) and `capital_per_market`, independent of dryrun's 4 hardcoded spread levels. Calls `ExecutionManager.execute_directional_orders()` with the heavy-side token ID. In live mode, the engine caps the budget to `min(dryrun_budget, LIVE_CONFIG["capital_per_market"])` to avoid risk rejection. `capital_per_market` is operator-tunable via `LIVE_CAPITAL_PER_MARKET` (see "Operator-tunable risk knobs" above); the dryrun shadow uses its own separate value from `DRYRUN_CONFIG`, which is why the `min(...)` clamp is needed.

`neg_risk` is derived from `source_exchange == "neg_risk"` (passed from the trade parser through orchestrator → engine). This must be correct — orders signed against the wrong exchange contract are rejected on-chain.

The **scanner** (`scanner.py`) also dispatches to live via `_dispatch_to_live()` when `execution_manager` is set. It recomputes bids with live capital/spread settings (not dryrun's), resolves both token IDs from the condition_id via `_resolve_token_ids()`, and calls the same `execute_spread_orders()` / `execute_directional_orders()` entry points. The scanner's live dispatch is gated on new dryrun orders being created (unlike the RT path, which is fully decoupled).

Bid updates flow from `LiveOddsPoller._trigger_bid_update()` → `ExecutionManager.update_bid()` (same-direction repricing) or → `ExecutionManager.cancel_and_place_flipped()` (direction flip). Both acquire the per-market lock for the DB phase, then submit to the CLOB outside the lock. With `SAFE_REPLACE_ENABLED` (default on), all three cancel-replace sites — `update_bid` normal path, contrarian flip path, and `execute_directional_orders`' pre-place cleanup loop — submit the new order FIRST and cancel the old order only on CLOB acceptance. If the new submit is rejected (post-only crosses the book, etc.), the old order stays live on the exchange. See "Safe Cancel-Replace" below in ExecutionManager for the mechanism.

### Component 1: PolymarketExecutor

**File**: `src/execution/executor.py`

Async wrapper around the synchronous `py-clob-client-v2` SDK. All SDK calls go through `asyncio.get_running_loop().run_in_executor()` with a dedicated `ThreadPoolExecutor(max_workers=4)`. This pool is separate from IntelBot's Web3 executor to avoid contention.

Methods: `create_and_submit_order()`, `cancel()`, `cancel_all()`, `get_order()`, `get_open_orders()`, `get_balance_allowance()`, `get_tick_size()`, `get_best_ask()`.

All SDK calls route through `_run_sync()`, which detects `PolyApiException` with `status_code == 429` and increments the `clob_rate_limit_hit` counter before re-raising. The wrapper observes only — no swallow, no retry. If the heartbeat ever shows a non-zero value, that is the signal to add exponential-backoff retry here; the underlying call still fails in the interim and is handled by callers' existing error paths.

`create_and_submit_order()` takes raw price/size, rounds to tick-size precision via `_round_price()` / `_round_size()`, builds `OrderArgs(token_id, price, size, side=Side.BUY|Side.SELL)`, then submits via the V2 SDK's single-call entry point `client.create_and_post_order(order_args, options=PartialCreateOrderOptions(tick_size=str(tick), neg_risk=neg_risk), order_type=OrderType.GTC, post_only=True)`. **`tick_size` is mandatory on every V2 order call** — failure to pass it raises an SDK error before the order reaches the network. V2 also dropped `feeRateBps`, `nonce`, `taker`, and `expiration` from the order struct (fees are baked into the V2 exchange itself; per-address uniqueness now comes from a millisecond timestamp). GTC (Good Till Cancelled) means the CLOB never auto-expires — the bot is the sole cancel authority via cancel-replace, edge-gone, stale-sweep, and startup reconcile (see Signal Model above). With `post_only=True`, orders that would cross the spread are rejected instead of filling as taker (preventing adverse fills at market prices + the V2 taker fee). EIP-712 signing takes ~1 second — this is the latency bottleneck.

**Guardrail — `Side` is the V2 enum, not a string constant.** V1 imported `BUY` / `SELL` from `py_clob_client.order_builder.constants`. V2 collapsed those into the top-level `Side` enum: `Side.BUY` / `Side.SELL`. `OrderArgs(side=...)` rejects bare strings. If a future refactor sees `from py_clob_client.order_builder.constants import BUY` and "fixes" the import to V2 by changing the module path only, it will land at `Side.BUY` is `0`, not `"BUY"` — silently wrong because `OrderArgs` accepts both forms but the on-chain side flag will not match what the bot's local accounting expects.

**Guardrail — do not re-introduce GTD expiry.** Before the Phase 2 switch, orders used `OrderType.GTD` with a 4-minute expiration as the "CLOB-enforced safety net." That relied on the bot being *slow enough* that orders died before drifting — which also meant the bot could never let a good bid rest. Under GTC, the bot's cancel-replace cadence (poller: 15s), edge-gone cancel, and `_sweep_stale_orders` 10-minute hard cap are the only things that retire an order. If you are tempted to add an expiration back "just in case," the symptom you're patching is either (a) the stale sweeper not running, or (b) the poller not firing — fix those, not the expiry.

`get_best_ask()` fetches the current order book for a token and returns the lowest ask price. Handles both dict and object response forms from the SDK. Used by the postOnly retry loop in `_place_order()`.

`get_balance_allowance()` passes `BalanceAllowanceParams(asset_type=AssetType.COLLATERAL)`. Under V2 this queries pUSD instead of USDC.e — the SDK abstracts the underlying token. Returned shape (`.balance`, `.allowance`) is unchanged. The SDK still crashes with `'NoneType' object has no attribute 'signature_type'` if called without the wrapping params object.

The `neg_risk` flag is passed via `PartialCreateOrderOptions(tick_size=..., neg_risk=...)`. Both fields are required: `tick_size` so the SDK's signed payload matches the market's price/size precision, `neg_risk` so the order is signed against the correct exchange contract (CTF vs Neg Risk).

V2 ClobClient construction is two-step: a key-only bootstrap client derives API creds via `create_or_derive_api_key()`, then a full client is built with `creds=`, `signature_type=`, `funder=`. The L1/L2 API credentials themselves are unchanged across V1 and V2 — existing creds work without re-derivation; the method name is what changed.

**Tick-size rounding** is the #1 source of order rejections. The Polymarket API returns `minimum_tick_size` as a float, but `_round_price()` / `_round_size()` need to call `.split('.')` on it. Both methods cast `tick_size = str(tick_size)` first. Without this cast, the API's float value causes `'float' object has no attribute 'split'`. Each market has its own tick size; prices and sizes must conform:

| Tick Size | Price Decimals | Size Decimals |
|-----------|---------------|---------------|
| 0.1       | 1             | 0             |
| 0.01      | 2             | 2             |
| 0.001     | 3             | 4             |
| 0.0001    | 4             | 6             |

`_round_price` uses `round(round(price / ts) * ts, decimals)`. Python's banker's rounding applies — `round(1234.5)` → 1234 (even). Polymarket likely uses truncation. If orders are rejected with tick-size errors, this is the first place to look.

### Component 2: RiskManager

**File**: `src/execution/risk_manager.py`

Every order passes through `check_order()` before submission. Returns `ALLOW`, `REJECT`, or `HALT`. Nine sequential gates:

1. **Halt flag** — reject everything if halted
2. **Price sanity** — reject if price < 0.02 or > 0.98
3. **Tick validation** — reject if price doesn't align to tick grid
4. **Balance** — reject if `usdc_cost > (USDC_balance - committed_capital)`. Committed capital = sum of `price × size` for all non-terminal, unfilled BUY orders. Polymarket checks balances continuously — orders backed by already-committed capital are rejected.
5. **Active market count** — reject if distinct markets with non-terminal orders ≥ `max_concurrent_markets` (operator-tunable via `LIVE_MAX_CONCURRENT_MARKETS`; see "Operator-tunable risk knobs"). Uses `count_active_markets()` which counts distinct `condition_id` from `live_orders` where status NOT IN (REJECTED, CANCELLED, SETTLED). This counts pending (CREATED, SUBMITTED, LIVE) and filled orders — a CREATED order reserves a market slot immediately, not only after its first fill. The naive approach of counting `live_positions` with status "OPEN" is silently blind to pending orders: positions only exist after a fill, so a burst of 40 pending orders all pass the gate.
6. **Per-market exposure** — reject if committed cost + new order > limit (default $10). Uses `get_market_exposure()` which sums `price × size` for all non-terminal orders on the same `market_title`. This counts committed cost at order time, not post-fill `usdc_spent` (which is 0 until the order fills). For filled orders, `price × size ≥ usdc_spent` always (limit fills are at-or-better), making this conservatively correct.
7. **Hourly loss** — HALT if realized P&L in last hour < -$25
8. **Daily loss** — HALT if realized P&L in last 24h < -$50
9. **Max drawdown** — HALT if drawdown > 30%

HALT is sticky — requires manual `reset_halt()` after investigation.

The ExecutionManager rounds price and size to tick precision **before** passing to the risk manager. Without this, the tick validation gate rejects prices that would be valid after the executor's rounding — this was a critical bug that silently dropped valid orders.

### Component 3: OrderTracker

**File**: `src/execution/order_tracker.py`

Tracks every order from creation through settlement. State machine:
```
CREATED → SUBMITTED/LIVE/MATCHED/REJECTED   (direct LIVE/MATCHED because
                                             record_submitted collapses
                                             sign-and-post into one step)
SUBMITTED → LIVE/MATCHED/REJECTED            (reserved for future async
                                             submit queues; currently
                                             unreachable from CREATED but
                                             kept in the table)
LIVE → PARTIAL_FILL/CANCELLED/FILLED
PARTIAL_FILL → FILLED/CANCELLED
FILLED → SETTLED
```

`VALID_TRANSITIONS[CREATED]` explicitly includes `LIVE` and `MATCHED` because `record_submitted()` transitions the order directly to the final status (LIVE/MATCHED/REJECTED) based on the CLOB response — it does not walk through SUBMITTED. A previous version of the table only allowed `{SUBMITTED, REJECTED}` from CREATED, which caused the `_transition` guard to log "Invalid transition CREATED → LIVE. Applying anyway" on literally every accepted order. That turned the guard into noise and masked any real state-machine bug. Do not re-narrow this set — either the direct path has to stay valid, or `record_submitted` must be split into two DB writes (CREATED→SUBMITTED then SUBMITTED→final), which is extra IO for no behavioural gain.

Feed sources: (1) direct API responses, (2) UserWebSocket fill events, (3) periodic polling backup. `record_submitted()` logs CLOB ACCEPTED/REJECTED status per order (with order ID and CLOB order ID). `record_fill()` logs fill events with 💵 emoji, outcome, price, fill progress, fee, and market title. `poll_active_orders()` guards against `None` responses from `get_order()` (the CLOB API can return `None` for orders it doesn't recognize) — without this, the next `.get("status")` call crashes the poll loop every 30 seconds.

**V2 fee accounting**: `record_fill(clob_order_id, fill_size, fill_price, fee_usdc=0.0, context=None)` accepts the explicit pUSD fee from each fill event. `usdc_spent` stays defined as gross (size × price); `fee_usdc` accumulates additively into a separate `live_orders.fee_usdc` column, and `live_positions.total_fees_usdc` aggregates across all fills feeding the position. Net wallet outflow per order is `usdc_spent + fee_usdc`. Most fills are zero — the bot places GTC limit orders that mostly rest on the book and fill as **maker**, and only takers pay V2 fees. The same accumulation logic is applied in `_apply_post_cancel_fill()` so post-cancel fills update `fee_usdc` alongside `size_filled` / `usdc_spent`.

**Concurrent fill protection**: WebSocket and poll can fire for the same fill simultaneously. Both read `size_filled=0`, both compute `new_filled=5`, both write — position gets 10 phantom shares. Fixed with per-order `asyncio.Lock` in `record_fill()`. The DB is re-read inside the lock for fresh state.

**Per-market position lock**: In spread regime, Yes and No orders for the same market can fill simultaneously. Both acquire their own per-order lock (different keys), both read the same position, last write wins. Fixed with a second lock level — `_position_locks[condition_id]` — around `_update_position()`.

**Fill clamping**: `fill_size` is clamped to `max(0, total_size - already_filled)`. Without this, duplicate events or bad data inflate positions with phantom shares.

**Position aggregation**: Position side is determined by `outcome_name` (Yes/No), NOT by order `side` (BUY/SELL). BUYing a "No" token is economically a short on "Yes" — the position goes to `no_shares`. The naive approach of keying on BUY/SELL corrupts position tracking: a "No" token BUY would be recorded as `yes_shares`.

**Reconciliation on startup**: Compares local DB against CLOB API. For orphans (on CLOB but not in DB), the flow is **adopt-then-cancel**: `_adopt_unknown_order()` inserts a minimal DB row first (with `cancel_reason="adopted_unknown"`), the executor cancels on the CLOB, and `record_cancelled(reason="startup_orphan")` flips the row to CANCELLED. The adoption step closes the race where a trailing fill lands during the cancel call — the fill has a real DB row to attach to via the normal fill path instead of hitting `fill_unknown_order`. For stale orders (in DB but not on CLOB), fill reconciliation computes `new_fill = size_matched - already_filled` to avoid double-counting partial fills that were recorded before the crash.

**Post-cancel fill reconcile**: When a fill arrives on an order whose local status is already `CANCELLED`, `_record_fill_inner()` routes to `_apply_post_cancel_fill()` instead of dropping the event. That helper re-reads `size_filled` / `usdc_spent`, clamps the incoming fill to remaining size, writes the accounting columns via `update_live_order()` (status stays CANCELLED — no `_transition` call, since CANCELLED is terminal), and feeds the position via `_update_position()` under the per-market position lock. Emits a `POST_CANCEL_FILL` audit event and a `💸🪦 POST-CANCEL FILL` log line. Net effect: the CLOB-side fill is captured in both the order row and the position even though the order cannot be resurrected. The `fill_on_cancelled` counter ticks for visibility but the money is preserved.

**Adoption for unknown fills**: When a fill arrives for a `clob_order_id` not in the DB, `_record_fill_inner()` calls `_adopt_unknown_order()` before deciding to drop. Source preference, in order:
  1. **`context` kwarg from `record_fill()`** (`via=context`): the fill WS event (`user_ws._handle_trade`) now plumbs `asset_id` (token_id), `market` (condition_id), `side`, `outcome`, and `market_slug` into `record_fill(..., context=...)`. The adoption helper synthesizes a minimal `LiveOrder` row directly from these fields. This is the primary path — it does NOT require a CLOB round-trip.
  2. **`executor.get_order(clob_order_id)`** (`via=get_order`): fallback for callers that don't pass context (e.g., `reconcile_on_startup`). Note: Polymarket's `/data/order/{id}` endpoint returns an empty body (no exception) for orders that have already been matched out of the active tier, so this fallback silently fails exactly when we most need it. The context path exists specifically to bypass this.
  3. **Last-resort synth from fill params alone**: if both above fail but `fill_size > 0`, a row is built with `size = fill_size` and a best-effort `token_id`. Bails on missing `token_id` (NOT NULL constraint) rather than crashing.

Every successful adoption logs `🩹 ADOPTED unknown CLOB order as id=N status=S via=X`, emits an `ADOPT_UNKNOWN_ORDER` audit event (with `source` in `details`), and ticks `adopted_unknown_order`. Additionally, when `_adopt_unknown_order` is called from `reconcile_on_startup` (source="startup"), the discriminator counter `adopted_unknown_order_startup` also ticks. The historical counter is kept verbatim so log archives and runbook baselines remain comparable across versions; the discriminator lets you subtract restart-driven adoptions from the total to isolate mid-session WS-fill races (`fill-race count = adopted_unknown_order − adopted_unknown_order_startup`). The original order is cancelled by the sweeper (see `stale_sweep_adopted`) or by startup reconcile on the next restart — adopted orders are NOT managed by the live poller, since they have no `dryrun_order` parent to drive updates from.

**Cancel-replace**: `cancel_replace()` checks both the CLOB cancel response AND local DB status for "already filled" signals. The local DB alone is insufficient — the WebSocket may not have updated the status yet between the cancel call and the DB check.

**Audit log serialization**: The `_audit()` method writes to the `live_audit_log` JSONB column. All `details` dicts are recursively sanitized via `_jsonb_safe()` before insertion — `datetime` objects are converted to ISO strings, `Decimal` to float. Without this, `datetime` objects from `_now()` (used in `created_at`) cause `TypeError: Object of type datetime is not JSON serializable` on PostgreSQL JSONB serialization, which kills the audit INSERT and aborts the order lifecycle — the order is created in DB but never submitted to the CLOB (zombie order).

### Component 4: UserWebSocket

**File**: `src/execution/user_ws.py`

WebSocket client for `wss://ws-subscriptions-clob.polymarket.com/ws/user` (host unchanged across V1/V2). Authenticates with CLOB API key/secret/passphrase (under V2 derived via `create_or_derive_api_key()`; the returned ApiCreds shape — `api_key` / `api_secret` / `api_passphrase` — is unchanged across V1/V2). Pushes fill and placement events to OrderTracker callbacks.

Reconnects with exponential backoff (max 120s). 30s ping keepalive. Max 500 market subscriptions per connection. When new markets are subscribed, sets a reconnect flag (Polymarket doesn't support dynamic subscribe).

Handles both maker fills (via `maker_orders[]`) and taker fills (fallback for immediately-marketable orders). Numeric fields from WebSocket events are wrapped in `try/except` for `ValueError` — malformed data must not crash the connection. The V2 `fee` field is also coerced via a non-negative-float helper (`_coerce_fee`) so a missing or malformed value defaults to 0 rather than blocking the underlying fill.

Each fill dispatches to `OrderTracker.record_fill()` with three things:

1. The numeric fill (`clob_order_id`, `fill_size`, `fill_price`).
2. **V2 explicit fee**: `fee_usdc=` extracted from the fill event's `fee` field (maker-level when present, event-level otherwise). Most maker fills carry zero; takers carry a positive value.
3. A `context` kwarg carrying the fields the trade event provides: `asset_id` (token_id), `market` (renamed to `condition_id`), `side`, `outcome`, and `market_slug`/`title` if present. The tracker's adoption path (`_adopt_unknown_order`) uses this context to synthesize a DB row for unknown orders without a CLOB round-trip. Maker-level fields (`maker.asset_id`, `maker.outcome`, `maker.side`) override the event-level equivalents when present, since a single trade event can contain multiple maker fills on different sides.

### Component 5: ExecutionManager

**File**: `src/execution/manager.py`

Central coordinator. Uses a **reserve/place split** architecture to serialize risk checks without blocking CLOB submissions.

**Per-market lock** (`_market_locks`, keyed by `condition_id`): An `asyncio.Lock` per market serializes the risk-check → DB-insert window. This prevents the 8-enrichment-worker race condition where multiple workers pass the risk gate before the first worker's order reaches the DB. The lock covers only the fast reservation phase (~5ms), not the slow CLOB signing + submission (~1.2s).

**Two-phase order submission**:
- `_reserve_order()` / `_reserve_order_inner()` — runs INSIDE the lock: risk check + CLOB min-size bump + `record_created()` (DB insert). Returns `(order_id, effective_size)`. Worker 2 waits for worker 1's DB insert, then its risk check sees worker 1's committed cost and rejects on per-market exposure. Both `check_order()` and `record_created()` MUST remain inside the same lock acquisition — moving either one outside re-opens the race condition (8 workers pass risk checks before any DB insert lands, creating 8x duplicate orders).
- `_place_order()` — runs OUTSIDE the lock: EIP-712 signing + CLOB submission + `record_submitted()`. Uses `post_only=True`. If postOnly rejects (bid crossed the spread), retries up to 5 times (`POST_ONLY_MAX_RETRIES`): fetches `best_ask` via `executor.get_best_ask()`, places at `best_ask - tick`, revalidates edge (`consensus_prob - new_price > 0`), and abandons if edge is gone, price hits zero, or retries exhausted. Each retry briefly re-acquires the market lock for its `record_created`. Logs `⚠️⚠️⚠️⚠️` on every retry. Returns `bool` — `True` iff the final attempt was CLOB-accepted (no `errorMsg`). Callers that route safe-replace branches off acceptance rely on this signal.

Two entry points per regime:

- `execute_spread_orders()` — submits Yes + No BUY with equal shares. Each side goes through reserve (under lock) → place (outside lock) independently. Risk rejection of one side doesn't block the other. A HALT on either side stops both (checked via `self._risk.is_halted` after each reservation). No dedup guard — spread signals are less prone to rapid-fire duplication because they require two-sided price convergence, and the per-market risk limit catches excess exposure.
- `execute_directional_orders()` — single heavy-side BUY with full budget. The dedup guard, pre-place cleanup, risk check, and DB insert all run inside the lock. CLOB submission runs outside.
  - **Condition-level dedup**: fingerprints each submission as `(condition_id, heavy_side)` with a 60-second cooldown (`DEDUP_COOLDOWN_S`). Catches sequential duplicates from rapid rn1 trades; the lock catches parallel duplicates from concurrent workers. Both are needed. The key deliberately excludes `consensus_prob` and `bid_price` — when multiple rn1 trades arrive in a burst on the same market, each produces slightly different poly prices (the CLOB moved between trades), but they represent the same signal. The naive alternative of including `round(consensus_prob, 4)` and `round(bid_price, 4)` in the key lets burst-of-same-signal through (5–23 concurrent orders per market), because tiny poly price differences between consecutive trades produce different fingerprints.
  - **Match-level dedup**: fingerprints each submission as `(_match_slug(market_title), heavy_side)` with a 30-second cooldown (`MATCH_DEDUP_COOLDOWN_S`). Prevents the same directional bet on different O/U lines or handicap lines of the same match (e.g., O/U 2.5 Yes + O/U 3.5 Yes within 30s, or Spread: Arsenal (-1.5) + Spread: Arsenal (-0.5)). `_match_slug()` strips O/U/BTTS suffixes (after `:`) from market titles and extracts the team name from `"Spread: Team (±N)"` format — "FC Barcelona vs. Club Atlético de Madrid: O/U 3.5" and "FC Barcelona vs. Club Atlético de Madrid: O/U 2.5" produce the same slug; "Spread: Arsenal (-1.5)" and "Spread: Arsenal (-0.5)" both produce "arsenal". Moneyline titles ("Will X win on date?") don't match the suffix/spread patterns and produce unique slugs, so they are NOT blocked by O/U or handicap orders on the same match — this is by design since they are different market types with different risk profiles.
  - **Pre-place cleanup**: cancels all unfilled orders on the same `condition_id` — both opposite-side (side-flip) and same-side (replacing orphan from a previous signal at a different price). Uses `get_pending_or_active_orders_by_condition()` which includes CREATED/SUBMITTED orders, not just LIVE/PARTIAL_FILL — this closes the race window where an in-flight cancel-replace replacement hasn't reached LIVE status yet. After each cancel, re-reads the order from DB — if a WebSocket fill marked it `FILLED`/`MATCHED` between the status query and the cancel call, `record_cancelled()` is skipped to avoid overwriting the fill. **Partial-fill protection**: same-side orders with `size_filled > 0` are kept (not cancelled) to preserve position tracking for already-filled shares. Cancel failures are logged but do not block the new order.

Both entry points check `_shutting_down` and `_risk.is_halted` early and return immediately. Without the `is_halted` check, concurrent calls that passed the `_shutting_down` guard before a HALT would still reach the risk manager and get noisy rejection logs instead of being silently short-circuited.

`update_bid()` performs cancel-replace for same-direction repricing. Accepts orders with status `LIVE` or `PARTIAL_FILL` — without PARTIAL_FILL, partially-filled orders are silently skipped even though they have unfilled remainder on the book. Under `SAFE_REPLACE_ENABLED` (default on) and for non-flip updates, it delegates to `_safe_update_bid()`. Directional flips (heavy outcome changes) still enter `_cancel_and_place_flipped_inner()`; the safe-flip branch there embeds `_safe_old_order_id` / `_safe_old_clob_id` sidecar keys into `_pending_flip_placement` so the outer caller can run the old-order cancel after `_place_order()` confirms acceptance. If the flag is off, both paths fall back to the legacy cancel-first flow.

`cancel_and_place_flipped()` is the direct entry for contrarian direction flips (used by `LiveOddsPoller._trigger_bid_update()`). Acquires the market lock, runs `_cancel_and_place_flipped_inner()`, releases the lock, calls `_place_order()` outside the lock, and — under safe-flip — runs `_retire_flipped_old_order()` only if `_place_order()` returned True.

### Safe Cancel-Replace

`SAFE_REPLACE_ENABLED` (env var, default `true`) flips the order of operations at all three cancel-replace sites. Instead of `cancel old → submit new` (where a post-only rejection on the new submit orphans the market), the flow is `submit new → on acceptance, cancel old`. If the new submit throws or returns `errorMsg`, the old order stays live; only the new order record is transitioned to REJECTED.

Three sites all follow the same pattern:

1. **`_safe_update_bid()`** (same-direction repricing on `update_bid`): reserves the new DB record under the market lock, releases the lock, submits to CLOB. On acceptance, calls `executor.cancel()` on the old `clob_order_id` then `record_cancelled(reason="cancel_replace")` on the old DB id. On rejection (exception or errorMsg), only records the new order as REJECTED — old stays LIVE.

2. **`_cancel_and_place_flipped_inner()` safe-flip branch** (contrarian flip): reserves the new flipped order in DB under the lock without cancelling the old. Embeds `_safe_old_order_id` and `_safe_old_clob_id` into `_pending_flip_placement` as sidecar keys. Both outer callers (`update_bid` flip branch, `cancel_and_place_flipped` direct entry) pop these keys, call `_place_order()`, and on acceptance invoke `_retire_flipped_old_order()`. The helper attempts `executor.cancel()` on the old CLOB id. If the CLOB cancel succeeds, it marks the old DB row CANCELLED with reason `contrarian_flip` and increments `side_flip_cancelled`. **If the CLOB cancel fails after the new order is live, the DB row is left LIVE** — stale-sweep and reconciliation handle it later. Marking CANCELLED in DB when CLOB still thinks the order is live would create state drift worse than a delayed cleanup.

3. **`execute_directional_orders()` pre-place cleanup**: the pre-place loop collects conflicting orders (opposite-side flips, same-side replacements) into a `deferred_cancels` list instead of cancelling them immediately. After `_place_order()` returns `True`, `_run_deferred_cancels()` iterates the list and performs the CLOB cancel + DB record_cancelled + counter increment for each target. If `_place_order()` returns `False`, none of the targets are cancelled and all prior orders stay live on the book.

`_place_order()` returns `bool` specifically to enable this routing — `True` iff a CLOB-accepted order exists after the (possibly retried) submit sequence. Callers that don't need the signal (e.g., `execute_spread_orders()`) can ignore it.

**Log markers** (all 🛡️ prefix): `🛡️🪜 Safe-replace submit threw` / `🛡️🪜 Safe-replace rejected by CLOB` (old preserved), `🛡️🔄 Safe-flip: reserved new ... will be cancelled on acceptance` (entering safe-flip), `🛡️🔄 Safe-flip complete: old retired after new accepted` (happy path), `🛡️🗑️ Safe-flip: CLOB cancel of old failed after new accepted` (state-drift-avoidance case).

**Guardrail — never invert this order again.** The original `cancel → submit` flow is plausible-looking but catastrophically wrong: when the book is fast-moving and the new bid would cross, the CLOB post-only check rejects the submit AFTER the old cancel has already landed, leaving the market with no live order at all. Production observed this as orphaned markets across multiple flips. The `cancel_then_reject` heartbeat counter and paired `Order CANCELLED reason=contrarian_flip` → `CLOB REJECTED` log sequences are direct evidence. Do not revert to cancel-first even if a caller's control flow looks simpler — the `submit → cancel` asymmetry is the whole point.

**Consequence — transient two-order window.** Between the new order's acceptance and the old order's cancel on CLOB, the market briefly has two live orders from the bot. Cardinality enforcement (`enforce_single_order_per_outcome`) catches this on the next poll if the cancel step lags. For opposite-side flips this is a micro self-hedge window (milliseconds); for same-side replacements it's two bids at different prices. Neither has been observed to cause issues in practice — the alternative (an orphan) is strictly worse.

Sub-minimum orders (< 5 shares) are bumped to `CLOB_MIN_SIZE` in `_reserve_order_inner()` — this ensures the order lands on the book at a marginally higher cost instead of being silently dropped.

`shutdown()` calls `cancel_all()` with individual-cancel fallback, marks all active orders (LIVE, PARTIAL_FILL, SUBMITTED, CREATED) as cancelled in DB, stops WebSocket and polling tasks.

`_emergency_halt()` cancels all active orders including SUBMITTED and CREATED (not just LIVE/PARTIAL_FILL). Without including pre-live states, the DB records would stay in non-terminal state even though `cancel_all()` cancelled them on the CLOB.

Startup: reconciles stale orders, launches background poll loop (30s), UserWebSocket task, stale-order sweeper (60s interval, cancels any LIVE/PARTIAL_FILL order older than 10 minutes via `_sweep_stale_orders()`), and the observability heartbeat loop (5-min interval, see "Observability" section). Under GTC there is no CLOB-side expiry; the sweeper is the hard ceiling that guarantees no order from this bot sits past `MAX_ABSOLUTE_ORDER_AGE_S` regardless of poller state.

### Component 6: LiveOrderStore

**File**: `src/storage/live_store.py`

Database adapter implementing the `OrderStore` and `RiskDataStore` protocols expected by OrderTracker and RiskManager. Wraps the main Database's `async_sessionmaker` to operate on `live_orders`, `live_positions`, and `live_audit_log` tables. Terminal statuses (REJECTED, CANCELLED, SETTLED) are defined once as `_TERMINAL_STATUSES` and shared across all queries.

Key methods:
- `count_active_markets()` — counts distinct `condition_id` from non-terminal orders
- `get_market_exposure()` — sums committed cost (`price × size`) from non-terminal orders per market
- `get_active_orders_by_condition(condition_id, outcome_name=None)` — returns LIVE/PARTIAL_FILL orders for a condition_id, ordered by `size_filled DESC, created_at DESC` (best survivor first). Used by cardinality enforcement and targeted live order queries
- `get_pending_or_active_orders_by_condition(condition_id, outcome_name=None)` — returns all non-terminal orders (CREATED, SUBMITTED, LIVE, PARTIAL_FILL) for a condition_id. Used by the pre-place cleanup loop in `execute_directional_orders()` to catch in-flight cancel-replace replacements that haven't reached LIVE status yet — without this, the wider query, a replacement order created by `update_bid()` under the lock but not yet submitted to the CLOB would be invisible, allowing a duplicate
- `get_stale_live_orders(max_age_seconds=600)` — returns LIVE/PARTIAL_FILL orders older than the threshold using the live order's `created_at` (not dryrun's, which resets on every bid update). Returns each order with computed `age_seconds`. Used by the stale-order sweeper

Each method opens its own session and commits. This means a crash between an order update and its position update leaves the position understated until the next fill or reconciliation. Acceptable for v1 — the reconciliation on startup catches inconsistencies.

### Component 7: Observability

**File**: `src/execution/observability.py`

Process-wide instrumentation shared by `manager.py`, `order_tracker.py`, `executor.py`, `user_ws.py`, `live_odds_poller.py`, and `universal_odds.py`. Three artefacts:

- **`COUNTERS`** (`_OrderCounters` singleton): an in-memory `collections.Counter[str]` of event tallies. Increments via `COUNTERS.increment(name)`. Never persisted — counters reset on process restart, which is intentional: a restart under a new code version must start from a clean baseline so counter movement attributes cleanly to the new build.
- **`heartbeat_loop(interval_s=300)`** (async task): every 5 minutes, logs one multi-line snapshot of totals and deltas since the previous snapshot, sorted by total descending. Output prefix `📊`. Started from `ExecutionManager.start()` as `self._heartbeat_task`. First heartbeat after startup shows "no counters incremented yet" when the process hasn't emitted any events yet.
- **`tag(order_id=, clob_order_id=, condition_id=, market_title=)`**: returns a compact `[ord=N clob=0x... cond=0x... mkt='...']` suffix appended to every lifecycle log line, so a single order's full history is one grep. Truncates `clob_order_id`/`condition_id` to 18 chars and `market_title` to 30 chars to match existing log style.

Counter names (kept in sync with `KNOWN_COUNTERS` tuple in `observability.py`).

**Lifecycle counters** — referenced from the prose above, follow the order/fill/cancel-replace flow:

| Counter | Site | Meaning |
|---|---|---|
| `fill_unknown_order` | `order_tracker.record_fill` | fill arrived for an order with no DB row AND no WS context AND no CLOB record — a true phantom. Adoption absorbs every non-phantom case, so non-zero is genuine ledger drift |
| `fill_on_cancelled` | `order_tracker.record_fill` | fill arrived after local cancel. The `_apply_post_cancel_fill` path applies it to `size_filled`/`usdc_spent`/position — counter marks the race, not dropped money |
| `fill_on_terminal_nonfinal` | `order_tracker.record_fill` | fill arrived on REJECTED or SETTLED. Dropped by design — no money moved on REJECTED, SETTLED already accounted for |
| `fill_on_already_full` | `order_tracker.record_fill` | fill arrived when order already at target size |
| `fill_clamped` | `order_tracker.record_fill` | fill size exceeded remaining; clamped (usually duplicate WS delivery) |
| `adopted_unknown_order` | `order_tracker._adopt_unknown_order` | total adoption count — minimal DB row synthesized so the fill applied instead of being dropped. `via=context` (WS event) or `via=get_order` (CLOB fallback) in the log line |
| `adopted_unknown_order_startup` | `_adopt_unknown_order` from `reconcile_on_startup` | discriminator subset (subset, never disjoint): ticks when adoption was triggered by startup reconciliation finding an orphan from a prior crashed session. `adopted_unknown_order − adopted_unknown_order_startup` = true WS-fill-race count |
| `invalid_transition` | `order_tracker._transition` | state machine violation, applied anyway. Should stay 0 under current `VALID_TRANSITIONS` |
| `cancel_then_reject` | `manager.update_bid` replacement path | cancelled old then replacement rejected (ORPHAN). Should stay 0 under safe-replace |
| `post_only_reject` | `manager` submit paths + `executor._run_sync` | CLOB rejected submit because post-only would cross the book. Under safe-replace no longer implies an orphan |
| `stale_sweep_cancelled` | `manager._sweep_stale_orders` | age-based safety-net cancelled an order > 10 min old (strategy + adopted combined) |
| `stale_sweep_adopted` | `manager._sweep_stale_orders` | subset of above where the swept order was adopted (not originated by this bot's strategy) |
| `side_flip_cancelled` | `manager` pre-place loop + `_retire_flipped_old_order` | opposite-side cancel for directional flip (legacy pre-place cleanup + safe-flip retire) |
| `replace_cancelled` | `manager` pre-place loop + `_run_deferred_cancels` | same-side cancel-replace on new signal |
| `edge_gone_cancel_fired` | `live_odds_poller._trigger_bid_update` | live CLOB order cancelled because edge disappeared |
| `edge_gone_cancel_failed` | `live_odds_poller._trigger_bid_update` | edge-gone cancel threw |
| `token_resolved_via_clob` | `live_odds_poller._resolve_token_via_clob` | flip token resolved by hitting `/markets/{cond}` after DB + sibling lookup both missed. High values suggest upstream DB token ingestion is behind |
| `flip_stale_cp_edge_fallback` | `manager._cancel_and_place_flipped_inner` | caller invoked the flip path without `new_consensus_prob` / `new_edge`; the helper persisted old-row values and the new row will fail the `edge == cp − price − spread_level` audit. Should stay 0 — every flip caller is expected to pass new-side values (Bug 2) |
| `flip_blocked_disable_cancellations` | `live_odds_poller._trigger_bid_update` | contrarian flip suppressed by `LIVE_DISABLE_CANCELLATIONS` (kill switch on by default). Live AND dryrun mutations are skipped; the resting old-side order survives until fill or stale-sweep |
| `edge_gone_blocked_disable_cancellations` | `live_odds_poller._trigger_bid_update` | edge-gone cancel suppressed by `LIVE_DISABLE_CANCELLATIONS` |
| `bid_update_blocked_disable_cancellations` | `live_odds_poller._trigger_bid_update` | same-direction cancel-replace suppressed by `LIVE_DISABLE_CANCELLATIONS` (live skipped; dryrun in-place update still runs so reporting stays honest) |
| `side_flip_blocked_disable_cancellations` | `manager.execute_directional_orders` pre-place loop | opposite-side cleanup suppressed by `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS` (kill switch on by default). New heavy-side order still submits; same-side replace cleanup is unaffected |
| `fetch_odds_cache_hit` | `universal_odds.fetch_odds` | served from D (10-second burst TTL). Dominates under partial-fill storms — expected nonzero during live soccer |
| `fetch_odds_poller_hit` | `universal_odds.fetch_odds` | served from A (poller handoff via `set_poller()`). Every increment is one saved per-event `/odds` call |
| `fetch_odds_poller_hit_h2h` | `universal_odds.fetch_odds` | discriminator subset of `_poller_hit`: ticks when `"h2h"` is in `markets` (incl. spread-narrowing `["h2h", "spreads"]` — h2h takes precedence). Primary smoke test for orchestrator narrowing — must climb after deploy |
| `fetch_odds_poller_hit_spreads` | `universal_odds.fetch_odds` | discriminator subset: spreads-only request served from A |
| `fetch_odds_poller_hit_other` | `universal_odds.fetch_odds` | discriminator subset: A-hit on a request with no `h2h` or `spreads`. Defensive — should stay near zero |

**Diagnostic counters** (heartbeat lookup; not narrated above) — bookmaker breadth, staleness/frozen attribution, cancel-replace gating, cycle health. All ticked from `live_odds_poller`, `consensus_odds`, or `executor` and surface via the 5-min heartbeat.

| Counter family | Members | Site | Meaning |
|---|---|---|---|
| `consensus_books_*` | `_le_3` / `_4_5` / `_6_7` / `_ge_8` | `consensus_odds.extract_consensus_*` (success-return) | one tick per extraction, bucketed by `len(bookmakers_used)`. Steady state under the 9-book request set: `_ge_8` ~55%, `_6_7` ~35%, `_le_3` ≤8%, `_4_5` near zero. Persistent `_le_3` >10% means the widening didn't take effect or a requested key is dead |
| `l1_excluded_<book>` | one per named book + `_other` | `consensus_odds.BookmakerTracker.update` | edge-triggered: one tick per `(event_id, market_type)` fresh→stale transition. Named books: `pinnacle`, `bet365`, `betfair_ex_eu`, `betfair_ex_uk`, `matchbook`, `draftkings`, `fanduel`, `williamhill` (covers `williamhill_us` too), `paddypower`, `unibet_eu`, `unibet_uk`. `_other` is the catch-all for any consensus-whitelisted name not in that list (`betfair`, `unibet`, alternates) — sustained nonzero suggests the trusted-book whitelist drifted from the canonical request list. `bet365` and `unibet_eu` are retired keys, never tick under current `settings.BOOKMAKERS` |
| `l1_consensus_unavailable` | — | `consensus_odds.calculate_consensus_prob` | returned `None` because every trusted bookmaker was excluded. Ticks during goal/VAR clusters and league-wide suspensions |
| `l2_frozen_*` | `_h2h` / `_totals` / `_handicap` | `live_odds_poller.MatchOddsTracker.update[/_handicap]` | edge-triggered: one tick per channel non-frozen→frozen transition. Handicap is fed from `_trigger_bid_update()` per-order, not the poll loop |
| `poller_cycle_completed` | — | `live_odds_poller.poll` | one tick per full cycle. Heartbeat delta = number of league-poll sweeps in the window |
| `poller_cycle_overrun` | — | `live_odds_poller.poll` | wall-clock cycle exceeded `pinnacle_poll_live_seconds` (15s). Leading indicator of parse-time creep; should stay 0 |
| `cancel_replace_cooldown_skip` | — | `live_odds_poller._trigger_bid_update` | 15s per-market cooldown suppressed a bid-update OR flip attempt (shared) |
| `cancel_replace_delta_gate_skip` | — | `live_odds_poller._trigger_bid_update` | passed cooldown but `abs(new_bid − live_price) < MIN_CANCEL_REPLACE_PRICE_DELTA`. Without this counter the cooldown-vs-delta funnel was unattributable |
| `edge_gone_had_live_order` | — | `live_odds_poller._trigger_bid_update` | stale-edge event with a live counterpart (attribution split for `edge_threshold_skip_update`) |
| `edge_gone_no_live_order` | — | `live_odds_poller._trigger_bid_update` | stale-edge event with no live counterpart (expected inflation from dryrun-only markets) |
| `edge_threshold_skip_update` | — | `live_odds_poller._trigger_bid_update` | bid update skipped because computed edge > max-edge sanity cap |
| `flip_skipped_token_unresolved` | — | `live_odds_poller._trigger_bid_update` | flip aborted: opposite-side `token_id` couldn't be resolved (DB + sibling + CLOB fallback all missed) |
| `flip_skipped_edge_too_high` | — | `live_odds_poller._trigger_bid_update` | flip aborted: flipped-side edge > sanity cap |
| `clob_rate_limit_hit` | — | `executor._run_sync` | HTTP 429 on any signed-order call. Observes only — no retry yet; nonzero is the trigger to add exponential backoff |

Every order-lifecycle log line in the execution layer calls `tag(...)` with the relevant IDs. This lets `findstr /C:"ord=14" logs\bot-YYYYMMDD.log` (or `grep`) return that single order's full life: ingest → engine decision → create → submit → accept/reject → fill → cancel.

**Guardrail — counter naming.** If you add a new counter, append it to both `KNOWN_COUNTERS` and the `describe_counter()` switch in `observability.py`. The heartbeat's human-readable suffix (after the `—`) comes from that switch; without an entry the heartbeat shows the raw name only. Existing counters have fixed names because they're parsed from logs by the remediation runbook — don't rename. When you need per-source attribution on an existing counter, add a **discriminator subset** (e.g., `adopted_unknown_order_startup` ticks in addition to `adopted_unknown_order`, not in place of it) so the historical total remains comparable and the subset is recovered by subtraction. Discriminators are subsets, never disjoint siblings — every site that ticks a subset must also tick the parent total in the same call, or the subtraction formula in the counter table silently breaks. The kill-switch counters (`*_blocked_disable_cancellations`) and `flip_stale_cp_edge_fallback` are currently incremented but missing from `KNOWN_COUNTERS` / `describe_counter()` — heartbeat shows them by raw name only. Adding them to both is mechanical follow-up.

**Guardrail — post-cancel fills must be reconciled, not dropped.** `_record_fill_inner` routes fills on CANCELLED orders through `_apply_post_cancel_fill`, which updates `size_filled` / `usdc_spent` / `fee_usdc` / position without calling `_transition` (CANCELLED is terminal). The naive "return early if terminal" pattern silently drops money — the CLOB fill already happened on-chain. If you refactor this path, preserve: (a) the status stays CANCELLED, (b) clamping still applies to prevent duplicate-delivery overflow, (c) the audit event (`POST_CANCEL_FILL`) fires with the fee captured in `details.total_fee`, (d) the per-market position lock wraps `_update_position` and `total_fees_usdc` accumulates on the position. REJECTED fills ARE still dropped by design — no money moved on a rejected order.

**Guardrail — adopt unknown orders from WS context, not `get_order`.** `_adopt_unknown_order` prefers the WS fill event's own fields (`asset_id`, `market`, `side`, `outcome`) over calling `executor.get_order(clob_order_id)`. The CLOB `/data/order/{id}` endpoint returns an empty body (not an exception) for orders that have already been matched out of the active tier — exactly the orders we most need to adopt. Production observed 0 successful adoptions across dozens of unknown fills when the code relied on `get_order` alone. If you refactor adoption, keep the context path as the primary source; `get_order` is the fallback, not the other way around.

### Lifecycle & Wiring

On `--live` startup:
1. `orchestrator._init_execution_layer()` builds the full component graph: bootstrap `ClobClient(host, chain_id, key)` derives API creds via `create_or_derive_api_key()`, then a full `ClobClient(host, chain_id, key, creds, signature_type, funder)` is constructed → `PolymarketExecutor` → `RiskManager` → `OrderTracker` → `UserWebSocket` → `ExecutionManager`
2. The derived `ApiCreds` object exposes `api_key` / `api_secret` / `api_passphrase` (unchanged shape across V1/V2; the V2 method name is what changed). The orchestrator uses `isinstance(api_creds, dict)` to handle both forms safely. The WebSocket auth message uses camelCase keys (`apiKey`/`secret`/`passphrase`). Existing CLOB API creds work without re-derivation across the V1→V2 cutover.
3. Live DB tables are created via `LiveBase.metadata.create_all`. The `live_orders.fee_usdc` and `live_positions.total_fees_usdc` columns must exist; the schema migration `migrations/v2_add_fee_columns.sql` is the operator's responsibility before first `--live` run.
4. `ExecutionManager.start()` runs reconciliation, launches WS + poll + stale-sweep + heartbeat tasks (all tracked in `self._poll_task`, `_ws_task`, `_sweep_task`, `_heartbeat_task` for clean shutdown)
5. The `execution_manager` is passed to `DryRunEngine` via constructor (for order dispatch). It is set on `LiveOddsPoller` via attribute assignment after construction (`self.dryrun_engine.live_odds_poller.execution_manager = execution_manager`), not via constructor — the poller is created inside the engine before the execution manager exists
6. On shutdown, `execution_manager.shutdown()` runs first (cancels all CLOB orders, cancels all four background tasks), then IntelBot's normal cleanup

The enrichment pipeline in the orchestrator extracts both Yes and No token IDs from `market_info.raw_data["tokens"]` by matching outcome labels ("yes"/"0" → Yes, "no"/"1" → No). If label matching fails and the market has exactly 2 tokens, a positional fallback assigns index 0 → Yes, index 1 → No (Polymarket convention). This fallback is critical for O/U markets where token labels may be "Over"/"Under" instead of "Yes"/"No" — without it, token extraction fails and live dispatch is skipped with "missing token ID". Both `source_exchange`, `token_id_yes`, and `token_id_no` are passed through the engine's method chain to the live dispatch point.

**No self-trade filter needed**: WalletMonitor topic-filters on rn1's address only — our wallet is a different address, so our own CLOB fills never enter the pipeline. An explicit `maker_address` check was tried and removed (commit `dbd754a`) as dead code.

### Latency Budget

```
rn1 trade detected         → 0ms
enrichment (parallel)      → ~400ms
order signing (EIP-712)    → ~1,000ms
CLOB API roundtrip         → ~200ms
                             ────────
total                      → ~1,600ms after rn1's trade
```

This is after rn1 has already acted. Acceptable for a market-making strategy (orders rest on the book). The bot is never front-running.

### Key Config (see `config/live_settings.py`)

Env-var knobs (`LIVE_*`, plus `SAFE_REPLACE_ENABLED`) are canonical in §"Configuration Files" below. This list covers hardcoded module-level constants and behavioural defaults not exposed via env:

- Edge threshold 2% (regime split, matches dryrun) | Spread offset 0.8% | Directional spread level 1.0% (single level, not 4 like dryrun)
- Risk halts: $25/hr, $50/day, 30% drawdown | Price bounds [0.02, 0.98]
- Poll interval 30s | Order type GTC + `post_only=True`, no server-side expiry
- Order staleness: 3 min live / 15 min pre-game age-gated re-validation
- Cancel-replace gates: $0.01 min price delta, 15s per-market cooldown (shared between bid-update and flip paths; matches `pinnacle_poll_live_seconds` so at most one update per poll cycle)
- Dedup cooldowns: 60s condition-level, 30s match-level (cross-O/U-line)
- Shadow dryrun enabled by default (live decisions also written to dryrun tables)

Operator-tunable risk knobs (`max_concurrent_markets`, `capital_per_market`, `max_per_market_usd`) are described in "Operator-tunable risk knobs (live trading)" above. The `MAX_CONCURRENT_MARKETS` env var is shared with `dryrun_settings.py` (see Known Limitations #16).

### Known Gaps

- **Live position settlement**: No component settles `live_positions`. The `ResolutionWatcher` only knows about `dryrun_positions`. When market resolution events are detected, live positions remain with blank `pnl`/`settled_at`. This requires extending `ResolutionWatcher` or building a parallel settlement path.
- **Shadow dryrun writes**: The `shadow_dryrun` config flag exists but the actual write path (writing live decisions to dryrun tables for comparison) is not yet implemented.
- **Deadman switch**: A separate process that cancels all orders if the main process hasn't sent a heartbeat in N minutes. Not implemented as a dedicated process. Under GTC there is no CLOB-side auto-expiry. If the bot crashes, orders sit until the next process start, at which point `reconcile_on_startup()` adopts-and-cancels them. If the process is running but the sweeper loop has stalled, orders older than 10 min go uncancelled until the stall clears. Both scenarios are surfacable via the heartbeat (`stale_sweep_cancelled` absent after a long session means the sweeper is not running).
- **Age-based re-validation relies on `commence_time`**: The `_is_match_live()` heuristic uses `commence_time` to determine live vs pre-game staleness thresholds. If `commence_time` is missing, the order is treated as pre-game (conservative — 15-minute TTL instead of 3-minute). Still bounded by the 10-minute stale-sweep hard cap.
- **Adopted orders are unmanaged by the live poller**: `_adopt_unknown_order()` creates a `live_orders` row but no matching `dryrun_order` parent. Since `live_odds_poller._trigger_bid_update()` iterates open dryrun orders (not live orders), adopted orders never receive bid updates, flips, or edge-gone cancels. They die only via `_sweep_stale_orders` (10-min hard cap) or startup reconcile on the next restart. `stale_sweep_adopted` tracks the rate — if it dominates `stale_sweep_cancelled`, most of the safety-net activity is adoption cleanup rather than strategy-order staleness.
- **CLOB rate-limit has detection but no retry**: `executor._run_sync()` ticks `clob_rate_limit_hit` on any 429 response and re-raises. The underlying call fails and is handled by the caller's existing error path. If the counter ever goes non-zero, add exponential-backoff retry in `_run_sync` before the re-raise.
- **WS reconnect on new subscriptions**: Polymarket's user-channel WebSocket does not support dynamic subscribe. Every new market subscription triggers a full reconnect, creating a fill-loss window. The 30s polling backup + adoption path make this tolerable — fills landing during the gap are picked up on the next `poll_active_orders()` and can be adopted if a race puts them outside the local DB.

### Counter health baselines

Heartbeat reads. Used to verify the cancel-replace / fill-reconciliation work stays landed and to distinguish failure modes from healthy diagnostics.

**Stays at 0 in heartbeat deltas** (any nonzero is a real bug):
`cancel_then_reject`, `invalid_transition`, `edge_gone_cancel_failed`, `clob_rate_limit_hit`, `poller_cycle_overrun`, `fetch_odds_poller_hit_other`, `flip_stale_cp_edge_fallback`.

**Fires nonzero under healthy operation** (diagnostic — known-handled races, quota savings, or breadth distribution, not failure modes):
`fill_on_cancelled`, `fill_unknown_order`, `adopted_unknown_order`, `adopted_unknown_order_startup`, `stale_sweep_adopted`, `token_resolved_via_clob`, `fetch_odds_cache_hit`, `fetch_odds_poller_hit`, `fetch_odds_poller_hit_h2h`, `fetch_odds_poller_hit_spreads`, the `consensus_books_*` family, the `l1_excluded_*` family, `l1_consensus_unavailable`, the `l2_frozen_*` family.

**Fires nonzero only when kill switches are on** (default = on; expected nonzero under default config):
`flip_blocked_disable_cancellations`, `edge_gone_blocked_disable_cancellations`, `bid_update_blocked_disable_cancellations`, `side_flip_blocked_disable_cancellations`. Setting `LIVE_DISABLE_CANCELLATIONS=false` and `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS=false` returns these to the "stays at 0" set.

`poller_cycle_completed` ticks once per league-poll sweep — denominator for cycle-rate health.

---

## Critical Code Paths

| What | Where |
|------|-------|
| WebSocket connection | `wallet_monitor.py:_websocket_monitor()` |
| Topic filtering | `wallet_monitor.py:_websocket_monitor()` (inline `eth_subscribe` calls with topic arrays) |
| Key rotation (Alchemy) | `wallet_monitor.py:AlchemyKeyRotator` |
| Key rotation (TheOdds) | `soccer_odds.py:APIKeyRotator` |
| Market classification | `market_classifier.py:classify()` |
| Classifier category fallback | `market_classifier.py:_guess_category_from_teams()` — token-set membership against `_SOCCER_TEAM_TOKENS` / `_ESPORTS_TEAM_TOKENS` |
| Match audit recorder | `src/match_audit/recorder.py:MatchAuditRecorder.record()` — best-effort, 2s timeout, own sessionmaker on the shared engine |
| Match audit row build | `src/match_audit/models.py:MatchAuditRow.from_context()` — maps `TradeContext` + `MatchResult` to the `match_audit` row |
| Match audit wiring | `orchestrator.py:_enrich_universal()` calls `self.match_audit.record(ctx, result.match_result)` after `enrich_trade` returns |
| Refresh log wiring | `orchestrator.py` builds `RefreshLogRecorder` alongside `MatchAuditRecorder`, hands it to `UniversalOddsService(refresh_log_recorder=…)`, and calls `set_poller(self.dryrun_engine.live_odds_poller)` after `DryRunEngine.initialize()` so poller-cache handoff is live |
| Consensus outlier wiring | `orchestrator.py` builds `ConsensusOutlierRecorder` alongside the other audit recorders, runs `ConsensusOutlierBase.metadata.create_all`, then assigns the recorder to `dryrun_engine.live_odds_poller.consensus_outlier_log` after `DryRunEngine.initialize()`. The poller's `_record_consensus_outlier()` helper synthesizes events from `_extract_consensus()` results and calls `recorder.record(event)` |
| Orchestrator market narrowing | `orchestrator.py:_narrow_markets_for()` (module-level helper) maps `parsed_market.market_type` to the markets list passed to `enrich_trade(..., markets=...)` |
| Participant extraction | `universal_odds.py:_extract_participants()` |
| Strict event matching | `universal_odds.py:find_match()` |
| Will-win ranking | `universal_odds.py:find_match()` will-win branch — candidates ranked by `_will_win_score()` (information-fraction); top score must clear `_MIN_WILL_WIN_SCORE = 0.50` and be a strict top under `_TIEBREAKER_EPSILON = 0.01`. Confidence buckets off score + margin: high `≥0.85+0.30`, medium `≥0.65+0.20`, low otherwise (`strict_will_win_low`) |
| VS-match score gate | `universal_odds.py:find_match()` vs-market branch — `_check_vs_match` boolean check, then `_vs_match_score()` (sum of per-team scores, max over home/away). Below `_MIN_VS_MATCH_SCORE = 1.00` → `parse_stage_reached="vs_match_below_threshold"` |
| Word-rarity scoring | `universal_odds.py:_word_score()` (`1/log(1+freq)`), `_team_score()` (overlap_information / query_information), `_will_win_score()`, `_vs_match_score()`. Frequency table `self._word_frequency` rebuilt by `_rebuild_word_frequency()` at the end of every `refresh_events()` cycle |
| Per-event odds fetch cache | `universal_odds.py:fetch_odds()` — 10-second TTL keyed on `(event.id, tuple(sorted(markets)))`; failures + empty bookmakers not cached; FIFO-evicted at 500 entries; ticks `fetch_odds_cache_hit` |
| Poller cache handoff | `universal_odds.py:set_poller()` + `_read_poller_cache()` — A-cache read inside `fetch_odds` between D and API; 20s freshness preferring `bookmakers_raw_timestamp` over `timestamp`; skips when `alternate_totals` is in markets; populates D on hit; ticks `fetch_odds_poller_hit` and the `_h2h` / `_spreads` / `_other` discriminator subset |
| Poller widening (request shape) | `live_odds_poller.py:_fetch_odds()` — reads `LIVE_POLLER_REGIONS` (default `config.settings.REGIONS = "us,uk,eu"`) and `LIVE_POLLER_BOOKMAKERS` (default = `config.settings.BOOKMAKERS` joined). Empty `LIVE_POLLER_REGIONS` drops the param (TheOddsAPI defaults to US-only); empty `LIVE_POLLER_BOOKMAKERS` falls through to `settings.BOOKMAKERS` (identical to leaving the env unset) — explicit narrow rollback requires setting `LIVE_POLLER_BOOKMAKERS` to a real CSV |
| Poller cycle health | `live_odds_poller.py:poll()` — `time.monotonic()` start at entry; ticks `poller_cycle_completed` and (when duration exceeds `pinnacle_poll_live_seconds`) `poller_cycle_overrun` at the end of every full cycle |
| L1 staleness threshold | `consensus_odds.py:STALE_UNCHANGED_THRESHOLD` — `int(os.getenv("LIVE_STALE_UNCHANGED_THRESHOLD", "4"))` at import; consumed by `BookmakerTracker.update` and `is_fresh` |
| L1 exclusion counters | `consensus_odds.py:BookmakerTracker.update` — edge-triggered via per-entry `excluded` flag; calls `_l1_exclusion_counter_for(bookmaker_key)` which maps to `l1_excluded_<name>` or `l1_excluded_other` |
| Bookmaker breadth bucket | `consensus_odds.py:_bump_consensus_books_bucket(n)` — invoked at the bottom of `extract_consensus_h2h` / `extract_consensus_totals` / `extract_consensus_handicap` on the success-return path |
| L2 frozen thresholds | `live_odds_poller.py:FROZEN_UNCHANGED_THRESHOLD` (default 4, env `LIVE_FROZEN_UNCHANGED_THRESHOLD`) and `FROZEN_MOVE_TOLERANCE` (default 0.005, env `LIVE_FROZEN_MOVE_TOLERANCE`) — read at import. `MatchOddsTracker.update` and `update_handicap` ticks `l2_frozen_<channel>` on the non-frozen→frozen transition only |
| Consensus outlier recorder | `src/consensus_outlier_log/recorder.py:ConsensusOutlierRecorder.record()` — best-effort, 2s timeout, own sessionmaker; iterates `event.triggered_reasons()` and writes one row per reason. No-op when `enabled=False` |
| Consensus outlier event source | `live_odds_poller.py:_record_consensus_outlier()` — invoked from `poll()` after each h2h `_extract_consensus()` (success and `None` short-circuit). Synthesizes a `ConsensusOutlierEvent` with `bookmakers_seen` from raw response, `bookmakers_used` / `bookmakers_excluded_l1` from the consensus result, current `is_h2h_frozen` from the MatchOddsTracker, and the `consensus_none` flag from the extraction's None-return |
| Event index refresh | `universal_odds.py:refresh_events()` — generates `refresh_cycle_id = uuid.uuid4()` per invocation; dispatches per-sport via `fetch_sport_events` under `gather(return_exceptions=True)`; atomic swap of `self.events` and `self.participant_index` after all sports resolve |
| Refresh log recorder | `src/refresh_log/recorder.py:RefreshLogRecorder.record()` — best-effort, 2s timeout, own sessionmaker on the shared engine; swallows all exceptions |
| Refresh log instrumentation | `universal_odds.py:fetch_sport_events()` — try/except BaseException wraps API + parse; builds `RefreshAttempt` on every exit; re-raises after audit write so `gather()` semantics are preserved |
| `_api_request` outcome capture | `universal_odds.py:_api_request()` — optional `status_out` scratch-dict kwarg; when provided, every terminal return populates `outcome`, `http_status`, `api_key_index`. Only `fetch_sport_events` opts in |
| Consensus calculation | `consensus_odds.py:calculate_consensus_prob()` |
| Bookmaker staleness | `consensus_odds.py:BookmakerTracker` |
| Category filtering | `orchestrator.py` (`DRYRUN_CONFIG["categories"]` check in `_process_single_trade`) |
| O/U market toggle | `orchestrator.py` (O/U regex + `LIVE_ENABLE_OVER_UNDER` check before `_enrich_universal`), `universal_odds.py:fetch_odds()` (markets param), `engine.py:_create_orders_realtime()` (safety net) |
| Handicap market toggle | `orchestrator.py` (`parsed_market.market_type == "spread"` + `LIVE_ENABLE_HANDICAP` check before `_enrich_universal`), `live_odds_poller.py:_fetch_odds()` (spreads in markets param), `engine.py:_create_orders_realtime()` (safety net) |
| Handicap consensus | `consensus_odds.py:resolve_handicap_side()` → `extract_consensus_handicap()` — called from orchestrator (enrichment), poller (bid updates), scanner (backup) |
| Handicap frozen detection | `live_odds_poller.py:MatchOddsTracker.update_handicap()` — fed from `_trigger_bid_update()` per-order; `is_frozen(event_id, "handicap")` checked in engine RT, scanner, bid updates |
| Pinnacle-only fallback probs | `orchestrator.py:_calculate_implied_probs()` |
| Order book capture | `orchestrator.py:_capture_market_state()` |
| DryRun order trigger | `orchestrator.py` → `engine.py:on_enriched_trade()` |
| DryRun regime branch | `engine.py:_create_orders_inner()` — `max_edge < edge_threshold` → spread, else → directional |
| DryRun order dedup | DB partial unique index `uq_dryrun_orders_active_slot` on `(market_title, outcome_name, spread_level) WHERE status IN ('open', 'filled')` |
| DryRun frozen odds gate | `live_odds_poller.py:MatchOddsTracker` + `is_frozen()` (checked in engine + scanner + bid updates) |
| DryRun self-fill | `engine.py:_create_orders_inner()` (replays rn1 trade against new orders) |
| DryRun bid updates | `live_odds_poller.py:_trigger_bid_update()` — sources poly price from `dryrun_snapshots` (fallback: `poly_price_at_creation`); spread: symmetric update + recompute shares; directional: cancel-replace with new outcome on flip (regardless of fills) |
| Live directional dedup | `manager.py:execute_directional_orders()` — two-layer dedup: (1) condition-level `(condition_id, heavy_side)` with 60s cooldown, (2) match-level `(_match_slug, heavy_side)` with 30s cooldown (blocks correlated O/U lines and handicap lines on same match) |
| Live pre-place cleanup | `manager.py:execute_directional_orders()` — cancels both opposite-side and same-side unfilled orders on same condition_id before submitting; uses `get_pending_or_active_orders_by_condition()` to include CREATED/SUBMITTED; partial-fill protection skips same-side orders with `size_filled > 0`; fill-race guard re-reads after cancel |
| Live CLOB min-size bump | `manager.py:_reserve_order_inner()` — bumps orders below 5 shares to `CLOB_MIN_SIZE` before DB insert |
| Live postOnly retry | `manager.py:_place_order()` — retries up to 5 times on postOnly rejection, re-fetching best_ask and revalidating edge |
| Live per-market lock | `manager.py:_get_market_lock()` — serializes risk-check → DB-insert per `condition_id`, prevents 8-worker race |
| Live contrarian flip | `manager.py:cancel_and_place_flipped(..., new_consensus_prob, new_edge)` → `_cancel_and_place_flipped_inner()` (under lock; persists new-side cp/edge with `flip_stale_cp_edge_fallback` counter on caller omission) → `_place_order()` (outside lock). Under `SAFE_REPLACE_ENABLED`, `_retire_flipped_old_order()` runs only if `_place_order()` returned True |
| Bug 2 acceptance | `check_flip_residual.py` — standalone audit of contrarian-flip rows for `edge == cp − price − spread_level` (within 1¢). Run after deploy of any flip-path change |
| Live cancel kill switches | `config/live_settings.py:LIVE_CONFIG["disable_cancellations"]` (env `LIVE_DISABLE_CANCELLATIONS`, default `True`) gates flip + edge-gone + same-direction repricing in `live_odds_poller._trigger_bid_update()`. `LIVE_CONFIG["disable_side_flip_cancellations"]` (env `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS`, default `True`) gates the opposite-side branch of `manager.execute_directional_orders()` pre-place cleanup |
| League label normalization | `market_classifier.normalize_league_label()` + `_LEAGUE_CANONICAL_MAP` → applied at `Database.insert_soccer_odds_state` (single chokepoint). Backfill: `scripts/backfill_league_labels.py` (`--dry-run` reports without writing) |
| Counterfactual no-logic mirror | `database.insert_dryrun_order_pair()` (regular + no-logic write at order creation), `fill_simulator.check_fill_from_trade_no_logic()` / `check_fill_from_book_no_logic()` (parallel fill ledger), `resolution_watcher._resolve_no_logic_orders()` (per-row P&L). Export: `main.py:dryrun_no_logic_export_csv` via `--dryrun-no-logic-export` |
| Odds raw log capture | `src/odds_raw_log/recorder.py:OddsRawLogRecorder.record()` — best-effort, 2s timeout, no-op when `LIVE_ODDS_RAW_LOG_DB=false`. Wired into `live_odds_poller._fetch_odds()` after each successful league-poll response. Export: `main.py:odds_raw_log_export_csv` via `--odds-raw-export` |
| Bookmakers snapshot | `consensus_odds.extract_bookmakers_snapshot()` — every trusted bookmaker's decimal odds + implied probs + vig-removed fair probs for the order's specific market (filtered by `market_type` / `ou_line` / `handicap_line` / `handicap_team`). Bypasses the `LIVE_CONSENSUS_BOOKMAKERS` allowlist so the export shows the full picture even when consensus is Pinnacle-only. Captured at order creation in both `live_orders.bookmakers_snapshot` and `dryrun_orders.bookmakers_snapshot` |
| Dryrun-only heartbeat | `dryrun/engine.py:_start_background_tasks()` — when `execution_manager is None` (dryrun-only run), launches `heartbeat_loop(300)` so counter snapshots emit even without the live layer. In `--live` mode, `ExecutionManager.start()` owns the heartbeat task to avoid double-printing |
| Live safe cancel-replace | `manager.py:_safe_update_bid()` (site A), `_cancel_and_place_flipped_inner()` safe-flip branch (site B), `execute_directional_orders()` → `_run_deferred_cancels()` (site C). See ExecutionManager §"Safe Cancel-Replace" for the canonical mechanism |
| Live observability | `src/execution/observability.py` — `COUNTERS.increment(name)` call sites in tracker/manager/poller; `heartbeat_loop(300)` task started from `ExecutionManager.start()`; `tag(...)` appended to every lifecycle log line |
| V2 fee accounting wiring | `user_ws._handle_trade()` extracts `fee` via `_coerce_fee()` from each maker / taker fill event → forwards as `fee_usdc=` kwarg → `order_tracker.record_fill()` → `_record_fill_inner()` accumulates into `live_orders.fee_usdc` and (via `_update_position`) `live_positions.total_fees_usdc`. Same accumulation in `_apply_post_cancel_fill()`. `usdc_spent` stays gross (size × price); the wallet's true outflow is `usdc_spent + fee_usdc` |
| V2 trade decoder | `src/core/abi.py` defines V2 OrderFilled / OrdersMatched / FeeCharged; `src/core/trade_parser.py:parse_from_receipt()` decodes against the V2 ABI, applies the relaxed double-count filter, derives BUY/SELL from `side` (uint8) wrt target as maker/taker, computes `usdc_amount` and `shares` from the maker-perspective amount mapping, and tallies `decoded_orderfilled` / `decoded_ordersmatched` / `decoded_feecharged` / `decode_other` so a non-zero `decode_other` is the canary for an ABI-shape change. `parse_transaction()` (calldata path) short-circuits to `[]` under V2 |
| V2 wallet bootstrap | `scripts/migrate_wallet_to_v2.py` — idempotent: approve Onramp on USDC.e → wrap to pUSD → approve V2 exchanges on pUSD → `setApprovalForAll` for both V2 exchanges on the Conditional Tokens contract |
| V2 receipt diagnostic | `scripts/diagnose_v2_receipt.py <tx_hash>` — full per-log dump (address, all topics, data words, decoded event for V2 exchange logs). Run when `parse_from_receipt`'s "0 trades" warning shows `decode_other > 0` |
| TheOddsAPI preflight | `src/enrichment/soccer_odds.py:APIKeyRotator.preflight()` + `_preflight_probe()` on `UniversalOddsService` and `SoccerOddsClient`, wired from `orchestrator.py:IntelCopyBot.start()` |
| Rotating log file | `main.py:setup_logging()` — `RotatingFileHandler` at `logs/bot-YYYYMMDD.log`, UTF-8, 50MB × 5 backups. Console console-filtered via `_ConsoleSilenceFilter`; file captures full INFO |
| DryRun age re-validation | `live_odds_poller.py:_check_stale_orders()` — forces `_trigger_bid_update()` when orders exceed age threshold |
| DryRun odds feed | `orchestrator.py` → `engine.py:on_enrichment_result()` → `live_odds_poller.py` |
| Outcome resolution | `orchestrator.py:resolve_outcomes()` |
| is_live_trade derivation | `main.py:export_to_csv()` |
| Live API usage check | `main.py:show_api_usage()` |
| Live execution init | `orchestrator.py:_init_execution_layer()` — builds ClobClient → Executor → Risk → Tracker → WS → Manager |
| Live order dispatch (RT) | `engine.py:_create_orders_inner()` — after regime branch, dispatches independently of dryrun order creation |
| Live order dispatch (scanner) | `scanner.py:_dispatch_to_live()` — recomputes bids with live capital, resolves token IDs, routes to ExecutionManager |
| Live cardinality enforcement | `live_odds_poller.py:_trigger_bid_update()` → `execution_manager.enforce_single_order_per_outcome()` — runs once per condition_id before per-order loop; cancels extras, keeps best-filled survivor |
| Live bid updates | `live_odds_poller.py:_trigger_bid_update()` → `execution_manager.update_bid()` (cancel-replace on matching live CLOB orders). Triggered by consensus movement ≥2% OR age-gated re-validation. Gated by per-market cooldown (15s, matches poll interval) and price delta threshold ($0.01 vs live CLOB price) to prevent cancel-replace churn |
| Live stale sweep | `manager.py:_sweep_stale_orders()` — background task, every 60s cancels LIVE/PARTIAL_FILL orders older than 10 min (`MAX_ABSOLUTE_ORDER_AGE_S = 600`). Log line tags each cancellation with `origin=strategy|adopted regime=... outcome=... price=... size=... age=...`; adopted-origin cancels increment `stale_sweep_adopted` in addition to `stale_sweep_cancelled`. Pure cleanup, no replacements |
| Live risk gate | `risk_manager.py:check_order()` — 9 sequential gates, HALT is sticky. Gates 5-6 query `live_orders` (not `live_positions`) for pending+filled exposure |
| Live poll None guard | `order_tracker.py:poll_active_orders()` — skips orders where CLOB API returns None (unrecognized order ID) |
| Live fill tracking | `user_ws.py` → `order_tracker.py:record_fill()` (per-order lock + per-market position lock) |
| Live fill context plumbing | `user_ws.py:_handle_trade()` — extracts `asset_id`/`market`/`side`/`outcome` (maker-level overrides event-level) and passes as `context=` kwarg to `record_fill()` |
| Live post-cancel fill reconcile | `order_tracker.py:_apply_post_cancel_fill()` — fills arriving on CANCELLED orders update `size_filled`/`usdc_spent`/position, emit `POST_CANCEL_FILL` audit, log `💸🪦`. Status stays CANCELLED (no `_transition` call) |
| Live adopt unknown order | `order_tracker.py:_adopt_unknown_order()` — synthesizes minimal `live_orders` row from WS context (primary) → `executor.get_order` (fallback) → fill params (last resort). Emits `ADOPT_UNKNOWN_ORDER` audit, logs `🩹` with `via=` tag. Always ticks `adopted_unknown_order`; when called with `source="startup"` (from `reconcile_on_startup`) additionally ticks `adopted_unknown_order_startup` as a discriminator |
| Live startup reconcile | `order_tracker.py:reconcile_on_startup()` — adopt-then-cancel for orphans so trailing fills during the cancel race have a DB row; stale-order fills synced via `size_matched - already_filled` diff |
| Live token-via-CLOB fallback | `live_odds_poller.py:_resolve_token_via_clob()` — hits `/markets/{cond}`, reads `tokens[].token_id` by outcome, caches (positive unbounded, 5-min negative per cond). Ticks `token_resolved_via_clob` |
| Live CLOB rate-limit detection | `executor.py:_run_sync()` — wraps every SDK call, ticks `clob_rate_limit_hit` on `PolyApiException.status_code == 429`, re-raises (no retry) |
| Live shutdown | `orchestrator.py:stop()` → `execution_manager.shutdown()` → cancel_all + mark DB + stop tasks |
| Live emergency cancel | `main.py:live_cancel_all()` — standalone, no bot needed |

---

## Known Limitations

1. **Saudi Pro League** — `soccer_saudi_arabia_pro_league`. TheOddsAPI does not provide scores/results for this league (only odds). Match enrichment works, but resolution fallback via TheOddsAPI is unavailable — Polymarket CLOB resolution is the only settlement path for these markets.
2. **Tennis** — Limited coverage for non-Grand Slam tournaments.
3. **Esports live state** — PandaScore free tier provides match info only (no kills/gold/towers/rounds).
4. **Alchemy CU** — No programmatic usage API. Check dashboard manually.
5. **TheOddsAPI `match_status`** — Unreliable (often shows "not_started" during live matches). `is_live_trade` uses timestamp comparison instead.
6. **Chinese Super League** — Teams may not be in TheOddsAPI. Classification now correct (soccer) but enrichment may fail on match lookup.
7. **Consensus equal weighting** — All bookmakers weighted equally. Pinnacle (sharp) is not given extra weight over consumer books (DraftKings, FanDuel). This is intentional for robustness against single-bookmaker suspension but may dilute sharp signal.
8. **Handicap line availability** — `extract_consensus_handicap()` requires the exact handicap line from the Polymarket title to be offered by at least one trusted bookmaker. If no bookmaker offers that line (e.g., Polymarket has -1.5 but all bookmakers only offer -0.5), the market is skipped with `handicap_line_match=False`. TheOddsAPI does not have a documented `alternate_spreads` endpoint (unlike `alternate_totals` for O/U), so non-standard handicap lines cannot be priced.
9. **Live position settlement** — `ResolutionWatcher` only settles `dryrun_positions`. Live positions (`live_positions` table) have `pnl`/`settled_at` fields but no component populates them on market resolution. Requires wiring into the resolution event path.
10. **Live `usdc_spent` is BUY-only** — `record_fill` computes `usdc_spent = fill_size × fill_price`, which is correct for BUY orders. The bot only places BUY orders. If SELL orders are ever added, the formula must change to `fill_size × (1 - fill_price)`. Under V2, `usdc_spent` stays defined as gross (size × price); fees land in the separate `fee_usdc` column. True wallet outflow per order is `usdc_spent + fee_usdc`. Most fills are zero-fee (maker fills don't pay).
11. **Tick size rounding** — Python's `round()` uses banker's rounding (round half to even). Polymarket likely uses truncation. A one-tick discrepancy is possible at exact half-tick boundaries.
12. **No native order amendment** — Polymarket has no modify endpoint. Price/size changes require cancel + new order; the safe-replace flow handles the orphan window (see ExecutionManager §"Safe Cancel-Replace").
13. **B-team/reserve teams** — Teams like "Real Sociedad de Fútbol B" are not covered by TheOddsAPI. Enrichment will always fail for reserve-team markets.
14. **Polymarket minimum order size** — All CLOB markets enforce a minimum of 5 shares per order. `_reserve_order_inner()` bumps sub-minimum orders to 5 shares (`CLOB_MIN_SIZE`) instead of letting them be rejected, which slightly exceeds the per-order budget. At very low `capital_per_market` values (e.g., $2), most orders will be bumped — monitor `📏 Size bump` log lines if cost control is critical.
15. **Scanner live dispatch still coupled to dryrun dedup** — The scanner's `_dispatch_to_live()` only fires when new dryrun orders are created. Unlike the RT path (fully decoupled), repeated scanner runs for the same market won't re-attempt live dispatch. This is acceptable because the scanner is a 5-minute backup, not the primary path.
16. **Shared `MAX_CONCURRENT_MARKETS` / `CAPITAL_PER_MARKET` env vars** — Both `config/live_settings.py` (live defaults: 3 markets, $10) and `config/dryrun_settings.py` (dryrun defaults: 50 markets, $5) read the same `MAX_CONCURRENT_MARKETS` and `CAPITAL_PER_MARKET` env vars via `_env_int()` / `_env_float()`. Setting either in the environment overrides BOTH layers to the same value. If you lift the live cap, verify the dryrun shadow is sized appropriately, or expose a separate env var before decoupling.
17. **Translation-style team-name mismatches** — when Polymarket and TheOddsAPI disagree on language, accent stripping isn't enough. `København → Kobenhavn` (after `ø → o`) doesn't overlap with TheOddsAPI's English `Copenhagen`; same class for `Köln/Cologne`, `München/Munich`. Atomic-letter normalisation (Brøndby, Viborg, Vejle, Silkeborg, Odense) handles spelling differences but not translations. Fix would be a narrow alias map; deferred pending a call on alias map vs. accepting these as coverage gaps.
18. **Bundesliga total-failure under investigation** — first match_audit session showed ~375 vs_match_failed / will_win_no_candidates rows on Bayern/Bayer with a single Union Berlin success, even though `expected_sport_keys` contained `soccer_germany_bundesliga` (so the category gate was passing). Bayern/Bayer shouldn't fail under the word-overlap fix — the first word of each club name is language-invariant. Mechanism unknown; candidates are (a) `soccer_germany_bundesliga` refresh returning empty events for this session, (b) an index-race window at refresh time, (c) title forms we don't parse. Do NOT ship a German alias map until `match_audit` rows for these teams are pulled and the `candidate_count` and `failure_reason` fields inspected — the fix spec explicitly calls this out. `theodds_refresh_log` now complements `match_audit` for this investigation: candidate (a) shows up as `outcome='empty'` rows for `sport_key='soccer_germany_bundesliga'` in the cycles preceding the failed trades, candidate (b) shows up as `outcome='exception'` or missing rows for that sport_key in a given `refresh_cycle_id`.
19. **Ligue 1 Strasbourg / Nice** — PSG and Nantes match hundreds of times per session; RC Strasbourg Alsace and OGC Nice match zero times, same `sport_key`, same session. No code-level cause found by inspection (`_parse_vs_market` + `_check_vs_match` both trace cleanly for these names). Leading candidate is an index-refresh race at the time those trades fired. Query `match_audit` after the next session for `market_title ILIKE '%strasbourg%' OR '%nice%'` and check whether `candidate_count` is zero (index miss) vs. nonzero (overlap issue). `theodds_refresh_log` can now corroborate: if the failing trades' timestamps fall inside a `refresh_cycle_id` where `soccer_france_ligue_one` has `outcome != 'ok'` or low `event_count`, the index-refresh-race hypothesis is confirmed without speculation.
20. **Moroccan Botola Pro coverage gap** — Polymarket lists markets on leagues TheOddsAPI doesn't cover; these trades reach `find_match` and resolve as failures in `match_audit`. Full list and the diagnostic query pattern live in Match Audit → Known coverage gaps. Not a matcher bug; only fixable by adding coverage.
21. **Leagues covered by TheOddsAPI but intentionally not polled** — 32 sport_keys exist in TheOddsAPI's soccer catalog that `SOCCER_LEAGUES` does not include. Notable among them: Serie B (Italy), Ligue 2 (France), 3. Liga (Germany), K-League, Brazil Série B, Chile Primera, Norwegian Eliteserien, Polish Ekstraklasa, Russia Premier League, League of Ireland, various FIFA international tournaments, UEFA Nations League, Copa Libertadores / Sudamericana, Africa Cup of Nations, women's competitions. Decision rule for future additions: one match_audit session showing ≥50 rows of failing trades on a specific unpolled league triggers inclusion. Do not speculatively add leagues.
22. **Copa Libertadores / Copa Sudamericana coverage gap** — When a Brazilian or Argentine club plays a continental cup fixture, the event lives in `soccer_conmebol_copa_libertadores` or `soccer_conmebol_copa_sudamericana`, NOT in the domestic league's sport_key. Since we poll only domestic Brazilian and Argentine leagues, continental-cup matches for those clubs will fail to match. This has not yet been observed in the audit (rn1 traded exclusively domestic fixtures in sessions observed), but is a known gap. Revisit if audit data surfaces it.
23. **`soccer_odds_states` per-bookmaker columns predate the 9-book widening** — The table carries seven JSONB columns (`odds_pinnacle`, `odds_bet365`, `odds_betfair`, `odds_draftkings`, `odds_fanduel`, `odds_williamhill`, `odds_unibet`) installed when the request set was 7 keys including the now-retired `bet365` and `unibet_eu`. The four books added in widening (`betfair_ex_uk`, `matchbook`, `paddypower`, `unibet_uk`) have no dedicated column and aggregate only into the consensus row. Forensic-data implication: per-bookmaker historical analysis on data captured after the widening cannot isolate the four new books at the row level — their contribution is visible in `consensus_implied_*` but not separable from the others. Retired keys (`bet365`) write `NULL` rows. Note: the per-order `bookmakers_snapshot` JSONB column on `dryrun_orders` and `live_orders` does carry every trusted bookmaker the API returned at order-open time, so per-book attribution is available at order granularity even though the `soccer_odds_states` row-shape lags. The fix is a schema migration adding four columns; not warranted by current analysis needs.
24. **Retired bookmaker keys can be revived only via `settings.BOOKMAKERS`** — `consensus_odds.BOOKMAKERS` (the 14-name whitelist) tolerates `bet365` and `unibet_eu` as defensive accept-if-returned entries, but the active request shape is driven by `settings.BOOKMAKERS` (9 names). If TheOddsAPI ever re-adds `bet365` or revives `unibet_eu`, the whitelist alone will not pull them in — the rotator only ever sees what was requested. Revival requires an explicit edit to `settings.BOOKMAKERS` plus a fresh empirical-validation probe per the guardrail in Soccer Odds. Until then, the retired keys' `l1_excluded_*` counters stay flat and the `odds_bet365` / `odds_unibet_eu` JSONB columns (where they exist in `soccer_odds_states`) stay `NULL`. Same logic applies to any future retirement-and-revival cycle on any other key.
25. **Bug 2 fix covers flips only — same-side cancel-replace cp/edge still inherits old-row values.** The flip path (`update_bid` → `cancel_and_place_flipped` → `_cancel_and_place_flipped_inner`) now threads `new_consensus_prob` / `new_edge` end-to-end, with `flip_stale_cp_edge_fallback` as the missing-caller signal. The same-side reprice paths (`_safe_update_bid` and `order_tracker.cancel_replace`) still build their `new_order_data` dict from `old_order.consensus_prob` / `old_order.edge`, so a same-side cancel-replace where consensus moved between cycles can still write a row that fails `edge == cp − price − spread_level`. Mechanical fix: thread the same kwargs through `update_bid`'s non-flip branch into `_safe_update_bid`, and thread them through `cancel_replace`'s `new_order_data` builder. Tracked separately from the Bug 2 flip fix; the investigation note in `investigations/edge-write-stale-snapshot.md` describes both classes.
26. **`bookmakers_snapshot` is not threaded through cancel-replace paths.** The column is captured correctly on initial reserve (`open_spread_pair` / `open_directional` → `_reserve_order_inner`) but `_safe_update_bid`, `_cancel_and_place_flipped_inner`, and the `cancel_replace` `new_order_data` builder do NOT pass `bookmakers_snapshot` into the new row. Every cancel-replace child row therefore has `bookmakers_snapshot = NULL`, so the "captured at order-open time" forensic guarantee silently degrades to "captured at first-creation time" — repriced or flipped orders lose their bookmaker forensic context. Fix: re-fetch from `LiveOddsPoller.last_odds[event_id]["bookmakers_raw"]` (or thread through) at the three sites.
27. **Live consensus default is Pinnacle-only.** `LIVE_CONSENSUS_BOOKMAKERS=pinnacle` (`.env.example` default) means `calculate_consensus_prob` averages across Pinnacle alone. Other trusted books are still recorded in `bookmakers_snapshot` and `theodds_raw_log` for forensic export, but they don't drive the bid. Set `LIVE_CONSENSUS_BOOKMAKERS=*` to widen back to every trusted bookmaker; comma-separated lists also work (e.g. `pinnacle,betfair_ex_uk`). Under Pinnacle-only, `l1_excluded_pinnacle` ticking immediately collapses consensus to `None` — there is no other book to fall back on. The L2 frozen layer is largely redundant in this regime (collapsed to `None` consensus already gates new orders upstream), which is why `LIVE_FROZEN_UNCHANGED_THRESHOLD` defaults to 4 (= L1) rather than carrying headroom. Bump L2 if you widen the allowlist.
28. **Cancel kill switches are on by default.** `LIVE_DISABLE_CANCELLATIONS=true` and `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS=true` (defaults in `config/live_settings.py`) disable the contrarian flip, edge-gone cancel, same-direction repricing, and pre-place opposite-side cleanup paths on the live CLOB. Resting orders survive until they fill or the absolute-age stale sweep retires them at 10 min. Set both to `false` in `.env` to restore signal-driven cancellations.
29. **Longshot guardrail is off by default.** `MIN_CONSENSUS_PROB_DIRECTIONAL=0` (`config/dryrun_settings.py` default). Set to a positive fraction (e.g. `0.10`) to re-enable the rejection of full-size directional bets where the heavy side's consensus probability is in the gutter — without it, a 3% edge at 4% consensus gets the same capital as a 3% edge at 55% consensus.
30. **Polymarket V1 → V2 migration completed (cutover April 28, 2026).** Exchange contracts moved to V2 (`0xE111...` CTF, `0xe222...` Neg Risk), collateral migrated from USDC.e to pUSD, the SDK swapped from `py-clob-client` to `py-clob-client-v2`, and explicit V2 fee accounting landed (`live_orders.fee_usdc`, `live_positions.total_fees_usdc`, `dryrun_orders.shadow_fee_usdc`). All V1 open orders were cancelled by Polymarket during the maintenance window — first `reconcile_on_startup()` post-cutover marked every previously-LIVE order as orphan (expected, noisy log). The V2 OrderFilled event shape is NOT byte-identical to V1 (replaced `(makerAssetId, takerAssetId)` with explicit `(side, tokenId)` and added trailing `builder` / `metadata`); the Trade Parser section above documents the wire format. See `docs/migrations/intelbot_v2_migration_spec.md` for the migration sequence, diagnostics, and live-ramp procedure.
31. **O/U `bookmakers_snapshot` drops Pinnacle when its primary line ≠ Polymarket's line.** `extract_bookmakers_snapshot` short-circuits on the first market matching the API key tuple `("totals", "alternate_totals")`, then per-outcome filters by exact `ou_line`. Pinnacle returns `totals` first with their main line (typically 2.5 for soccer), so when Polymarket's line is non-standard the loop selects `totals`, fails the line filter, collects no decimal_odds, and Pinnacle is dropped from the snapshot. Matchbook and Williamhill return dense `alternate_totals` and dominate O/U snapshots as a result. This is per-order forensic-data degradation only; the consensus calculation in `calculate_consensus_prob` does **not** filter by line and uses Pinnacle's totals odds regardless of which line they're for, which is a separate accuracy concern. Mechanical fix: iterate ALL markets in `api_keys` and let the line filter pick across both `totals` and `alternate_totals`. Apply the same fix to `calculate_consensus_prob` to stop pricing O/U markets against the wrong line.
32. **`dryrun_orders.shadow_fee_usdc` is declared but not populated.** The column was added during the V2 migration so the dryrun shadow can ultimately mirror V2's taker-only fee model for like-for-like P&L comparison against `live_orders.fee_usdc`. The compute is deferred — no code currently writes a non-zero value, so every row sits at the column's default of 0. The fee formula referenced in the model's docstring (`fee = C * feeRate * p * (1 - p)` where `feeRate` comes from `client.get_clob_market_info(condition_id).fd.r`, populated only on shadow orders that would have hit as taker) is the work item. Until then, shadow vs live P&L diverges by exactly the fee amount on every taker-side fill in `live_orders`.
33. **`OPERATIONS.md` Database Cleanup table list lags the schema.** The cleanup runbook in `OPERATIONS.md` enumerates four DROP TABLE groups (8 + 3 + 3 + 4 tables). The diagnostic group is missing `theodds_raw_log`, and the dryrun group is missing `dryrun_orders_no_logic`. Both tables exist in the schema today; a manual wipe per the runbook leaves them populated. Cosmetic for fresh-DB setup (since `metadata.create_all` recreates them) but produces stale state if used as the only cleanup step before forensic-data analysis.

---

## Debugging

**Trade not enriching**: Check `--api-usage` for key exhaustion → check logs for participant extraction failure → verify league is in `PRIORITY_SPORTS` (missing league = zero events fetched, e.g., `soccer_spain_segunda_division` for La Liga 2) → check market title against known regex patterns → check if category is in `DRYRUN_CONFIG["categories"]` (soccer-only filter).

**Trades failing because league isn't polled**: If `match_audit` shows a cluster of failures with `parse_stage_reached='vs_match_failed'` or `'will_win_no_candidates'` on recognizable club names (e.g., Saudi clubs, Danish clubs), cross-reference the teams against the current `SOCCER_LEAGUES` list. A team not in any polled league cannot be matched. Fix is to add the league's sport_key to `SOCCER_LEAGUES`, not to edit the matcher. Verify the sport_key exists in TheOddsAPI's catalog at <https://the-odds-api.com/sports-odds-data/sports-apis.html> before adding.

**DryRun not creating orders**: Check logs for `RT-orders SKIP` or `RT-orders [ML/OU/HC] SKIP` messages. Common reasons: category not tradeable, market type unsupported, confidence too low, match too old, frozen odds, no live poly price, edge exceeds sanity cap, handicap/O/U toggle off. In live mode, breadcrumbs from the `live.engine` logger show each skip reason with `↳ skip:` prefix.

**Consensus issues**: In live mode, the `🔍 consensus match` line shows bookmaker count and probabilities. For detailed per-bookmaker breakdown, use `--log-level DEBUG` to see `Trade N CONSENSUS:` log lines. If bookmakers_used is low, check `BookmakerTracker` staleness exclusions. Consensus falls back to Pinnacle-only if all bookmakers suspended.

**Misclassified markets**: The `Spread:` regex catches all spread titles. If a soccer team lacks indicators (FC, SC, CF, United, City, Real, Athletic, Sporting), it will be classified as basketball and filtered out. Add the team's indicator to the soccer_indicators list in `_try_basketball_spread()`.

**WebSocket dying silently**: Staleness detection triggers after 30 min. Look for "Staleness timeout" in logs. Check Alchemy dashboard for rate limiting.

**Keys exhausting fast**: Verify topic-filtered subscriptions are active. Check for duplicate `get_block` calls. Confirm `run_in_executor` wrapping.

**Live order rejected by CLOB**: Check tick-size rounding first (`_round_price`/`_round_size` in `executor.py`) — V2 makes `tick_size` mandatory on every order call. If the error mentions "invalid tick", the market's tick size may have changed since it was cached (tick sizes change dynamically when markets become one-sided). If the error mentions "insufficient balance", check `--live-status` for committed capital — open orders reserve pUSD. Sub-5-share orders are auto-bumped to 5 shares by `_reserve_order_inner()` — look for `📏 Size bump` log lines. If bumped orders still fail, check tick-size rounding or other CLOB rejections.

**Live orders not placing**: Check the `live.engine` breadcrumb logs for `↳ skip:` lines — they show exactly which filter rejected the market (category, confidence, frozen odds, edge sanity, etc.). If the market passes all filters but no `💰 LIVE` line appears, check `--live-status` for halt state. If `is_halted=True`, the risk manager triggered a circuit breaker — check logs for `RISK HALT:` messages. Use `reset_halt()` after investigation. Also check that `POLY_PRIVATE_KEY` is set and that the V2 wallet setup has been completed: pUSD balance + V2 exchange allowances + CTF `setApprovalForAll` for both V2 exchanges. The `scripts/migrate_wallet_to_v2.py` script is idempotent — re-run if any allowance is missing. See `OPERATIONS.md` §"Wallet Setup" for the full procedure. If `LIVE_ENABLE_SPREAD_REGIME=false`, only directional-regime markets (edge ≥ 2%) will place live orders. If orders are created in DB but never submitted to the CLOB (zombie orders), check for serialization errors in the audit log — `_jsonb_safe()` must handle all types in the `details` dict.

**Live fills not detected**: Check if UserWebSocket is connected (look for `User WS connected` in logs). If disconnected, check CLOB API credentials. The 30s polling backup will catch fills even if WebSocket is down, but with delay.

**Stale orders after crash**: On restart, `reconcile_on_startup()` runs automatically. Check logs for `Reconciliation:` summary. Orphan orders (on CLOB but not in DB) are cancelled. Stale orders (in DB but filled on CLOB) are synced.

**Orders re-validating too frequently**: Check that `created_at` is being reset after successful cancel-replace in `_trigger_bid_update()`. If `⏰` lines appear every poll cycle for the same order, the `created_at` update is missing from the `update_dryrun_order_bid()` call. Age thresholds: 3 min live (`max_order_age_live_s`), 15 min pre-game (`max_order_age_pregame_s`).

---

## Configuration Files

| File | Purpose |
|------|---------|
| `config/settings.py` | All API keys (551 `THEODDS_API_KEYS` slots with 464 live keys and 87 empty placeholders), contract addresses, timing, rate limits, league lists. Canonical request shape for both endpoints: `REGIONS = "us,uk,eu"` and `BOOKMAKERS` (9 books — pinnacle, betfair_ex_eu, betfair_ex_uk, matchbook, draftkings, fanduel, williamhill, paddypower, unibet_uk). Also: `POLY_PRIVATE_KEY`, `POLY_FUNDER_ADDRESS`, `POLY_SIGNATURE_TYPE`, `POLY_WALLET_ADDRESS` for live execution; `MATCH_AUDIT_ENABLED` (default `true`); `REFRESH_LOG_ENABLED` (default `true`); `LIVE_CONSENSUS_DIAGNOSTICS_DB` (default `false`); `LIVE_ODDS_RAW_LOG_DB` (default `false`) |
| `config/dryrun_settings.py` | Dry run wallet size, dual-regime thresholds (edge_threshold, spread_regime_offset, spread_levels), polling intervals, fill mode, category filter, order staleness thresholds (max_order_age_live_s, max_order_age_pregame_s), longshot filter (min_consensus_prob_directional, default 0). `capital_per_market` defaults to $5; `max_concurrent_markets` defaults to 50. Bookmaker / region lists live in `config/settings.py` (`BOOKMAKERS`, `REGIONS`) — `LiveOddsPoller._fetch_odds` reads those directly with `LIVE_POLLER_BOOKMAKERS` / `LIVE_POLLER_REGIONS` overrides |
| `config/live_settings.py` | Live execution config: wallet credentials, strategy params (mirrors dryrun dual-regime), risk limits, execution tuning (poll interval, `order_type="GTC"` with no server-side expiry), order staleness thresholds, shadow_dryrun flag, cancel kill switches. **Operator-tunable via `.env`** (defaults are starting points, not invariants; see "Operator-tunable risk knobs (live trading)" for `max_concurrent_markets`, `capital_per_market`, `max_per_market_usd`): every key in `LIVE_CONFIG` wrapped by `_env_int` / `_env_float` / `_env_bool` honors its `LIVE_<KEY>` env override. Notable env knobs: `LIVE_MAX_CONCURRENT_MARKETS`, `LIVE_CAPITAL_PER_MARKET`, `LIVE_MAX_PER_MARKET_USD`, `LIVE_MAX_HOURLY_LOSS_USD`, `LIVE_MAX_DAILY_LOSS_USD`, `LIVE_MAX_DRAWDOWN_PCT`, `LIVE_EDGE_THRESHOLD`, `LIVE_ENABLE_SPREAD_REGIME`, `LIVE_ENABLE_OVER_UNDER`, `LIVE_ENABLE_HANDICAP`, `LIVE_DISABLE_CANCELLATIONS` (default `true`), `LIVE_DISABLE_SIDE_FLIP_CANCELLATIONS` (default `true`). Threshold tunables read directly by `consensus_odds.py` / `live_odds_poller.py` at import: `LIVE_STALE_UNCHANGED_THRESHOLD` (L1, default 4), `LIVE_FROZEN_UNCHANGED_THRESHOLD` (L2, default 4 — matches L1 under the Pinnacle-only consensus default; bump if widening), `LIVE_FROZEN_MOVE_TOLERANCE` (default 0.005). Consensus allowlist: `LIVE_CONSENSUS_BOOKMAKERS` (default `pinnacle`; `*` for every trusted book; comma-separated list to mix). Poller request shape: `LIVE_POLLER_REGIONS` (default `us,uk,eu`), `LIVE_POLLER_BOOKMAKERS` (default = `BOOKMAKERS` joined). Orthogonal non-`LIVE_`-prefixed: `SAFE_REPLACE_ENABLED` (read in `src/execution/manager.py`, default on). Changes take effect on next process start only — `LIVE_CONFIG` and threshold module-level constants are read once at import. |
| `.env.example` | Template for environment variable overrides. Currently sets `LIVE_CONSENSUS_BOOKMAKERS=pinnacle` and the four wallet/key fields |
| `requirements.txt` | Python dependencies — includes `py-clob-client-v2` (V2 SDK, replaces V1's `py-clob-client`) |
| `migrations/v2_add_fee_columns.sql` | Idempotent (`ADD COLUMN IF NOT EXISTS`) schema migration for V2 fee columns: `live_orders.fee_usdc`, `live_positions.total_fees_usdc`, `dryrun_orders.shadow_fee_usdc`. Apply on existing DBs before any post-V2 run — `--dryrun` also references `shadow_fee_usdc` via the SQLAlchemy default, and the dryrun engine's auto-migration `_migrate_v2_columns` does NOT include it |
| `scripts/migrate_wallet_to_v2.py` | Idempotent V2 wallet setup — wraps USDC.e → pUSD via the CollateralOnramp, approves both V2 exchanges to spend pUSD, runs CTF `setApprovalForAll` for both V2 exchanges. See `OPERATIONS.md` §"Wallet Setup" |
| `scripts/diagnose_v2_receipt.py` | One-shot V2 receipt inspector — dumps every log's address/topics/data and runs the V2 ABI decoder over logs from the V2 exchanges. Used when `parse_from_receipt`'s "0 trades" canary fires with non-zero `decode_other` |
| `docs/migrations/intelbot_v2_migration_spec.md` | Archived V2 migration record — the migration playbook + cutover values. Kept for posterity; the canonical source of truth is now this spec |

---

## Test Suite

Run: `python -m pytest tests/ -v`. Layout, stub conventions (`py_clob_client_v2` import-time stub in `tests/execution/conftest.py`), and the pinned regression cases referenced by Guardrails throughout this spec live in [`tests/README.md`](tests/README.md). The IntelBot core (wallet monitor, rest of enrichment) has no automated tests — validated through production operation on the live trade stream. The trade parser is now exercised by `tests/migration/test_v2_orderfilled_decode.py` (synthetic V2 receipts decoded against the real ABI) and `tests/migration/test_parse_transaction_short_circuit.py` (pin that the calldata-decoding path returns `[]` under V2).

`tests/migration/` holds the V2 migration diagnostic suite (D1–D14): signature-hash pins for the V2 OrderFilled / OrdersMatched / FeeCharged events, end-to-end decode regression tests, plus network-gated diagnostics (skip cleanly when env prereqs are absent) for SDK install, read-only and authenticated CLOB calls, on-chain wallet state (pUSD balance, allowances, CTF setApprovalForAll), schema migration presence, and the bot's own HTTP `PolymarketClient` against live V2 endpoints. See `tests/migration/README.md` for the full runbook and triage map (failing diagnostic → file to investigate first).
