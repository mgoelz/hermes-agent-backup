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
5. **Paper-trade via cron before real bets**: a monitor script + 15-min cronjob that only logs signals and simulated P&L. Compare live vs backtest for ~2 weeks, then decide. Two suppression patterns so silent ticks never reach the user: (a) `no_agent=true` + script that prints nothing when nothing happened (empty stdout sends nothing — watchdog pattern); (b) for LLM-agent crons, the prompt must instruct: reply with exactly `[SILENT]` (nothing else) on no-change ticks — the user explicitly does not want 'no news' reports. **Time-unit pitfall (bitten once): keep everything in ONE unit** — an early bug added a milliseconds constant to `time.time()` seconds, silently deferring every exit by 83 days.

### Validated finding (2026-09): fade/mean-reversion — and v3 refinements
Price surges (≥8pp in <1h) in liquid Manifold markets (vol >50k) revert within 2h ~75% of the time; fading them averaged +4-8%/trade in backtest (338 events across 20 markets), before slippage. Pattern is structural (overreaction + LP pressure), not market-specific. Working monitor: `~/manifold/fade_monitor.py` + cronjob.

**Live lessons (12 paper trades, +13.6 M$ net):**
- **Fixed time-hold exits are inferior to TP/SL**: take-profit at +5pp in your favor, stop at −10pp against. A −6pp stop measured on this market set was TOO TIGHT — post-stop price audit showed 3 of 4 stopped trades were later 'saved' by the reversal (markets with 15-30pp amplitude need room). When a stop fires, audit what the price did 2h later: if stops keep getting saved, widen the stop and shrink the stake instead of abandoning the thesis.
- **Chop sieve**: count ≥8pp pushes in the last 24h on the market; **>3 → no fade** (volatility trap, not mean-reversion regime). One whipsawing news market (navier-stokes) produced 4 of the 5 losses. This one filter would have prevented the whole losing streak.
- **Concentration kills stats**: one early home-run trade was 82% of net profit; one choppy market took 5 of 12 trades. Cap 1 position per market, max ~4 open, daily loss limit (−30 M$ on a 500 account), stake ~M$10 during system-shakedown, scale only after live hit-rate confirms backtest.
- **v4 entry rules (added after the losing stretch)**: (1) entry window 15¢–85¢ ONLY — fading at extreme prices (e.g. NO at 97¢) is catastrophically asymmetric: pennies of upside, unlimited downside, and the stop can't help; (2) never fade self-referential markets ('THIS ONE', mana-goal markets) — they can fulfill themselves because traders buy YES *because of* the market, so there is no external reality to revert; (3) news sieve: max 1 push per market per 24h — repeated pushes mean an information cycle, not an impulse, and fading news loses (AI-breakthrough hype moved ~5 sister markets together; fades against it all lost).
- **Mini-stakes are noise, not safety (confirmed at 20 trades)**: 16 sub-M$3 trades netted ±0 while 6 large trades carried everything. After the filter set stabilizes, shrink the trade COUNT and grow the stake (M$25+) on concentrated edges instead of spraying M$10 at everything.

## Execution velocity: resting limits beat market orders (maker mode)

Cron polling (even 3-min) is too slow for course explosions — one market went 60¢→99¢ inside minutes in a single bulk fill; no polling frequency could have caught it. The fix is structural: **place resting limit orders in the book BEFORE the move** so the book executes in milliseconds.

Working pattern (`maker_bot.py`, no_agent cron every 3 min):
- **Fade-limits**: on markets with 1–2 pushes/24h, rest a limit 4pp improved behind the current price in fade direction — the next spike fills you AT your price instead of you chasing it.
- **Straddle-maker** on high-amplitude markets (>5 pushes/24h — too choppy to fade, perfect to make): rest BOTH sides (YES at p−8pp, NO at p+8pp). Whichever way the wave breaks, one side fills; volatility pays you instead of stopping you out. Costs nothing while resting.
- Keep MAX ~6 live orders; 48h expiry then re-place.

**Duplicate-execution pitfall (bitten once)**: `POST /v0/bet` with `limitProb` returns an empty body — do not rely on the response for an order ID. If the bot's state stores `bet_id=None`, every tick re-places the same limit and both copies fill (cost: 2× intended exposure). Fix: dedupe against live book state each tick — `GET /v0/bets?username=<name>`, filter `isFilled==false && isCancelled==false` with a `limitProb`, and build an open-keys set of `(contractId, outcome, round(limitProb,2))`; skip placement if the key exists, and sync the local state against that set instead of tracking IDs.

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

