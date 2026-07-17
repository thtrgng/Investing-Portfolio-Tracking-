# Portfolio — Personal Multi-Asset Portfolio Tracker

## Project Specification

## 1. Vision

A single-user web application that tracks a real, personally-held investment portfolio spanning Vietnamese open-ended funds, physical gold, Vietnamese equities, cryptocurrency, and cash in two currencies. It answers three questions accurately and continuously: what do I own, what is it worth, and how much have I made or lost.

The accounting currency is VND. Every asset, including USD cash and USD-quoted crypto, is valued in VND. There is no multi-currency accounting.

This is not a trading platform. It executes nothing, connects to no broker, and holds no credentials. It is a bookkeeping system with a live price feed — the user records what they did, and the app tells them the truth about it.

Correctness outranks features. A tracker that is beautiful and wrong is worse than useless, because the user will believe it.

## 2. User Experience

### First Launch

The user signs in (Supabase Auth, single account) and lands on an empty portfolio with a prompt to seed it. Seeding is manual — the user types in what they currently hold. There is no CSV import in v1.

The seed flow produces ordinary transactions, not a special record type:

1. Pick an **opening date**
2. Record one `DEPOSIT` of `CASH_VND` equal to the total VND cost basis of everything held, plus VND cash on hand
3. Record one `BUY` per holding, dated at its real purchase date where known, otherwise the opening date, with `settle_amount` set to the real all-in cost
4. For USD cash, record a `BUY` of `CASH_USD`: quantity in USD, `settle_amount` in the VND actually spent — the system derives and stores the cost rate
5. Verify that the resulting `CASH_VND` balance matches the real bank balance exactly

Real purchase dates improve XIRR and DCDS lot ages. Cost basis matters more than dates — if a date is unknown, use the opening date rather than inventing one.

### What the User Can Do

- **See the portfolio board** — total NAV, today's change, total P&L split into realized and unrealized, XIRR, and allocation by category
- **Drill into any position** — quantity, average cost, market price with its own freshness stamp, market value, portfolio weight, unrealized P&L, realized P&L
- **Record a buy or sell** — pick the asset, pick the settlement account (VND or USD), enter quantity and the all-in amount that actually left or entered the account
- **Buy USD** — enter the USD received and the VND spent; the system stores the implied rate and tracks FX gain on the position from then on
- **Spend USD on crypto** — the app realizes the FX gain or loss on the USD at that moment and shows it in the preview before the transaction is recorded
- **Deposit and withdraw cash** — kept distinct from buying, so that funding the account is never mistaken for profit
- **Watch NAV over time** — a chart built from daily snapshots
- **Set target allocations** — per category, with a tolerance band and a drift warning
- **Override any price by hand** — for when a provider is down, or for assets the user would rather price themselves
- **Export everything to CSV** — this is personal financial data; it must be possible to get it out

### Visual Design

The colour language is taken from the Vietnamese exchange price boards (HOSE/HNX). This is not decoration. **Green means up and red means down** — the inverse of the Western convention. The user has spent years reading boards this way; inverting it would make every screen misread at a glance.

- **Dark board background**: deep navy, around `#0B1220`, never pure black
- **Every number carries its own freshness**: a timestamp or staleness badge beside each price. Asset classes are never in sync — on a Saturday crypto is live while the VN exchanges are closed and DCDS has no NAV. That is the normal state, and the UI must show it rather than imply that everything is live
- **Data-dense, quiet chrome**: flat surfaces, hairline dividers, no gradients, no shadows. The numbers are the content
- **Signature element — the NAV ticker**: NAV rendered large in tabular mono, shifting up/down colour with a short transition, freshness badge attached
- **Desktop-first**, functional on mobile
- **Typography must render stacked Vietnamese diacritics** (ế, ộ, ữ). Many display faces fail this — it is a hard constraint, not a preference. Display and body: **Be Vietnam Pro**. All numbers: **JetBrains Mono** with `font-variant-numeric: tabular-nums`, so columns align

### Color Scheme
- Board Background: `#0B1220`
- Surface: `#141C2E`
- Up (tăng): `#00C176`
- Down (giảm): `#F0334B`
- Reference (tham chiếu): `#F5C518`
- Ceiling (trần): `#C05CE8`
- Floor (sàn): `#22D3EE`
- Ink: `#E6EDF7` / Ink Dim: `#7C8AA3`

---

## 3. Architecture Overview

### Static Frontend, Managed Backend

