# Lab 2: Quantify the Dashboard Reads

---

## 1. Quality Requirements

Each requirement follows the structure: **measure + target + operating condition**.
A fast "unavailable" result can satisfy the latency target and still fail a consistency or availability requirement — these are tracked separately.

---

### 1.1 Read Latency

**Measurement boundaries**

- **Start:** The Dashboard server receives the first byte of the HTTP request from the client.
- **End:** The Dashboard server sends the last byte of the HTTP response to the client.
- This is *within-system latency*. It excludes client-side network travel time and browser rendering. The client cannot be instrumented in the first version.

**Requirement**

> Under the steady read workload (up to the market-open target RPS per scale), measured at the Dashboard server boundary, **p95 of all Dashboard read responses** must be delivered within **2 000 ms**.

**Percentile choice — "most" = p95**

"Most correct reads should complete within 2 seconds" maps to p95 by convention: 95 of 100 reads meet the target and a defined timeout handles the remaining 5%.

**Behavior at the 2 000 ms boundary**

A read that cannot produce a result within 2 000 ms returns a timeout error (HTTP 504 Gateway Timeout). That response is counted against the latency budget and separately counted as an unavailable result. It does not reset or extend the latency window.

**Assumption:** p50 is expected to be $\le 300\text{ ms}$ under normal load. This is an initial design target; it is not a client contractual commitment.

---

### 1.2 Availability (Uptime-Style)

**Definition of "usable"**

The Dashboard is *usable* during a one-minute interval if at least one of the six read endpoints (Overview, Filter, Stock price, History, Watchlist, Search) returns an acceptable response within the latency target.

**Market-hours window**

NYSE and NASDAQ regular session: **09:30–16:00 ET, Monday–Friday, excluding US market holidays.**
This is approximately 6.5 hours $\times$ 5 days $\times$ ~52 weeks $\approx$ **1 690 hours/year**, or roughly **7.3%** of calendar time.

**Uptime targets**

| Window | Target | Justification |
|---|---|---|
| **Market hours** | **99.9%** | Price staleness is bounded by the 15-minute provider delay; a brief outage during market hours is a degraded experience, not a critical financial loss path (no trading). One extra nine (99.99%) would cost disproportionately for a display-only product. |
| **Off-market hours** | **99.0%** | During off-hours the Dashboard shows history and watchlists; no real-time price pressure. Users are more tolerant and the traffic is an order of magnitude lower. |

**30-day budget calculation**

Each window is measured separately over its own elapsed time within a 30-day period (≈ 21 trading days / market days):

- **Market hours elapsed time:** $21\text{ days} \times 6.5\text{ h} \times 3\ 600\text{ s} = 491\ 400\text{ seconds}$
- **Off-market hours elapsed time:** $2\ 592\ 000\text{ s (total 30 days)} - 491\ 400\text{ s} = 2\ 100\ 600\text{ seconds}$

| Window | Elapsed seconds in 30 days | Downtime budget formula | Downtime budget |
|---|---|---|---|
| **Market hours (99.9%)** | $\approx 491\ 400\text{ s}$ | $491\ 400 \times (1 - 0.999)$ | **$\approx 491.4\text{ s} \approx 8\text{ min } 11\text{ s}$** |
| **Off-market hours (99.0%)** | $\approx 2\ 100\ 600\text{ s}$ | $2\ 100\ 600 \times (1 - 0.990)$ | **$\approx 21\ 006\text{ s} \approx 5\text{ h } 50\text{ min}$** |

**Why different targets for each window**

Market hours carry real-time user expectations (prices, watchlist updates). Off-market hours serve lower-urgency reads (historical charts, watchlist browsing). A uniform 99.9% target for off-hours would consume engineering budget that delivers marginal user value.

---

### 1.3 Consistency

#### Stock price — maximum accepted staleness

> After the Market Data Provider publishes a new price, the Dashboard **must display a price that is no older than 15 minutes** from the provider's stated price timestamp (the provider's time, not the Dashboard clock).

- A price with a timestamp older than 15 minutes relative to the current provider time **must be shown with a visible staleness indicator** (e.g., "delayed" label), not silently as current.
- A price with **no provider timestamp** or for which no data has ever been received must be displayed as **unavailable**, not as zero and not as the last known price.
- A fast "unavailable" response satisfies the latency requirement. It is a separate failure for the consistency requirement.

