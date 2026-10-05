# Options Market-Maker Risk System: Build Specification

**Audience:** an AI coding agent (or engineer) that already knows the firm's internal data pool (REST + WebSocket: option chains, per-name vol surfaces, positions, fills, expiries) and will implement the system. The data-pool access layer is deliberately left generic here; everything else is prescriptive.

**Status of this document:** researched against primary sources as of October 2026 (exchange and clearing rule filings, academic papers, open-source code). Every rule that depends on an exchange/clearinghouse parameter is marked with the source so the implementer can re-verify when the rule changes. Items marked **[verify]** could not be confirmed against a primary source and must be checked before being relied upon in production.

---

## 0. How to use this document

### 0.1 Reading order and scope

Sections 1 through 6 build the valuation foundation (reference data, market-data hygiene, pricing, surface, marks). Sections 7 through 10 define risk measurement and control (Greeks, scenarios, limits, books). Sections 11 and 12 define P&L and P&L explain. Section 13 covers the expiry lifecycle including 0DTE. Sections 14 through 17 cover hedging policy, event risk, daily operations, and testing. Section 18 is the phased build plan with acceptance criteria. Appendices hold formulas, schemas, default parameters and sources.

Build in the order of Section 18. Do not build the risk display before the ledger, the forward extraction, and the mark policy exist. Almost every "unexplained P&L" and "wrong Greek" problem in a market-making risk system traces back to one of those three.

### 0.2 Non-negotiable design principles

1. **Everything is reproducible.** Any number the system shows (a mark, a Greek, a P&L line) must be regenerable from a stored tuple `(ledger_seq, mark_version, surface_version, engine_version, calendar_version)`. If you cannot regenerate yesterday's official P&L bit-for-bit, the system is not finished.
2. **Positions are derived, never stored as truth.** The truth is an append-only event ledger (fills, busts, corrections, exercises, assignments, corporate actions, internal transfers). Positions, cash and P&L are projections of the ledger.
3. **Marks come from a surface, not from mids.** Mid is an input to the surface fit, never the mark. Every mark records its source and version.
4. **One forward and one discount factor per expiry.** All puts and calls of an expiry are priced off the same forward. Put-call parity is enforced by construction.
5. **Time is a model choice.** Which "day count" you use for theta, which intraday clock you use for 0DTE, and whether weekends count, must be one decision applied consistently across pricing, Greeks, marks and P&L explain. Mixing clocks is the single most common source of fake P&L.
6. **Risk for a 0DTE book is scenario P&L and gamma dollars, not vega.** Vega-based limits are blind near expiry.
7. **Every limit, mark override and model change is an auditable event** with who, when, old value, new value, reason, approver.
8. **Compute on a fixed cadence, not per message.** Load must scale with book size, not with market busyness.

### 0.3 Notation

| Symbol | Meaning |
|---|---|
| `S` | spot (or synthetic underlying for adjusted deliverables) |
| `F_T` | forward to expiry `T` (implied from the option chain or from the same-expiry future) |
| `DF_T` | discount factor to `T` |
| `K` | strike |
| `k = ln(K / F_T)` | log-forward-moneyness |
| `τ` | time to expiry in years under the chosen clock |
| `σ(k, τ)` | implied vol from the fitted surface |
| `w(k, τ) = σ² τ` | total implied variance |
| `mult` | contract multiplier (100 for standard US equity options) |
| `q` | signed position in contracts (positive long) |
| `Δ, Γ, ν, Θ, ρ` | delta, gamma, vega, theta, rho |
| `Δ$ = Δ · S · mult · q` | dollar delta |
| `Γ$ = Γ · S² / 100 · mult · q` | dollar gamma per 1% spot move (change in Δ$ for a 1% move) |
| `ν$` | vega in dollars per 1.00 vol point (0.01 absolute) |
| `Θ$` | theta in dollars per 1 day under the chosen clock |

### 0.4 Assumptions about the data pool

The implementer's data pool is assumed to supply, per underlying: option chain with bid/ask/size/timestamps per listing venue or consolidated, underlying/future quotes, a vendor or internal vol surface per name, positions and fills from the OMS, and an expiry list. Wherever this document says "from the data pool," the implementer should map to the internal API. Where the data pool's vol surface is used, Section 5 still applies: the system must validate the external surface (arbitrage checks, fit quality) and must be able to re-fit from raw quotes, because an external surface cannot be trusted at EOD on illiquid names or on expiry day.

---

## 1. Architecture

### 1.1 Reference component layout

```
                ┌──────────────────────────────────────────────────────────┐
                │                      DATA POOL (existing)                │
                │  REST: chains, surfaces, positions, fills, expiries      │
                │  WS:   quotes, trades, fills, underlying ticks           │
                └───────┬───────────────┬───────────────┬──────────────────┘
                        │               │               │
              ┌─────────▼───────┐ ┌─────▼──────┐ ┌──────▼─────────┐
              │ Feed adapters   │ │ Fill       │ │ Reference data │
              │ seq/gap detect  │ │ ingestion  │ │ instrument     │
              │ staleness stamps│ │ idempotent │ │ master, cal,   │
              └─────────┬───────┘ └─────┬──────┘ │ corp actions   │
                        │               │        └──────┬─────────┘
              ┌─────────▼───────┐ ┌─────▼──────────────┐│
              │ Market-data     │ │ EVENT LEDGER       ││
              │ cache (per instr│ │ append-only, seq   ││
              │ ts, seq, src)   │ │ fills/busts/exer/  ││
              └─────────┬───────┘ │ assign/CA/transfer ││
                        │         └─────┬──────────────┘│
              ┌─────────▼───────┐       │               │
              │ Forward &       │ ┌─────▼──────────────▼┐
              │ surface fitter  │ │ Position projector  │
              │ (per expiry /   │ │ (book × instrument) │
              │ per underlying) │ └─────┬───────────────┘
              └─────────┬───────┘       │
                        │   ┌───────────▼───────────┐
              ┌─────────▼───▼─────────┐             │
              │ Pricing & Greeks      │             │
              │ (vectorized batch per │             │
              │  underlying, cadence) │             │
              └─────────┬─────────────┘             │
                        │                           │
              ┌─────────▼─────────────┐   ┌─────────▼──────────┐
              │ Aggregation tree      │   │ Mark service       │
              │ firm→book→name→expiry │   │ (versioned marks,  │
              │ →strike, $Greeks,     │   │  official EOD)     │
              │ ladders, scenarios    │   └─────────┬──────────┘
              └───┬─────────┬─────────┘             │
                  │         │          ┌────────────▼───────────┐
        ┌─────────▼──┐ ┌────▼───────┐  │ P&L engine             │
        │ Limit      │ │ Risk       │  │ live / official /      │
        │ monitor &  │ │ snapshots  │  │ explain / fees / carry │
        │ actions    │ │ (pub/sub)  │  └────────────────────────┘
        └────────────┘ └────────────┘
```

### 1.2 Event ledger (position keeper)

Model this on the LMAX architecture: all state is derivable by replaying input events; restart is snapshot plus journal replay; the same journal can be replayed in a development environment for debugging (Fowler, "The LMAX Architecture", martinfowler.com/articles/lmax.html).

**Event types** (all immutable, each with `ledger_seq`, `event_ts` (when it happened), `recv_ts` (when we learned of it), `source`):

- `FILL` with dedupe key `(venue, exec_id, exec_type)`. FIX busts arrive as ExecType `H` (Trade Cancel) and corrections as `G` (Trade Correct) referencing `ExecRefID`. Model these as compensating events that reference the original fill, never as in-place edits.
- `EXERCISE`, `ASSIGNMENT` (from OCC/clearing reports, next morning), `EXPIRY_REMOVAL` (worthless expiry).
- `CORP_ACTION` (position transform: old instrument qty → new instrument qty with basis preserved).
- `INTERNAL_TRANSFER` (paired fills between books at the official mark, with `transfer_id`).
- `HEDGE_ALLOCATION` (allocation of a shared hedge to books, so re-allocation is itself an event).
- `CASH` (fees, dividends, borrow, interest, true-ups).

**Idempotency rule:** a replayed or duplicated fill with the same dedupe key is ignored and logged. Late fills are new events with their own `recv_ts`, which makes "P&L as known at 16:05" and "P&L as known after T+1 reconciliation" both reproducible queries (bitemporal: event time vs knowledge time).

**Snapshot cadence:** nightly snapshot of projected positions keyed by `ledger_seq`; intraday checkpoints every N minutes. Target recovery time under one minute.

### 1.3 Market-data cache and staleness

Each cached quote carries `exchange_ts`, `recv_ts`, `seq`, `venue`. Gap detection is `seq != last_seq + 1`, which triggers a snapshot re-request. Staleness is per instrument: `now - recv_ts`. A quote is also considered stale if the underlying has moved more than `m · σ · sqrt(Δt)` since the quote's timestamp (default `m = 2`) and the quote has not updated.

Treat two missed WebSocket heartbeats as a dead feed: raise a limit-monitor flag that freezes "live" P&L publication and marks the underlying's risk as `STALE` in every display.

### 1.4 Pricing cadence and incremental updates

Reprice a whole underlying's book as one vectorized batch (NumPy/Polars/Arrow arrays of strikes, expiries, vols) on a fixed cadence (default 100 to 250 ms per underlying, coalescing ticks), rather than per market-data message. Optiver describes exactly this "constant rate" design so that systems scale with the size of the trading operation rather than with the busyness of the market.

Between full reprices, update cached values by Taylor expansion:

```
ΔP ≈ Δ·ΔS + ½Γ·ΔS² + ν·Δσ + Θ·Δt
```

Force a full reprice when any of these hold: `|ΔS/S| > 25 bp` since the last full reprice; the surface version changed; the instrument is within 1 trading day of expiry; or `½Γ·ΔS²` exceeds 10% of `|Δ·ΔS|` (gamma-dominated regime where Taylor fails fast).

### 1.5 Aggregation tree

`firm → desk → book → underlying → expiry → strike → instrument`. Store every Greek in dollar terms so they sum (Section 7.1). Also materialize the `underlying` aggregate across all books, because exchange risk controls and clearing margin see the firm's clearing account, not the firm's internal books.

### 1.6 Storage

- **Hot:** in-memory columnar per underlying; Redis or equivalent for pub/sub of risk snapshots.
- **Intraday time series:** a time-series DB optimized for "latest value per key" and fast appends (QuestDB or kdb+ class).
- **History:** Parquet partitioned by `date=YYYY-MM-DD/underlying=XXX/` for quotes, surfaces and risk snapshots; by `date` only for fills and marks (they are small).
- **Official tables** (EOD marks, official P&L, signed-off risk) are separate from intraday tables and are append-only with versions.

### 1.7 Bitemporal records

Every reference-data and mark table carries `valid_from, valid_to` (when the fact was true in the world) and `tx_from, tx_to` (when the system knew it). This is how a Monday EOD report is reproduced after Tuesday's correction. Open-ended intervals use a sentinel date (9999-12-31).

---

## 2. Reference data

### 2.1 Instrument master

The OCC/OSI symbol is 21 characters: root padded to 6, `YYMMDD`, `C`/`P`, strike × 1000 as 8 digits. Example: `AAPL  260918C00260000`. Adjusted series carry the root plus an integer suffix (`AAPL1`, `AAPL2`), incremented per subsequent adjustment, and a non-standard deliverable.

**The deliverable is a list, not a multiplier.** OCC memo #50867 (Root Inc. 1-for-18 reverse split) is a template: contract multiplier 1, strike divisor 1, premium multiplier still 100, deliverable "5 shares + cash in lieu of ~0.5556 fractional share," with an interim underlying reference price `ROOT1 = 0.055556 × ROOT` until the cash leg fixes. A pricer that assumes `100 × S` is wrong for every adjusted contract.

```sql
instrument (
  instrument_id      BIGINT PRIMARY KEY,
  osi_symbol         CHAR(21) UNIQUE,
  root               VARCHAR(6),          -- 'AAPL', 'AAPL1'
  underlying_id      BIGINT,
  expiry_date        DATE,
  expiry_time        TIME,                -- product-class specific (Section 2.2)
  strike             NUMERIC(12,4),
  right              CHAR(1),             -- 'C' / 'P'
  premium_multiplier INT,                 -- 100 even for reverse-split-adjusted
  exercise_style     CHAR(1),             -- 'A' american / 'E' european
  settlement_type    CHAR(1),             -- 'P' physical / 'C' cash
  settlement_timing  CHAR(2),             -- 'AM' / 'PM' (index products)
  deliverable        JSONB,               -- [{type:'STOCK',symbol:'ROOT',qty:5},
                                          --  {type:'CASH',amount:12.34,fixed:true}]
  adjustment_memo_id VARCHAR(32),
  is_adjusted        BOOL,
  listing_venues     TEXT[],
  valid_from, valid_to, tx_from, tx_to    TIMESTAMPTZ
)
```

**Pricing adjusted contracts:** value the option on the deliverable basket `B = Σ qᵢ·Pᵢ + cash`, compare the strike to `B`, map delta/gamma to each component, and treat the cash leg as zero-vol. Quotes remain "per share × 100" even when the deliverable is 150 shares.

### 2.2 Calendar service

Encode per product class: `session_open`, `session_close`, `expiring_series_close`, `exercise_cutoff`, `settlement_type {AM_OPEN, PM_CLOSE}`, `settlement_publish_window`, `holiday_roll_rule {PREV_BDAY, NEXT_BDAY}`, `early_close_flag`. Derive each series' actual expiry date at listing time from the roll rule, and version the table so historical replays use the rules in force at the time.

Facts that must be encoded (verified against Cboe/Nasdaq/OCC documents, see Appendix D):

| Item | Rule |
|---|---|
| US equity/ETF option trading close | 16:00 ET (13:00 ET on early-close days) |
| SPX/XSP/RUT/VIX non-expiring series | trade to 16:15 ET; SPX curb to 17:00 ET; GTH 20:15 to 09:25 ET |
| SPXW/XSP expiring PM series | stop at 16:00 ET (15:00 CT); 13:00 ET on half days |
| SPX monthly (AM-settled) | last trade Thursday; settlement = SET from component opening prices Friday |
| Exercise cutoff (FINRA 2360(b)(23)) | 17:30 ET on expiry day; close + 90 min on early-close days (14:30 ET) |
| OCC exercise-by-exception processing | roughly 18:00 to 23:30 CT Friday evening |
| Monday-expiry holiday roll | **forward** to Tuesday (Cboe Rule 4.5(d)) |
| Wed/Thu/Fri-expiry holiday roll | **back** to the preceding business day |
| 2026 Friday holidays | Juneteenth (Fri Jun 19), Independence Day observed (Fri Jul 3); Good Friday Apr 3 |
| 2026 early closes | Nov 27, Dec 24 at 13:00 ET |
| Single-stock Mon/Wed expirations | since Jan 26 2026 on nine names (AAPL, AMZN, AVGO, GOOGL, META, MSFT, NVDA, TSLA, IBIT), up to two Mondays and two Wednesdays beyond the current week, re-tested quarterly on market-cap and volume criteria; no listing on a day with an after-close earnings announcement |
| VIX options | Wednesday AM settlement via SOQ of SPX options; strategy-order cutoff 08:20 CT; weekly VIX uses 30 days + 390 minutes in its time-to-expiry |

The asymmetry in holiday rolls (Monday rolls forward, others roll back) is a known trap; test it explicitly.

### 2.3 Corporate actions

Source of truth: OCC Information Memos (infomemo.theocc.com), applied on the primary-market ex-date before the start-of-day position load.

Rules per OCC By-Laws Article VI §11A (current text in SR-OCC-2025-017):

