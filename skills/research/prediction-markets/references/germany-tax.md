# German tax treatment — prediction markets & crypto (researched 2026-09; NOT legal advice)

Re-check before tax filing: rates/freigrenzen change annually; DAC8 makes exchange data EU-visible from 2026.

## Crypto on regulated exchanges (Coinbase/Kraken/Binance — § 23 EStG, settled law)

- **1-year holding period**: gains on coins held >12 months are fully tax-free, unlimited amount. Frist starts the day AFTER acquisition. The single most powerful legal lever — but incompatible with short-cycle trading.
- **Coin-to-coin swaps are taxable disposables** (BTC→ETH resets everything) — the most common surprise.
- **FIFO, wallet-based** (BMF 2022, Rz. 61): oldest coins count as sold first, per wallet. Separate HODL and trading wallets so old tax-free coins aren't 'consumed' by new trades.
- **1.000 €/year Freigrenze — all-or-nothing**: gains below it are tax-free; at 1.000,00 € the ENTIRE gain becomes taxable (not just the excess). Counts ALL private Veräußerungsgeschäfte (metals, art, etc.) together.
- **Staking/lending/mining**: separate 256 € Freigrenze (§ 22 Nr. 3), taxed at receipt (market value that day). No 10-year holding extension (BMF final letter 2022). New holding period starts at receipt for rewards.
- **Tax rate within holding period**: personal income tax rate (0–45%).
- **DAC8**: from 2026, exchanges report EU-wide. Documentation is non-optional.

## Prediction-market winnings (Polymarket/Kalshi/Manifold) — UNSETTLED, three competing readings

1. **Tax-free gambling win** (like Sportwetten): argument = zero/negative expected value, no skill. Weak for us: a documented systematic algorithm suggests skill, undermining the gambling classification.
2. **§ 22 Nr. 3 sonstige Einkünfte** (256 € Freigrenze): applies if the Finanzamt sees 'Gewinnerzielungsabsicht'. LIKELY our reality — a documented bot with analytics is the profile that triggers this.
3. **§ 23 EStG privates Veräußerungsgeschäft**: crypto tax tools auto-classify Polymarket bets as trades (USDC ↔ conditional-token NFT). Each bet = acquisition, each resolution/sell = disposal.

All three readings exist in current literature; pick ONE consistently, document the reasoning, disclose everything. The pekuna.de overview is the best German-language treatment found.

**Gewerblich risk (the real danger)**: BFH X R 43/12 (poker) — sustained, skill-based, profit-seeking activity can be classified gewerblich (§ 15 EStG): income tax + Gewerbesteuer + Umsatzsteuer questions. A persistent automated bot with positive P&L is exactly this profile. At four-figure annual profits, consult a Steuerberater about a structure (GbR/GmbH) BEFORE the Finanzamt decides unilaterally.

## Module pattern (built for Manifold, portable)

- `tax_log.py`: pulls ALL filled bets via API (paginated), writes `tax_log.csv` (datum, markt, event, outcome, einsatz, shares, kurs, erlös, classification per row).
- `tax_fifo.py`: FIFO-matches BUY↔SELL per (market, outcome), computes per-disposal cost basis / proceeds / G/V / holding-days / 1-year-rule status → `tax_fifo.csv` + `tax_summary.json` (per-year: realized G/V, steuerpflichtig vs steuerfrei split).
- `EUR_PER_MANA` constant: 0.01 placeholder on Manifold play money; becomes the actual EUR rate per transaction on real-money venues.
- Run both after every trading session; the CSV is the source of truth, state files are just caches.
- Known gap: resolution payouts don't carry a `profit` field in the bets API — match resolved markets' payouts against positions at resolution time.

## Standing rules

- Log every real trade the day it happens — reconstruction from chain history later is hours of work.
- Keep the checked-and-skipped list too: it proves fair-priced markets were declined, which supports the 'not gambling, systematic process' narrative either way.
- For final classification of meaningful sums: Steuerberater with crypto/prediction-market experience. This file is analysis, not advice.