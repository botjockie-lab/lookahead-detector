---
name: lookahead-detector
description: "Audit a trading backtest or strategy for lookahead bias (using information that was not yet known when a simulated decision was made). Use before trusting any backtest result, after writing or changing a backtest loop, when a backtest looks too good, or when live fills disagree with backtest trades. Runs a static timing audit, a shift test, and a live-vs-backtest reconciliation. Works on any language and any LLM/agent."
---

# lookahead-detector

Find and prove lookahead bias in a backtest. Written to be followed by any LLM or human; no
tool in this file is required. Use whatever file-reading and shell tools you have.

**Lookahead bias** = a simulated decision uses information that would only exist LATER than
the time the simulation says the decision happened. The backtest looks profitable because it
is trading on the future.

## Why this skill exists (the case it came from)

One of our test strategies, a short-straddle backtest on 15-minute candles, showed profit
factor 1.60, net +$650, max drawdown -7.8%. Corrected, the same strategy was profit factor
0.915, net -$80, max drawdown -34.6%. It had been run in forward-testing mode with live test
capital, to gather live execution logs, on the strength of the first number.

The bug: the loop iterated candles labelled by their OPEN time. The entry gate was computed
from that candle's CLOSE, but the trade was timestamped and priced at the candle's OPEN. So
every decision used information 15 minutes newer than the price it filled at. The live
daemon, which can only act on a CLOSED candle, entered 15 minutes after the backtest did.

How it was found, and the order to repeat it in:

1. Line up real fills against backtest trades for the same period (reconciliation, step 4).
2. Four of five matched entries differed by exactly one bar length. A constant offset equal
   to the bar size is the signature of a candle-timestamp convention bug.
3. Read the loop and ask, for every value used in a decision: "at the timestamp the loop
   says this happens, is this value already final?"

Nothing in the numbers looked wrong. Good-looking results are not evidence of no bias.

## Step 0 - Gather

Find and read: the backtest loop, the signal/indicator code it calls, the data loader, and
whatever the live/paper version of the same strategy does. Write down, in one line each:

- Bar resolution(s) used (signal bars, price/fill bars, any higher-timeframe data).
- What the bar timestamp means in the data (open time? close time? Check the exchange or
  vendor docs and look at one real row: a 15m bar stamped 10:00 that covers 10:00-10:15 is
  OPEN-stamped; one stamped 10:15 is CLOSE-stamped).
- At what simulated time each decision is made and each order fills.

## Step 1 - Static timing audit (the checklist)

For each decision in the loop, trace every input and mark it KNOWN or FUTURE at decision time.

1. **Open vs close stamps.** A bar's close, high, low, volume and any indicator computed from
   them are only final at bar END. If the row is open-stamped and the decision uses these
   values at the open timestamp, it is lookahead. (The case above.)
2. **Fill price vs decision time.** The fill price must come from data available at or after
   the decision, never from before it and never from a bar that had not closed yet. Taking
   "the close of the bar containing the decision time" is lookahead if that bar is still open.
3. **Intrabar ordering.** If one bar's high and low both touch a stop and a target, the
   backtest cannot know which came first. Check it assumes the worst case or uses finer data.
4. **Higher-timeframe data.** A daily/weekly value (pivot levels, daily high/low/close, VWAP,
   ATR, regime flag) attached to intraday rows must come from the PREVIOUS completed period.
   Look for merges that attach today's daily close to today's intraday rows. `shift(1)` on the
   daily frame before merging is the usual correct form.
5. **Rolling/ranking windows.** Percentiles, z-scores, min/max, normalisation, optimiser fits
   and scalers must use only past rows. Watch for `.rank()`, `.transform()`, `fit()` or
   `scale()` applied to the whole series and then sliced.
6. **Negative shifts and reverse indexing.** Grep for `shift(-`, `.iloc[i+1`, `[::-1]`,
   `bfill`, `backfill`, `interpolate`, `.at[i+`, any index larger than the current one.
7. **Resampling labels.** `resample(...)` and `rolling(...)` defaults differ on which edge
   labels the bucket (`label=`, `closed=`). A bucket labelled with its start but containing its
   whole interior is lookahead if used at the start.
8. **Signal-then-execute on the same bar.** A signal computed from bar N's close must execute
   at bar N+1's open (or later), not at bar N's open or close-before-the-signal.
9. **Reference data that gets revised or listed later.** Instrument lists, strike lists,
   contract availability, delisted symbols, "the cheapest/ATM contract" chosen from data that
   includes contracts not yet listed at that time (also survivorship bias).
10. **State carried across the loop.** Anything assigned before the loop from the full
    dataset (a global max, an entire-period average, a precomputed column that used future
    rows) and read inside it.