#### Watchlist — read-your-own-writes guarantee

> After the Dashboard confirms a Watchlist change to the same authenticated User (e.g., adding or removing a stock), **the next Watchlist read by the same User session must reflect that change**.

- "Confirmed" means the Dashboard has returned HTTP 2xx to the client for the write.
- This guarantee applies to the **same User session** only. It does not guarantee that a second device or session belonging to the same account sees the change immediately.
- Another User's session is explicitly excluded: a User cannot read or change another User's Watchlist.

---

## 2. Steady RPS Estimates

**Formula:**

$$\text{RPS} = \text{concurrent Users} \times \text{participating share} \times \frac{\text{actions per User}}{\text{seconds}}$$

---

### Calculation Details per Scale

#### Scale: 300 concurrent Users
- **Overview:** $300 \times 0.70 \times \frac{1}{30} = 7.000\text{ RPS}$
- **Filter:** $300 \times 0.50 \times \frac{3}{60} = 7.500\text{ RPS}$
- **Stock price:** $300 \times 0.20 \times \frac{1}{1} = 60.000\text{ RPS}$
- **History:** $300 \times 0.20 \times \frac{1}{300} = 0.200\text{ RPS}$
- **Watchlist:** $300 \times 0.60 \times \frac{1}{60} = 3.000\text{ RPS}$
- **Search:** $300 \times 0.10 \times \frac{3}{60} = 1.500\text{ RPS}$
- **Steady total:** $79.200\text{ RPS}$

#### Scale: 3,000 concurrent Users
- **Overview:** $3\ 000 \times 0.70 \times \frac{1}{30} = 70.000\text{ RPS}$
- **Filter:** $3\ 000 \times 0.50 \times \frac{3}{60} = 75.000\text{ RPS}$
- **Stock price:** $3\ 000 \times 0.20 \times \frac{1}{1} = 600.000\text{ RPS}$
- **History:** $3\ 000 \times 0.20 \times \frac{1}{300} = 2.000\text{ RPS}$
- **Watchlist:** $3\ 000 \times 0.60 \times \frac{1}{60} = 30.000\text{ RPS}$
- **Search:** $3\ 000 \times 0.10 \times \frac{3}{60} = 15.000\text{ RPS}$
- **Steady total:** $792.000\text{ RPS}$

#### Scale: 30,000 concurrent Users
- **Overview:** $30\ 000 \times 0.70 \times \frac{1}{30} = 700.000\text{ RPS}$
- **Filter:** $30\ 000 \times 0.50 \times \frac{3}{60} = 750.000\text{ RPS}$
- **Stock price:** $30\ 000 \times 0.20 \times \frac{1}{1} = 6\ 000.000\text{ RPS}$
- **History:** $30\ 000 \times 0.20 \times \frac{1}{300} = 20.000\text{ RPS}$
- **Watchlist:** $30\ 000 \times 0.60 \times \frac{1}{60} = 300.000\text{ RPS}$
- **Search:** $30\ 000 \times 0.10 \times \frac{3}{60} = 150.000\text{ RPS}$
- **Steady total:** $7\ 920.000\text{ RPS}$

---

### Steady RPS Summary Table

| Read | 300 Users | 3,000 Users | 30,000 Users |
|---|---|---|---|
| Overview | 7.00 | 70.00 | 700.00 |
| Filter | 7.50 | 75.00 | 750.00 |
| Stock price | 60.00 | 600.00 | 6 000.00 |
| History | 0.20 | 2.00 | 20.00 |
| Watchlist | 3.00 | 30.00 | 300.00 |
| Search | 1.50 | 15.00 | 150.00 |
| **Steady total** | **79.20** | **792.00** | **7 920.00** |

> **Note:** Stock price dominates every scale ($\approx 75.76\%$ of steady RPS) because 20% of concurrent users issue one request per second. This is the primary driver for throughput and the main candidate for caching.

---

## 3. Market-Open RPS Estimates

**Rules applied:**

1. **Steady traffic continues** during market open.
2. Additional Overview burst: **30% of concurrent Users refresh once in 10 seconds** ($\text{Users} \times 0.30 \times \frac{1}{10}$).
3. Additional Watchlist burst: **60% of that Overview group also refresh once in 10 seconds** ($\text{Users} \times 0.30 \times 0.60 \times \frac{1}{10}$).
4. Do not double-count steady Overview/Watchlist traffic (already in Steady total).
5. Add a **10% capacity margin** to the subtotal.
6. **Round up only the final market-open target** ($\lceil \text{result} \rceil$).

