# Manifold API — endpoints, auth, worked examples

## Auth
Header on private endpoints: `Authorization: Key <api-key-uuid>`.
`GET https://api.manifold.markets/v0/me` → `{username, balance, totalDeposits, id, ...}` (worked 2026-09, account HackDaMarket).

## Markets
```
GET /v0/markets?limit=500&sort=last-bet-time        # list (NO description field)
GET /v0/market/{id}                                  # single market WITH description
```
- Rejected params return `400` `Error validating request` (e.g. `filter=open`).
- Fields: `outcomeType` ('BINARY'), `isResolved`, `probability`, `volume`, `closeTime` (ms epoch), `pool {YES, NO}`, `id`, `slug`, `url`.
- `closeTime` can be absurd (year 2100+) — treat as 'no real deadline'.

## Bets (price history)
```
GET /v0/bets?contractId=<marketId>&limit=1000&before=<lastBetId>
```
- Filter param is `contractId` (NOT `market`, NOT `marketId` — both 400).
- Pagination: `before` takes the **bet ID string** of the last item; passing a timestamp → HTTP 404.
- Each bet: `probBefore`, `probAfter`, `createdTime`, `amount`, `shares`, `outcome`, `isFilled`, `isCancelled`, zero `fees`. Price series = sort by createdTime, probAfter per bet.

## Placing bets (verified live 2026-09-05)
`POST /v0/bet {contractId, amount, outcome}` — works immediately after signup (only comments are locked 7 days). Response body may be EMPTY even on success → verify via `GET /v0/bets?username=<name>` and check balance in `/v0/me`.

Two measured slippage cases (2026-09-05):
- THIN book: Millennium-Problem market, M$1 + 4×M$25 tranches (2s pauses) → price 45¢→37¢→36¢, ~159 NO-shares for M$101 (Ø 36¢ vs 55¢ pre-trade). Tranches limit but don't remove impact; on thin books prefer resting limit orders.
- NORMAL book: Greens-Sachsen-Anhalt market — M$2 probe then M$13 both filled at exactly mid (eff 65.5-65.7¢), price moved 65.46→65.53 only. Active markets absorb M$15-20 slip-free.
- **The `pool {YES, NO}` fields do NOT predict execution depth.** Pool-math (`n'=n+A; s=y−k/n'`) predicted 80-400% slippage on markets that filled slip-free at mid, and a market it flagged as unusable accepted M$20 at 54-56¢. Never pre-compute slippage from pools — probe empirically (M$2-5 test bet, check fill price + probBefore→probAfter), then tranche.

Limit orders use the same endpoint: `POST /v0/bet {contractId, amount, outcome, limitProb}` — `isFilled:false` in the response confirms a resting order (verified 3× 2026-09-05). Confirm via `GET /v0/bets?username=<name>` filtered to `isFilled==false && isCancelled==false`. Cancel endpoint unverified — test before relying on it.

## Comments
`POST /v0/comment {contractId, markdown}` → **403 'Commenting on other users' markets unlocks 7 days after signup'** during the lock. Reading comments is always allowed: `GET /v0/comments?contractId=<id>`. Schedule creator-clarification questions via cron for day 7+.

## Description parsing
`description` is a TipTap doc dict:
```python
def desc_text(j):
    d = j.get('description'); parts = []
    def walk(o):
        if isinstance(o, dict):
            if o.get('type')=='text' and 'text' in o: parts.append(o['text'])
            for v in o.values(): walk(v)
        elif isinstance(o, list):
            for v in o: walk(v)
    walk(d)
    return ' '.join(parts)
```
`textDescription` field is usually empty — do not rely on it.

## Practice
Sleep ~0.25s between bet pages; 3 pages × 1000 bets covers recent history of liquid markets (bets come newest-first).