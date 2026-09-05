---
name: prediction-markets
description: Use when trading or researching prediction markets.
---

# Prediction Markets

Working with real-money and play-money prediction markets from Germany. The regulated US venues (Kalshi, Polymarket US) do NOT serve German/EU residents (Polymarket is geoblocked; VPN workarounds violate ToS and risk fund forfeiture — out per Michael's ethics rules). **Manifold (manifold.markets)** is the clean venue: play-money (M$), no fees, no KYC, free documented API, legal in DE. Kalshi fees if ever relevant: taker = 7%·p·(1-p) per contract (max 1.75¢ at 50¢), maker = 25% of taker; not available in DE either.

## Venue facts (Manifold)

- Signup: Google/Apple login ONLY (no email signup) — a headless agent cannot create an account; the user must sign up and hand over an **API key** (Settings → API Keys, format UUID). Auth header: `Authorization: Key <uuid>`.
- Trading costs: zero platform fees; CPMM market maker. But **slippage on thin books is real and measured**: a M$101 entry (4×25 tranches) moved one market 45¢→36¢ (~15% vs mid). Always split entries into tranches, compare achieved avg price to pre-trade mid; prefer resting limit orders on thin books.
- Account API: `GET /v0/me` returns balance, username, totalDeposits.
- **Comment lock**: comments on OTHER users' markets are blocked for **7 days after signup** (POST /v0/comment → 403 'unlocks 7 days after signup'); bets work immediately. Plan the criteria-clarification question for day 7+ via cron.
- Live bets: `POST /v0/bet {contractId, amount, outcome}` — verified working immediately after signup; response body may be empty → confirm via `GET /v0/bets?username=<name>`.

## Workflow: strategy validation (backtest before betting)

Never trade a hypothesis with real M$ before validating against historical bets:

1. Pull open markets: `GET /v0/markets?limit=500&sort=last-bet-time` (verify `outcomeType=='BINARY'`, `isResolved==false`, volume threshold).
2. Pull trade history: `GET /v0/bets?contractId=<id>&limit=1000` — each bet has `probBefore`/`probAfter`/`createdTime`, so the full price series is reconstructable from bets alone.
3. Define the signal mechanically (e.g. price move ≥ 8pp within <1h), measure what follows (momentum vs mean reversion), with entry/exit rules and a per-trade stake.
4. Reality-check the backtest: expect live edge ≈ 1/3 of backtest — live fills are worse than probAfter exits. **Do NOT trust `/v0/markets` `pool` numbers for slippage math** (falsified 2026-09: pool k=y·n math predicted 80-400% slippage on a market that filled M$15 at exactly mid, zero movement). The v0 pool fields do NOT reflect execution depth. The only reliable test is empirical: probe with M$2-5, compare fill's effective price and the market's probBefore→probAfter movement, then scale in tranches. Active markets routinely absorb M$20 slip-free at mid.
5. **Paper-trade via cron before real bets**: a monitor script + 15-min cronjob that only logs signals and simulated P&L. Compare live vs backtest for ~2 weeks, then decide. Cron pattern: `attach_to_session=true, deliver=origin`; script prints ONLY on open/close so silent ticks deliver nothing. **Time-unit pitfall (bitten once): keep everything in ONE unit** — an early bug added a milliseconds constant to `time.time()` seconds, silently deferring every exit by 83 days.

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

### Execution playbook (proven 2026-09)

1. **Price-history forensics as evidence**: list historical ≥5pp spikes with dates. Markets whose YES spikes repeatedly collapsed on the SAME clause (Greenland: 50¢ hype-top → 2¢, 53 spikes) are prime candidates — the clause provably kills every hype wave.
2. **Two execution modes**: (a) *static mispricing* → tranche market buys (Millennium-Problem case: NO at 45¢ when criteria made YES near-impossible); (b) *news-hype overreaction* → resting NO limit orders placed at levels the hype historically reaches, filling INTO the next spike — become the liquidity that absorbs the overreaction instead of paying slippage for an immediate position.
3. **Safeguard before sizing**: publicly ask the creator how the edge case resolves (respect the 7-day comment lock — schedule it). Answer pins their interpretation; 'market intent' signals → exit/cap.
4. Skip the pool-math check — probe empirically instead (see Workflow step 4): M$2-5 test bet, measure fill price + price movement, then tranche the rest. Confirm tranche 1 before scaling.

## Reference-market mismatch (third edge class)

Same binary event, different venues with different prices. Manifold is a thin, retail-heavy market; professional benchmarks (CME FedWatch for Fed decisions, Polymarket for US politics/macro, poll aggregators/wahlrecht.de for elections) are better calibrated. When Manifold deviates materially from the benchmark on the SAME question, trade toward the benchmark — the mismatch, not a forecast, is the edge. Proven 2026-09: Manifold 'Fed hike September' at 53¢ vs CME FedWatch 59.4% and Polymarket equivalent 72% → YES bought at ~54.5¢.

Rules: (1) verify the criteria match the benchmark's question (settled vs expected, specific meeting vs year-end, spot vs futures — e.g. WTI markets bind to EIA *spot*, not NYMEX futures); (2) check `closeTime` is near the resolution event so capital isn't locked (prefer <30 days — 'Wetten, die nicht ewig binden'); (3) examples of benchmark pairs: FOMC decisions → CME FedWatch, US elections → Polymarket/538-style aggregates, German state elections → wahlrecht.de/MDR poll tables (also gives the fine-grained party-threshold data Manifold markets resolve on).

## API gotchas (hard-won)

- `/v0/markets` rejects unknown params (`filter=` → 400). Bets pagination uses `before=<last bet id>` (string ID, NOT timestamp — timestamps 404).
- List endpoint omits `description`; fetch `GET /v0/market/{id}` per market for criteria text.
- `description` is a TipTap doc dict (`{type:doc, content:[...]}`), not a string — walk it for text nodes; `textDescription` is often empty.

References: `references/manifold-api.md` (endpoints + examples), `references/resolution-arbitrage.md` (verified cases + scan recipe).