```
┌──────────────────────────────────────────────────┐
│  Cloudflare Pages                                │
│  Vite + React + TypeScript (static SPA)          │
└───────────────────┬──────────────────────────────┘
                    │ HTTPS (supabase-js)
┌───────────────────▼──────────────────────────────┐
│  Supabase — region: Singapore                    │
│  ├── Postgres         numeric, source of truth   │
│  ├── Edge Functions   Deno — price providers     │
│  ├── pg_cron + pg_net scheduler                  │
│  └── Auth + RLS       single account             │
└───────────────────┬──────────────────────────────┘
                    │ egress from Singapore
      ┌─────────────┼──────────────┬───────────────┐
   Binance      Vietcap        FMarket      SJC / BTMC
   (crypto,     (VN equities)  (DCDS NAV)   (gold)
    P2P FX)
```

- **Frontend**: Vite + React + TypeScript, static build, deployed to Cloudflare Pages
- **Backend**: Supabase Edge Functions (Deno) for price fetching and NAV snapshots
- **Database**: Supabase Postgres. All money and quantities are `numeric`
- **Engine**: a pure TypeScript module in `engine/`, imported by both the frontend and the snapshot function. No I/O, no clock reads, no database access
- **Scheduler**: `pg_cron` + `pg_net` calling Edge Functions on a schedule
- **Region is Singapore and this is load-bearing** — see the table below

### Why These Choices

| Decision | Rationale |
|---|---|
| Ledger is the only writable table | Positions, NAV, and P&L are always derived by replaying `transactions`. Never store a mutable `holdings.quantity` and adjust it per trade — one bug corrupts the balance permanently with no way back. With replay, a bad transaction is edited and the truth recomputes. A few hundred transactions replay in milliseconds |
| Postgres `numeric`, never float | Money is not approximable. VND has no decimals, BTC has eight. `decimal.js` in TypeScript, `numeric` in Postgres. Floats appear only in the charting layer |
| Weighted-average cost basis | Vietnam does not tax capital gains by lot — the equity sale tax is 0.1% of proceeds, unrelated to cost. FIFO would add real complexity for no benefit. Lot ages remain recoverable from the ledger when needed |
| Realize FX P&L immediately | Spending USD on crypto is treated as selling the USD at the spot rate of that moment. FX gain locks in there; crypto cost basis is USD × spot. The alternative buries FX gains inside the crypto cost basis, making "BTC went up" and "USD went up" indistinguishable |
| Cash USD is an asset, not a currency | Its "price" is the VND/USD rate. Buying USD becomes an ordinary `BUY`: quantity in USD, settlement in VND, cost rate derived. No FX code path exists anywhere — the engine has one shape for everything |
| `settle_amount` is all-in | It is the money that actually moved, fees and taxes included. `fee_vnd` and `tax_vnd` are reporting fields the engine never reads. Cash balances then reconcile exactly, and fees surface automatically as unrealized loss the moment a position opens — which is economically true |
| Value at the exit price | Gold and USD are valued at `bid` — what the shop or counterparty would pay. Gold spreads are wide; valuing at the ask makes the portfolio permanently, invisibly overstated |
| P2P rate for valuation, not VCB | The user buys USDT on P2P, so the P2P rate is what they actually face. The VCB rate is stored in `fx_reference` for display only and never enters the valuation path |
| Two gold assets, not one | SJC bars and 9999 rings have materially different prices and spreads. Merging them corrupts both allocation and P&L. Both are stored in **chỉ** |
| TypeScript everywhere, no Python | `vnstock` is the obvious library for VN market data but it is Python and cannot run on Deno or Workers. It is also open source and is only an HTTP wrapper, so its endpoints were read from source and are called directly (Section 6). This removes a second runtime and a second deployment for the sake of three tickers |
| Price fetching in Supabase Edge Functions, not Cloudflare Workers | `api.binance.com` returns HTTP 451 to US IPs. Workers egress from an unpredictable colo. Edge Functions egress from the chosen region — Singapore — which is both reachable and adjacent to the database |
| Vite SPA over Next.js | Nothing here needs SSR. It sidesteps the Next-on-Cloudflare adapter entirely |
| Manual override on every provider | Sources will break. Every asset degrades to a hand-entered price rather than to an error or, worse, a silently stale number |

---

## 4. Directory Structure