Rules: (1) verify the criteria match the benchmark's question (settled vs expected, specific meeting vs year-end, spot vs futures — e.g. WTI markets bind to EIA *spot*, not NYMEX futures); (2) check `closeTime` is near the resolution event so capital isn't locked (prefer <30 days — 'Wetten, die nicht ewig binden'); (3) examples of benchmark pairs: FOMC decisions → CME FedWatch, US elections → Polymarket/538-style aggregates, German state elections → wahlrecht.de/MDR poll tables (also gives the fine-grained party-threshold data Manifold markets resolve on). Election-threshold variant that won (Grüne Sachsen-Anhalt, +49%): when polls put a party repeatedly just ABOVE a threshold (5%-Hürde) and the market prices ~65%, poll-aggregate edge + 'barrier proximity' is a tradeable short-cycle bet — wahlrecht.de/dawum.de carry the per-institute numbers.

## Capital-deployment channels (portfolio frame, 2026-09)

Keep cash deployed across four channel types instead of idle; reserve ~40% cash for new arbitrage arrivals (they appear weekly):
1. **Static criteria-arb** (Millennium-NO pattern) — concentrated, long horizon, manual sizing after safeguard question.
2. **Hype-absorbing limit orders** (Greenland/Iran pattern) — rest NO limits at historically-reached hype levels; costs nothing while resting, no capital bound until filled.
3. **Reference-market arb** (Fed/BTC pattern) — recurring on every FOMC/CPI/NFP date; also works on long-dated crypto targets (BTC $120K: Manifold 8¢ vs Polymarket 13-18¢ on identical question = buy the discount, asymmetric payoff).
4. **'Bond channel'** — park cash in 90-99¢ markets closing <45 days (yield 1-4% per weeks vs 0% idle). Selection scan: probability 0.90–0.995, close ≤45d, volume >M$5k. Only take events with a *near-deterministic* driver (benchmark index milestones, incumbents with stable coalitions); avoid 94-96¢ event-contingent markets (Apple-event style: 4% yield doesn't pay for a tail −100%).

**Thin-book partial-fill gotcha (bitten 2026-09)**: a market-order `POST /v0/bet` without `limitProb` on a thin book (vol ~M$10k, pool YES tiny) filled only M$0.41 of a M$30 order — the unfilled budget is simply refunded, NOT held as an open order. Consequence: the position silently ends up a fraction of intended size. On any market below ~M$20k volume, either use `limitProb` (resting order) or verify the fill size in the response/bets list before counting the exposure as deployed.

### Measurable-resolution markets (holder-count pattern, proven 2026-09)

Best subclass: markets whose resolution variable can be **queried directly via the API before betting**. Example: 'Will YES have more holders than NO?' — resolution = counting distinct net-position holders, computable by aggregating `GET /v0/bets?contractId=<id>` per userId (+shares YES, −shares NO). Measured 36 YES vs 15 NO holders with the market at 81¢ → buy YES. Structural bonus: 1 person = 1 holder regardless of size, so whales can't flip it — only many small new entrants on the other side can. Before betting, write the counting query and confirm the number actually supports your side; if the measurable quantity says the market price is fair, skip.

### Cross-market consistency check (avoid false arb)

Before trading a 'sister' market, verify pricing is internally consistent across related markets on overlapping events. Case: 'Alcaraz ≥2 Grand Slams in 2026' at 59¢ — verified 2026 slam winners (AO Alcaraz, RG Zverev, Wimbledon Sinner) meant he needed the US Open title, and the direct 'Alcaraz wins US Open' market sat at 58¢ — consistent pricing, zero edge, skip. Reconstruct the fact base (who won what) from Wikipedia/search BEFORE paying for a market whose answer is already implied by a sibling market.

## API gotchas (hard-won)

- `/v0/markets` rejects unknown params (`filter=` → 400). Bets pagination uses `before=<last bet id>` (string ID, NOT timestamp — timestamps 404).
- List endpoint omits `description`; fetch `GET /v0/market/{id}` per market for criteria text.
- `description` is a TipTap doc dict (`{type:doc, content:[...]}`), not a string — walk it for text nodes; `textDescription` is often empty.
- `POST /v0/bet` with `limitProb` returns an EMPTY body (no bet id) — confirm placement via the open-orders query, and dedupe placements against the live open-orders list, not stored IDs.

References: `references/manifold-api.md` (endpoints + examples), `references/resolution-arbitrage.md` (verified cases + scan recipe).