11. **Exit logic on the same bar as entry.** A stop or target checked on the entry bar using
    prices from before the entry moment.
12. **Data caches.** A cache that was filled mid-way (partial candles for a still-live
    contract) will silently produce partial or different results later. Confirm the cache is
    complete and that results do not change when it grows.

## Step 2 - Shift test (empirical, works on any backtest)

Lookahead makes results better than reality. Delaying the information should hurt a
lookahead strategy badly and a clean one only a little.

1. Run the backtest as is. Record trades, win rate, profit factor, net, max drawdown.
2. Delay the entry decision by exactly one bar: act on the signal one bar later and price the
   fill one bar later.
3. Run again. Compare.

Read the result:

- Small, smooth degradation: probably clean on this axis.
- Large collapse (e.g. profit factor 1.6 -> 0.9): the edge was living in the one-bar window
  the original code was not allowed to see. Treat as lookahead until proven otherwise.
- Improvement: suspicious in a different way (the signal may be mean-reverting noise).

Also run a **truncation test**: run the backtest on the first N days, then on the first N+k
days. Trades in the first N days must be IDENTICAL in both runs (same entries, same prices).
If adding future data changes past trades, something in the pipeline reads the future.

## Step 3 - Replay test (single-decision proof)

Pick one backtest trade. Rebuild the data frame containing ONLY rows with timestamps up to
that trade's decision time (cut off everything after). Re-run the signal on that truncated
data. The signal and the fill price must be exactly what the backtest recorded. If they
differ, you have located the exact line that reads the future.

## Step 4 - Live-vs-backtest reconciliation (best detector when you have real fills)

1. Export real fills for a recent period (exchange trade history or order history).
2. Group them into round trips: entry time, exit time, symbol, entry price, exit price,
   quantity, exit reason. Convert all times to one timezone (UTC).
3. Run the backtest for the same period with the live settings and the same data.
4. Match trades and tabulate: entry time, backtest time, gap in minutes, entry price,
   exit price.
5. Interpret:
   - Gap equal to one bar length on most matches -> timestamp-convention bug (this case).
   - Backtest entry prices systematically better than live -> priced from a bar that was
     not closed yet, or from the future side of the spread.
   - Backtest trades that live never took -> data the live system could not see, or filters
     live does not apply. Backtest-only or live-only trades both need an explanation.
   - Same trades, different exits -> stop/target intrabar ordering or fill-model difference.
6. List every difference with a cause or an explicit "unexplained". Do not average them away.

## Step 5 - Report

Produce a short report with these sections, in numbers, with no recommendations unless the
user asks:

1. Verdict per strategy: CLEAN / BIASED / INCONCLUSIVE.
2. For each issue: file and line, the exact quantity read, the time it is read at, the time it
   becomes known, and the size of the gap.
3. Shift-test table: metric before and after the one-bar delay.
4. Any live-vs-backtest gap table.
5. What you did NOT check.

## Step 6 - Fix and prove

Follow the repository's test-first rules if it has them.

1. Write a test that fails because of the bug (a unit test around the function that picks the
   fill price or the decision time, with a tiny hand-built dataset where the right answer is
   obvious).
2. Fix the code. A common correct pattern: loop over bars, set `decision_time = bar_open +
   bar_length`, fill from the last bar that closed at or before `decision_time`, scan later
   bars only after the fill.
3. Re-run the test, the shift test, and the full backtest. Report old and new numbers
   side by side.
4. Mark every earlier result produced by the buggy version as invalid: parameter sweeps,
   saved run directories, documents quoting numbers, and any live settings tuned on them.
   Tell the user which live systems were deployed on those numbers.

## Guardrails

- Do not trust a strategy because its results are strong. Treat unusually high profit factor
  with small drawdown on intraday data as a prompt to audit, not as good news.
- Never stop or change a live trading system on your own authority. Report the finding and let
  the owner decide.
- If you cannot read the data or the live fills, say so; mark that part INCONCLUSIVE.
- The audit looks for bias in timing and information. It does not prove a strategy will make
  money; overfitting, costs and regime change are separate checks.

## Quick grep starters

```
grep -nE "shift\(-|iloc\[[^]]*\+ ?1|bfill|backfill|interpolate|\[::-1\]" <files>
grep -nE "resample|rolling|rank\(|transform\(|fit\(|merge|merge_asof|\.reindex" <files>
grep -nE "timestamp *(<=|>=|<|>)|\.timestamp|floor\(|ceil\(" <files>
```

Each hit is a place to apply the Step 1 question: at the time this line claims to run, is
every value it reads already final?
