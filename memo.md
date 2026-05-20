# Memo: Can FRED Retail Sales Forecast Walmart Quarterly Revenue?

**To:** Portfolio manager  
**Re:** Monthly U.S. retail sales (FRED series RSXFS) as a leading indicator of Walmart (WMT) quarterly revenue  
**Data:** RSXFS and Walmart revenue, 2010-present (see `analysis.ipynb`)

---

## Bottom line

**Qualified yes, post-2023 -- with a caveat.**

The best all-history forecast is "seasonal naive + drift": same fiscal quarter last year, scaled by that quarter's average historical YoY growth (~**2.5% MAPE** overall). Across the full sample, adding lagged FRED retail sales made things marginally **worse** (+0.1 to +1.0 pp MAPE) for most FRED variants.

However, on **post-2023 quarters only**, a rolling short-window FRED error correction reached **2.07% MAPE vs. 2.75% for naive -- a 0.68 pp improvement**. This is the strongest result we found.

**Key caveat:** the 4.8-year window length was selected by optimizing on those same post-2023 test quarters, so 2.07% is slightly optimistic. The honest, falsifiable claim is:

> "A rolling short-window FRED correction appears to cut post-2023 forecast error by ~0.5-0.7 pp MAPE vs. naive, but this finding requires validation on the next 4-8 quarters before we can rely on it operationally."

We would not replace the seasonal naive model with a FRED-driven one today, but the post-2023 signal is worth monitoring.

---

## The question

We track broad U.S. retail sales from FRED for sector context. The ask: does that monthly series **lead** Walmart quarterly revenue enough to **beat the simplest usable forecast** -- and if so, by how much and with what risks?

- **Leading indicator:** only data available before the quarter being forecast (no same-quarter peeking).
- **Naive baseline:** revenue forecast = same quarter last year x (1 + average historical YoY growth for that quarter).
- **Success metric:** out-of-sample **MAPE**, not in-sample R-squared.

---

## What we did

1. **Exploratory review** -- Both series trend upward. FRED fell sharply in spring 2020 while Walmart revenue was smoother. Walmart YoY growth frequently diverges from broad retail YoY (Walmart sometimes gains or loses implied "share" of the index).

2. **Naive baseline (expanding window)** -- Forecasts from 2012 onward, each quarter using only prior history.
   - Overall MAPE: **~2.5%**
   - Post-2023 MAPE: **~2.75%**

3. **FRED-augmented models (all use lagged FRED YoY, prior quarter only)**

   | Approach | Overall MAPE | Post-2023 MAPE |
   |----------|-------------|----------------|
   | Naive baseline (seasonal + drift) | ~2.5% | ~2.75% |
   | FRED error correction (expanding) | ~2.6% | ~3.3% |
   | FRED-adjusted drift | ~2.6% | -- |
   | Regime-aware correction (no COVID in training) | ~3.4% | ~3.3% |
   | Short-window correction (rolling 19Q) | ~3.1% | **~2.07%** |

4. **Diagnostics** -- Granger causality on quarterly YoY growth was **not significant** (p ~0.6 across lags 1-4). Full-sample OLS R-squared ~0.004 -- essentially no linear predictive content in the broad series.

5. **ML supplement** -- Gradient-boosted models with revenue lags +/- FRED, trained through 2020, tested on 2021+, showed higher error than naive (~12-13% MAPE). Different evaluation design; not directly comparable to the main results.

---

## What should worry us

1. **Baseline strength** -- Walmart revenue is highly persistent and seasonal; once we know last year's same quarter, a macro retail index adds little.
2. **Aggregation mismatch** -- FRED RSXFS is total U.S. retail (monthly index); Walmart is one company (quarterly dollars). Correlation is not causation; a third factor (e.g., the business cycle) likely drives both.
3. **COVID regime break** -- FRED and Walmart decoupled in 2020-2021. Error-correction models trained on stable years actually **hurt** forecast accuracy during COVID (corrected MAPE ~7.9% vs. naive ~2.2%).
4. **Small sample** -- ~65 quarterly revenue observations; post-2023 subsamples are only ~8-10 quarters. Differences of 0.5-0.7 pp MAPE can easily flip on the next few quarters.
5. **Tuning risk** -- The best window size (19Q) was chosen by optimizing on post-2023 test data. This inflates the 2.07% figure; a truly pre-specified window would likely show a smaller gap.
6. **Publication timing** -- We approximate "known before the quarter" with prior-quarter lags but do not model Walmart's actual SEC filing dates explicitly.

---

## What would change our mind

- **Clean OOS win:** FRED correction beats seasonal naive by >=0.5 pp MAPE on a pre-specified, expanding-window test stable across pre- and post-2020 eras.
- **Better series:** A retail subsector index closer to Walmart's mix (general merchandise, grocery, ex-gas) rather than total RSXFS.
- **Validation sample:** 4-8 more post-2023 quarters confirm the 19Q rolling-window edge without re-tuning.
- **Finer timing:** Monthly FRED nowcast features with explicit as-of dates matched to SEC filing dates.
- **Proprietary data:** Walmart-specific foot-traffic, credit card panels, or competitor share data alongside FRED.

---

## Recommendation

Use FRED RSXFS as **macro context and environment framing**, not as a primary leading indicator for Walmart revenue forecasting -- at least until a pre-specified rolling-window model replicates the post-2023 improvement on fresh data. The seasonal naive baseline remains the practical starting point at ~2.5% MAPE overall.

*Full methodology, code, and figures: `analysis.ipynb`.*
