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

## Placing bets (paper → live switch)
`POST /v0/bet {contractId, amount, outcome}` — not yet exercised live (paper phase until live numbers confirm backtest).

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