- No adjustment for ordinary dividends, nor for any cash distribution below **$0.125 per share** (equivalently $12.50 per standard 100-share contract; the per-share form is the current rule text).
- Whole-number splits (2-for-1): contract count multiplied, strike divided, unit of trading unchanged.
- Non-whole splits (3-for-2): strike reduced, deliverable increased (100 → 150 shares).
- Reverse splits: only the deliverable is reduced; strike and premium multiplier unchanged.
- Special distributions: either strike reduced by the per-share value, or deliverable augmented to include the distributed property (creates "100 shares + $X cash" or mixed-stock deliverables).
- Cash mergers: deliverable becomes fixed cash per share equal to the deal price; ITM options become fixed-value instruments, OTM become worthless (not accelerated, but effectively dead).
- Rounding: strikes to the nearest increment, ties round up; units round down with the fractional share's cash value added to the deliverable.

**Ledger treatment:** a `CORP_ACTION` event transforms positions and preserves total cost basis (basis per new contract = old basis × old units / new units). Failure to preserve basis produces a one-day fake P&L equal to the entire basis difference.

**Early-exercise and dividend scanning** (Section 13.5) depends on an accurate projected-dividend table (ex-date, amount, pay date) per underlying. Pull from the data pool or vendor and store with a source and confidence flag.

---

## 3. Market-data hygiene

Bad inputs produce bad surfaces, which produce bad marks, which produce bad Greeks and fake P&L. Filtering is not optional. On a real-data experiment of 17,712 quotes across 386 slices and 80 symbols, two filters (exclude zero bids; exclude relative spread above 50%) removed 32.9% of quotes and cut median SVI fit error from 0.83 to 0.39 vol points (flashalpha.com, SVI liquidity filtering experiment).

### 3.1 Quote filter rules (defaults)

Apply per quote before any fit. Record the rejection reason so the fitter's input set is auditable.

| Rule | Default | Action |
|---|---|---|
| Zero or missing bid | bid ≤ 0 | exclude from fit; keep as upper-bound constraint `fitted_price ≤ ask` |
| Wide relative spread | `(ask − bid) / mid > 0.5` | exclude |
| Wide vol spread | `IV(ask) − IV(bid) > X` | X = 5 vol pts liquid index, 10 single names; downweight above X/2, exclude above X |
| Bid below intrinsic | `IV(bid)` undefined | treat as one-sided quote (ask-only constraint) |
| Locked/crossed | `bid ≥ ask` | drop and flag venue |
| Stale | `recv_ts` older than N s vs underlying tick, or unchanged while underlying moved > 2σ√Δt | exclude |
| Tiny size | size < 1 to 5 contracts (configurable per name) | downweight by `min(1, size/5)` |
| Rotation windows | first 2 min after open, last 1 min before close for the fit target used in marks | exclude from EOD fit input unless the quote was live ≥ 2 min before close (CME uses this standard for settlement inputs) |
| ITM side | puts with `K > F`, calls with `K < F` | exclude from fit; use only for parity cross-check |

**Fit target:** when top-of-book is unbalanced, use the size-weighted microprice `P = I·ask + (1−I)·bid` with `I = Q_bid / (Q_bid + Q_ask)` (Stoikov 2018) rather than the plain mid.

### 3.2 Underlying synchronization

The underlying used to compute an option's implied vol must be simultaneous with the option quote. For index options, never use the cash index: the cash close is 16:00 ET while SPX options trade to 16:15, and the index print lags constituent trades. Use the same-expiry future (ES for SPX) or, better, the forward implied from the option chain itself (Section 3.3). For single names, use the primary-listing NBBO mid or microprice timestamped within a few hundred milliseconds of the option quote.

### 3.3 Implied forward and discount factor extraction

Per expiry, European put-call parity reads `C − P = DF·(F − K)`, which is linear in `K`: `C − P = a + b·K` with `b = −DF` and `a = DF·F`, so `F = −a/b` and `DF = −b`.

Do **not** run OLS on mids. Use the interval-consistent form: find `(F, DF)` consistent with every strike's `[bid, ask]` interval simultaneously, excluding the fewest strikes necessary to restore feasibility (the volkit `estimate_future_from_option_quotes` pattern). Excluded strikes are almost always stale or fat-fingered quotes and should be reported.

Then decompose: `F = (S − PV(known dividends)) · exp((r − borrow)·T)`, so implied borrow is the residual after subtracting known cash dividends. ORATS does this as a "residual yield" anchored at 50-delta with higher weight on near-ATM and tighter-spread quotes, reports an R² of the fit, and smooths the borrow across expiries to avoid calendar arbitrage. A strongly negative residual yield (e.g. −5%) is a hard-to-borrow signal; a residual-rate slope in delta indicates American early-exercise contamination or stale wings, not a real rate.

Decision rules:

1. Fit `(F, DF)` from strikes within ±10–15% of ATM where both legs have two-sided non-stale quotes.
2. If the synthetic width `F_ask − F_bid` exceeds 2× the stock/future spread, fall back to `S − PV(divs)` with yesterday's borrow curve and flag the expiry.
3. Enforce smooth borrow across expiries so `DF·F` does not create calendar arbitrage.
4. For American single-stock options, fit the forward from OTM options only (less early-exercise premium) and cross-check with the futures or stock-plus-dividends forward.
5. Store `(F, DF, borrow, n_strikes_used, excluded_strikes, width)` per expiry per fit as part of `surface_snapshot`.

---

## 4. Pricing and implied volatility

### 4.1 Model and coordinates

Price and invert in **Black-76 on the forward** (undiscounted price `C / DF_T`, moneyness `x = ln(F/K)`), not Black-Scholes on spot. Dividends, borrow and rates collapse into the observable `F_T` and `DF_T`, and put and call IVs coincide by construction when `F` is right. Deribit, CME and every serious surface fitter work this way.

### 4.2 Implied-vol solver

Use Jäckel's "Let's Be Rational" (Wilmott 2015): normalized coordinates `β = price / sqrt(F·K)`, ITM converted to OTM by subtracting normalized intrinsic, rational-cubic seed across four regions, third-order Householder polish; the reference implementation needs exactly **two iterations** for full double precision on all admissible inputs. Reference implementations: `py_lets_be_rational` / `py_vollib` (Python), the `jaeckel` and `implied-vol` Rust crates (track the 2024 C++ reference with a fourth-order Householder step).

Admissible range: `intrinsic < price < upper bound` (for an undiscounted call, `max(F−K,0) < C < F`). The libraries raise `PriceIsBelowIntrinsic` / `PriceIsAboveMaximum`. **Never silently clamp these.** A mid below intrinsic means a wrong forward, a stale or one-sided quote, or American early-exercise premium. Route it to diagnostics, not to a tiny vol.

In vega-degenerate regions (deep wings, very short τ) a one-tick price error maps to many vol points. The solver handles the numerics; the *fit* must handle the information content by weighting (Section 5.3).

### 4.3 American options (single stocks and ETFs)

US equity and ETF options are American with discrete dividends. Index options (SPX, SPXW, XSP, NDX, RUT, VIX) are European.