---

### Market-Open Full Calculation Table

| Market-open calculation | 300 Users | 3,000 Users | 30,000 Users |
|---|---|---|---|
| Steady read traffic | 79.20 | 792.00 | 7 920.00 |
| Additional Overview refresh flow | 9.00 | 90.00 | 900.00 |
| Additional Watchlist refresh flow | 5.40 | 54.00 | 540.00 |
| **Market-open subtotal** | **93.60** | **936.00** | **9 360.00** |
| 10% capacity margin | 9.36 | 93.60 | 936.00 |
| Subtotal + Margin | 102.96 | 1 029.60 | 10 296.00 |
| **Rounded-up market-open target** | **103 RPS** | **1 030 RPS** | **10 296 RPS** |

---

## 4. Storage Estimates

### 4.1 Research and Definition of "Stock"

#### Scope & Definitions

In a market-data product, "stock" can refer to common stock, preferred stock, ADRs, ETFs, ETNs, funds, warrants, or rights.

| Decision | Choice | Justification |
|---|---|---|
| **Exchanges** | USA only — NYSE and NASDAQ | Largest, most standardized equity markets; ideal for MVP scope. |
| **Included types** | Common stock, Preferred stock, ADRs | Standard equities sharing standard OHLCV pricing semantics. |
| **Excluded types** | ETFs, ETNs, Closed-end funds, REITs, Warrants | Different underlying structures and valuation mechanics (NAV vs price). |
| **Delisted / Inactive** | Excluded from live views/search; retained in history | Keeps live indices clean while preserving chart accuracy for historical holdings. |
| **Supported Stocks count** | **~8,000 active listings** | Standard baseline for active NYSE + NASDAQ common, preferred, and ADRs. |

