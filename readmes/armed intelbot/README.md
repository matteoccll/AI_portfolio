# Polymarket Trading Bot

Three-mode system for copy-trading a target Polygon wallet on sports prediction markets.

## Architecture

IntelBot (monitoring core) connects to Alchemy via WebSocket with two topic-filtered
subscriptions on the CTF Exchange contract — one defensive (target at topic[1]), one
load-bearing (target as maker at topic[2]) — cutting inbound log volume from thousands
of events per hour to ~10–20. Detected transactions are pushed into a 500-capacity
asyncio queue and drained by 8 parallel enrichment workers. Each worker decodes the
raw log against the V2 CTF Exchange ABI (`src/core/abi.py`), resolves market metadata
via the Polymarket CLOB and Gamma APIs, and enriches with real-time bookmaker odds from
TheOddsAPI (soccer: h2h / totals / spreads) or PandaScore (esports) before writing to
PostgreSQL.

The Dry Run Engine hooks into that pipeline via `on_enriched_trade`, which fires
~400ms after each rn1 fill. It computes a consensus probability from an allowlisted
set of bookmakers (default: Pinnacle), applies a dual-regime bidding model (spread
capture below the edge threshold, directional-only above it), and simulates fills
against three Polymarket order-book tiers — optimistic (rn1's fill price ≤ our bid),
realistic (0.5% below our bid), conservative (cumulative volume at/below bid exceeds
`order_size × queue_factor`). Five background tasks run concurrently: MarketScanner
(5-min sweep of DB candidates), LiveOddsPoller (15s cadence for live matches,
5-min for pre-game), PricePoller (60s order-book backup), ResolutionWatcher (5-min
CLOB/Gamma resolution check), and an observability heartbeat.

The Live Execution Layer wraps `py-clob-client-v2` (synchronous SDK bridged to
asyncio via a dedicated `ThreadPoolExecutor`) with a nine-gate pre-trade risk manager
(price sanity → tick size → available balance → concurrent market cap → per-market
exposure → hourly loss → daily loss → max drawdown, plus a halted-state check at
gate 0) and a circuit-breaker HALT that cancels all open orders on any loss-limit
breach. Real-time fills arrive via the Polymarket User Channel WebSocket; the
cancel-replace path uses submit-new-then-cancel-old (`SAFE_REPLACE_ENABLED=true`)
so the position is never zero during a bid update.

## Stack
- **Language**: Python 3.10+ (async/await throughout; `X | None` union syntax)
- **Database**: PostgreSQL via SQLAlchemy 2.0 async + asyncpg
- **Chain access**: web3.py v6 (POA middleware for Polygon) over Alchemy RPC + WebSocket,
  with an `AlchemyKeyRotator` that rotates keys on 429/capacity limits with a 60s cooldown
- **Event source**: Polygon V2 CTF Exchange + Neg Risk Exchange logs decoded against the
  V2 `OrderFilled` ABI (`src/core/abi.py`, `src/core/trade_parser.py`)
- **Odds enrichment**: TheOddsAPI behind an `APIKeyRotator` over a 551-slot key list
  (425 active); PandaScore for esports
- **Markets / execution**: Polymarket CLOB + Gamma APIs; `py-clob-client-v2==1.0.0`
  (synchronous SDK bridged to asyncio via a dedicated `ThreadPoolExecutor`)
- **HTTP / WS**: aiohttp, httpx, websockets
- **Support libs**: pydantic v2, python-dotenv, structlog
- **Tests**: pytest + pytest-asyncio (231 commits; test coverage across enrichment,
  dryrun, execution, and V2 migration paths)

## Run modes
```
  python main.py                  # monitor only + auto-resolve every 5 min
  python main.py --dryrun         # monitor + paper trading engine
  python main.py --live           # monitor + dryrun shadow + real CLOB orders
```

## Key diagnostics
```
  # General
  python main.py --backfill N            # scan last N blocks before monitoring
  python main.py --resolve               # run outcome resolution only
  python main.py --stats                 # current statistics
  python main.py --api-usage             # TheOddsAPI + Alchemy usage
  python main.py --export [FILE]         # export all trades to CSV
  python main.py --validate [HOURS]      # Neg Risk BUY/SELL + source_exchange check

  # Dry run (paper trading)
  python main.py --dryrun-report         # P&L report
  python main.py --dryrun-orders         # all simulated orders
  python main.py --dryrun-positions      # current positions
  python main.py --dryrun-resolve        # force-resolve active positions (one-shot)
  python main.py --dryrun-reset          # clear all dry run data
  python main.py --dryrun-config         # show dry run configuration
  python main.py --dryrun-export [PFX]   # export orders/positions/snapshots
  python main.py --dryrun-no-logic-export [PFX]   # counterfactual table export

  # Live execution
  python main.py --live-status           # open orders, positions, balance
  python main.py --live-report           # live P&L report
  python main.py --live-orders           # all CLOB orders (status, fills, edge)
  python main.py --live-positions        # positions with costs and P&L
  python main.py --live-export [PFX]     # orders/positions/audit log export
  python main.py --live-cancel-all       # emergency: cancel all open orders

  # Diagnostic table exports
  python main.py --match-audit-export [FILE]
  python main.py --refresh-log-export [FILE]
  python main.py --consensus-outlier-export [FILE]
  python main.py --odds-raw-export [FILE]
```

## Status
Working in dry-run mode. Live execution layer built and tested against paper positions.
