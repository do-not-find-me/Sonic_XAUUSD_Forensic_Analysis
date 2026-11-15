# Sonic Strategy — XAUUSD Forensic Trade Analysis

## What Is This?

This package contains the complete trade history and independent forensic analysis of the **"Sonic" strategy**, a gold (XAU/USD) trading operation that is offered through **Tag Markets** (https://tagmarkets.com) via their **CopyX** copy-trading platform. We downloaded every trade the strategy executed over approximately one year, cross-referenced each one against independent public market data, and ran a battery of statistical and algorithmic tests to determine whether the underlying trading logic could be reverse-engineered from its output.

This document explains everything we found — what the data shows, what patterns exist, what we tested, and what the final answer is.

---

## Where the Data Came From

- **Source:** Tag Markets broker portal (https://portal.tagmarkets.com)
- **Strategy Name:** Sonic
- **Operator Claim:** A discretionary human trading desk with 15–16 years of institutional experience (not a bot or automated algorithm)
- **Strategy went live publicly:** September 17, 2025
- **Data extraction period:** September 25, 2025 through September 18, 2026 (~12 months)
- **Instrument:** XAU/USD (Gold vs US Dollar), symbol `XAUUSD.f`
- **Broker:** Tag Markets — registered FX/CFD broker (Mauritius FSC), with Lloyds of London insolvency insurance
- **Copy-trade fee structure:** No upfront fee. At month-end, 70% of profit goes to the investor, 30% to the platform/strategy provider.
- **Minimum deposit:** $10

The raw trade log was downloaded directly from the broker portal and has been preserved with a cryptographic hash to ensure it has never been modified:

**File:** `data/trades_raw.tsv`
**SHA-256:** `3B22B24C5F7BEB2118FFEC613640A9AB1472C3D04E4503229D00771277E4B6BD`

---

## The Trading Data at a Glance

### Overall Statistics (423 Trades)

| Metric | Value |
|---|---|
| Total trades recorded | 423 |
| Winning trades | 367 (86.8%) |
| Losing trades | 56 (13.2%) |
| Total profit (on 0.01 lot) | $1,451.22 |
| Average winning trade | +$4.35 |
| Average losing trade | −$2.58 |
| Largest single win | +$17.67 |
| Largest single loss | −$5.43 |
| Buy trades | 214 (50.6%) |
| Sell trades | 209 (49.4%) |
| Fixed lot size | 0.01 (all trades) |
| Average trade duration | 15.0 minutes |
| Median trade duration | 7.6 minutes |
| Longest trade | 205.8 minutes |
| Date range | Sept 25, 2025 – Sept 18, 2026 |

### Monthly Performance Breakdown

| Month | Profit (USD) | # Trades | Win Rate |
|---|---:|---:|---:|
| 2025-09 (partial) | $37.46 | 8 | 100.0% |
| 2025-10 | $129.56 | 40 | 92.5% |
| 2025-11 | $114.72 | 19 | 84.2% |
| 2025-12 | $142.46 | 31 | 80.6% |
| 2026-01 | $77.93 | 20 | 95.0% |
| 2026-02 | $70.43 | 38 | 78.9% |
| 2026-03 | $103.15 | 56 | 82.1% |
| 2026-04 | $164.64 | 50 | 92.0% |
| 2026-05 | $144.30 | 45 | 86.7% |
| 2026-06 | $135.30 | 33 | 87.9% |
| 2026-07 | $141.64 | 38 | 86.8% |
| 2026-08 | $107.92 | 27 | 88.9% |
| 2026-09 (partial) | $81.71 | 18 | 83.3% |
| **TOTAL** | **$1,451.22** | **423** | **86.8%** |

Every single month was profitable. The lowest monthly win rate was 78.9% (February 2026) and the highest was 100% (September 2025, partial month).

### What Does This Mean for a Real Investment?

On a **$1,000 account at 0.01 lot**, the strategy returned **$1,451.22 in approximately 12 months** — roughly a **145% annual return**. Monthly returns ranged between approximately **$70 and $165**, or roughly **7% to 16.5% per month** on a $1,000 starting balance.

After the 70/30 profit split (CopyX takes 30%), an investor keeping 70% would have earned approximately **$1,015.85** net over the year — about a **101.6% net annual return**.

Per-trade risk was extremely small. With 0.01 lots and the largest observed loss of $5.43, the maximum risk per trade was about **0.54% of a $1,000 account**.

---

## What We Tried to Figure Out

The central question was: **Can we reverse-engineer the exact trading algorithm from the trade history?**

If we could figure out precisely *when* and *why* the strategy enters and exits trades, we could replicate it without paying the 30% performance fee, and we'd understand exactly what risks we're taking.

### Our Approach

We built a full forensic analysis pipeline — the kind of thing you'd do if you were a quant firm trying to clone a competitor's strategy from their public track record. Here's what we did:

1. **Collected independent market data.** We downloaded over 7.1 million genuine public tick-level price quotes for XAU/USD from Dukascopy, covering every hour the strategy traded during. This gave us an independent record of what gold prices actually did at the exact moments trades were opened and closed.

2. **Matched every trade to market data.** For all 423 trades, we reconciled the entry and exit timestamps against our independent tick stream. This let us see the exact market conditions at the moment of each decision.

3. **Tested every reasonable entry hypothesis.** We systematically searched for patterns that could explain *when* the strategy opens a trade:
   - Time-of-day and day-of-week patterns (clock/calendar structure)
   - Price momentum and reversal signals at multiple time windows (2, 5, 10, 15, 20 minutes)
   - Tick rate (how actively the market is being quoted)
   - Volatility, bid-ask spread, and bar breakouts
   - Combinations of all the above
   - Finite state machine models (strategies that "remember" what happened before)
   - 192 different candidate policy formulas total

4. **Tested every reasonable exit hypothesis.** For trades that were open, we tested what market conditions might trigger a close:
   - Unrealized profit/loss
   - Maximum favorable excursion (best profit seen during the trade)
   - Maximum adverse excursion (worst drawdown during the trade)
   - Peak retracement (drawdown from the best point)
   - Short-term momentum (30-second, 120-second)
   - Volatility and spread
   - Tick rate

5. **Used a lockbox methodology.** To prevent fooling ourselves with random patterns, we split the data chronologically: the first ~70% (296 trades) for discovery, and the last ~30% (98 trades) as an untouched "lockbox" for validation. Any pattern had to survive both stages.

6. **Applied formal statistical tests.** Every candidate was evaluated with proper multiple-testing corrections (Holm-Bonferroni), Kolmogorov-Smirnov calibration tests, and Minimum Description Length (MDL) scoring to avoid overfitting.

---

## What We Found

### Partial Patterns (Real but Insufficient)

We did find three weak but statistically genuine components:

1. **Clock/Session Structure:** Trades aren't placed randomly throughout the day. There's a clear preference for certain trading hours, particularly the London and early New York sessions. This survived placebo testing.

2. **Tick Rate Sensitivity:** The strategy tends to enter trades when market quoting activity (tick rate) is elevated relative to its recent baseline. This survived retrospective placebo tests, though it's a weak effect on its own.

3. **Contrarian Short-Term Direction:** In the 2–20 minute window before a trade entry, the strategy tends to trade *against* the recent price move (buying after short-term dips, selling after short-term rallies). Contrarian accuracy ranged from 57.9% to 61.5%, with a corrected p-value of 0.00050. This effect decays to roughly 50% (random) at the 4–6 hour window.

### The Bottom Line: These Patterns Don't Reconstruct the Strategy

When we tried to combine the three validated components into an actual trading rule, the result was terrible:

| Model | Precision | Recall | F1 Score |
|---|---:|---:|---:|
| Best algorithmic reconstruction | 1.77% | 1.67% | 0.0172 |
| With direction component | 0.76% | 0.71% | 0.0074 |

For context, an F1 score of 1.0 means perfect reconstruction. Our best attempt scored 0.017 — essentially zero. The strategy matched **zero** trades in the validation lockbox.

### Why It Can't Be Done (Mathematically)

1. **Not enough data.** Our power analysis showed that even if a clean, stationary algorithmic rule existed, you'd need at least **1,600 independent trade events** to reliably identify it using our framework. We have 423. The math simply doesn't work.

2. **No exit rule found.** We tested every market-state variable we could measure against the exit timing. The combined model added only **0.034 bits of information per exit** beyond a simple age/calendar baseline. The overall statistical test returned p = 0.583 — completely insignificant. The tested market variables do not explain when positions close.

3. **Peak-distance (drawdown from best price) was the last hope.** It improved calibration slightly but still failed the acceptance threshold (KS test p = 0.010, needed p ≥ 0.05).

4. **"Discretionary human" is consistent with the data.** The operator claims the strategy is run by experienced human traders making judgment calls. Our analysis is entirely consistent with this claim. A human trader would naturally show clock preferences (they work certain hours), would trade when the market is active (tick rate), and might use a contrarian approach (buying dips) — but the exact moment-to-moment decisions would depend on screen reading, intuition, and context that no amount of historical data can capture.

---

## Final Verdict

> **Partial clock, tick-rate, and contrarian-direction components identified. No complete entry trigger or exit algorithm is recoverable from this ledger under any tested framework.**

This means:

- The strategy is **real and consistently profitable** (86.8% win rate, every month positive, 145% annual return on observed lot size).
- We can describe *roughly when* it tends to trade (active London/NY hours, elevated tick rate, contrarian to recent moves).
- We **cannot** reconstruct the exact decision logic that determines "trade now" vs "don't trade now."
- We **cannot** reconstruct the exit logic.
- Continuing to search for patterns in this same data would produce **false discoveries** (overfitting), not real insights.
- The only way to confirm or extend these findings would be with a **completely independent second ledger** (a different account, different time period) — which we do not have.

---

## What's in This Package

```
handoff_package/
├── README.md                          ← This file
├── data/
│   └── trades_raw.tsv                 ← All 423 canonical trades (tab-separated)
├── reports/
│   ├── CTPI_RECOVERABILITY_REPORT.md  ← Entry analysis final report
│   ├── CTPI_EXIT_REPORT.md            ← Exit investigation report
│   ├── CTPI_PEAK_DISTANCE_REPORT.md   ← Terminal peak-distance test
│   └── CTPI_NEXT_TRACKS_REPORT.md     ← Sample size power + replay
├── verdicts/
│   ├── recoverability_verdict.json    ← Machine-readable entry verdict
│   ├── exit_recoverability_verdict.json ← Machine-readable exit verdict
│   └── peak_distance_verdict.json     ← Machine-readable final test
├── evidence/
│   ├── direction_shifted_event_placebo.json
│   ├── direction_shifted_event_windows.csv
│   ├── sample_size_power_curve.json
│   ├── validated_component_replay.json
│   ├── lockbox_feature_tests.csv
│   └── peak_distance_model_comparison.csv
└── code_samples/
    ├── ctpi_immediate_execution.py    ← Entry analysis implementation
    ├── ctpi_direction_placebo.py      ← Direction placebo test
    ├── ctpi_next_tracks.py            ← Power curve + replay
    ├── ctpi_exit_investigation.py     ← Exit investigation
    └── ctpi_exit_peak_distance.py     ← Terminal peak-distance
```

### How to Read the Trade Data

The file `data/trades_raw.tsv` is a tab-separated file with these columns:

```
ticket_id  direction  open_time  close_time  symbol  lot_size  open_price  profit_usd
```

For example:
```
100829495  Sell  2026-09-17T16:21:20  2026-09-17T16:29:43  XAUUSD.f  0.01  4373.27  6.53
```

This trade opened a Sell position on gold at $4,373.27 on September 17, 2026 at 4:21 PM, closed it 8 minutes later, and earned $6.53 profit on a 0.01 lot.

---

## Key Code Evidence

Here is a condensed example of how we tested for entry patterns. This is the core of the contrarian direction test — the one pattern that actually showed statistical significance:

```python
# For each trade entry, compute price return in a window BEFORE the trade
# If return was negative (price dropped), and the trade was a Buy → contrarian
# If return was positive (price rose), and the trade was a Sell → contrarian
for window_minutes in [2, 5, 10, 15, 20]:
    results = []
    for epoch in canonical_epochs:
        pre_return = get_return_before(epoch.time, window_minutes, ticks)
        is_contrarian = (
            (pre_return < 0 and epoch.direction == "Buy") or
            (pre_return > 0 and epoch.direction == "Sell")
        )
        results.append(is_contrarian)

    accuracy = sum(results) / len(results)
    # Result: 57.9% to 61.5% contrarian accuracy (vs 50% random)
    # Max-statistic corrected p = 0.00050
```

And here is the exit investigation conclusion in code form:

```python
# Combined market-state model vs age/calendar baseline
# Result: information_gain = 0.0338 bits/exit (essentially zero)
# Lockbox adjusted p = 0.583 (totally insignificant)
# Time rescaling KS p = 0.000303 (calibration FAIL)
# Verdict: No tested market-state feature identifies the exit mechanism
```

---

## For the Skeptic

**"Maybe you just didn't test the right thing?"**
Possible, but unlikely to matter. We tested 192 candidate entry policies spanning threshold rules, conjunctions, finite state machines, clock models, point-process intensity models, and direction filters. On the exit side, we tested every observable market variable. After 116 passing unit tests and formal multiple-testing corrections, continuing the search would violate basic statistical principles — you'd start finding "patterns" that are just noise.

**"Could the traders be using information you can't see?"**
Absolutely. They might be reading news, watching order flow on the broker's platform, monitoring correlated assets, using proprietary indicators, or simply making gut calls based on decades of experience. None of that is observable in public tick data.

**"Is the strategy worth copying through CopyX?"**
That's an investment decision, not a technical one. The data shows a consistently profitable track record over 12 months with low per-trade risk. Whether that continues is unknowable. The 30% profit share is the cost of access. We could not find a way to replicate it without that cost.

---

## Technical Integrity

- All analysis was performed against cryptographically hashed canonical data
- 7.1M+ independent public tick quotes were used for market reconstruction
- Chronological lockbox methodology prevented data snooping
- All statistical tests used proper multiple-testing corrections
- 116 automated tests passed at the conclusion of the investigation
- The entire analysis codebase is available in the full repository at: https://github.com/shreyash1234566/xauusd-forensic-reconstruction

---

*Analysis completed November 2026. This document represents the frozen state of the investigation. No further feature mining on this ledger is scientifically justified.*
