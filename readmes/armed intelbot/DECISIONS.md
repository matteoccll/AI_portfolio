# Engineering decisions and failures

## Migration from py-clob-client V1 to py-clob-client-v2

When Polymarket deprecated its V1 contracts, the entire execution surface had to be
replaced: new CTF Exchange and Neg Risk Exchange addresses, USDC.e → pUSD as
collateral, the `py-clob-client-v2` SDK (single-call `create_and_post_order`,
mandatory `tick_size`, `Side` enum, two-step `create_or_derive_api_key` client
construction), and three new fee accounting columns (`live_orders.fee_usdc`,
`live_positions.total_fees_usdc`, `dryrun_orders.shadow_fee_usdc`) because V2 charges
fees that V1 did not. The initial cutover commit audited `src/core/abi.py` as
"unchanged" on the basis that V2 `OrderFilled` was byte-identical to V1 — runtime
warnings from the live bot immediately disproved it. V2 adds three new fields (`side`,
`tokenId`, `builder`) that shift all downstream positional offsets, so the ABI decode
produced garbage fills until the event shape was corrected against the actual V2
`Events.sol` source.

Commit range: `a341233...28e7cf1`

## Risk gates counted filled positions, not committed orders

Gates #4 and #5 in `RiskManager.check_order` were measuring the wrong thing.
Gate #4 (`max_concurrent_markets`) queried `LivePosition` with `status=OPEN`, which
only exists after a fill — so 40+ pending orders on PSG O/U 3.5 all passed because
nothing had filled yet. Gate #5 (`max_per_market`) summed `usdc_spent`, which is 0
until fill. Both gates were therefore inert for the entire window between order
submission and first fill, which is exactly the high-risk window they exist to guard.
Fixed to count distinct `condition_id`s from non-terminal `LiveOrder` rows (gate #4)
and to sum `price × size` committed cost for all non-terminal orders (gate #5). The
fix reasoning is still quoted verbatim in the current gate comments in
`src/execution/risk_manager.py`.

Commit: `ea69dd1`

## GTD server-side expiry was a crutch, not a safety net

The initial design assumed a CLOB-enforced server-side auto-expiry (GTD, 3 minutes)
was a necessary floor — "no order survives beyond 3 minutes even if the bot crashes."
In practice the timeout was tuned repeatedly under pressure (3 min → 2 min → 4 min →
10 min) as each value proved too tight or too loose, revealing it was compensating for
bot-side mechanisms not yet built rather than providing genuine safety. Once Phase 1
delivered safe cancel-replace, post-cancel fill reconciliation, unknown-order adoption,
and the stale-order sweeper, the GTD expiry was removed entirely and orders reverted to
GTC. The three remaining safety nets — LiveOddsPoller cancel-replace (15s cadence),
edge-gone cancel, and `_sweep_stale_orders` (600s ceiling) — are all bot-owned and
event-driven, not a server timer.

Commit range: `ecda737...cf2f19a`
