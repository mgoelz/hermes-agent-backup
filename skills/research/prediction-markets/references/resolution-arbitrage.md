# Resolution-criteria arbitrage — verified cases + scan recipe (2026-09)

## Verified cases

1. **Zelensky suit** (Polymarket 2025, >$100M volume): 'Will Zelensky wear a suit before July?' He did, colloquially. Resolved NO on a strict 'business suit' definition. (Sportico covered the UMA resolution fights.)
2. **Ukraine minerals deal** (Polymarket 2025-03): 'deal announced by month end' priced 15-20¢ (signing had collapsed). A whale forced a YES resolution via the ambiguous clause 'announcement without enactment' (two dictionary senses). >$500k reallocated NO→YES. An identical market closing 3 days earlier had resolved NO un-contested. (mr.ozi Substack chronology.)
3. **MicroStrategy BTC sale** (Polymarket): sale provably occurred on-chain, but criteria required confirmation *within the market timeframe*; MSTR confirmed 2 days late → NO. Time-window traps exclude events that objectively happened.
4. **Manifold 'Tesla more fully autonomous rides than Waymo 2026'** (~p=0.04, vol>1M): criteria define 'fully autonomous' strictly (no human at controls, no monitor aboard, non-trivial distance, no fixed track). Definition ≈ the whole edge; headline readers get run over.
5. **Manifold 'LLM beats super-GM at chess'** (p≈0.49): 'I will ignore fun games, at my discretion' + creator clarification excluding purpose-built chess AIs. Creator discretion = hidden veto.
6. **Manifold 'Did COVID come from a lab?'** (p≈0.29): resolves only at '98% definitive answer… many years after'. Open-ended capital lockup not reflected in the price.
7. **Russian town capture via fake map** (Manifold 2025): manipulated ISW map accepted as resolution evidence; reversed after discovery. Source-quality trap.
8. **Manifold 'Millennium Problem solved before 2028'** (p=45¢, 2026-09-05): criteria require OFFICIAL CMI announcement; preprints explicitly excluded ('will not qualify'); CMI rules impose a 2-year post-publication scrutiny window → YES practically impossible before the 1.1.2028 deadline despite the live 'Claude solved Navier-Stokes' hype. Entered M$101 NO at Ø36¢ (tranches, see manifold-api.md). Safeguard: creator-clarification comment scheduled for day 7 (comment lock); if creator signals 'intent' resolution → exit.
9. **Manifold Greenland markets 2026** (p≈2-3¢): 'binding legal instrument / sovereignty' only; bases, MOUs, leases, SOFA explicitly NO. Price history: 53 spikes ≥5pp, hype top 50¢ (Jan 2026 Trump headlines) → 2¢. Template for the limit-order-into-hype mode: resting NO limits at YES≈40¢ fill during the next wave.
10. **Manifold 'Iran's regime fall in 2026'** (p≈4.5¢, vol 3M): YES needs BOTH core-institution dissolution AND loss of majority-population control per Reuters/AP/BBC/NYT consensus; creator's own example says a Syria-style territorial-loss scenario resolves NO. June 2026 escalation spike 14¢→4.5¢. Same fade-the-hype template.

## Trap taxonomy (scan keywords)
| Trap | Keywords in criteria |
|---|---|
| Definition | 'defined as', 'for the purposes of this market', enumerated criteria lists |
| Time | 'UTC', 'Pacific Time', 'by 11:59', 'announced', 'before/on', deadline-vs-confirmation |
| Source binding | 'per wikipedia', 'according to', 'official announcement', named site/exchange |
| Creator discretion | 'at my discretion', 'I will decide', 'in my judgement', 'I resolve' |
| Resolution lock | 'resolves when definitive', 'once we know', no close date |

## Scan recipe
1. `GET /v0/markets?limit=500&sort=last-bet-time`, keep open BINARY with vol > threshold.
2. `GET /v0/market/{id}` each, parse rich-text description, keyword-match the taxonomy.
3. For hits: compare title's colloquial reading vs criteria literal reading vs current price. Edge exists where price ≈ colloquial probability but criteria make the literal reading materially different.
4. Manually read top hits; optionally ask the creator in comments to clarify ambiguous criteria BEFORE betting (community norm — also pins their interpretation).
5. Size like any trade (1-3% of bankroll); edge decays once the market reads the fine print.

## Ethics note (Michael's hard rule)
Reading fine print is clean — no deception, no resolution manipulation (the mineral-deal whale behavior and fake-map submission are the anti-patterns to never touch).