```
portfolio/
├── web/                      # Vite + React + TS (Cloudflare Pages target)
├── engine/                   # Pure calculation module — no I/O, no clock, no DB
│   ├── src/
│   └── tests/                # Golden tests + invariant tests
├── supabase/
│   ├── migrations/           # Schema + seed
│   └── functions/            # Edge Functions (Deno)
│       ├── prices-crypto/
│       ├── prices-fx-p2p/
│       ├── prices-vn-stock/
│       ├── prices-gold/
│       ├── prices-dcds/
│       ├── fx-reference/
│       └── snapshot-nav/
├── planning/
│   └── PLAN.md               # This document
├── test/                     # Playwright E2E
├── .env.local                # gitignored; .env.example committed
└── .gitignore
```

Rename the root to taste — nothing depends on it.

### Key Boundaries

- **`engine/`** is the heart and is deliberately isolated. It exports pure functions of the shape `(transactions, prices) → positions | nav | xirr`. It performs no I/O, reads no clock, and knows nothing about Postgres, HTTP, or React. This is what makes it exhaustively testable, and it is why the engine can be finished and proven correct before any provider exists. Its only dependency is `decimal.js`. Both `web/` and `supabase/functions/snapshot-nav/` import it — Deno resolves `decimal.js` via an `npm:` specifier.
- **`web/`** is a self-contained Vite project. It talks to Postgres through `supabase-js`, calls the engine for every derived number, and computes nothing financial on its own. Internal component structure is up to the Frontend Engineer.
- **`supabase/functions/`** holds one Edge Function per price domain. Each one fetches, **normalises to the asset's declared unit**, and writes to `prices`. Provider quirks live here and nowhere else. `snapshot-nav` is the exception: it reads the ledger, calls the engine, and writes a row to `nav_snapshots`.
- **`supabase/migrations/`** owns the schema and the seeded `assets` rows. The schema is the contract between the engine and the database.
- **`planning/`** is the shared contract for all agents.
- **`test/`** holds Playwright E2E specs. Engine unit tests live in `engine/tests/`.

---

## 5. Environment Variables

```bash
# Frontend — public by design, safe in the browser, protected by RLS
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key

# Edge Functions — injected automatically by Supabase at runtime.
# Only needed for local `supabase functions serve`.
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
```

### Behavior

- The anon key is meant to be public. Every table has RLS enabled; the anon key alone grants nothing without an authenticated session
- The service role key bypasses RLS and must never reach the browser or the repository. It exists only inside Edge Functions
- No price source requires an API key. Any code that asks for one is a mistake — re-read Section 6
- The Supabase project **must** be created in the Singapore region. This is not a preference and cannot be changed after creation

---

## 6. Market Data

### One Interface, Many Providers

Every provider implements the same interface and is selected per asset via `assets.provider`:

```ts
interface Quote {
  assetId: string
  bid?: Decimal
  ask?: Decimal
  mid: Decimal
  ts: Date
  source: string
}

interface PriceProvider {
  id: string
  fetch(assets: Asset[]): Promise<Quote[]>
}
```

**The provider normalises to `assets.unit` before returning.** Gold sources quote VND per *lượng*; the asset is denominated in *chỉ*; the provider divides by 10. The engine contains no unit conversions and no per-asset special cases. Any `if (asset.category === 'GOLD')` inside the engine is a bug in the wrong file.

### Fallback Chain

Every asset degrades in the same order, and never throws to the UI:

**primary → fallback → last known price (flagged stale) → manual entry required**

A stale price is always shown with its age. A missing price is shown as missing. A fabricated price is never shown.

### Endpoint Reference

These endpoints were read directly from the `vnstock` source (open source, HTTP-only wrappers) and are called here without the library. Everything below is keyless and unauthenticated.

**Required headers.** Several of these hosts reject requests without a browser-shaped `User-Agent` plus matching `Referer` and `Origin`. Omitting them is a common silent failure:

| Source | Referer / Origin |
|---|---|
| Vietcap | `https://trading.vietcap.com.vn/` |
| FMarket | `https://fmarket.vn/` |
| SJC | `https://sjc.com.vn` |

**`binance`** — BTC, ETH, SUI, SOL
```
GET https://api.binance.com/api/v3/ticker/price?symbols=["BTCUSDT","ETHUSDT","SUIUSDT","SOLUSDT"]
```
One request for all four. `mid = price`, quoted in USD. Do not create an API key — this is public market data and a key adds only risk. Returns HTTP 451 to US IPs; see Section 3. Fallback: CoinGecko `simple/price`.

**`binance_p2p`** — CASH_USD, the valuation rate
```
POST https://p2p.binance.com/bapi/c2c/v2/friendly/c2c/adv/search
{"asset":"USDT","fiat":"VND","tradeType":"SELL","page":1,"rows":10,"payTypes":[]}
```
`tradeType: "SELL"` returns the ads the user could sell into — the rate actually receivable. Take the **median of the top five**, not the best ad; the top of the book is usually bait or a trivial size. Sets `bid`, and `bid` is what values the position.