Market-standard pipeline ("de-Americanization"): find the flat vol `σ*` such that an American pricer (binomial/trinomial tree with discrete cash dividends, or the Andersen–Lake–Offengenden fixed-point method as in QuantLib's `QdFpAmericanEngine`) matches the market price, then emit the European Black price/IV at `σ*` and fit the surface to those. Burkovska et al. (arXiv 1611.06181) show this is accurate at low rates but introduces material errors at high rates, growing with maturity and larger ITM than OTM. Rules:

- De-Americanize from **OTM** quotes only.
- Use a dividend-aware tree (discrete cash dividends on projected ex-dates), at least several hundred steps for EOD marks; OCC's own EOD marks use a proprietary binomial model derived from Cox-Ross-Rubinstein, and OptionMetrics uses multi-thousand-step trees.
- In high-rate regimes (above roughly 4%), prefer a direct American solver inside the IV loop for ITM puts rather than de-Americanized European IVs.
- Fast analytic fallback for intraday: Bjerksund–Stensland 2002 (more accurate than Barone-Adesi–Whaley). Reserve the tree for EOD marks and for the early-exercise scan.

### 4.4 Time measure (the clock)

Pick one clock and use it everywhere: pricing, Greeks, marks, theta in P&L explain, surface aging.

**Day-level clock.** A pure 365-day calendar clock decays Friday to Monday by 3 days and produces the well-documented "weekend effect" artifact in Monday IVs. A pure business-day clock (252) ignores real weekend variance. Practitioner compromise (Moontower): a **308-day year** = 251 trading days × 1.0 + 114 weekend/holiday days × 0.5, with overnight weighted about 0.25 day. Under it, Friday close to Monday open elapses 1.25 days instead of 2.25. Floor convention: decay the weekend out during the week by scaling daily theta by about 6.5/5 so that by Friday 16:00 the weekend is already gone. Either is acceptable; mixing them is not.

**Intraday clock (critical for 0DTE and 1DTE).** Wall-clock time overstates early-day decay and understates closing-leg decay. Intraday variance is U-shaped (high at open and close, trough about 13:00 ET) and overnight is roughly 73% of wall time but only about 30% of daily SPX variance. Define

```
τ_eff(t) = Σ_{buckets i after t} w_i      (within the day)   +   remaining whole days (day clock)
```

where `w_i` are 5-minute variance-share weights estimated from realized 5-minute returns (refit monthly), scaled so a full day sums to one trading day. Feed `τ_eff` into pricing and Greeks. Quoted 0DTE IV rising into the close while total variance falls is then recognized as an artifact of the wall-clock annualization, not a market signal.

**Event time.** Scheduled events (earnings, FOMC, CPI) add a variance lump at a known time. Section 15 handles this via event-variance decomposition; the clock above handles only diffusive seasonality.

**Crypto:** 24/7 trading means τ is near wall-clock with a 7-day seasonality profile (weekend and US-hours weights). The equity trading-time shortcut does not transfer.

---

## 5. Volatility surface construction

### 5.1 Why a parametric arbitrage-free surface

Marks, Greeks and scenario P&L for thousands of strikes must come from a continuous, arbitrage-free object, not from sparse and noisy quotes. The industry default for equity and index options is SVI per slice with an SSVI/eSSVI backbone for cross-expiry consistency. SABR belongs in rates (Hagan's expansion produces negative densities at low strikes; arbitrage-free SABR variants exist but add nothing for equities).

### 5.2 Parameterizations

**Raw SVI** (Gatheral): `w(k) = a + b·{ρ(k−m) + sqrt((k−m)² + σ²)}` with `b ≥ 0`, `|ρ| < 1`, `σ > 0`, `a + bσ·sqrt(1−ρ²) ≥ 0`, and the Lee bound `b(1+|ρ|) ≤ 2` in total-variance slope (QuantLib enforces `b(1+|ρ|) ≤ 4` on its own convention; use 2 for the slope of `w` in `k`). Equity initial guesses: `a ≈ w_ATM`, `m ≈ 0`, `ρ ≈ −0.5`, `σ ≥ 0.01`.

**SVI-JW** (jump-wings) re-expresses the same slice in trader units: ATM variance `v_t`, ATM skew `ψ_t`, put wing slope `p_t`, call wing slope `c_t`, minimum variance `ṽ_t`. Use JW coordinates for day-over-day regularization and for displaying "what moved" (level, skew, wings).

**SSVI** (Gatheral–Jacquier 2014): `w(k, θ_t) = (θ_t/2)·{1 + ρφ(θ_t)k + sqrt((φ(θ_t)k + ρ)² + 1 − ρ²)}` with ATM total variance `θ_t` and a function `φ`. No calendar arbitrage requires `∂_t θ_t ≥ 0` and `0 ≤ ∂_θ(θφ) ≤ (1/ρ²)(1 + sqrt(1−ρ²))φ`. No butterfly arbitrage (sufficient): `θφ(1+|ρ|) < 4` and `θφ²(1+|ρ|) ≤ 4`. With power-law `φ(θ) = η / (θ^γ (1+θ)^{1−γ})` and `γ = 1/2`, the surface is fully arbitrage-free iff `η(1+|ρ|) ≤ 2`.

**eSSVI** (Hendriks–Martini 2019): makes ρ maturity-dependent, slice parameters `(θ_i, ρ_i, ψ_i)` with `ψ = θφ`. Butterfly per slice: `ψ ≤ 4/(1+|ρ|)` and `ψ² ≤ 4θ/(1+|ρ|)`. Calendar between consecutive slices (necessary and sufficient, Corbetta et al. arXiv 1804.04924): `θ_i < θ_{i+1}`, `ψ_i < ψ_{i+1}`, and `|(ρ_{i+1}ψ_{i+1} − ρ_iψ_i) / (ψ_{i+1} − ψ_i)| ≤ 1`. Mingone (arXiv 2204.00312) gives a global reparameterization in which every constraint becomes a box constraint, which is the cleanest thing to implement with a standard least-squares solver.

**Recommended:** eSSVI (global, Mingone parameterization) as the backbone for every underlying; optional per-slice SVI-JW refinement for the liquid front expiries with a crossing penalty against neighboring slices.

### 5.3 Fitting procedure

1. **Inputs:** filtered OTM quotes (Section 3.1), one `(F, DF)` per expiry (Section 3.3), IVs via Section 4.2 in forward coordinates, de-Americanized where applicable.
2. **Weights:** `w_i = vega_i / spread_vol_i` where `spread_vol_i = IV(ask_i) − IV(bid_i)`. Vega weighting makes vol-space error equivalent to price-space error at first order; inverse-spread weighting makes errors scale with market uncertainty. Multiply by the size downweight from Section 3.1.
3. **Loss:** robust (Huber or soft-L1) with scale `f_scale` equal to the typical IV spread (0.5 to 1.0 vol point for liquid index, larger for single names). SciPy `least_squares(loss='huber')` is sufficient.
4. **Warm start** from the previous fit and add a temporal regularizer `λ·‖JW_t − JW_{t−1}‖²` so that a flickering wing quote does not move marks. Default λ such that a one-off single-quote outlier moves ATM vol by less than 0.1 vol point.
5. **Arbitrage checks on a dense grid** `k ∈ [−3, 3]`, not just at quoted strikes: Durrleman butterfly function `g(k) ≥ 0` (Appendix A) and total variance non-decreasing in τ at fixed `k`. Reject and fall back (Section 5.6) if `min g(k) < −1e-6` or any calendar crossing exceeds tolerance.
6. **Wings:** extrapolate linearly in total variance beyond the last liquid strike with slope at most 2 (Lee's moment formula), practically 1.5 to leave headroom; never polynomial or flat-vol extrapolation. Cap IV for very deep ITM/OTM as OCC does.
7. **Time interpolation** between fitted slices: linear in total variance at fixed `k`. Never interpolate in vol.

### 5.4 Quality metrics (store with every surface version)

- vega-weighted RMSE in vol points (target: median under 0.4 vol pt on liquid names after filtering; 85% of slices under 1 vol pt is a realistic benchmark);
- fraction of fitted IVs inside `[IV_bid, IV_ask]` (target > 90% on liquid slices);
- `min g(k)` on the grid and the maximum calendar violation;
- parameter drift vs the prior fit in JW units;
- number of quotes used and excluded, with reasons;
- forward width and borrow per expiry.

### 5.5 Intraday updates between refits

Spot moves between fits. You must choose how the surface moves with spot:

- **Sticky strike:** `σ(K)` fixed; implied vol at a given strike unchanged.
- **Sticky delta / sticky moneyness:** `σ(k)` fixed; the smile slides with the forward.
- **Empirical:** Bergomi's skew-stickiness ratio `R` defines `δσ_ATM = −R · Skew · (δS/S)` type dynamics, with `R = 1` sticky strike, `R = 0` sticky delta, `R = 2` for local-vol at short maturities; empirically for SPX and DAX, `R` exceeds 2 at short maturities (Vargas et al. arXiv 1311.4078).

Default for intraday updates on index products: sticky-strike for moves under 0.5%, with an ATM vol adjustment `δσ_ATM = β_spotvol · (δS/S)` where `β_spotvol` is regressed daily per expiry bucket (Section 14.2). For single names default to sticky-delta between fits. Record the regime in `surface_snapshot.update_mode`. Refit from quotes at least every 1 to 5 minutes for liquid names and whenever spot moves more than 0.5% since the last fit.

### 5.6 Fallbacks

If a fit fails validation: (a) keep the previous surface version aged forward in total-variance time with a `STALE_SURFACE` flag; (b) if staleness exceeds 15 minutes intraday or applies at EOD, fall back to a reduced-parameter SSVI fit on ATM and 25-delta quotes only; (c) if even that fails, use the data pool's vendor surface with an explicit `VENDOR` source tag and widen the mark reserve (Section 6.4). Never let a failed fit silently freeze Greeks.

### 5.7 Surface aging overnight

Age the previous close's surface to the next open by rolling in total variance along the term structure at fixed `k`: `w(k, τ − Δτ)` via the time-interpolation rule, with `Δτ` measured in the chosen clock (Section 4.4) and event lumps removed if the event has passed (Section 15). This aged surface is the "unchanged surface" baseline for P&L explain's roll-down term (Section 12.4).

---

## 6. Marking policy

### 6.1 Why not mid

A zero-bid option quoted 0.00 × 0.05 has mid 0.025 but fair value might be 0.001. One-sided, stale and wide quotes make mids fiction at EOD, and multiple listing venues with no consolidated options quote make even the "best" mid ambiguous. On Aug 5 2024 the VIX printed above 65 pre-market while S&P futures implied under 35, because market makers widened quotes asymmetrically and lifted mids (BIS Bulletin 95). Any mark sanity check must look at spread width, not just the mid.

Every clearer marks to a smoothed curve: OCC takes the composite mid across exchanges, computes IV, smooths the IV curve iteratively per class/type/expiration, caps vol for deep ITM/OTM, and publishes edited marks that are "not a tradable value" (IBKR KB 1199; OIC). CME settles OTM options to the implied-vol skew combined with the futures settlement and ITM options via put-call parity ("settle to the curve"), and since April 2020 settles expiring OTM options at zero. Deribit's mark is the Black-76 price off the same-expiry future with bandwidth checks. The system must do the same.

### 6.2 Mark algorithm (every instrument, every cycle)

```
mark_iv    = σ_surface(k_i, τ_i)                      # Section 5
mark_price = Black76(F_T, K, τ_eff, mark_iv, right) × DF_T   (American: tree/QdFp at mark_iv)
if two-sided, non-stale quote exists:
    if bid ≤ mark_price ≤ ask:   source = SURFACE
    else:                        mark_price = clamp(mark_price, bid, ask); source = SURFACE_CLAMPED
                                 flag with distance in vol points
elif one-sided (ask only):       mark_price = min(mark_price, ask); source = SURFACE_ASK_BOUND
else:                            source = SURFACE_NOQUOTE
mark_price = max(mark_price, discounted_intrinsic)   # never below intrinsic for Europeans;
                                                     # undiscounted intrinsic for Americans
```

Marks outside `[bid, ask]` are allowed only with a documented reason (stale or one-sided quote, size-weighted book) and desk-head sign-off; the override is an event.

Expiring instruments on expiry day follow Section 13.4 (settlement value), not this algorithm, after the expiring-series close.

### 6.3 Official EOD marks

1. **Snapshot time:** one canonical EOD timestamp per product class for spot, forwards, rates, dividends and surfaces (equity options 16:00 ET; index products that trade to 16:15 use 16:15 or 16:00 by policy, but the same choice every day). FRTB requires risk-model and official valuation snapshots to be aligned; mirror that internally.
2. **Inputs:** only quotes live at least 2 minutes before the close, filtered per Section 3.1; the closing auction print of the underlying; same-expiry futures settlement for index products.
3. **Fit** per Section 5 with the regularizer toward the prior official surface.
4. **Mark** per Section 6.2. Freeze as `mark_version = official:YYYY-MM-DD:v1`, `is_official = true`.
5. **Compare** to OCC/CME settlement marks (when available, next morning) with tolerance bands: flag IV differences above 2 vol points front-month index, 5 vol points single names. Reconcile; do not copy. Clearer settlements are curve-smoothed and not tradable, but a systematic bias against them is a sign your forward or wings are wrong.
6. **Compare** to the day's trades (own fills and tape): a mark more than one spread away from the last meaningful trade requires a reason.
7. **Re-marks** (IPV, refit, override) create `v2`, `v3`; P&L always references a named mark version (Section 11.5).

### 6.4 Valuation reserves (the market maker's version of prudent valuation)

Basel CAP50 allows mid-market valuation only for "a significant market-maker in a particular position type" that "can close out at mid-market"; otherwise the prudent side of bid/offer applies, with adjustments for close-out costs, concentrated and stale positions, and model risk. The EU prudent-valuation RTS (2016/101) makes it quantitative: market-price-uncertainty AVA at the 90%-confidence exit price; close-out-cost AVA equal to 50% of the 90%-confidence bid-offer spread; a concentration AVA when the prudent exit period exceeds 10 days.

Translate for the book:

- **Close-out reserve** per vega bucket: `Σ |net ν$_bucket| × ½ × IV_spread_90pct_bucket`, where the 90th-percentile IV bid-ask spread is measured from the day's filtered quotes.
- **Concentration add-on:** when a bucket's net vega exceeds what the market absorbs in roughly 10 days of typical option volume at normal spreads, scale the reserve by the implied exit horizon.
- **Model reserve for wings:** the price difference between slope-1.5 and slope-2.0 total-variance extrapolations on positions beyond the last liquid strike.
- **Pin reserve:** Section 13.3.

Reserves are reported as a separate P&L line (Section 11.3) and recomputed daily; they are not folded into marks.

### 6.5 Mark governance

- Flag any mark whose IV moved more than 2 vol points (front-month index) or 5 vol points (single names) day over day; flag any mark outside bid/ask.
- Independent price verification: Basel requires at least monthly; for a liquid book do daily automated IPV against clearer settlements and vendor surfaces, with monthly human review.
- Overrides require reason code and second-person approval; retain pre- and post-override values.
- Every official surface and mark version is immutable once published.

---

## 7. Greeks and risk measures

### 7.1 Conventions (fix these once, document them in the UI)

| Measure | Definition | Notes |
|---|---|---|
| `Δ$` | `Δ · S · mult · q` | For adjusted deliverables, per component of the basket |
| `Γ$ (1%)` | `Γ · S² / 100 · mult · q` | Change in `Δ$` for a 1% spot move. The curvature P&L of a 1% move is `½ · Γ$ · 1%`. Say which one you display; this spec uses the delta-change form |
| `ν$` | `∂V/∂σ · 0.01 · mult · q` | Dollars per 1 vol point |
| `ν$_norm` | `ν$ · sqrt(T_ref / T)` with `T_ref = 30 days` | 30-day-equivalent vega; limits are set in these units |
| `Θ$` | `∂V/∂t · (1 day in the chosen clock) · mult · q` | Same clock as pricing (Section 4.4) |
| `Vanna$` | `∂Δ/∂σ · 0.01 · S · mult · q` | `Δ$` change per 1 vol point. Explains "I was flat and the market fell 1% and now I am long deltas" |
| `Volga$` | `∂ν/∂σ · 0.01² · mult · q` | Vega change per vol point; exposure to vol-of-vol and smile convexity |
| `Charm$` | `∂Δ/∂t · (1 day) · S · mult · q` | Delta you will have to re-hedge overnight with nothing moving |
| `ρ$` | per 1 bp | Minor for short-dated equity books; matters for deep ITM Americans and boxes |

Black-Scholes forms for the second-order Greeks (with dividend yield `q_y`): vanna `= −e^{−q_y T} φ(d₁) d₂ / σ`; volga `= vega · d₁ d₂ / σ`; charm `= q_y e^{−q_y T} Φ(d₁) − e^{−q_y T} φ(d₁) [2(r−q_y)T − d₂ σ√T] / (2T σ√T)` for calls. For anything off a fitted surface or an American tree, compute by bump-and-revalue (Section 7.3).

### 7.2 Which delta

There is no single delta. Display three and state which one the hedger uses:

1. **Model delta** `Δ_BS`: sticky-strike, the Black delta off the surface.
2. **Hedge delta** `Δ_hedge = Δ_BS + ν_BS · (∂σ/∂S)_regime`, where the vol-spot response is taken from the regime model (Section 14.2): for index books `(∂σ_ATM/∂ ln S) = SSR × skew` with the skew-stickiness ratio `SSR` between 1 and 2 at the front end (empirically above 2 at very short maturities for SPX and DAX, and declining toward 1 at 2 years); for single names closer to sticky-delta (SSR 0.5 to 1).
3. **Minimum-variance delta** (Hull & White 2017): `Δ_MV = Δ_BS + ν_BS · (a + b·Δ_BS + c·Δ_BS²) / (S√T)`, with `(a, b, c)` regressed in-house on a rolling 36-month window per index (the paper's fitted coefficients are published only as a chart). On S&P 500, `Δ_MV` averages about 8% below `Δ_BS`, and out-of-sample hedging-variance reduction is roughly 15 to 30% for calls and 9 to 25% for puts depending on moneyness. The effect is much weaker for single stocks; apply it to index books, and for single names only to calls with shrinkage.

Report `Δ_hedge − Δ_BS` as **skew delta** so the trader sees how much of the hedge is a model choice.

### 7.3 Numerical Greeks

- Analytic Black Greeks for European options off the surface.
- Bump-and-revalue for American options, discrete dividends, and all sensitivities to surface parameters. Default bumps (ORE sensitivity config baseline): spot relative 1%, vol absolute 1 vol point, rates 1 bp, time 1 day; central differences.
- Trees: use Leisen–Reimer (odd step count, 201 to 501) so a node sits on the strike; CRR gamma oscillates and is unusable at modest step counts. Take delta and gamma from the first time-step nodes rather than re-running with bumped spot, to avoid tree-realignment noise. Barone-Adesi–Whaley gives no Greeks; Bjerksund–Stensland can be bumped but is for quoting-time speed, not risk.
- **Near expiry:** ATM gamma, vanna and charm scale like `1/√T` and diverge; vega vanishes like `√T`. Floor `τ` at one hour for Greek computation, clip `Γ$` per contract for display, and switch the primary measure to the spot ladder (Section 8.1). For `T < 1 day`, drop vega from limits or replace it with the P&L of a 1-day-equivalent vol change. Numerix's worked example: at noon with 2 hours to expiry an ATM SPX call has `Δ ≈ 0.50`; a 1% move takes it to about 0.95 or 0.05.

### 7.4 Bucketing and surface risk factors

Bucket by expiry (`0DTE / 1–7d / 8–30d / 31–90d / 91–180d / 180d+`) × moneyness (10Δ put, 25Δ put, ATM, 25Δ call, 10Δ call). Report:

- `Γ$` and `Θ$` per expiry bucket (and `Γ$` for expiries under 7 days as its own headline number);
- `ν$` and `ν$_norm` per expiry × moneyness cell;
- `Vanna$`, `Volga$` per expiry;
- **Skew vega:** P&L for 25Δ-put vol +1, 25Δ-call vol −1, ATM unchanged, per expiry;
- **Term vega:** P&L for front vol +1, back vol −1 pivoting at 3 months;
- **Factor vegas:** project cell vegas onto 2 to 3 PCA factors (level, term slope, skew) estimated from daily surface changes per underlying. Cont & da Fonseca (2002) found SPX and FTSE surface variation is driven by a small number of orthogonal factors interpretable as level, orientation (skew) and convexity, with the first factor explaining most of the variance; a modern cross-sectional study on about 500 S&P constituents (Avellaneda et al., arXiv 2002.00085) finds about nine components for the cross-section. Three factors per index are adequate for risk reporting; single names need a market factor plus idiosyncratic residual.

Raw vega across expiries hides front-end risk. Moontower's example: a one-day move in daily vol from 1% to 3% lifts a 1-week IV from 16 to 26 but a 1-month IV from 16 to only 18.7. Two thousand vega in a weekly is more vol risk than ten thousand vega in a 6-month.

---

## 8. Scenario, stress, tail risk and margin

### 8.1 Spot × vol ladder (the primary risk display)

Full revaluation (not Taylor) at every grid point, per underlying and aggregated:

| Dimension | Index default | Single-name default |
|---|---|---|
| Spot | −8% to +6% in 1% steps (regulatory range), extended to ±15% in 2.5% steps | ±15% in 2.5% steps, extended to ±30% in 5% steps; event names to ±2× implied move |
| Spot (0DTE/1DTE) | ±3% in 0.25% steps, plus every listed strike within ±2% (dense ladder at ±0.1% around heavy strikes) | same, strike-aligned |
| Vol | parallel ±5, ±10 vol points; relative ±20%, ±40% | parallel ±10, ±20; relative ±30%, ±60% |
| Vol shift mode | (a) sticky-strike parallel; (b) sticky-delta; (c) skew-consistent: `δσ_ATM = SSR · skew · δ ln S`; (d) term-consistent: scale by `sqrt(30/T)` per expiry | same |
| Time roll | T+1 day, T+3 days, Friday→Monday under the chosen clock | same |

Expiring series are revalued at intrinsic (settlement) in the ladder; non-expiring at the shifted surface. Report "dollars at risk per 1% move" `= |Δ$| · 1% + ½ · Γ$ · 1%` as the 0DTE headline.

### 8.2 Stress scenarios (named, re-run nightly and on demand)

- Index vol level +50% relative with skew steepening (SSR 1.5 means a −5% spot move re-marks ATM vol by about 1.5 × the 5% skew).
- Single-name gap ±25 to 40% with vol crush or spike.
- Correlation shock: index vol up, dispersion down (implied correlation +15 points).
- Event scenarios (Section 15): realized move ±1×, ±1.5×, ±2× the implied move with post-event vol set to the ex-event vol.
- Historical replays: 2020-03-16 (VIX 82.69, S&P −12%), 2024-08-05 (VIX from 23.9 to above 65 pre-market on asymmetric quote widening), 2018-02-05 (vol ETP unwind), each as a full-revaluation vector of spot, vol level, skew and term changes.

### 8.3 VaR and expected shortfall

Delta-normal understates options tail risk; delta-gamma is unreliable for short maturities and high confidence. Use **full-revaluation historical simulation** on daily vectors of (spot return, parallel vol change, skew change, term change) per underlying over 250 to 500 days, 1-day and 2-day horizons, 99% VaR and 97.5% ES. Keep a delta-gamma-vega-vanna Taylor VaR for intraday incremental checks only. FRTB's useful imports are the longer liquidity horizon for vol than for spot (20 vs 10 days for large-cap equity) and stressed-period calibration; the capital number itself is irrelevant for a prop book.

### 8.4 Shadowing clearing margin and haircuts

Run the regulators' and clearers' ladders inside your own engine from the same positions and surface, so the trader's scenario loss and the clearing requirement are two views of one computation.

- **Customer portfolio margin / TIMS** (FINRA 4210(g), Cboe 10.4): theoretical P&L at 10 equidistant points over +6%/−8% for high-cap broad-based indices, ±10% for non-high-cap broad-based indices, ±15% for individual equities and narrow-based indices; requirement = max(worst point, $0.375 × multiplier per contract); leveraged ETFs scaled by leverage.
- **SEC 15c3-1 Appendix A haircuts** (broker-dealer capital): same 10-point machinery; clearing market makers ±15% equities and narrow indices, ±10% high-cap diversified indices; offsets 100% within a class, 95% between index options and baskets, 90% among high-cap diversified indices, 75% among non-high-cap; a per-contract minimum charge (text reads as ¼ point × multiplier **[verify against eCFR]**).
- **OCC STANS** (clearing-member margin): 2-day 99% expected shortfall from 10,000 Monte Carlo scenarios, nine implied-vol pivots per product, copula with tail dependence, anti-procyclical vol floor with a long lookback. Approximate it with your own 2-day historical-simulation ES and reconcile nightly to the clearing firm's file; attribute differences to marks, pricing model (OCC uses a CRR-derived binomial for Americans) and offsets.
- **CME SPAN** (if hedging with ES): 16 scenarios = {price unchanged, ±1/3, ±2/3, ±3/3 of the price scan range} × {vol up, vol down} plus two extreme ±3× price scenarios weighted 33%; scan risk is the worst; add intra-commodity spread charge and delivery charge, subtract inter-commodity credit, floor at the short option minimum.
- **Deribit PME** (crypto): ±15% spot grid (16% in the segregated model) × relative vol shocks (+45% / −30% scaled by `(30/DTE)^VegaPower`), initial margin 120% of maintenance, per-strike short-option contingency.

Treat per-contract minimums (SPAN short option minimum, PM $0.375 floor) as a hard cost of deep-OTM short wings and feed them into quoting.

---

## 9. Limits and controls

### 9.1 Hierarchy and limit types

`firm → desk → book → trader → underlying → expiry`; each level's limits sum to no more than the parent's.

| Type | Scope | Notes |
|---|---|---|
| Net `Δ$` (beta-adjusted to index for the firm total) | all levels | intraday wider than EOD |
| `Γ$ (1%)` | all levels; separate limit for expiries < 7 days | 0DTE flat by a fixed time (many clearers require about 15:45 ET) |
| `ν$_norm` | all levels | 30-day equivalent |
| Skew vega, term vega, factor vegas | underlying and up | |
| `Θ$` | book and up | max daily decay as % of capital |
| Scenario loss at named grid points | all levels | includes +6/−8, ±10, ±15 regulatory points and wider stress |
| Daily max loss (P&L stop) | trader, book, desk, firm | |
| Concentration | per strike, per expiry, % of open interest | |
| Event exposure | per name | short gamma and short event-vega into earnings |
| Pin exposure | per strike on expiry day | Section 13.3 |
| Time-at-risk | per position | "tolerable for seconds, not overnight or through a scheduled announcement" |
| Margin and haircut utilization | firm | vs capital |

### 9.2 Limit schema

```yaml
limit:
  id: L-0042
  metric: delta_usd | gamma_usd_1pct | vega_usd_norm | skew_vega_usd | theta_usd_day |
          scenario_loss | pnl_day | contracts_executed | notional_executed | exec_count |
          pct_of_quote_executed | risk_trips | pin_exposure_usd | event_vega_usd
  scope: {level: FIRM|DESK|BOOK|TRADER|UNDERLYING|EXPIRY|INSTRUMENT, key: "AAPL"}
  window: {kind: ABSOLUTE_DAY | ROLLING_MS, ms: 1000}
  threshold: {soft: 800000, hard: 1000000}
  action_on_soft: [ALERT, WIDEN_QUOTES(bps: 20), SKEW_QUOTES]
  action_on_hard: [PULL_QUOTES(scope), ONE_SIDED, LOCKOUT]
  reset_policy: MANUAL | AUTO_AFTER_MS
  owner, approver, effective_from, version, change_reason, tx_ts   # append-only
```

Breach workflow: soft → alert and auto-widen or skew quotes toward reducing the exposure; hard → pull quotes at the matching scope (class → account → firm), write a `LOCKOUT` event, require a human reset with reason. Every limit change is an append-only row.

### 9.3 Pre-trade controls (SEC Rule 15c3-5)

Automated pre-trade controls must prevent orders exceeding pre-set credit or capital thresholds (aggregate per client and for the broker-dealer) and erroneous orders (price and size parameters, duplicates), must *reject* before entry rather than remediate, must be under the broker-dealer's direct and exclusive control, and are reviewed at least annually with CEO certification. Thresholds may be adjusted intraday under documented procedures. A vendor-hosted risk UI alone does not satisfy the rule; the checks must live in the order gateway.

### 9.4 Mirror the exchange market-maker protections

Internal counters should coincide with exchange trips so the two never surprise each other. Verified parameter types:

- **Cboe Risk Monitor Mechanism (Rule 5.34(c))**: per class, per EFID, per EFID group; counters for volume, notional, execution count, percentage (sum across series of the % of outstanding quote executed per side, not netted across sides) and risk trips; user interval minimum 100 ms plus absolute daily; up to 8 rules per risk root + EFID; pre-trip marketable orders still execute even beyond the parameters; same-EFID crosses do not count; class-level rules pull SPY and SPY7 together but SPX and SPXW are separate.
- **Nasdaq family (BX/ISE/GEMX/MRX "Rapid Fire")**: percentage or volume threshold mandatory, delta and vega thresholds optional, specified time period ≤ 30 s; "delta" counter is `|(bought calls + sold puts) − (sold calls + bought puts)|` in contracts and "vega" counter is `|bought − sold|` contracts (contract counts, not Greeks); multi-trigger across issues; re-entry requires exchange staff to set a re-entry indicator.
- **ISE Market Wide Risk Protection (Rule 714(d))**: mandatory counts of orders entered and contracts traded over rolling member-set windows; breach rejects all new orders, optionally cancels resting, stays engaged until manually re-enabled.
- **MIAX Aggregate Risk Manager**: allowable engagement percentage (default 105%) over a 1 s default (max 15 s) window; enhanced ARM adds class-protection and firm-wide triggers after 3 to 99 breaches.
- **Self-trade prevention**: Cboe `PreventMatch` (tag 7928) with cancel newest/oldest/both/smallest/decrement modifiers at firm or EFID level with optional trading-group id; set group ids per book so hedge and options books do not self-match.

Design rule: the internal pre-trade layer mirrors these counters (per-class % of quoted size, contracts, delta-contracts, vega-contracts over 1 to 30 s) and additionally enforces the dollar-Greek limits the exchanges cannot.

### 9.5 Encoding risk into quotes

Baldacci, Bergault & Guéant (2021, "Algorithmic market making for options") show that with continuous delta hedging the residual risk is vega, the whole book collapses to one state variable (portfolio vega), and optimal quotes skew roughly linearly in portfolio vega: more vega → wider asks on vega-positive options and tighter bids. Their follow-up (arXiv 2009.00907) notes that short-dated inventories cannot be managed inside an aggregated first-order-Greek inventory, so the front end needs its own treatment (gamma, ladder). Practical import: `quote_skew_i = −k · (portfolio exposure in the relevant risk unit) · (option i's exposure)`, with normalized vega as the unit for `T > 7 days` and gamma dollars for the front, and `k` calibrated so the skew at the vega limit is about 1 vol point.

---

## 10. Books and portfolios

### 10.1 Structure

```yaml
book:
  book_id: MM_TECH_AAPL                      # leaf
  type: OPTIONS_MM | HEDGE | FACILITATION | ERROR | TRANSFER_SUSPENSE
  owner_trader, desk, strategy, legal_entity, clearing_account
  parent: DESK_TECH -> FIRM                  # consolidation hierarchy
  capacity_tag: M | F | C                    # exchange capacity used for fills from this book
  pnl_currency: USD
```

- **Firm/prop vs facilitation**: separate books; Cboe requires an information barrier between market-making and customer business (RG00-052). The capacity tag must flow to the exchange `Capacity` field so contra-capacity-weighted risk counters behave correctly.
- **Hedge book vs options book**: stock and futures hedges in a HEDGE book that is *always* aggregated with its options book at the underlying level for Greeks and margin. The split exists for P&L attribution (gamma scalping vs spread capture), not for risk.
- **Error book**: every bust, mis-allocation, and fat-finger lands here first, at the official mark, and is cleared by an explicit event.

### 10.2 Rules

1. **Netting vs gross**: Greeks and margin net at underlying and firm level (exchange controls and clearing see the clearing account, not books); position-count and notional limits are enforced gross per book so two books cannot offset a limit breach by accident.
2. **One underlying across books**: compute the underlying aggregate across all books for exchange-facing limits and margin; book-level limits remain for discipline.
3. **Internal transfers**: paired fills `(book A sells, book B buys)` at the *official mark* (never trader-chosen prices), `source = INTERNAL`, zero net firm position, auditable by `transfer_id`.
4. **Shared hedges**: allocate by pro-rata `Δ$` at the time of the hedge, recorded as `HEDGE_ALLOCATION` events so re-allocation is itself an event.
5. **P&L segregation**: every P&L line (Section 11) is computed per leaf book and rolled up; the firm-level P&L must equal the sum of leaves plus the error book.
6. **FRTB framing** (useful even for a prop firm): the desk is the level at which model approval is granted and at which P&L attribution is tested; unexplained P&L at desk level is the diagnostic that surfaces booking errors, missing risk factors and stale marks first.

---

## 11. P&L

### 11.1 Definitions (adopt the FRTB vocabulary)

| Term | Definition | Use |
|---|---|---|
| **Actual P&L (APL)** | Daily P&L including intraday trading and new trades, valuation adjustments and time effects (theta, carry, funding); fees excluded | "Official EOD P&L" on official marks; "live P&L" recomputed continuously on live marks |
| **Hypothetical P&L (HPL)** | Revalue the positions held at the prior close with today's market data, positions static | The baseline for P&L explain |
| **Risk-theoretical P&L (RTPL)** | P&L produced by the risk engine's own valuation of the static book | Compared to HPL to validate the risk model |
| **New-trade P&L** | `Σ (mark − trade price) × qty × mult` over today's fills | APL − HPL − fees − carry ≈ new-trade P&L |

### 11.2 Clean (position-based) P&L, not lot matching

For a desk with thousands of fills across hundreds of series, FIFO or average-cost realized/unrealized splits are accounting artifacts. Use the clean identity over any interval `[t0, t1]`:

```
P&L = Σ_i [ q_i(t1)·M_i(t1) − q_i(t0)·M_i(t0) ] × mult_i  −  Σ (signed cash paid for fills in the interval)
```

This is what IBKR calls mark-to-market P&L: every position is assumed settled at each close and reopened the next day, and it splits into *transaction MTM* (fills in the period, mark minus trade price) and *prior-period MTM* (mark change on carried positions). Realized and unrealized become derived views under a chosen matching rule, not inputs. Commissions are displayed separately, never embedded in basis.

### 11.3 P&L lines (every one computed per leaf book, rolled up)

1. **Trading P&L** (clean, Section 11.2) on the chosen mark version.
2. **Fees and rebates** (estimated per fill, trued up monthly against the clearing statement). Facts to encode and re-verify on each change:
   - SEC Section 31: applies to the *sell* side of option premium and stock; $20.60 per $1 million effective Apr 4 2026 (rate was $0 for part of FY2026 pending appropriation).
   - OCC clearing fee: $0.02 per contract, capped at $55 flat for trades of 2,751+ contracts (a 2025 proposal to change this was withdrawn). The flat cap makes per-contract cost size-dependent; allocate the actual clearing-level charge pro rata.
   - ORF (options regulatory fee): per contract side, **customer-range clears only**, so a market maker pays zero ORF on its own flow; rates differ by exchange and change semi-annually.
   - Exchange transaction fees and rebates: depend on origin code, maker/taker, simple/complex and monthly volume tier, so the final fee is unknown until month-end. Book estimates plus a monthly true-up line.
3. **Carry**: stock borrow (accrues on *settled* position, every calendar day, on collateral of 102% × prior settlement rounded up to the next whole dollar, at rate/360; punitive for low-priced stocks), margin interest, dividends on hedges (accrue on ex-date when option marks already reflect the drop; cash-settle on pay date, otherwise explain shows a fake ex-date loss), payment-in-lieu on shorts.
4. **Mark-change P&L**: `Σ q × (M_new_version − M_old_version)` at the same timestamp, when marks are re-marked by IPV, surface refit or override. Trading P&L must be invariant to a re-mark.
5. **Reserves change** (Section 6.4): close-out, concentration, wing-model, pin.
6. **Expiry and settlement P&L** (Section 13.6), including assignment variance and pin P&L.
7. **Corporate-action P&L** (should be ~0 if basis is preserved; a non-zero value is a bug detector).
8. **Error-book P&L** and **bust/adjustment P&L** (each referencing the original fill).

### 11.4 Live vs official

- **Live P&L** recomputes line 1 on live marks (Section 6.2) at the pricing cadence, plus accrued theta and carry for the elapsed fraction of the day under the variance clock (Section 12.5). It is provisional by construction: OCC's final trade-input deadline is 17:30 ET and post-trade adjustments run to about 20:00 ET, so an official position can only be struck after that.
- **Official P&L** runs after the EOD mark freeze (Section 6.3), on `mark_version = official:date:vN`, with positions from the ledger at the EOD `ledger_seq`, reconciled to the clearing statement the next morning. Signed off by trader (explain), product control (marks, IPV) and operations (breaks closed). The signed P&L is pinned to `(ledger_seq, mark_version, surface_version, engine_version)`.

### 11.5 Mark versions and late events

Every P&L report names its mark version. A late fill or a correction is a new ledger event with its own `recv_ts`; it never rewrites a signed P&L. Instead it produces a "prior-day adjustment" line on the day it is learned, with a pointer to the original business date. This is bitemporality applied to P&L.

---

## 12. P&L explain (attribution)

### 12.1 Taylor expansion

```
ΔV ≈ Δ·δS + ½Γ·δS² + ν·δσ + Θ·δt + ρ·δr + Vanna·δS·δσ + ½Volga·δσ² + residual
```

Second-order and cross terms matter for an options book. A reference implementation on an 8-leg SPY book reports a first-order residual of 1.58% of daily |P&L| falling to 0.196% after adding gamma, vanna and volga (MeilinP/pnl-explain). The second-order residual is the number to monitor; the first-order residual is noise.

Compute theta **at fixed σ and r** to avoid double counting with vega and rho.

### 12.2 Simultaneous vs sequential

- **Simultaneous (Greek × factor move, cross terms explicit)** is order-invariant; use it intraday (cheap).
- **Sequential full-revaluation waterfall** (bump time → reprice; then rates; then spot; then vol surface; then new trades; then other) has no Taylor residual but assigns cross effects to whichever factor is bumped last. Fix the order in policy and never change it mid-period. Use it at EOD.
- The difference between the two is itself a model-quality diagnostic; ORE's market-risk examples compute both sensitivity-based and full-revaluation explain for this reason.

### 12.3 Vol P&L decomposition

Attribute vega P&L by (expiry × moneyness) node using the surface as the book is marked, then decompose the surface move into:

1. **Level** (ATM parallel), **skew** (25Δ risk-reversal), **convexity** (25Δ butterfly), **term** (ATM slope across expiries), plus per-node residual; or equivalently project onto the PCA factors of Section 7.4.
2. **Skew-adjusted delta term:** report both the `Δ_BS` term and the `Δ_hedge` term (Section 7.2); the gap is "vanna / skew-dynamics P&L." For SPX, the implied-versus-realized vol differential is often not the main driver of a delta-hedged position's P&L; skew dynamics are (Daviaud, Risk.net 2023; SSR above 2 at short maturities). In sticky-strike regimes long-skew positions pay theta on the skew premium with no re-marking in their favor; in hybrid regimes skew theta is offset by re-marking.
3. **Roll-down / vol carry:** `Σ ν_node × (σ_aged_surface(node) − σ_prev_surface(node))` using the *unchanged* previous surface aged one day (Section 5.7). This isolates the carry the trader is harvesting from the market vol move.
4. **Refit (model/mark) P&L:** compute vega P&L against raw node moves first, then revalue with the new fitted surface; the difference is refit P&L and belongs to the mark-change line, not to trading explain.

### 12.4 Unexplained thresholds and escalation

Percent-of-P&L thresholds blow up on flat days. Express unexplained in units of the desk's daily P&L volatility `σ_P&L` (60-day EWMA) and of the expected hedging error (Section 12.6):

| Condition | Action |
|---|---|
| `|unexplained| > 0.25 σ_P&L` | analyst review same day |
| `> 0.5 σ_P&L` | trader sign-off with written cause |
| `> 1 σ_P&L`, or 3 consecutive days same sign | model and mark review |
| unexplained changes the sign of total P&L | mandatory re-run with full revaluation before official P&L is published |

FRTB's PLA test is a useful periodic validation of the risk engine against official valuation over 250 days per desk: Spearman correlation of RTPL vs HPL (green above 0.80, red below 0.70) and Kolmogorov–Smirnov statistic (green below 0.09, red above 0.12). Expect well-hedged books to fail the KS test often even with a perfect model, because small P&L is noise-dominated (BPI's analysis). Use it as a trend diagnostic, not a pass/fail gate. FRTB permits aligning RTPL inputs to HPL only for data-provider or time-fixing differences, never for "operational noise"; mirror that internally so explain is not tuned to zero.

### 12.5 Theta in explain

Theta must use the same clock as pricing (Section 4.4). Under the 308-day clock, Friday→Monday elapses 1.25 days; under the floor convention, daily theta is scaled by 6.5/5 so the weekend is decayed out by Friday close. Either way, Friday must not show "theta gain" and Monday "vol loss" that net to zero but pollute both lines.

Intraday, allocate theta by cumulative **variance fraction** `f(t)` of the session (from the fitted U-shaped profile plus event lumps): `Θ_explain(t1, t2) = Θ_day × [f(t2) − f(t1)]`. For 0DTE, do not attribute intraday residual to "vol"; vega is negligible relative to theta by midday (Moontower's 1-DTE example: 30-minute theta $0.007 vs vega $0.042 per vol point, on a $0.67 straddle).

### 12.6 Gamma scalping and the expected hedging-error band

Hedging at implied vol, a long option earns `½ Γ S² (σ²_realized − σ²_implied) dt` per instant. Log per hedge interval:

```
gamma_pnl_i = ½ · Γ$_i · (ΔS_i / S_i)² × 100      (in Δ$-per-1% units)
theta_pnl_i = Θ$_i · Δt_i
```

and report cumulative realized-variance P&L against cumulative theta, with the hedge cadence recorded, since different cadences sample different realized vols (Moontower's IWM example: daily hedging sampled 50.1% realized vol; hedging once on day 5 sampled 79.2%).

Derman–Kamal (1999): the standard deviation of total hedged P&L with `N` equally spaced rehedges is about `sqrt(π/4N) · vega · σ`, roughly `0.886 · vega · σ / √N`. For a 1-month ATM option hedged daily (N ≈ 21) that is about 19% of premium; four times as many hedges halves it. QuantLib's `DiscreteHedging` example reproduces it. **Unexplained inside this band is not a model problem.** Publish the band next to the unexplained number.

### 12.7 Validation of the explain itself

- Invariance tests: explain of a flat book is zero; explain is additive across books; a position transfer at mark produces zero explain.
- Golden days: replay 2020-03-16, 2024-08-05, a monthly expiry Friday, a Russell reconstitution Friday, and an ex-dividend day on a name with deep-ITM short calls.
- Back-test: distribution of unexplained vs `σ_P&L` and vs the Derman–Kamal band over 250 days; trend of the second-order residual.

---

## 13. Expiry lifecycle, 0DTE, exercise and assignment

### 13.1 Why 0DTE is a different problem

As `T → 0`, ATM gamma and charm diverge like `1/√T` while vega vanishes like `√T`. A vol-based limit framework is blind to a 0DTE book. Context: 0DTE's share of SPX volume went from 5% (2016) to above 40% after Tue/Thu expirations (2022), roughly 50% in Aug 2023 and around 60% by 2025, with about 2 million contracts a day; Cboe reports more than 95% of 0DTE trades in defined-risk structures and dealer net gamma at 15:30 ET typically between −$1.1 bn and +$2.4 bn per 1% (median +$173 mm), with extremes of −$5 bn to +$7.7 bn; on Aug 15 2023 a single strike traded 100k+ contracts while dealers were net short only about 3k, and dealer gamma flipped from +$2 bn to −$500 mm between 15:00 and 15:30. Lesson for the system: net your own open/close flags by strike rather than treating gross volume or open interest as exposure, and expect your own book to be far more one-sided than the aggregate.

### 13.2 0DTE risk measures and modeling

- **Primary:** dense spot ladder (Section 8.1) with expiring series at intrinsic; `Γ$ (1%)`; "dollars at risk per 1% move"; theta per *minute*; pin exposure per strike.
- **Time:** `τ_eff` from the intraday variance profile (Section 4.4). Clock time overprices morning decay and underprices the closing leg.
- **Last 30 to 60 minutes:** treat the expiring series as intrinsic plus a residual digital. Delta of the ATM strike becomes a step; hedge to the **expected assignment delta** `P(ITM at close)` rather than Black delta; cap gamma-hedging frequency (gamma exceeds any realistic rebalancing cadence); run the ladder at ±0.1% steps around strikes carrying your largest net position.
- **Hard rule:** a configured flatten time for 0DTE gamma (many clearing firms mandate 15:45 ET); the limit monitor escalates from soft to hard at that time.
- **Literature** (for the implementer's background, not required reading): Bandi–Fusari–Renò "0DTE Option Pricing" (jumps shape the OTM 0DTE smile; continuous vol dynamics shape ATM); Dim–Eraker–Vilkov 2024 (higher 0DTE gamma open interest associates with lower intraday realized vol; 0DTEs carry the highest volatility risk premium of any tenor); Beckmeyer–Branger–Gayda 2023 (retail 0DTE flow); Numerix 2026 white paper on gamma hedging of 0DTE.

### 13.3 Pin risk

Definition: the underlying closes at or near a strike where you are short, so assignment is unknown until OCC results post. The uncertainty window is structural: holders have until **17:30 ET** on expiry day to submit contrary instructions (FINRA 2360(b)(23)), and the stock keeps trading after 16:00, so a strike $0.02 OTM at the close can be exercised against you after a late rally, and an ITM long can go unexercised after a late drop.

Quantify per strike `K`: `pin_exposure(K) = |net contracts at K| × 100 × S` (gross shares you may be delivered or called), weighted by `P(|S_close − K| < ε) ≈ 2ε · φ(d) / (S σ √τ_eff)` with `ε` about half a strike width or the after-hours σ. Flag strikes where the net share exposure exceeds a fraction of expected after-hours stock liquidity. Pinning itself is produced by long-gamma dealers resting hedges around the strike (Avellaneda–Lipkin 2003); it is strongest where hedging flow is large relative to stock volume.

Handling: trade out of the straddle at the pinned strike before the close; otherwise hedge to the expected-assignment delta `P(ITM)` and true up after OCC results; carry a **pin reserve** (Section 6.4) released on T+1. PM-settled index options (SPXW, XSP, RUTW, NDXP) have no after-hours flip because they settle to the official close and exercise automatically; the residual is the gap between the 16:00 option close and the final index print, which incorporates component closing auctions.

### 13.4 Settlement conventions

| Product | Style | Settlement | Value | Last trade |
|---|---|---|---|---|
| SPX (3rd Fri) | European | AM | SET: opening sale price of each component on expiry day | Thursday, 17:00 ET incl. curb |
| SPXW (Mon–Fri, EOM) | European | PM | SPX closing value | expiry day 16:00 ET; 13:00 ET half days |
| XSP (1/10 SPX) | European | PM | SPX close / 10 | expiring 15:00 CT; non-expiring 15:15 CT |
| RUT / RUTW | European | AM / PM | RLS opening prices / closing value | Thu / expiry day |
| NDX / NDXP | European | AM / PM | XQO from Nasdaq Official Opening Price (last sale if no NOOP) | Thu / expiry day |
| VIX / VIXW | European | AM (Wed) | VRO via SOQ of SPX options | Tuesday 16:00 CT |
| Equity / ETF | American | physical, T+1 | 100 shares (or adjusted deliverable) at `K` | 16:00 ET |

Cash settlement amount = `(settlement value − K) × 100`. **AM-settlement risk:** SET/RLS/XQO use each component's opening print, arriving over the first 15 to 60+ minutes; the 09:30 index print is not the settlement, and the value is published later in the morning. Thursday-close positions in expiring AM-settled series carry overnight gap risk with no ability to trade the option. Mark them Thursday at fair value under overnight vol (not intrinsic), hedge the residual delta with futures, and use SET as the "spot" for Friday's explain of those lines.

**VIX settlement specifics:** settlement strip = SPX options expiring exactly 30 days later; opening prices from a modified HOSS single-price auction; strategy-order cutoff 08:20 CT; weekly VIX uses a time-to-expiry of 30 days + 390 minutes because SPXW constituents trade to 15:00 CT; expiry moves to the preceding business day if the Wednesday or the Friday 30 days later is a holiday.

### 13.5 Exercise and assignment mechanics

- **Exercise-by-exception (OCC Rule 805):** expiring equity options ITM by at least **$0.01** against the underlying's closing price on the last trading day are auto-exercised unless the clearing member instructs otherwise (threshold cut from $0.05 in 2008). Index and cash-settled options follow Rule 1804(c).
- **Cutoff (FINRA 2360(b)(23)):** 17:30 ET on expiry day; close + 90 minutes on early-close days; brokers set earlier internal cutoffs (commonly 16:30 to 17:00); electronic submissions may reach the exchange until 19:30 ET; firms must disclose their assignment allocation method (FIFO, automated random, or manual random).
- **OCC processing:** exercise-by-exception for Friday expirations runs about 18:00 to 23:30 CT Friday; OCC assigns to clearing members by a random wheel (customer accounts first); results reach accounts Saturday. Contracts expire Friday (since 2015), and since the May 28 2024 move to T+1 settlement OCC **removed the late-exercise process entirely** (Rule 903 amendment). A missed contrary instruction is final; make the pre-17:30 instruction check a hard control.
- **Early exercise economics (American equity options):** exercise a call on the last cum-dividend day when dividend `D` exceeds the remaining time value, which by parity is about `P(K) + K·(1 − e^{−rT})` (same-strike put plus carry on the strike); exercise a deep-ITM put when the interest on `K` exceeds the same-strike call value. System flags: a "dividend risk" report two days before each ex-date listing short ITM calls where `P(K) + carry < D − buffer`; a "dividend play" detector for abnormal volume and open interest in deep-ITM calls into ex-date; a "put interest" flag for short deep-ITM puts in high-rate regimes. Expect assignment overnight on flagged lines and pre-position the hedge (long stock earns the dividend; being assigned removes the call's delta).

### 13.6 Expiry-day sequence for the risk system (US equity/ETF, PM-settled index)

1. **Morning:** ladder with expiring series at intrinsic; pin exposure per strike; list strikes within ±1 band of spot with large net position; confirm settlement type per series; for AM-settled series ingest SET/RLS/XQO when published and book settlement P&L.
2. **15:00–15:45 ET:** soft-to-hard escalation of the 0DTE gamma limit; dense ladder refresh every cycle; hedge to expected-assignment delta on near-ATM strikes.
3. **16:00 ET close:** PM-settled index expiring options → mark at `(settlement − K)⁺ × 100` once the official close is published (not at the last option trade). Equity expiring options → intrinsic vs the 16:00 close *plus* an assignment-uncertainty overlay for strikes within the after-hours band (about ±0.5 σ_after-hours).
4. **Position morphing:** every expiring ITM equity option becomes a *provisional* stock position (±100 × deliverable shares at `K`, delta 1, settling T+1); short positions near the money get a probability-weighted stock delta `P(assigned)`. The morphed book drives the 16:00–17:30 after-hours hedge, so the system needs live extended-hours quotes in that window.
5. **17:30 ET:** contrary-instruction deadline; log every instruction submitted; hard check that every long ITM option the desk does *not* want exercised has a do-not-exercise instruction on file with the broker before the broker's cutoff.
6. **Friday night / Saturday job:** ingest OCC exercise and assignment results; replace provisional positions with actual; compute the **expiration-weekend residual delta** (actual − hedged) and queue Monday-open hedges; book "assignment variance P&L" = (actual exercised − provisional) × payoff.
7. **Reconciliation assertion:** after results are loaded, no position may remain in any series with `expiry_date ≤ today`. Any residual is a break (an unexercised long ITM the broker failed to submit, or a do-not-exercise instruction).
8. **Monday:** delivered stock appears at `K` as cost basis; its first-day P&L vs the opening print is "pin P&L"; release the pin reserve.

Friday-night risk must carry both branches (assigned and not assigned) for every near-the-money short; the weekend book is a distribution over stock positions, not a point estimate.

### 13.7 Crypto (Deribit) notes

Expiry 08:00 UTC; delivery price = 30-minute TWAP of the Deribit index (07:30–08:00 UTC, sampled every 4 s). From 07:30 the expiring series use the estimated delivery price as underlying, so delta decays linearly toward a digital. Inverse (coin-settled) options pay `max(S−K, 0)/S` in coin; the coin-denominated hedge ratio is the exchange's "net transaction delta" `= BS delta − option mark (in coin)`, not the USD delta, and hedges go in inverse futures or perps (which import funding carry: 8-hourly quoted, accruing continuously, zero within ±0.05% of index, capped ±0.5% per 8 h). Portfolio margin is a ±15% (or 16%) spot grid × relative vol shocks.

---

## 14. Hedging policy

The risk system does not execute hedges, but it must compute the hedge targets, bands and costs, and it must attribute the hedge P&L. Encode the following.

### 14.1 Delta bands, not continuous hedging

**Whalley–Wilmott (1997)** asymptotic no-transaction band around the chosen delta, for proportional cost `λ` (fraction of traded value) and absolute risk aversion `γ`:

```
Δ_hedge ∈ [Δ − H_WW, Δ + H_WW],     H_WW = ( (3/2) · e^{−r(T−t)} · λ · S · Γ_net² / γ )^{1/3}
```

Trade to the nearest band edge, not to the center. Width scales as `λ^{1/3}` and `Γ^{2/3}`. Valid for small `λ` (below about 1%); for larger costs it underestimates the band. **Compute the band from the net portfolio gamma `Γ_net = Σ Γ_i` per underlying.** Applying per-leg bands to a spread and intersecting them collapses the band near the money and amounts to continuous hedging.

**Zakamouline (2005/2006)** improves on this with a gamma-independent floor plus a gamma term and a volatility shift (for a short option the shift raises σ, flattening delta and reducing trades; for a long option it lowers σ):

```
Δ_hedge = Δ(σ_m) ± (H_w + H_0),  σ_m² = σ² (1 + H_σ)
H_0 = λ / (γ S σ² (T−t))
H_w = 1.08 · λ^0.31 · σ^−0.25 · (Γ/γ)^0.5
H_σ = 6.85 · λ^0.78 · σ^−0.25 · (γ S² Γ)^0.15
```

Published constants differ slightly between the 2005 working paper and the 2006 journal version; treat them as calibration targets.

**Operational rule per underlying:**

```
band_i = clip( k · (λ_i · S_i · Γ_net,i² / γ)^{1/3}, band_min_i, band_max_i )
```

with `λ_i` from the name's realized half-spread plus impact (re-estimated monthly from the hedge fills), `γ` from the desk's capital and VaR appetite, and `k` a tuning constant. Hedge gamma-heavy names tighter and illiquid names wider. For 0DTE, widen the band with `σ · S · sqrt(remaining minutes)` and tighten only in the last 30 to 60 minutes when delta approaches a step.

**Expected hedging noise** (Derman–Kamal): `σ_P&L ≈ 0.886 · vega · σ / √N` for `N` rehedges over the option's life; the error is a gamma-weighted integral of squared returns, so it scales with realized vol and with the time profile of gamma. Publish this band in the explain (Section 12.6).

### 14.2 Which delta to hedge (regime model)

Hedge `Δ_hedge = Δ_BS + ν_BS · β_spotvol / S` where `β_spotvol = ∂σ_ATM / ∂ ln S`.

- Estimate `β_spotvol` per expiry bucket daily: robust (Huber) regression of daily ATM-vol change on `Δ ln S` over a rolling 60 to 120 day window, estimated separately for up and down days (SPX 2024 showed inverted realized skew). Convert to SSR by dividing by the current ATM skew; cap SSR to [1, 2] for index books, [0.5, 1] for single names.
- This reproduces the Bartlett (SABR) delta when SSR → 1 (`Δ_Bartlett = Δ_BS + vega · ∂σ/∂K|_{K=F}`, proven variance-optimal in lognormal SABR) and the Hull–White minimum-variance delta as the empirical special case. Under-hedge calls and over-hedge puts relative to Black on equity indices.
- Derman's regime guidance for switching the *intraday* surface update: range-bound with unchanged realized vol → sticky strike; trending without a change in realized vol → sticky delta; jumpy or falling market → shift the whole curve by `β_spotvol · Δ ln S` (implied-tree-like). Record the regime in the surface snapshot.

### 14.3 Hedging instrument (index)

- SPX option = $100 × index ≈ 10 SPY options ≈ 2 ES ($50/pt); MES is $5/pt. One SPX delta ≈ 2 ES ≈ 20 MES.
- ES: near-24-hour trading, no borrow or dividend leakage, but basis risk (`F = S·e^{(r−q)T}`, converging at the quarterly roll) and roll risk. SPY: trades in the cash session matching PM settlement but imports dividend timing and 100-share granularity. SPX/XSP/VIX options themselves trade in GTH 20:15–09:25 ET, so overnight re-hedging in the option is possible.
- Reconcile forwards before netting delta across ES options (Black-76 on the future) and SPX options (forward from dividend/financing).
- Book each hedge's cost (half-spread + impact) into a **hedge-cost** P&L bucket separate from gamma P&L; compare realized hedge cost to the `λ` in the band formula monthly.

### 14.4 Vega, skew and term hedging

- Normalize vega to 30-day equivalent (Section 7.1) and hedge the PCA factors (level first), not raw vega per expiry; two to three factors explain roughly 70 to 85% of surface variance, so hold a residual limit on the un-hedged factors. The first eigenvector's amplitude decreases in `T`, which is why `1/√T` weighting is the default (IBKR Risk Navigator exposes both plain `1/√T` and a fitted term curve).
- Hedge skew with 25Δ risk reversals, convexity with 25Δ flies, accounted in RR and fly vega units. In implied-tree regimes skew hedges also carry delta.
- Cross-name hedging of single-name vega with index vega is a dispersion (correlation) position: `σ_I² = Σ w_i² σ_i² + 2ρ̄ Σ_{i<j} w_i w_j σ_i σ_j`; Cboe's COR indices solve for `ρ̄`. Keep a correlation-shock scenario (`ρ̄ + 15` points) in limits.
- **When not to hedge vega:** the expected variance reduction must exceed round-trip spread plus impact plus the variance risk premium you pay. Carr & Wu (RFS 2009) report SPX `mean ln(RV/SW) = −0.59` per month (a long variance position lost over half its value in log terms). Prefer netting intra-book and hedging only the level-factor residual beyond a limit.

### 14.5 Realized-vol monitoring

Daily (OHLC) estimators and relative efficiency vs close-to-close: Parkinson ≈ 5×, Garman–Klass ≈ 7×, Rogers–Satchell ≈ 6× (drift-robust), Yang–Zhang up to ≈ 14× (drift- and gap-robust; use it for names with overnight gaps). Formulas in Appendix A. Intraday: 5-minute realized variance (nothing reliably beats it across 400+ estimators tested by Liu–Patton–Sheppard 2015). Forecast with EWMA λ = 0.94 or GARCH(1,1). Monitor the implied–realized spread per expiry bucket; `ln(RV/IV²) ≈ −0.5` is normal index richness, so flag when 1-month realized exceeds implied for more than a few days.

---

## 15. Event risk

### 15.1 Implied move and event variance

ATM straddle ≈ `S · σ · √T · sqrt(2/π) ≈ 0.80 · S · σ · √T` (Brenner–Subrahmanyam). Hence the 1σ implied move ≈ `1.25 × straddle`, while the *expected absolute move* ≈ `0.80 × straddle`. Document which one each display uses; the "0.85 × straddle" rule of thumb is the second.

Total variance is additive in time: `σ_imp² · T = σ_diff² · T + σ_event²` (event variance in return units, not annualized). Extract `σ_event²` from two expiries straddling the event or from a non-event expiry's diffusive vol:

```
σ_event² = σ_1² T_1 − σ_diff² T_1          (T_1 the first expiry after the event)
ex-event vol = sqrt( (σ_imp² T − σ_event²) / T )
```

Equivalently the event is an extra "day" of weight `w_e = σ_event² / σ_diff,daily²` in the variance calendar (Section 4.4). Moontower's worked example: 36% IV, 40 days, 24% ex-earnings vol → `σ_event ≈ 171%` annualized → about 10.7% daily → about 8.6% implied straddle move, with the one day carrying about 57% of total variance.

### 15.2 Marking and risking through events

- Mark a separate **event variance** per name and expiry; display gamma and vega both including and excluding the event.
- Ladder at ±1×, ±1.5×, ±2× the implied move with post-event vol set to the ex-event vol (vol crush scenario). Empirical crush at the first post-event quote on front expiries: META about 42%, AAPL about 23%, NVDA about 15% of IV; 60-day options barely move. Across about 29k US reports 2023–2026 the options-implied move averaged ±9.2% vs realized ±6.3%; only about a quarter of reports closed beyond the implied move; overpricing grows with implied-move size, while small implied moves on megacaps were breached.
- Limit short event-gamma and short event-vega per name so that the crush scenario stays under a fixed fraction of daily VaR.
- Scheduled macro events (FOMC, CPI, payrolls) carry roughly 1.5 to 3× a normal day's variance in the front weekly; about half of the FOMC-day implied-variance uplift is premium rather than realized variance (Wright, NBER w28306), so do not mark the post-event vol as if the whole uplift disappears.
- Index rebalances and triple witching are liquidity events at the close: Nasdaq's closing cross handled a record $334 bn at the June 2026 Russell reconstitution. Widen delta bands into them and plan MOC/LOC usage around the cutoffs (NYSE MOC/LOC entry by 15:50 ET; Nasdaq MOC until 15:55, modify until 15:50).
- Surfaces must not list or imply an event-free expiry on a day with a known after-close announcement (the exchanges themselves skip single-stock daily expirations on such days).

---

## 16. Daily operations

### 16.1 Start of day (before quoting is enabled)

1. Load clearing-firm positions and cash; apply OCC adjustment memos effective today to the instrument master; ingest exercise/assignment results; rebuild positions by ledger replay and **assert `positions(clearing) == ledger(replayed)`** per instrument. Quoting stays disabled until this passes or every break is classified.
2. Roll expiries: remove zero-quantity expired series; book exercise/assignment deliveries; assert no position remains in any series with `expiry_date < today`.
3. New listings, adjusted contracts, changed dividend forecasts, today's events and their variance weights, today's close type (early close, auction-only events).
4. Overnight gap: ES-implied open (ES adjusted for fair-value basis); run the ladder; pre-compute the delta to trade at the open; recompute the expiration-weekend residual on Mondays.
5. Warm surfaces: age last official surface (Section 5.7); fit from pre-open quotes once they are two-sided; validate.
6. Reset hedge bands (`λ_i` from yesterday's realized spreads and impact, `Γ_net` from the current book); widen for event names.
7. Upload exchange risk profiles (Cboe risk-profile uploads 02:30–09:00 ET take effect 09:05; some venues require completion by 09:00) and assert acknowledgement as a gate.

### 16.2 End of day

1. **T−45 min:** residual delta into the close via MOC/LOC (cutoffs above) or ES into 16:00/16:15; near-expiry and pin review; 0DTE flatten check.
2. **Close:** freeze EOD snapshot (one canonical timestamp per product class); refit surfaces; check calendar monotonicity; re-estimate `β_spotvol`/SSR per expiry bucket; recompute event weights for tomorrow; freeze official marks `v1`.
3. **Official P&L and explain** (Sections 11, 12); investigate unexplained beyond thresholds; reserves recomputed.
4. **Scans:** dividend risk and early exercise (ex-dates within two days), earnings and macro events, corporate actions, index adds/deletes, stale and expired instrument cleanup.
5. **Margin and capital:** shadow TIMS/PM, 15c3-1 haircuts, STANS proxy, SPAN; compare to the clearing firm's file; limit utilization report.
6. **Archive:** risk snapshot with the reproducibility tuple, surfaces, marks, quotes used; diff vs clearing.
7. **Sign-off:** trader (explain), product control (marks/IPV), operations (breaks).

### 16.3 Reconciliation

- **Real-time:** consume exchange drop-copy as an independent fill source with `source = DROPCOPY`; match to OMS fills on `(venue, exec_id)`; unmatched after N seconds is an intraday break.
- **Three-way EOD:** OMS ledger vs drop copy vs clearing-firm position file.
- **Break taxonomy:** `QTY`, `PRICE`, `MISSING_OURS`, `MISSING_THEIRS`, `SYMBOL_MAP` (adjusted root), `CORP_ACTION`, `LATE_FILL`, `BUST`, `EXERCISE_ASSIGN_VARIANCE`, `FEE_VARIANCE`. SLAs: intraday fill breaks within 15 minutes; EOD position breaks before next open; cash breaks T+1.
- **Busts and adjustments:** under Cboe Rule 6.5 parties may nullify or adjust before 08:30 ET next day; the exchange adjusts non-customer trades toward theoretical price and nullifies customer trades where adjustment would violate the customer's limit. A bust is a reversing event referencing the original fill; an adjustment is a reversal plus a new fill, both tagged with reason.

### 16.4 Reliability

- Recovery = nightly snapshot + journal replay, target under one minute; hot replica consuming the same journal.
- Clock sync: FINRA 4590/6820 require business clocks within 50 ms of NIST with synchronization logs retained; use PTP on trading hosts and store `recv_ts` from the disciplined clock alongside `exchange_ts` and `seq`.
- Monitoring: per-feed staleness, surface fit failures (`arb_free = false`, RMSE jump), ledger-vs-drop-copy lag, risk publish latency p99, exchange risk trips by reason code.
- Shadow mode for every new pricer, fitter or mark policy: run on production inputs with outputs not acted upon; compare per-instrument theo, IV and Greek distributions; promote only after agreed tolerances; replay recorded production days (including the regression days of Section 17.2) into the shadow.

---

## 17. Testing and model validation

### 17.1 Unit and property tests (block the build on failure)

- **Pricer:** put-call parity to one tick for Europeans off the same forward; central-difference Greeks vs analytic with tolerance 1e-5 (QuantLib's test-suite pattern: bumps of 1e-4 relative in spot, vol, rate and ±1 day for theta); IV round-trip price → IV → price; boundary cases `T → 0`, `σ → 0`, deep ITM/OTM, strikes on adjusted contracts with multiplier ≠ 100 and multi-leg deliverables; American ≥ European; American call = European call when no dividends.
- **Forward extraction:** synthetic chains with known `(F, DF)` and injected stale/fat-finger strikes; the solver must recover `(F, DF)` and name the excluded strikes.
- **Surface:** every published surface passes `g(k) ≥ 0` on `k ∈ [−3, 3]` and calendar monotonicity; regularizer keeps a single outlier from moving ATM by more than 0.1 vol point; Lee bound respected.
- **Marks:** never below intrinsic; parity-consistent; inside `[bid, ask]` unless flagged; expiring series marked at settlement after the expiring close.
- **Ledger:** idempotent on duplicate fills; busts and corrections produce reversing events; corporate-action transforms preserve total basis; internal transfers net to zero firm-wide.
- **Calendar:** Monday-holiday forward roll and Wednesday/Friday-holiday back roll; early-close day cutoffs (13:00 close, 14:30 exercise cutoff); DST transitions; AM vs PM settlement flags.
- **Explain:** zero on a flat book; additive across books; zero on a transfer at mark; theta clock identical to pricing clock.

### 17.2 Regression days (golden datasets, replayed in CI)

- 2020-03-16 (VIX 82.69, S&P −12%, full backwardation).
- 2024-08-05 (VIX from 23.9 to above 65 pre-market while futures implied under 35; asymmetric quote widening lifted mids). Direct test of mid-based marking and of the spread-aware sanity checks.
- 2018-02-05 (vol ETP unwind).
- A monthly and a quarterly expiry Friday; a Russell reconstitution Friday; a 1:00 pm early-close day; a Monday-holiday week.
- A reverse-split adjustment day and a special-dividend adjustment day on a held name.
- An ex-dividend day on a name with deep-ITM short calls (assignment path).

### 17.3 Back-testing of risk measures

- VaR exception counts at 99% over 250 days; ES back-test.
- Distribution of unexplained vs `σ_P&L` and vs the Derman–Kamal band; trend of second-order residual.
- Periodic FRTB-style PLA (Spearman and KS) as a trend diagnostic per desk.
- Mark quality: daily distance between official marks and OCC/CME settlements; fraction inside bid/ask; frequency of overrides.

### 17.4 Governance (SR 11-7 as the template)

Treat the surface fitter, the mark policy, the hedge-delta model and the event-variance model as models: each has an owner, documentation, a validation record (conceptual soundness, ongoing monitoring, outcomes analysis), a version in `engine_version`, and a change log. Model changes go through shadow mode and are approved by someone other than the author.

---

## 18. Build plan with acceptance criteria

Each phase is complete only when its acceptance tests pass in CI against the golden datasets.

**Phase 1: Reference data and ledger (foundation)**
Instrument master with deliverable lists and bitemporal fields; calendar service; corporate-action ingestion; event ledger with idempotent fill ingestion, busts, corrections, transfers; position projector; nightly snapshot and replay; SOD assertion against clearing positions.
*Accept:* replay of 10 recorded days reproduces positions exactly; duplicate and out-of-order fills handled; corporate-action day preserves basis; Monday-holiday calendar tests pass.

**Phase 2: Market data hygiene, forward extraction, implied vol**
Quote filters with recorded rejection reasons; interval-consistent `(F, DF)` per expiry; LBR implied vol in forward coordinates; de-Americanization pipeline with dividend-aware tree.
*Accept:* synthetic-chain tests recover `(F, DF)`; IV round-trip exact to 1e-10; below-intrinsic quotes routed to diagnostics, never clamped.

**Phase 3: Surface and marks**
eSSVI backbone (Mingone parameterization), optional SVI-JW front refinement, dense-grid arbitrage checks, Lee-bounded wings, total-variance time interpolation, warm start and regularizer, fallbacks; mark algorithm with clamping and flags; official EOD mark freeze with versions; reserves; comparison to clearer settlements.
*Accept:* median vega-weighted RMSE under 0.4 vol pt on liquid names after filtering; 100% of published surfaces arbitrage-free on the grid; marks reproducible from `surface_version`; 2024-08-05 replay flags the asymmetric-widening mids.

**Phase 4: Pricing, Greeks, aggregation, ladders**
Vectorized batch pricing per underlying on a cadence; Taylor updates with forced-reprice rules; dollar Greeks including vanna/volga/charm; three deltas; bucketing and PCA factor vegas; spot × vol ladder with four vol-shift modes and time rolls; near-expiry handling (τ floor, clipping, strike-aligned ladder).
*Accept:* Greeks vs finite differences within tolerance; full book reprice within the cadence budget at peak quote rates; ladder agrees with full revaluation at every point.

**Phase 5: P&L and explain**
Clean P&L; fee, carry, mark-change, reserve, expiry, corporate-action and error lines; live vs official; simultaneous intraday explain and sequential EOD waterfall; vol decomposition (level/skew/convexity/term, skew-adjusted delta, roll-down, refit); theta on the variance clock; unexplained thresholds; Derman–Kamal band.
*Accept:* invariance tests; official P&L reproducible from the tuple; golden-day explains within thresholds; late fill produces a prior-day adjustment, never a rewrite.

**Phase 6: Limits, controls, books**
Limit schema and monitor; breach workflow with actions; exchange-protection mirrors; 15c3-5 checks in the gateway; book hierarchy, netting rules, transfers, hedge allocation.
*Accept:* every limit change audited; simulated breach triggers the correct scoped action; internal trip coincides with the exchange mirror on recorded data.

**Phase 7: Expiry lifecycle and events**
Expiry-day sequence (morphing, after-hours hedge window, 17:30 instruction check, Saturday job, reconciliation assertion); pin exposure and reserve; AM-settlement handling; dividend and early-exercise scans; event variance per name; crush scenarios.
*Accept:* replayed expiry Friday ends with zero expired positions and correct assignment variance; ex-dividend replay flags the assignable calls two days ahead.

**Phase 8: Margin shadows, VaR/ES, stress, operations**
TIMS/PM ladder, 15c3-1 haircuts, STANS proxy, SPAN; historical-simulation VaR/ES; named stresses and historical replays; SOD/EOD checklists automated; reconciliation break management; monitoring and shadow mode.
*Accept:* nightly margin shadow within agreed tolerance of the clearing file with attributed differences; VaR back-test exception count within the expected band.

---

## Appendix A: Formulas

**Black-76 (undiscounted, forward coordinates):**
`C = DF · [F Φ(d₁) − K Φ(d₂)]`, `P = DF · [K Φ(−d₂) − F Φ(−d₁)]`, `d₁ = [ln(F/K) + ½σ²τ] / (σ√τ)`, `d₂ = d₁ − σ√τ`.

**Put-call parity (European):** `C − P = DF · (F − K)`.

**Raw SVI:** `w(k) = a + b{ρ(k−m) + sqrt((k−m)² + σ²)}`; constraints `b ≥ 0`, `|ρ| < 1`, `σ > 0`, `a + bσ√(1−ρ²) ≥ 0`, `b(1+|ρ|) ≤ 2`.

**SVI-JW from raw:** `v_t = w(0)/t`; `ψ_t = (1/√w_t)(b/2)(−m/√(m²+σ²) + ρ)`; `p_t = b(1−ρ)/√w_t`; `c_t = b(1+ρ)/√w_t`; `ṽ_t = (a + bσ√(1−ρ²))/t`.

**SSVI:** `w(k, θ) = (θ/2){1 + ρφ(θ)k + sqrt((φ(θ)k + ρ)² + 1 − ρ²)}`; power law `φ(θ) = η / (θ^γ (1+θ)^{1−γ})`, `γ = ½`, arbitrage-free iff `η(1+|ρ|) ≤ 2`.

**eSSVI slice constraints:** butterfly `ψ ≤ 4/(1+|ρ|)`, `ψ² ≤ 4θ/(1+|ρ|)`; calendar between slices `θ_i < θ_{i+1}`, `ψ_i < ψ_{i+1}`, `|(ρ_{i+1}ψ_{i+1} − ρ_iψ_i)/(ψ_{i+1} − ψ_i)| ≤ 1`.

**Durrleman butterfly function:** `g(k) = (1 − k w'/(2w))² − (w'²/4)(1/w + 1/4) + w''/2 ≥ 0`.

**Lee's moment formula:** `limsup σ²T / |k| ≤ 2` in either wing; extrapolate total variance linearly with slope ≤ 2.

**Quasi-explicit SVI (Zeliade):** for fixed `(m, σ)`, `y = (k−m)/σ`, `w = a + d·y + c·sqrt(y²+1)` is linear in `(a, d, c)` with `d = ρbσ`, `c = bσ`; inner constrained linear least squares, outer 2-D search in `(m, σ)`.

**Implied forward regression:** `C − P = a + bK` ⇒ `DF = −b`, `F = −a/b`; `q_implied = (1/T) ln(S e^{rT} / F)`; with discrete dividends `F = (S − PV(divs)) e^{(r − borrow)T}`.

**Microprice:** `P = I·ask + (1−I)·bid`, `I = Q_bid / (Q_bid + Q_ask)`.

**Second-order Greeks (BS, yield q):** vanna `= −e^{−qT} φ(d₁) d₂ / σ`; volga `= vega · d₁ d₂ / σ`; charm (call) `= q e^{−qT} Φ(d₁) − e^{−qT} φ(d₁)[2(r−q)T − d₂σ√T] / (2Tσ√T)`; speed `= −(Γ/S)(d₁/(σ√T) + 1)`.

**Minimum-variance delta:** `Δ_MV = Δ_BS + ν_BS (a + bΔ_BS + cΔ_BS²) / (S√T)`.

**SSR / hedge delta:** `Δ_hedge = Δ_BS + ν_BS · SSR · skew / S`, with `skew = ∂σ_ATM/∂k`.

**Whalley–Wilmott band:** `H = ((3/2) e^{−r(T−t)} λ S Γ_net² / γ)^{1/3}`.

**Derman–Kamal hedging error:** `σ_P&L ≈ sqrt(π/4N) · vega · σ`.

**Gamma scalping identity:** `dP&L = ½ Γ S² (σ²_realized − σ²_implied) dt`; discrete `Σ [½ Γ_i S_i² (ΔS_i/S_i)² − Θ_i Δt_i]`.

**Event variance:** `σ_imp² T = σ_diff² T + σ_event²`; straddle ≈ `0.80 S σ √T`; 1σ move ≈ `1.25 × straddle`.

**Pin probability:** `P(|S_close − K| < ε) ≈ 2ε φ(d) / (S σ √τ_eff)`.

**Realized-vol estimators (daily OHLC, n days):**
Parkinson `σ² = (1/(4 ln2 · n)) Σ ln²(H/L)`;
Garman–Klass `σ² = (1/n) Σ [½ ln²(H/L) − (2ln2 − 1) ln²(C/O)]`;
Rogers–Satchell `σ² = (1/n) Σ [ln(H/C) ln(H/O) + ln(L/C) ln(L/O)]`;
Yang–Zhang `σ² = σ_co² + k σ_oc² + (1−k) σ_RS²`, `k = 0.34 / (1.34 + (n+1)/(n−1))`, `σ_co²` the variance of `ln(O_i/C_{i−1})`, `σ_oc²` of `ln(C_i/O_i)`.

**Implied correlation:** `σ_I² = Σ w_i² σ_i² + 2ρ̄ Σ_{i<j} w_i w_j σ_i σ_j`.

---

## Appendix B: Core schemas

```sql
fill (
  fill_id UUID PRIMARY KEY, venue, exec_id, exec_ref_id, exec_type CHAR(1),   -- F fill, H cancel, G correct
  order_id, book_id, instrument_id, side, qty, price, trade_ts, recv_ts, ledger_seq BIGINT,
  capacity CHAR(1), contra_capacity, liquidity_flag, clearing_firm, account,
  source,                       -- OMS | DROPCOPY | CLEARING | INTERNAL
  UNIQUE (venue, exec_id, exec_type)
)

position (book_id, instrument_id, as_of_ts, qty_net, qty_long, qty_short, avg_cost,
          ledger_seq_applied, PRIMARY KEY (book_id, instrument_id, as_of_ts))

mark (instrument_id, mark_ts, mark_px, mark_iv, source,  -- SURFACE | SURFACE_CLAMPED | SURFACE_ASK_BOUND
      surface_version, mark_version, is_official BOOL, override_reason, approved_by, tx_ts)

surface_snapshot (underlying_id, version UUID, fit_ts, model,   -- ESSVI | SVI_JW | SSVI_REDUCED | VENDOR
                  params JSONB, update_mode,                      -- STICKY_STRIKE | STICKY_DELTA | BETA
                  fit_rmse_vegaw, pct_inside_spread, min_g, max_cal_violation,
                  n_quotes_used, excluded JSONB, forwards JSONB,  -- per expiry {F, DF, borrow, width}
                  arb_free BOOL, spot_ref, event_weights JSONB, tx_ts)

risk_snapshot (snapshot_id, book_id, scope, scope_key, as_of_ts,
               price, delta_usd, delta_hedge_usd, gamma_usd_1pct, vega_usd, vega_usd_norm,
               theta_usd_day, vanna_usd, volga_usd, charm_usd, rho_usd,
               ladder JSONB, pnl_day, pnl_explained JSONB, pin_exposure JSONB,
               surface_version, mark_version, ledger_seq, engine_version, calendar_version)

limit_event (limit_id, version, metric, scope, window, soft, hard, actions JSONB,
             owner, approver, effective_from, change_reason, tx_ts)   -- append-only

reserve (book_id, as_of_date, kind,   -- CLOSEOUT | CONCENTRATION | WING_MODEL | PIN
         amount, inputs JSONB, mark_version)
```

Storage: Parquet partitioned `date=/underlying=/` for quotes, surfaces and risk snapshots; `date=/` for fills and marks; official tables separate and append-only.

---

## Appendix C: Default parameters (starting points; calibrate per name)

| Parameter | Default |
|---|---|
| Pricing cadence per underlying | 100–250 ms, coalesced |
| Forced full reprice | spot move > 25 bp, surface version change, < 1 day to expiry, gamma term > 10% of delta term |
| Quote filter: relative spread | exclude > 50% |
| Quote filter: vol spread | downweight > 2.5 / exclude > 5 vol pts (index); 5 / 10 (single names) |
| Stale quote | > N s behind underlying, or unmoved after a 2σ√Δt underlying move |
| Forward fit strike range | ±10–15% of ATM, two-sided quotes only |
| Synthetic width fallback | > 2× stock/future spread |
| Fit loss | Huber, `f_scale` 0.5–1.0 vol pt index, larger single names |
| Weights | `vega / spread_vol × size_factor` |
| Regularizer | outlier moves ATM < 0.1 vol pt |
| Arbitrage grid | `k ∈ [−3, 3]`, tolerance 1e-6 |
| Wing extrapolation slope | ≤ 1.5 (hard cap 2) |
| Day clock | 308-day (trading 1.0, weekend/holiday 0.5, overnight 0.25) or 6.5/5 scaled theta |
| Intraday clock | 5-min variance shares, refit monthly |
| τ floor for Greeks | 1 hour |
| Mark day-over-day IV flag | 2 vol pts front index, 5 single names |
| Settlement comparison flag | same thresholds vs OCC/CME |
| Ladder (index) | −8…+6% in 1%, ±15% in 2.5%; vol ±5, ±10 abs, ±20%, ±40% rel |
| Ladder (0DTE) | ±3% in 0.25%, strike-aligned, ±0.1% near heavy strikes |
| SSR cap | [1, 2] index; [0.5, 1] single names |
| MV-delta regression window | 36 months rolling, monthly refresh |
| β_spotvol window | 60–120 days, Huber, up/down split |
| Hedge band | WW form, `λ` from realized half-spread + impact, monthly recalibration |
| 0DTE flatten time | 15:45 ET (confirm with clearing firm) |
| Unexplained thresholds | 0.25 / 0.5 / 1.0 × σ_P&L (60-day EWMA) |
| VaR/ES | historical simulation, full reval, 250–500 days, 1d and 2d, 99% VaR, 97.5% ES |
| EWMA λ (vol forecast) | 0.94 |
| Reserve | close-out = ½ × 90th-pct IV spread × |net vega| per bucket; concentration beyond ~10 days of volume |
| Event scenarios | ±1×, 1.5×, 2× implied move with ex-event vol |
| Break SLAs | 15 min intraday fills; before next open positions; T+1 cash |

---

## Appendix D: Verified facts most likely to change (re-check on each release)

| Fact | Value as researched | Source |
|---|---|---|
| OCC auto-exercise threshold | $0.01 ITM vs closing price | SEC Rel. 34-57163 |
| Exercise cutoff | 17:30 ET; close + 90 min on early-close days | FINRA Rule 2360(b)(23) |
| Late exercise | removed with T+1 (May 28 2024) | SEC Rel. 34-99701 (OCC Rule 903) |
| OCC special-dividend adjustment threshold | $0.125 per share (= $12.50 per 100-share contract) | OCC By-Laws Art. VI §11A, SR-OCC-2025-017 |
| Single-stock Mon/Wed expirations | launched Jan 26 2026, nine names, criteria re-tested quarterly | SEC Rel. 34-104624, 34-104643 |
| Holiday roll | Monday → next business day; others → preceding | Cboe Rule 4.5(d) |
| 2026 Friday holidays / early closes | Jun 19, Jul 3 / Nov 27, Dec 24 13:00 ET | Cboe hours page |
| SEC Section 31 | $20.60 per $1 mm, effective Apr 4 2026 | SEC fee rate advisory FY2026 |
| OCC clearing fee | $0.02/contract, $55 flat ≥ 2,751 contracts | OCC filings 2025 |
| Portfolio margin ranges | +6/−8% high-cap index; ±10% non-high-cap; ±15% equities; $0.375 floor | SEC Rel. 34-54125 |
| Cboe RMM minimum interval | 100 ms; up to 8 rules per risk root + EFID | Cboe Titanium risk spec |
| FINRA clock tolerance | 50 ms vs NIST | FINRA 4590 / 6820 |
| FRTB PLA thresholds | Spearman 0.80 / 0.70; KS 0.09 / 0.12 | BIS MAR32 |
| Deribit expiry / delivery | 08:00 UTC; 30-min TWAP 07:30–08:00 | Deribit support |

---

## Appendix E: Sources

**Vol surface, implied vol, marking**
- Gatheral & Jacquier, "Arbitrage-free SVI volatility surfaces" — https://arxiv.org/abs/1204.0646
- Corbetta, Cohort, Laachir, Martini, robust SSVI calibration — https://arxiv.org/abs/1804.04924
- Hendriks & Martini, extended SSVI, JCF 2019 — https://ideas.repec.org/a/rsk/journ0/6555351.html
- Mingone, global eSSVI parameterization — https://arxiv.org/abs/2204.00312
- Roper, arbitrage-free implied vol surfaces — https://external.maths.usyd.edu.au/u/pubs/publist/preprints/2010/roper-9.pdf
- Lee, moment formula — https://ideas.repec.org/a/bla/mathfi/v14y2004i3p469-480.html
- Homescu, IV surface construction survey — https://arxiv.org/pdf/1107.1834
- Burkovska et al., de-Americanization — https://arxiv.org/abs/1611.06181
- Risk-sensitive option MM with eSSVI — https://arxiv.org/html/2510.04569v1
- Jäckel, Let's Be Rational — http://www.jaeckel.org ; py_lets_be_rational — https://github.com/vollib/py_lets_be_rational ; py_vollib — https://github.com/vollib/py_vollib ; Rust port — https://docs.rs/jaeckel
- QuantLib SVI/SABR/QdFp headers — https://github.com/lballabio/QuantLib (ql/experimental/volatility, ql/pricingengines/vanilla)
- pysabr — https://github.com/ynouri/pysabr ; svi-py — https://pypi.org/project/svi-py/ ; regit-svi — https://docs.rs/regit-svi ; volsurface-lab — https://github.com/rodi008/volsurface-lab
- volkit implied forward — https://volkit.readthedocs.io/en/stable/examples/estimate_future_from_option_quotes.html
- ORATS implied dividends/borrow — https://orats.com/blog/implied-dividends-ORATS-approach-to-market-consensus
- FlashAlpha SVI production guide and liquidity-filter experiment — https://flashalpha.com/articles/engineering-arbitrage-free-volatility-surfaces ; https://flashalpha.com/articles/svi-liquidity-filtering-real-data-experiment
- IBKR on OCC closing prices — https://ibkb.interactivebrokers.com/de/node/1199 ; OIC mark-to-market — https://prd-web.optionseducation.org/news/mark-to-market-where-every-option-finds-its-true-value
- CME option settlement procedures — https://cmegroupclientsite.atlassian.net/wiki/spaces/EPICSANDBOX/pages/457412876 ; expiring OTM = 0 — https://www.cmegroup.com/notices/clearing/2020/03/Chadv20-094.html
- Deribit mark prices — https://support.deribit.com/hc/en-us/articles/25944746962973-Mark-Prices ; inverse options — https://support.deribit.com/hc/en-us/articles/31424939096093-Inverse-Options
- Basel CAP50 prudent valuation — https://www.bis.org/basel_framework/chapter/CAP/50.htm ; EU RTS 2016/101 — https://www.legislation.gov.uk/eur/2016/101
- Stoikov, micro-price — https://ideas.repec.org/a/taf/quantf/v18y2018i12p1959-1966.html
- BIS Bulletin 95 (Aug 5 2024 VIX) — https://www.bis.org/publ/bisbull95.pdf

**Greeks, hedging, vol dynamics, events**
- Hull & White, Optimal Delta Hedging for Options (JBF 2017) — https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2658343
- Derman, Regimes of Volatility (1999) — https://hpirotte.ulb.be/INGESTriskmgt/readings/Derman%20(1999)%20-%20Volatility%20regimes.pdf
- Vargas, Allez, Bouchaud, skew and implied leverage — https://arxiv.org/abs/1311.4078 ; SSR computation — https://arxiv.org/abs/2406.16131 ; SPX SSR 2021–2026 — https://arxiv.org/pdf/2608.12493
- Hagan & Lesniewski, Bartlett's delta — https://arxiv.org/abs/1704.03110 ; Keller-Ressel — https://arxiv.org/abs/2207.13573
- Cont & da Fonseca, QF 2002 — https://ideas.repec.org/a/taf/quantf/v2y2002i1p45-60.html ; Avellaneda et al., PCA for IV surfaces — https://arxiv.org/abs/2002.00085
- Whalley & Wilmott 1997 — https://ideas.repec.org/a/bla/mathfi/v7y1997i3p307-324.html ; Zakamouline EFMA 2005 — https://www.efmaefm.org/0efmameetings/EFMA%20ANNUAL%20MEETINGS/2005-Milan/papers/284-zakamouline_paper.pdf ; Arzel & Lehdili — https://arxiv.org/abs/2603.29994
- Derman & Kamal, hedging error — https://emanuelderman.com/when-you-cannot-hedge-continuously-the-corrections-of-black-scholes ; QuantLib DiscreteHedging — https://github.com/lballabio/QuantLib/blob/master/Examples/DiscreteHedging/DiscreteHedging.cpp ; Bertsimas, Kogan, Lo 2000 — https://web.mit.edu/~alo/www/Papers/bertsimas-etal-97b.html
- Leisen & Reimer 1996 — https://www.wiwi.uni-bonn.de/sfb303/papers/1995/b/bonnsfb309.pdf
- Numerix, gamma hedging of 0DTE (2026) — https://www.numerix.com/sites/default/files/file/2026-01/Numerix_White_Paper_Gamma_Hedging_of_ODTE_Options.pdf
- Moontower: vega risk — https://moontowermeta.com/understanding-vega-risk/ ; weekend theta — https://moontowermeta.com/weekend-theta/ ; pricing 0DTEs — https://moontowermeta.com/pricing-0dtes/ ; P&L decomposition — https://moontowermeta.com/dynamic-hedging-option-p-l-decomposition/ ; earnings variance — https://moontower.substack.com/p/how-an-option-trader-extracts-earnings
- Carr & Wu, Variance Risk Premiums (RFS 2009) — https://engineering.nyu.edu/sites/default/files/2019-01/CarrReviewofFinStudiesMarch2009-a.pdf
- Wright, Event-day Options (NBER w28306) — https://www.nber.org/papers/w28306 ; Lucca & Moench — https://libertystreeteconomics.newyorkfed.org/2012/07/the-puzzling-pre-fomc-announcement-drift
- Avellaneda & Lipkin, stock pinning — https://math.nyu.edu/inmemoriam/avellaneda/stock_pinning.html
- Range-based vol estimators — https://portfoliooptimizer.io/blog/range-based-volatility-estimators-overview-and-examples-of-usage/ ; Liu, Patton, Sheppard 2015 — https://www.economics.ox.ac.uk/node/1373346
- Cboe implied correlation white paper — https://cdn.cboe.com/resources/indices/documents/Implied_Correlation-WhitePaper-v1.0.5.pdf
- Earnings crush and implied-vs-realized studies — https://flashalpha.com/articles/how-much-does-iv-drop-after-earnings-real-crush-numbers ; https://earnings-watcher.com/wiki/options-implied-move-vs-actual-earnings-study-2026
- Baldacci, Bergault, Guéant, Algorithmic market making for options — https://arxiv.org/pdf/1907.12433 ; high-dimension follow-up — https://arxiv.org/abs/2009.00907 ; Stoikov & Sağlam 2009 — https://ideas.repec.org/a/kap/revdev/v12y2009i1p55-79.html ; El Aoud & Abergel — https://ideas.repec.org/p/hal/journl/hal-01061852.html

**0DTE, expiry, settlement, corporate actions**
- Cboe 0DTE research — https://www.cboe.com/insights/posts/volatility-insights-evaluating-the-market-impact-of-spx-0-dte-options/ ; https://www.cboe.com/insights/posts/0-dt-es-decoded-positioning-trends-and-market-impact/
- Bandi, Fusari, Renò, 0DTE option pricing — https://econbiz.de/Record/0dte-option-pricing-bandi-federico/10014348685 ; Bandi et al. 2026 — https://arxiv.org/abs/2603.29430
- Dim, Eraker, Vilkov 2024 (summary) — https://harbourfrontquant.substack.com/p/impact-of-zero-dte-options-on-the ; Beckmeyer, Branger, Gayda (summary) — https://www.cxoadvisory.com/?p=44172
- OCC exercise-by-exception $0.01 — https://www.sec.gov/file/34-57163 ; FINRA Rule 2360 — https://www.finra.org/rules-guidance/rulebooks/finra-rules/2360 ; FINRA Notice 11-35 — https://www.finra.org/rules-guidance/notices/11-35
- OCC Friday processing — https://www.govinfo.gov/link/fr/78/60001 ; random wheel — https://www.govinfo.gov/content/pkg/FR-2002-11-05/pdf/02-28099.pdf ; T+1 and late-exercise removal — https://www.sec.gov/files/rules/sro/occ/2024/34-99701.pdf ; OIC on T+1 — https://www.optionseducation.org/news/understanding-t-1-conversion
- OCC By-Laws Art. VI §11A (SR-OCC-2025-017) — https://www.sec.gov/files/rules/sro/occ/2025/34-104104-ex5b.pdf ; split examples — https://www.optionseducation.org/news/splits-happen ; OCC memo #50867 (ROOT) — https://miaxglobal.com/sites/default/files/alert-files/ROOT_revsplit__50867.pdf
- OSI symbology — https://www.fidelity.com/research/options/osi.shtml ; adjusted roots — https://ibkb.interactivebrokers.com/article/972
- Cboe SPX / XSP / RUT / VIX specs — https://www.cboe.com/tradable_products/sp_500/spx_options/specifications/ ; https://www.cboe.com/tradable_products/sp_500/mini_spx_options/specifications/ ; https://cboe.com/tradable_products/ftse_russell/russell_2000_index_options/rut_specifications/ ; https://www.cboe.com/tradable_products/vix/vix_options/specifications ; VIX SOQ circular RG18-015 — https://cdn.cboe.com/resources/regulation/circulars/regulatory/RG18-015.pdf ; SOQ methodology — https://cdn.cboe.com/api/global/us_indices/governance/SOQ-Settlement-of-Standard-AM-Settled-Index-Futures-and-Options.pdf
- Nasdaq NDX / XND — https://nasdaqtrader.com/content/phlx/NDXProductSheet.pdf ; https://nasdaq.com/docs/2023/08/14/XND_FactSheet.pdf
- Cboe hours and holidays — https://www.cboe.com/about/hours/us-options/
- Single-stock Mon/Wed expirations — https://www.sec.gov/files/rules/sro/cboe/2026/34-104643.pdf ; https://www.bloomberg.com/news/articles/2026-01-16/mag-7-stock-options-get-sec-nod-for-monday-wednesday-expiries
- Dividend risk / early exercise — https://international.schwab.com/story/ex-dividend-dates-understanding-dividend-risk ; https://ibkb.interactivebrokers.com/de/node/2081
- Deribit settlement / delta decay — https://support.deribit.com/hc/en-us/articles/29734325712413-Settlement ; https://support.deribit.com/hc/en-us/articles/25944751433757-Delta-decay-during-settlement ; DVOL — https://insights.deribit.com/?p=1813 ; portfolio margin — https://statics.deribit.com/files/DeribitPortfolioMarginModel.pdf

**P&L, attribution, fees, controls**
- BIS MAR32 (PLA definitions and thresholds) — https://www.bis.org/basel_framework/chapter/MAR/32.htm ; MAR33 — https://www.bis.org/basel_framework/chapter/MAR/33.htm ; EBA RTS on PLA — https://www.eba.europa.eu/sites/default/files/document_library//EBA-RTS-2020-02%20Final%20draft%20RTS%20on%20Backtesting%20and%20PLA%20requirements.pdf
- BPI on PLA tests — https://bpi.com/the-new-profit-and-loss-attribution-tests-not-ready-for-prime-time/ ; Zanders — https://zandersgroup.com/en/insights/blog/frtb-profit-and-loss-attribution-pla-analytics
- IBKR MTM P&L method — https://www.ibkr.info/en/node/56 ; borrow fee mechanics — https://www.interactivebrokers.com/en/pricing/short-sale-cost.php
- SEC Section 31 FY2026 — https://www.sec.gov/rules-regulations/fee-rate-advisories/2026-2 ; OCC clearing fee — https://policyrisk.com/federal-register/2025-02944 ; ORF (Cboe) — https://www.sec.gov/files/rules/sro/cboe/2026/34-105902-ex5.pdf
- Daviaud, Rethinking P&L attribution for options — https://www.risk.net/investing/7959307/rethinking-pl-attribution-for-options ; JRMV PLA validation for equity derivatives — https://www.risk.net/journal-of-risk-model-validation/5452281/validation-of-profit-and-loss-attribution-models-for-equity-derivatives
- MeilinP/pnl-explain — https://github.com/MeilinP/pnl-explain ; ORE MarketRisk examples — https://github.com/OpenSourceRisk/Engine/tree/master/Examples/MarketRisk ; Strata ExplainMap — https://strata.opengamma.io/apidocs/com/opengamma/strata/market/explain/ExplainMap.html
- Cboe obvious error (Rule 6.5) — https://www.govinfo.gov/content/pkg/FR-2022-01-25/html/2022-01324.htm ; OCC submission deadlines — https://boxexchange.com/assets/IC-2014-011-Change-to-OCC-Daily-Submission-Deadlines-Effective-Aug-1-2014.pdf
- SEC Rule 15c3-5 guide and FAQ — https://www.sec.gov/rules/final/2010/34-63241-secg.htm ; https://www.sec.gov/divisions/marketreg/faq-15c-5-risk-management-controls-bd.htm
- Cboe Rule 5.34(c) RMM — https://cdn.cboe.com/resources/regulation/rule_filings/approved/2026/SR-CBOE-2026-006.pdf ; Cboe Titanium risk management spec — https://cdn.cboe.com/resources/techspec/technical-specifications-cboe-titanium-u-s-options-risk-management-specification.md ; Cboe FIX spec — https://cdn.cboe.com/resources/techspec/technical-specifications-cboe-titanium-u-s-options-fix-specification.md ; Cboe QRM RG15-006 — https://cdn.cboe.com/resources/regulation/circulars/regulatory/RG15-006.pdf
- Nasdaq BX Rapid Fire — https://www.sec.gov/files/rules/sro/bx/2023/34-98179-ex5.pdf ; NOM multi-trigger — https://www.govinfo.gov/content/pkg/FR-2015-07-14/pdf/2015-17169.pdf ; ISE 714(d) — https://www.sec.gov/files/rules/sro/isegemini/2016/34-77488-ex5.pdf ; MIAX ARM — https://www.govinfo.gov/content/pkg/FR-2015-07-10/pdf/2015-16858.pdf ; Optiver on MM protections — https://optiver.com/insights/market-maker-protections/
- Portfolio margin ranges — https://www.sec.gov/rules/sro/nyse/2006/34-54125.pdf ; 15c3-1 Appendix A — https://www.customsmobile.com/regulations/17/240.15c3-1a ; OCC STANS — https://www.sec.gov/files/rules/sro/occ/2018/34-82801.pdf ; CME SPAN — https://www.cmegroup.com/clearing/files/span-methodology.pdf
- Cboe RG00-052 information barriers — https://res-certification.cboe.com/resources/regulation/circulars/regulatory/RG00-052.pdf
- FINRA 4590 / 6820 clocks — https://www.finra.org/rules-guidance/rulebooks/finra-rules/4590 ; https://www.finra.org/rules-guidance/rulebooks/finra-rules/6820
- Fed SR 11-7 — https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm

**Architecture and engines**
- Fowler, LMAX architecture — https://martinfowler.com/articles/lmax.html
- Optiver, engineering the three pillars — https://www.optiver.com/insights/technology-blog/engineering-the-three-pillars-of-trading-pricing-risk-and-execution/
- Jane Street, Incremental — https://blog.janestreet.com/introducing-incremental/
- Databento, streaming Greeks example — https://databento.com/blog/option-greeks ; microprice — https://databento.com/docs/examples/order-book/microprice
- NautilusTrader architecture and event sourcing — https://nautilustrader.io/docs/nightly/concepts/architecture ; https://nautilustrader.io/docs/latest/concepts/event_sourcing/
- ORE — https://github.com/OpenSourceRisk/Engine ; OpenGamma Strata — https://strata.opengamma.io/ ; QuantLib — https://github.com/lballabio/QuantLib ; hftbacktest — https://github.com/nkaz001/hftbacktest
- KX TSBS benchmark — https://kx.com/blog/benchmarking-kdb-x-vs-questdb-clickhouse-timescaledb-and-influxdb-with-tsbs/ ; Polars benchmarks — https://pola.rs/posts/benchmarks/
- OnixS on FIX drop copy — https://onixs.biz/insights/understanding-fix-drop-copy-in-financial-trading
