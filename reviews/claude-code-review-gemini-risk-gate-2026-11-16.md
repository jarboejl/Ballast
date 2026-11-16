# Claude Code review — Gemini risk gate + ledger (2026-11-16)

Scope: review only, no rewrite. I read Idris's brief first, then the code. Line numbers refer to `gatekeeper.py` and `risk_ledger.sql` as shipped in the package.

**Provenance check.** The two split files are byte-identical to the verbatim original: `risk_ledger.sql` matches lines 1–21 of `gemini-risk-gate-as-received-2026-11-16.txt`, and `gatekeeper.py` matches lines 24–167 (checked with `diff`). SHA-256 of the as-received file as I got it: `35acfe91f3b0a0787a6b0a9065b8c3efd5a225e51cbdad7f296aead2bceefdd5`.

**Idris's verified facts, reproduced (not re-argued):** taker 100 @ 50¢ = 175¢, maker-fee series = 44¢, standard maker = 0. `optimize_contract_batch(10, 50, 5000, taker)` returns 12. A limit price of 0 raises `ZeroDivisionError`. All confirmed. Every probe result below comes from the harness in the Appendix, run against the shipped `gatekeeper.py`.

---

## Part 1 — Idris's nine potholes

| # | Pothole | Verdict |
|---|---|---|
| 1 | No exit path | **CONFIRMED** (and worse than stated) |
| 2 | No tape-state input | **CONFIRMED** |
| 3 | Optimizer silently upsizes | **CONFIRMED** (badly understated) |
| 4 | Stateless, no aggregate exposure | **CONFIRMED** (plus: the per-order cap itself leaks) |
| 5 | Ledger can't reconcile outcomes | **CONFIRMED** (plus: it can't record no-trade decisions at all) |
| 6 | No input validation | **CONFIRMED** (one detail corrected) |
| 7 | No quote staleness | **CONFIRMED** (plus: an empty book passes) |
| 8 | Whale filter maker-only, undefined | **CONFIRMED** (plus: NaN passes it) |
| 9 | No glue between gate and ledger | **CONFIRMED** (plus: UNIQUE doesn't fix the exposure race) |

### 1. No exit path — CONFIRMED
- **Evidence:** the module has three functions (`calculate_kalshi_fee_cents`, `optimize_contract_batch`, `evaluate_trade_setup`), and every one is about entry. Nothing takes a position as input, and nothing returns a sell or close action. `seconds_to_close` is used only as an entry freeze (L88). Net edge is `gross_edge - fee_per_contract` (L123), where that fee is the entry fee only. No exit fee and no exit spread enter the calculation.
- **Worse than stated:** the bigger missing exit cost is the spread, not the exit fee. The gate never looks at spread width. Probe: bid 2 / ask 95, taker buy at 95, fair 0.99 → **approved, 3 contracts**. Under rule 1 that position has to come out before close, and the visible bid to exit into is 2¢. See also New Finding 4: the freeze window is a trap for anyone who adds exits by reusing this function.

### 2. No tape-state input — CONFIRMED
- **Evidence:** the signature of `evaluate_trade_setup` (L72–83) has no regime parameter of any type. A calm tape gets the same treatment as any other tape.
- **Note for reviewers:** the banner-text trap ("CALM TAPE — DON'T TRADE" against an engine matching "CALM TAPE") can't happen here only because there is no regime input at all. That is not a point in the gate's favor. `whale_imbalance_ratio` is an order-flow number, not a tape-state signal, and it doesn't stand in for one.

### 3. Optimizer silently upsizes — CONFIRMED, badly understated
Idris showed 10 → 12. The real range is far larger. The loop (L60–67) searches from the requested count up to the dollar cap and keeps the lowest fee per contract. Fee per contract only reaches its floor when `0.07·C·P(1−P)·100` lands on a whole number. At most prices that takes a large C, so the optimizer walks all the way out toward the cap.

Probe results (cap = 5000¢, taker unless noted):

| Requested | Price | Returned |
|---|---|---|
| 1 | 2¢ | **1,793** |
| 1 | 1¢ | **1,443** |
| 1 | 4¢ | 625 |
| 1 | 7¢ | 621 |
| 1 | 37¢ (maker-fee series) | 125 |

Across prices 1–99¢ and requests {1, 2, 5, 10, 20}, the optimizer upsized in **481 of 495** cases.

Three further problems:
- **Upsizing turns blocks into approvals.** Request 1 @ 50¢ taker with fair 0.549. At 1 contract the fee is 2¢, so net edge is 2.90¢ and the trade would be **blocked** at the 3¢ threshold. The optimizer picks 4 contracts, where the fee is 1.75¢ per contract, so net edge is 3.15¢ → **approved at 4×**. Size feeds back into the edge test, so the optimizer is effectively a lever for getting trades approved.
- **The upsize can't be seen downstream.** `TradeEvaluation` (L5–12) has no `requested_contracts` field, and neither does the ledger. Nothing downstream can tell that size changed.
- **It also downsizes silently** to `max_affordable` (L52), with no flag. Under rule 3, a silent change in either direction is a defect.

The docstring (L45–46) says the function stays "near requested_contracts". It doesn't.

### 4. Stateless, no aggregate exposure (R-P1-02) — CONFIRMED
- **Evidence:** the module imports only `dataclasses` and `decimal`. It never mentions `reserved_cents` and has no database access. There is no input for open positions or open orders.
- **The one cap that exists also leaks.** `target = min(max(1, requested), max_affordable)` (L52) is computed before fees. When `target` itself breaks the cap once fees are added, the loop breaks on its first pass (L62–63) and returns `target` anyway. Probe: request 100 @ 50¢, cap 5000 → returns 100, which costs **5,175¢** with fees. The per-order cap does not hold for this order.
- **Every place the design assumes R-P1-02 away:**
  - `max_order_cents` is the only limit anywhere.
  - `reserved_cents` is a column that nothing writes or reads.
  - The schema has no bot, server or strategy column and no event or close-time column. Exposure can't be summed per bot, across the three servers, or per BTC window, even in SQL.
  - "Open" isn't defined: `status` has no set of values, so no query can say which rows count as exposure.
  - Correlation is ignored: different strikes in the same 15-minute window, and back-to-back windows, are all the same BTC bet.
- I'm flagging these, not proposing a fix. R-P1-02 stays open.

### 5. Ledger can't reconcile outcomes — CONFIRMED, with a structural addition
- **Idris's list holds** (`risk_ledger.sql` L1–15): no fill fields, no exit fields, no settlement, no P/L, no `updated_at`, and `status` has no constraint.
- **Addition, which I think is the worst ledger defect:** the ledger cannot store a decision not to trade. `contracts INTEGER NOT NULL CHECK (contracts > 0)` (L7), but every blocked `TradeEvaluation` carries `optimal_contracts = 0` (L90, L97, L112). The table is also described as "one row per intended order." So blocks and model-down NONE decisions have nowhere to go. That breaks rule 5 (record NONE) and rule 6 (every decision reconcilable). It also biases any later study of the gate, because only approvals are ever kept.
- **Addition: no snapshot of the inputs.** There are no columns for fair prob, bid, ask, seconds to close, whale ratio, net edge, requested count or fee-schedule version. Everything is folded into the free-text `rationale`. Grading would mean parsing prose.
- **Addition: no paper flag.** In a paper-only operation, nothing in a row proves it was paper. `kalshi_order_id` (L4) assumes a real exchange order. Nothing in the paper design says what fills a paper order (see New Finding 6).

### 6. No input validation — CONFIRMED (one correction)
- **Correction:** Idris says a request for −5 "gets an approval path for 1." That holds only for zero-fee makers. On a taker at 50¢, −5 → clamped to 1 → upsized to **4**, and approved. A request for 0 → also **4**, approved.
- **More probes** (fair prob in, result out):
  - `model_fair_prob = 1.5` → **approved**, net edge 98.25¢
  - `inf` → **approved**, net edge `Infinity`
  - `nan` → raises `decimal.InvalidOperation`
  - `None` → `TypeError`
  - `requested_contracts=True` → approved, 4 contracts
  - `limit_price_cents=100` (with fair 1.2) → **approved**, even though the ledger's `CHECK BETWEEN 1 AND 99` would reject that row
- **Model-down behavior:** the only possible outcomes are approve, block or crash. Raising an exception is not recording NONE. Whether a crash fails open depends on a caller nobody has written yet.

### 7. No quote staleness — CONFIRMED, with an addition
- **Evidence:** there are no timestamp or age parameters anywhere (L72–83).
- **Addition — a missing book is treated as a maker opportunity.** `best_ask_cents == 0` makes the order a maker (L100) and also skips the crossed-book check (L94). Probes:
  - bid 0 / ask 0 → **approved MAKER_REST**
  - bid 50 / ask 0 → **approved MAKER_REST**
  - bid −1 / ask 0 → **approved MAKER_REST**
- An empty or failed book fetch is the most common bad-data case, and here it passes.
- `seconds_to_close` is also computed by the caller, with no clock reference, so it can go stale as well.

### 8. Whale filter maker-only and undefined — CONFIRMED, plus fail-open on NaN
- **Evidence:** L131 applies the filter only when `mode == "MAKER_REST"`. The 0.35 threshold is hard-coded. Nothing says which direction is good, and it isn't tied to the order's side.
- **Addition:** `nan < 0.35` is False, so a NaN ratio **passes** the filter. Probe: maker with whale ratio `nan` → **approved**.

### 9. No glue between gate and ledger — CONFIRMED, with an addition
- **Evidence:** there is no database import and no insert.
- **Addition for reviewers to test:** `UNIQUE(client_order_id)` stops one ID from being written twice. It does nothing for:
  - (a) a retry that generates a new ID for the same intent. How IDs are generated is not specified.
  - (b) the exposure race. Three servers each doing "SELECT sum(reserved) → check → INSERT" will each pass. Any future R-P1-02 check needs the check and the insert serialized: one transaction holding a lock or using serializable isolation, not a unique key.

---

## Part 2 — What Idris and Gemini both missed (ranked by how badly it would bite a running bot)

**1. An empty or missing book is approved as a maker trade.** (L94, L100)
Ask = 0 means "maker", skips the crossed-book check, and the bid is never looked at. A feed hiccup becomes an approval. This directly breaks rule 5. Probes are under pothole 7.

**2. A bad model value either approves or crashes; it never blocks.** (L121–125)
- 1.5 and `inf` are approved.
- `nan` and `None` raise exceptions.
- The fair prob is never range-checked to [0, 1].
- No path in the code returns a recorded NONE.

**3. No spread-width or exit-liquidity check.** (whole gate)
Under rule 1, every entry has to be exitable, and the round-trip cost is mostly the spread. Probe: 2/95 book, buy at 95 → approved. A gate that lets you buy where you can't get out conflicts with rule 1 even before anyone builds an exit path.

**4. The freeze window traps the exit path when it's built.** (L88)
The gate blocks everything under 300 s, no matter what the action is. If an exit path is built by sending exits through `evaluate_trade_setup`, it will block exits in the last five minutes, which is exactly when exit-before-close has to happen. Separately, an entry at 301 s passes with no check that there is time left to exit. Exits need their own gate rules; entries need a minimum time-to-exit.

**5. No-trade decisions can't be recorded, and inputs aren't captured.** (`risk_ledger.sql` L7; `TradeEvaluation` L5–12)
Covered under pothole 5. It ranks this high because it locks the "78% graded ERROR" failure into the schema from day one: anything blocked is invisible, and anything approved can't be re-derived without parsing text.

**6. Maker approvals assume a fill, and there's no paper fill model.** (L100–135)
A MAKER_REST approval prices edge as if the resting order fills at the limit price. Two problems follow:
- Resting orders fill mostly when price moves against them (adverse selection). The edge test ignores this.
- The project is paper-only, so something has to decide whether a paper maker order filled. Nothing does.

Without a stated fill rule, paper P/L on maker trades can't be trusted, and grading can't fix it after the fact.

**7. The gate has no `side`; the ledger does.** (L72–83 vs `risk_ledger.sql` L6)
- The edge is `fair − limit`, so `model_fair_prob` must already be the probability of the side being bought. The bid and ask must also be that side's book.
- Neither convention is written down. Kalshi's book is quoted as YES and NO bids, so a NO buy needs converted prices.
- Probe: a caller who passes P(YES) = 0.65 for a NO buy at 40¢ gets **approved with +23.3¢ net edge**. The true NO edge is 35 − 40 = −5¢.
- This is a contract ambiguity, not a proven bug, but the ledger row's `side` can never be checked against the evaluation.

**8. The per-order cap doesn't include fees.** (L48, L52, L62)
100 @ 50¢ under a 5000¢ cap → 5,175¢. Small dollars, but it's the only limit in the design.

**9. Mode is decided for the whole order, ignoring depth.** (L100)
No size-at-best-ask input exists. A 12-contract taker order against 1 contract on the ask fills 1 as taker and rests the other 11 as a maker at the limit price. The fee, the mode label and the whale-filter decision are then wrong for most of the order.

**10. The maker-fee flag defaults to the cheaper case.** (L82)
`series_charges_maker_fee=False` is the default and isn't derived from the ticker. If a caller forgets the flag, fees are understated and edge is overstated. I have **not** checked which series the BTC 15-minute markets fall under — Rook/Ledger, please verify against the current fee schedule.

**11. Unverified fee-rounding points.** (L33–34, L60–67) Flagged for attack, not asserted:
- (a) If Kalshi applies its cent ceiling per fill rather than per order, a resting order filled in pieces gets several ceilings, and the optimizer's whole premise (round once per order) is false.
- (b) Taker fees are computed at the limit price, while the exchange charges at the fill price. Fee is highest at 50¢, so a fill closer to 50¢ than the limit has a higher fee than the gate estimated.
- (c) The fee rates are hard-coded, and no version is stored with each row, so trades can't be regraded correctly after a fee change.

**12. Smaller items.**
- The gate and the schema disagree on valid limit prices: limit 100 is approved, but the schema rejects it.
- Both docstrings make false claims: L45 says "count <= max_order_cents"; L85 says "deterministic" for a function that takes no snapshot and can't be replayed from the ledger.
- Execution modes are free text: `"BLOCKED"`, `"MAKER_REST"` and `"TAKER_CROSS"` with no enum in either place.

---

## Bottom line

**Rebuild to the doctrine. Don't build this out as-is, and don't scrap it outright.**

**Worth keeping:**
- The fee function (verified exact)
- Decimal money math
- A written reason on every decision
- The `client_order_id` idempotency idea
- An entry freeze as a concept

**What has to change:**
- **The optimizer goes, entirely.** It is a sizing engine pretending to be fee hygiene, and it doubles as a lever for approving trades. Under rule 3 it has no repair path.
- **The ledger needs redesigning before any code is written against it.** As written it can't hold NONE, can't hold inputs, and can't hold outcomes, which is the grading failure rule 6 exists to prevent.
- **The structural parts the gate lacks:** exit, regime enum, side, timestamps, state and exposure.

Those missing parts are the architecture, not features waiting to be added. Building them onto this gate means building a new gate around it.

**Exposure (R-P1-02) stays unresolved.** Nothing here should be read as resolving it.

---

## Appendix — reproduction harness

Run from outside the package directory: `python3 -I probe.py /path/to/gatekeeper.py`.

```python
import sys, importlib.util
from decimal import Decimal
spec=importlib.util.spec_from_file_location("gk", sys.argv[1]); gk=importlib.util.module_from_spec(spec); spec.loader.exec_module(gk)
f=gk.calculate_kalshi_fee_cents; o=gk.optimize_contract_batch; e=gk.evaluate_trade_setup
print("fees", f(100,50,False), f(100,50,True,True), f(100,50,True))          # 175 44 0
print("opt 10@50", o(10,50,5000,False))                                       # 12
for req,price in [(100,50),(10,50)]:
    n=o(req,price,5000,False); print("cap",req,price,"->",n,"cost+fee",n*price+int(f(n,price,False)))  # 100 -> 5175
worst=[(o(r,p,5000,False)/r,r,o(r,p,5000,False),p) for p in range(1,100) for r in (1,2,5,10,20) if o(r,p,5000,False)>r]
worst.sort(reverse=True); print(len(worst), worst[:4])                         # 481; 1->1793 @2c
base=dict(limit_price_cents=50,best_bid_cents=48,best_ask_cents=50,seconds_to_close=600,whale_imbalance_ratio=1.0,max_order_cents=5000)
r=e(requested_contracts=1,model_fair_prob=0.549,**base); print(r.approved,r.optimal_contracts,r.net_edge_cents)  # True 4 3.15
for p in (1.5,float('inf'),float('nan')):
    try: r=e(requested_contracts=1,model_fair_prob=p,**base); print(p,r.approved,r.net_edge_cents)
    except Exception as x: print(p,"RAISES",type(x).__name__)
print(e(requested_contracts=1,model_fair_prob=0.6,**dict(base,best_bid_cents=0,best_ask_cents=0)).approved)        # True MAKER
print(e(requested_contracts=1,model_fair_prob=0.6,**dict(base,limit_price_cents=45,whale_imbalance_ratio=float('nan'))).approved)  # True
print(e(requested_contracts=1,model_fair_prob=0.99,**dict(base,best_bid_cents=2,best_ask_cents=95,limit_price_cents=95)).approved) # True
print(e(requested_contracts=1,model_fair_prob=0.65,**dict(base,limit_price_cents=40,best_bid_cents=38,best_ask_cents=40)).net_edge_cents) # 23.32
print(e(requested_contracts=-5,model_fair_prob=0.6,**base).optimal_contracts)  # 4
```