⚠️ **This is an internal endpoint and is the only unverified source in the project.** Verify it before building anything else (Section 13). If it is gone, CASH_USD falls back to weekly manual entry, which is still more accurate than substituting the VCB rate.

**`vietcap`** — QTP, SIP, HPG
```
POST https://trading.vietcap.com.vn/api/price/symbols/getList
{"symbols":["QTP","SIP","HPG"]}
```
One request for all three. Vietcap's constants include `UPCOM`, so QTP should be covered — but **QTP must be tested by itself**, as many VN sources cover only HOSE and HNX.

**`fmarket`** — DCDS, `product_id = 28`
```
GET  https://api.fmarket.vn/res/products/28
POST https://api.fmarket.vn/res/product/get-nav-history
     {"isAllData":1,"productId":28,"fromDate":null,"toDate":"YYYYMMDD"}
```
Note the **singular** `product` in the history path — the plural is `products` everywhere else, and this trips people up. `get-nav-history` with `isAllData:1` returns the entire NAV history, which means historical DCDS NAV can be backfilled into `nav_snapshots`. Fallback: scrape the public page at `https://fmarket.vn/quy/DCDS`, which carries the latest NAV and its date.

DCDS NAV is **not real time**. Matching runs Monday to Friday with a 14:30 cut-off, and the published NAV routinely lags one to three days. The UI must not imply otherwise. Staleness threshold: 4 days.

**`gold_sjc`** — GOLD_SJC, GOLD_RING
```
GET https://sjc.com.vn/GoldPrice/Services/PriceService.ashx
```
First-party SJC. **Response shape needs verification** — confirm the field names and which rows correspond to bars (miếng) versus rings (nhẫn 9999) before mapping.

Fallbacks, in order: BTMC (`http://api.btmc.vn/api/BTMCAPI/getpricebtmc?key=...` — note the key is hard-coded inside vnstock and may be rotated or throttled), then `https://www.vang.today/api/prices` (`SJL1L10` = SJC bars, `SJ9999` = SJC rings; free, 5-minute refresh, CORS enabled, but a personal site with no SLA and a robots.txt that disallows `/api/`), then manual.

**All gold sources quote VND per lượng. Divide by 10 for VND per chỉ.** `bid = buy ÷ 10`, `ask = sell ÷ 10`.

**`vcb_ref`** — reference only, never valuation
```
GET https://www.vietcombank.com.vn/api/exchangerates?date=YYYY-MM-DD
```
First-party VCB, JSON. USD returns `cash`, `transfer`, and `sell`. Writes to `fx_reference`. The `?date=` parameter accepts past dates, which makes historical USD/VND backfill possible — useful when seeding old transactions, since P2P has no history.

**`const_one`** — CASH_VND. Returns `mid = 1` forever. No network call, no row in `prices`.

### Schedule

Driven by `pg_cron`. Times are ICT (UTC+7).

| Job | Schedule | Stale after |
|---|---|---|
| `prices-crypto` | every 5 min | 15 min |
| `prices-fx-p2p` | every 5 min | 60 min |
| `prices-vn-stock` | every 15 min, 09:00–15:10, Mon–Fri | 30 min in session |
| `prices-gold` | every 30 min | 2 h |
| `prices-dcds` | 18:30 Mon–Fri, retry 08:00 | 4 days |
| `fx-reference` | 09:00 daily | — |
| `snapshot-nav` | 23:50 daily | — |

`snapshot-nav` is not optional and cannot be added later. Gold price history is unavailable beyond a short window, so a day not snapshotted is a day permanently missing from the NAV chart.

---

## 7. Database

### Postgres with Migrations

Schema and seed data live in `supabase/migrations/`. The `assets` table is reference data seeded at migration time. Every table has RLS enabled — single user, but enabled regardless.

### Schema

**assets** — Asset registry. Static, seeded, rarely changes.
- `id` TEXT PRIMARY KEY (`DCDS`, `GOLD_SJC`, `BTC`, `CASH_USD`, …)
- `symbol` TEXT, `name` TEXT
- `category` TEXT (`FUND` | `GOLD` | `VN_STOCK` | `CRYPTO` | `CASH`)
- `quote_ccy` CHAR(3) — the currency the **raw price** is quoted in. `BTC` → `USD`. `CASH_USD` → `VND`, because the price of one USD is a number of VND
- `unit` TEXT (`ccq` | `chi` | `cp` | `coin` | `USD` | `VND`)
- `qty_scale` SMALLINT — decimal places for quantity
- `valuation` TEXT (`mid` | `bid`) — which side values the position
- `provider` TEXT, `provider_cfg` JSONB (e.g. `{"product_id": 28}`)
- `is_cash` BOOLEAN, `sort_order` SMALLINT

