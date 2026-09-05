---
name: prediction-markets
description: Use when trading or researching prediction markets.
---

# Prediction Markets

Working with real-money and play-money prediction markets from Germany. The regulated US venues (Kalshi, Polymarket US) do NOT serve German/EU residents (Polymarket is geoblocked; VPN workarounds violate ToS and risk fund forfeiture — out per Michael's ethics rules). **Manifold (manifold.markets)** is the clean venue: play-money (M$), no fees, no KYC, free documented API, legal in DE. Kalshi fees if ever relevant: taker = 7%·p·(1-p) per contract (max 1.75¢ at 50¢), maker = 25% of taker; not available in DE either.

## Venue facts (Manifold)

- Signup: Google/Apple login ONLY (no email signup) — a headless agent cannot create an account; the user must sign up and hand over an **API key** (Settings → API Keys, format UUID). Auth header: `Authorization: Key <uuid>`.
- Trading costs: zero platform fees; CPMM market maker. But **slippage on thin books is real** — a M$20 order on a small pool can move the price enormously (compute: buying A of YES moves y→y+A, n→k/(y+A), k=y·n; effective price = A/shares).
- Account API: `GET /v0/me` returns balance, username, totalDeposits.

## Workflow: strategy validation (backtest before betting)

Never trade a hypothesis with real M$ before validating against historical bets:

1. Pull open markets: `GET /v0/markets?limit=500&sort=last-bet-time` (verify `outcomeType=='BINARY'`, `isResolved==false`, volume threshold).
2. Pull trade history: `GET /v0/bets?contractId=<id>&limit=1000` — each bet has `probBefore`/`probAfter`/`createdTime`, so the full price series is reconstructable from bets alone.
3. Define the signal mechanically (e.g. price move ≥ 8pp within <1h), measure what follows (momentum vs mean reversion), with entry/exit rules and a per-trade stake.
4. Reality-check the backtest: compute slippage from pool sizes before believing the edge. Backtest P&L uses probAfter as exit — live fills are worse. Expect live edge ≈ 1/3 of backtest.
5. **Paper-trade via cron before real bets**: a monitor script + 15-min cronjob that only logs signals and simulated P&L. Compare live vs backtest for ~2 weeks, then decide.

### Validated finding (2026-09): fade/mean-reversion
Price surges (≥8pp in <1h) in liquid Manifold markets (vol >50k) revert within 2h ~75% of the time; fading them averaged +4-8%/trade in backtest (338 events across 20 markets), before slippage. Pattern is structural (overreaction + LP pressure), not market-specific. Working monitor: `~/manifold/fade_monitor.py` + cronjob.

## Resolution-criteria arbitrage (the 'read the fine print' edge)

Markets price the *colloquial* reading of a question but resolve on the *literal* criteria. Systematically scannable trap taxonomy — see `references/resolution-arbitrage.md` for verified cases:

1. **Definition traps** — 'suit', 'fully autonomous', 'regime fall' (headline ≠ definition)
2. **Time traps** — timezone clauses, 'announced vs occurred', deadlines that excluded already-occurred events (MSTR case)
3. **Source binding** — 'per Wikipedia', 'official announcement only' (the source decides, not reality)
4. **Creator discretion** — 'at my discretion' = your counterparty is the market maker
5. **Resolution lock** — 'resolves when definitive' = open-ended capital lockup

Scan pattern: fetch markets, extract description (rich-text doc dict, needs a text-walking parser — see references), flag criteria keywords vs title, then manually read top-liquid hits where price implies the colloquial reading. Edge decays once the market notices the clause — be early. Ethically clean (nobody is deceived), and Manifold creators can be asked in comments to clarify ambiguous criteria before betting (community norm favors this — it also locks the creator into a reading).

## API gotchas (hard-won)

- `/v0/markets` rejects unknown params (`filter=` → 400). Bets pagination uses `before=<last bet id>` (string ID, NOT timestamp — timestamps 404).
- List endpoint omits `description`; fetch `GET /v0/market/{id}` per market for criteria text.
- `description` is a TipTap doc dict (`{type:doc, content:[...]}`), not a string — walk it for text nodes; `textDescription` is often empty.

References: `references/manifold-api.md` (endpoints + examples), `references/resolution-arbitrage.md` (verified cases + scan recipe).