**Citations & Sources:**
- *Instrument taxonomy:* NYSE & NASDAQ Security Specifications; Nasdaq Trader Symbol Directory definitions ([nasdaqtrader.com](https://www.nasdaqtrader.com/trader.aspx?id=symboldirdefs), observed Oct 2026).
- *Instrument count:* FINRA OTC & Exchange-Listed Stock Data ([finra.org](https://www.finra.org/investors/learn-to-invest/types-investments/stocks), observed Oct 2026): ~3,400 NASDAQ + ~3,300 NYSE common/preferred + ~1,300 ADRs $\approx$ **8,000 active stocks**.

---

### 4.2 Synchronized Data Definition

| Data set | Fields stored | Product purpose |
|---|---|---|
| **Stock reference data** | `ticker`, `exchange`, `isin`, `cusip`, `company_name`, `sector`, `industry`, `country`, `is_active`, `search_tokens` | Overview display, ticker/name search, sector filtering. |
| **Latest prices** | `ticker`, `last_price`, `currency`, `price_timestamp`, `bid`, `ask`, `day_open`, `day_high`, `day_low`, `day_volume`, `provider_received_at` | Real-time Stock price card, staleness check (provider time vs current time). |
| **Price history** | `ticker`, `date`, `open`, `high`, `low`, `close`, `adjusted_close`, `volume` | Historical charts, performance calculation. |
| **Watchlist** | `user_id`, `ticker`, `added_at` | User private watchlist persistence and read-your-own-writes guarantee. |

**Excluded Fields:** Order book depth, fundamental ratios (P/E, EPS), real-time tick-level logs (15-min snapshots are sufficient for MVP).

---

### 4.3 Storage Calculation

#### Byte Estimates per Record

- **Stock Reference Record:** $\approx 245\text{ B} \to \mathbf{256\text{ B}}$
- **Latest Price Record:** $\approx 85\text{ B} \to \mathbf{96\text{ B}}$
- **Price History Record (1 day bar):** $\approx 62\text{ B} \to \mathbf{64\text{ B}}$
- **Watchlist Record:** $\approx 48\text{ B}$ (estimated at 10 items/user across 3,000 users $\approx 30\ 000\text{ rows} \times 48\text{ B} \approx 1.44\text{ MB}$)

#### Record Count Assumptions
- **Supported Stocks:** 8,000
- **Retention:** 5 years $\times$ 252 trading days/year = **1,260 days**
- **Total History Records:** $8\ 000 \times 1\ 260 = \mathbf{10\ 080\ 000\text{ records}}$

#### Raw Storage Table

| Data set | Product decision and retention | Record-count calculation | Bytes per record | Raw storage |
|---|---|---|---|---|
| **Stock reference data** | 8 000 active stocks; updated daily; static replacement | 8 000 records | 256 B | $8\ 000 \times 256 = 2\ 048\ 000\text{ B} \approx \mathbf{2.0\text{ MB}}$ |
| **Latest prices** | 1 live snapshot per stock; overwritten every 15 min | 8 000 records | 96 B | $8\ 000 \times 96 = 768\ 000\text{ B} \approx \mathbf{0.75\text{ MB}}$ |
| **Price history** | Daily OHLCV; 5-year retention (1 260 trading days) | $8\ 000 \times 1\ 260 = 10\ 080\ 000$ | 64 B | $10\ 080\ 000 \times 64 = 645\ 120\ 000\text{ B} \approx \mathbf{615.0\text{ MB}}$ |
| **Watchlist** | User-driven; assumed avg 10 stocks / user for 3k users | 30 000 records | 48 B | $30\ 000 \times 48 = 1\ 440\ 000\text{ B} \approx \mathbf{1.44\text{ MB}}$ |
| **Total** | Base raw storage required | | | **$\approx 619.19\text{ MB}$ ($\approx 618\text{ MB}$ core)** |

#### Annual Storage Growth
- **New daily history bars:** $8\ 000\text{ stocks} \times 252\text{ days} \times 64\text{ B} \approx \mathbf{129.02\text{ MB/year}}$ ($\approx 123\text{ MiB}$)
- **Total 1-Year Raw Target:** $\approx 618\text{ MB} + 123\text{ MB} = \mathbf{741\text{ MB raw}}$

---

## 5. Potential Bottlenecks

| Quality | Potential bottleneck | Evidence from this lab | Possible effect | What to measure next |
|---|---|---|---|---|
| **Latency** | **Stock price read path under high steady load** | Stock price accounts for **75.76% of total steady RPS** (6,000 RPS out of 7,920 RPS at 30k Users). Under market-open bursts, queue times will spike if requests hit unindexed DB tables or single-threaded caches. | p95 latency exceeds 2 000 ms limit during peak load, causing HTTP 504 timeouts. | Measure p50 and p95 latency at 79, 792, and 10,296 RPS. Profile cache hit ratios vs DB connection pool saturation. |
| **Consistency** | **15-minute sync job execution delay** | The system refreshes 8,000 stocks every 15 minutes ($8\ 000 / 900\text{ s} \approx 8.88\text{ updates/sec}$). If provider sync stalls or fails for specific tickers, data staleness silently breaches 15 min. | Users see outdated price data without the required "delayed / unavailable" indicator, violating consistency rules. | Monitor p99 sync execution duration per ticker. Measure time delta between `provider_received_at` and current timestamp. |
| **Throughput** | **Shared read/write database connection pool during market open** | Market-open traffic reaches **10,296 RPS** while background ingestion simultaneously writes new price snapshots for 8,000 stocks. | Lock contention or pool exhaustion drops throughput below target RPS, causing cascading failures. | Perform stress testing of concurrent 10k RPS reads + background sync writes. Measure lock wait times and DB queue depth. |
| **Availability** | **External Market Data Provider dependency** | The Dashboard relies on a single external provider. A provider outage longer than 15 minutes renders live stock prices unavailable across the system. | Core dashboard views fail data freshness requirements, exhausting the 8-minute market-hours downtime budget. | Instrument external provider uptime separately. Implement circuit breakers and fallbacks to last known valid cached state. |

---

## Checklist Verification

- [x] Measurable requirements written for all core qualities (measure + target + operating condition).
- [x] Uptime targets chosen and justified for market hours (99.9%) and off-market hours (99.0%).
- [x] Downtime budgets calculated for both windows over a 30-day period.
- [x] Steady and market-open RPS calculations shown for 300, 3,000, and 30,000 Users.
- [x] Scope of supported Stocks researched and defined (~8,000 instruments with citations).
- [x] Initial, daily, and 1-year raw storage estimated.
- [x] Assumptions, units, windows, and final rounding rules clearly stated.
- [x] Potential bottlenecks analyzed for all 4 core qualities.