**transactions** — The only table the user writes to. Everything else is derived from it.
- `id` UUID PRIMARY KEY
- `ts` TIMESTAMPTZ — when the trade happened
- `type` TEXT (`DEPOSIT` | `WITHDRAW` | `BUY` | `SELL`)
- `asset_id` TEXT → assets — the asset **received** (BUY/DEPOSIT) or **given up** (SELL/WITHDRAW)
- `quantity` NUMERIC(38,12) — always > 0; direction comes from `type`, never from a sign
- `settle_asset_id` TEXT → assets — the cash account used. NULL for DEPOSIT/WITHDRAW
- `settle_amount` NUMERIC(38,12) — **all-in**, fees and taxes included. NULL for DEPOSIT/WITHDRAW
- `fx_rate` NUMERIC(20,8) — conversion to VND, default 1. Applies to `settle_amount` for BUY/SELL, to `quantity` for DEPOSIT/WITHDRAW. Always 1 when the counterparty is `CASH_VND`
- `fee_vnd`, `tax_vnd` NUMERIC(20,2) — **reporting only**; the engine must never read these
- `note` TEXT, `created_at`, `updated_at` TIMESTAMPTZ
- CHECK: BUY/SELL require both settlement fields; DEPOSIT/WITHDRAW require both to be NULL
- CHECK: `settle_asset_id` is distinct from `asset_id`

**prices** — Time series, one row per fetch.
- `asset_id` TEXT → assets, `ts` TIMESTAMPTZ — composite PRIMARY KEY
- `bid`, `ask` NUMERIC(28,8) — nullable
- `mid` NUMERIC(28,8) NOT NULL
- `source` TEXT (`binance`, `vietcap`, `fmarket`, `sjc`, `binance_p2p`, `manual`, …)
- `is_manual` BOOLEAN
- Values are in `assets.unit` and `assets.quote_ccy` — already normalised by the provider
- `CASH_VND` never appears here

**fx_reference** — VCB rates. Display only. Never touched by the valuation path.
- `ts` TIMESTAMPTZ PRIMARY KEY
- `vcb_buy`, `vcb_transfer`, `vcb_sell` NUMERIC(12,2)
- `source` TEXT

**nav_snapshots** — Immutable daily history. Not a cache; it cannot be recomputed after the fact.
- `d` DATE PRIMARY KEY (ICT)
- `nav_vnd`, `cost_vnd`, `unrealized_vnd`, `realized_cum_vnd`, `net_flow_cum_vnd` NUMERIC(24,2)
- `by_category` JSONB, `by_asset` JSONB
- `fx_used` NUMERIC(12,2) — the P2P rate applied that day
- `created_at` TIMESTAMPTZ

**targets** — Target allocation per category.
- `category` TEXT PRIMARY KEY
- `target_pct` NUMERIC(5,2), `band_pct` NUMERIC(5,2)

**provider_health** — Fetcher status, surfaced in the UI.
- `provider` TEXT PRIMARY KEY
- `last_ok`, `last_error_at` TIMESTAMPTZ, `last_error` TEXT
- `consecutive_failures` INT

### Indexes

- `transactions(ts, id)` — replay order; `id` is the tie-break that keeps same-timestamp ordering stable
- `transactions(asset_id, ts)`
- `prices(asset_id, ts DESC)` — backs the `latest_prices` view

### Views

- **`latest_prices`** — `DISTINCT ON (asset_id)` ordered by `ts DESC`. The single lookup for current prices

### Seed Data

Twelve `assets` rows: `DCDS` · `GOLD_SJC` · `GOLD_RING` · `QTP` · `SIP` · `HPG` · `BTC` · `ETH` · `SUI` · `SOL` · `CASH_VND` · `CASH_USD`.

Gold and `CASH_USD` seed with `valuation = 'bid'`; everything else `'mid'`. Both gold assets use `unit = 'chi'`. Crypto assets use `quote_ccy = 'USD'`; every other asset, including `CASH_USD`, uses `'VND'`.

No seeded transactions. The user enters their own (Section 2).

---

## 8. API Endpoints

The frontend uses `supabase-js` against Postgres directly — RLS is the authorization layer, so there is no bespoke REST tier. Edge Functions exist only for scheduled work and are invoked by `pg_cron`, not by the browser.

### Data Access (via supabase-js, RLS-enforced)
| Operation | Table / View | Notes |
|---|---|---|
| select | `transactions` | ordered `ts, id` — full replay input |
| insert / update / delete | `transactions` | the only user-writable table |
| select | `latest_prices` | current price per asset |
| select | `assets`, `nav_snapshots`, `fx_reference`, `provider_health` | read-only to the client |
| insert | `prices` | manual override only, with `is_manual = true` |
| upsert | `targets` | |

### Edge Functions (invoked by pg_cron)
| Method | Path | Description |
|---|---|---|
| POST | `/functions/v1/prices-crypto` | Fetch BTC, ETH, SUI, SOL → `prices` |
| POST | `/functions/v1/prices-fx-p2p` | Fetch P2P USDT/VND → `prices` for `CASH_USD` |
| POST | `/functions/v1/prices-vn-stock` | Fetch QTP, SIP, HPG → `prices` |
| POST | `/functions/v1/prices-gold` | Fetch SJC bars and rings → `prices` |
| POST | `/functions/v1/prices-dcds` | Fetch DCDS NAV → `prices` |
| POST | `/functions/v1/fx-reference` | Fetch VCB rates → `fx_reference` |
| POST | `/functions/v1/snapshot-nav` | Replay ledger, compute NAV, write `nav_snapshots` |

Every price function writes `provider_health` on both success and failure.

---

## 9. Calculation Engine

This is the core of the project. It is a pure module, it is written before the UI, and it is proven by tests before anything is built on top of it.

### Replay

Positions are rebuilt from scratch on every read. There is no incremental state.

```ts
type Pos = { qty: Decimal; costVnd: Decimal; realizedVnd: Decimal }

const avg = (p: Pos) => p.qty.isZero() ? D(0) : p.costVnd.div(p.qty)

const grossVnd = (t: Txn) =>
  t.type === 'BUY' || t.type === 'SELL'
    ? t.settleAmount.mul(t.fxRate)
    : t.quantity.mul(t.fxRate)

function replay(txns: Txn[]): Map<string, Pos> {
  const pos = new Map<string, Pos>()
  const get = (id: string) => pos.get(id) ?? { qty: D(0), costVnd: D(0), realizedVnd: D(0) }

  for (const t of sortBy(txns, ['ts', 'id'])) {
    const g = grossVnd(t)

    // ── settlement leg ──
    if (t.settleAssetId) {
      const s = get(t.settleAssetId)
      if (t.type === 'BUY') {
        // paying = disposing of the settlement asset at fx_rate → FX P&L locks in here
        invariant(s.qty.gte(t.settleAmount), `Insufficient ${t.settleAssetId} at ${t.ts}`)
        const basis = avg(s).mul(t.settleAmount)
        s.realizedVnd = s.realizedVnd.add(g.sub(basis))   // always 0 for CASH_VND
        s.costVnd     = s.costVnd.sub(basis)
        s.qty         = s.qty.sub(t.settleAmount)
      } else {                                             // SELL → receiving
        s.qty     = s.qty.add(t.settleAmount)
        s.costVnd = s.costVnd.add(g)
      }
      pos.set(t.settleAssetId, s)
    }

    // ── asset leg ──
    const a = get(t.assetId)
    if (t.type === 'BUY' || t.type === 'DEPOSIT') {
      a.qty     = a.qty.add(t.quantity)
      a.costVnd = a.costVnd.add(g)
    } else {                                               // SELL | WITHDRAW
      invariant(a.qty.gte(t.quantity), `Insufficient ${t.assetId} at ${t.ts}`)
      const basis = avg(a).mul(t.quantity)
      a.realizedVnd = a.realizedVnd.add(g.sub(basis))
      a.costVnd     = a.costVnd.sub(basis)
      a.qty         = a.qty.sub(t.quantity)
    }
    pos.set(t.assetId, a)
  }
  return pos
}
```

`CASH_VND` always carries an average cost of exactly 1, so the BUY branch yields `realized = 0` for it without any special case. If `avg(CASH_VND)` ever drifts from 1, the engine is broken.

### Valuation

```
price_vnd(CASH_VND)             = 1
price_vnd(a) where quote_ccy VND = raw(a)
price_vnd(a) where quote_ccy USD = raw(a) × price_vnd(CASH_USD)

raw(a) = a.valuation === 'bid' ? prices.bid : prices.mid

mv_vnd(a)         = qty(a) × price_vnd(a)
unrealized_vnd(a) = mv_vnd(a) − cost_vnd(a)
total_pnl(a)      = realized_vnd(a) + unrealized_vnd(a)

NAV      = Σ mv_vnd(a)                                     -- cash included
net_flow = Σ gross_vnd(DEPOSIT) − Σ gross_vnd(WITHDRAW)
```

`CASH_USD` is the single source of truth for USD→VND across the whole system. Crypto is converted at the same P2P rate that values the USD position, because that is the rate the user would actually face on the way out.

### The Invariant

```
NAV − net_flow  ==  Σ realized_vnd + Σ unrealized_vnd        (within 1 VND)
```

This holds for every possible ledger. It is the single most valuable test in the project — it catches nearly every class of engine bug. It is a test, not a comment. If it fails, fix the engine, never the test.

### Worked Examples

These are golden tests. Both must pass before any UI work begins.

**FX lifecycle**

| Step | Result |
|---|---|
| `DEPOSIT CASH_VND 100,000,000` | CASH_VND: qty 100M, cost 100M, avg **1** |
| `BUY CASH_USD 1,000` settle `CASH_VND 25,000,000` | CASH_USD: qty 1,000, cost 25M, avg **25,000**. CASH_VND: qty 75M, avg 1, realized **0** |
| rate rises to 26,000 | CASH_USD mv 26M, unrealized **+1M**. NAV 101M, net_flow 100M → P&L **1M** ✓ |
| `BUY BTC 0.01` settle `CASH_USD 620` @ fx 26,100 | basis = 25,000 × 620 = 15.5M; gross = 620 × 26,100 = 16.182M → realized **+682,000**. CASH_USD: qty 380, avg still **25,000**. BTC: cost **16,182,000** |

**Fees**

`DEPOSIT 100,000,000` then `BUY HPG 1,000` settle `26,039,000` (26,000 per share plus 39,000 in fees).
Market still 26,000 → mv 26M, cost 26.039M, unrealized **−39,000**.
NAV 99,961,000, net_flow 100,000,000 → P&L **−39,000**, exactly the fee. ✓

### XIRR

Simple return on capital is meaningless when deposits and withdrawals are ongoing, which they are. XIRR is the number that answers "how well is this portfolio actually doing".

```
flows = [(ts, −gross_vnd) for each DEPOSIT]
      + [(ts, +gross_vnd) for each WITHDRAW]
      + [(now, +NAV)]

solve for r:  Σ cf_i / (1 + r)^(days_i / 365) = 0
```

Newton-Raphson with a bisection fallback over `[-0.9999, 10]`. Return `null` when it fails to converge or the history is shorter than 30 days — never return a fabricated figure.

### Out of Scope for v1

`txn_type` is an enum with room to grow. Adding a value requires one `ALTER TYPE` and one branch in `replay` — **no data migration**. Deliberately deferred:

- **Cash dividends** — the first thing to add. QTP pays them regularly, and ignoring them understates its P&L
- **Estimated DCDS exit fee** by lot age (1.5% under 12 months, 0.5% at 12–24, 0% beyond). The data is already in the ledger; only the display is missing
- Stock dividends and splits, staking rewards on SUI and SOL, airdrops, benchmark comparison, TWR

---

## 10. Frontend Design

### Layout

A single-page application, dense and board-like. Component architecture is up to the Frontend Engineer, but the UI must include:

- **Board header** — NAV in large tabular mono with up/down colour, today's change, total P&L split realized/unrealized, XIRR, and a freshness badge
- **Allocation** — donut by category with the target ring overlaid, plus a drift warning when a category leaves its band
- **NAV chart** — line chart from `nav_snapshots`
- **Positions table** — grouped by category: quantity, average cost, market price with its own timestamp, market value, weight %, unrealized, realized, total P&L %
- **Transaction entry** — asset, settlement account, quantity, all-in amount. A live preview showing the implied rate, cost per unit, resulting balance, and — when settling in USD — **the FX gain or loss about to be locked in**. Negative balances are blocked here and again by a database constraint
- **Transaction history** — filter, edit, delete. Any edit triggers a full replay
- **Settings** — manual price entry, target allocation, provider health, CSV export

### Technical Notes

- Every derived number comes from `engine/`. The frontend performs no financial arithmetic of its own
- Recharts for charts. Floats are acceptable here and only here
- VND renders with thousands separators and no decimals. Crypto renders at the asset's `qty_scale`
- Tailwind, with the Section 2 tokens as CSS custom properties
- Respect `prefers-reduced-motion`; keyboard focus must be clearly visible
- UI copy uses active verbs describing what will actually happen: "Ghi giao dịch", not "Submit". Errors state what broke and how to fix it, without apologising

---

## 11. Deployment

### Frontend — Cloudflare Pages

Build `web/` and publish the static output. Set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` as build-time environment variables. No server, no adapter, no SSR.

### Backend — Supabase

Create the project **in the Singapore region**. This cannot be changed later and Section 3 explains why it matters.

```bash
supabase link --project-ref <ref>
supabase db push                  # migrations
supabase functions deploy         # edge functions
```

### Scheduling

`pg_cron` jobs call Edge Functions through `pg_net`, on the Section 6 schedule. Register them in a migration so the schedule is version-controlled rather than clicked into a dashboard.

### Backup

CSV export is a v1 feature, not a nice-to-have. This is irreplaceable personal financial data and the transaction ledger is the one thing that cannot be reconstructed from any external source.

---

## 12. Testing Strategy

### Unit Tests (`engine/tests/`, Vitest)

The engine is pure, so it can be tested exhaustively with no mocks and no fixtures beyond arrays of transactions.

- **The invariant** — `NAV − net_flow == Σrealized + Σunrealized` within 1 VND, asserted across every fixture including randomly generated ledgers
- **`avg_cost(CASH_VND) == 1`** after any sequence of transactions
- **Golden tests** — both Section 9 worked examples, asserted figure by figure
- **Overselling** — selling or withdrawing more than held must throw
- **Idempotence** — replaying twice yields identical results
- **Stable ordering** — transactions sharing a `ts` resolve deterministically via the `id` tie-break
- **XIRR** — cash flow sets with known answers; non-convergence returns `null`
- **Zero and full exits** — position closed to exactly zero leaves cost at zero and realized intact

### Provider Tests (`supabase/functions/`)

- **Unit** — parse a recorded response fixture into `Quote`, with unit normalisation asserted: 85,500,000 per lượng → `bid` 8,550,000 per chỉ
- **Live smoke tests**, run manually and not in CI: each provider hits its real endpoint once and asserts a plausible response. These exist to catch a source breaking, which will happen
- **Fallback chain** — with the primary stubbed to fail, the fallback is used; with both failing, the last known price is returned flagged stale; with no price at all, manual entry is demanded and no number is invented

### E2E Tests (`test/`, Playwright)

- Seed flow: deposit plus buys produce the exact expected `CASH_VND` balance
- Record a buy in VND: cash falls, position appears, NAV updates
- Record a buy in USD: preview shows the correct implied FX P&L; after recording, CASH_USD realized P&L matches
- Oversell is rejected at the UI
- Edit a historical transaction: every downstream figure recomputes
- A stale price is visibly badged rather than silently displayed
- CSV export downloads and round-trips

---

## 13. Build Order

### Phase 0 — Verification Spike (do this first)

Roughly an hour of `curl`, and it can change the phases that follow. None of it is optional.

- [ ] **Binance P2P** — the only genuinely unverified source. Confirm the endpoint, request shape, and response. If it is gone, decide between manual weekly entry and VCB-plus-spread before building the provider layer
- [ ] **Vietcap `getList`** — confirm the response, and **test QTP by itself** to prove UPCoM coverage
- [ ] **FMarket** — confirm `products/28` and `product/get-nav-history`
- [ ] **SJC `PriceService.ashx`** — confirm the response shape and identify which rows are bars versus rings
- [ ] **Binance from Singapore** — confirm no HTTP 451
- [ ] **VCB `api/exchangerates`** — confirm the `?date=` parameter honours the requested date

Record the findings in `planning/` before proceeding.

### Phase 1 — Schema
Migrations, seeded `assets`, RLS policies, `latest_prices` view. The engine's contract.

### Phase 2 — Engine
Pure module plus the full test suite from Section 12. **The invariant and both golden tests must be green before Phase 4 begins.** This phase carries most of the project's value and nearly all of its irrecoverable risk.

### Phase 3 — Providers
One at a time, each with its fixture test and a live smoke test. Order by certainty: crypto, then VN equities, then DCDS, then gold, then FX. Then `pg_cron` and `snapshot-nav`.

### Phase 4 — UI
Seed flow first — nothing else can be exercised until there is a portfolio to look at. Then the board, positions, transaction entry, history, settings.

### Phase 5 — Hardening
E2E suite, CSV export, provider health surfacing, staleness badges everywhere.

Do not build the UI before the engine. A beautiful interface over a wrong engine is a liability, because it makes the wrong numbers credible.
