# Prompt Log — YipitData Take-Home Exercise

**LLM assistant used:** GitHub Copilot (via Cursor) + Cursor AI (for data understanding)
**Total prompts:** 18

---

## Prompt Log

---

### Prompt 0a
> "What does the date value in retail_sales_fred.csv mean and what does the date value in walmart_revenue.csv mean?
>
> walmart_revenue.csv: 2010-01-31  113594000000 / 2010-04-30  99811000000
> retail_sales_fred.csv: 2010-01-01  302325 / 2010-02-01  302310
>
> How do I read this data? How should I understand it? What is the number 302325 exactly -- how did FRED end up with this number?"

**Context:** Before any modelling, I needed to understand what the raw values represent in each file so I could align them correctly and avoid unit errors.

**What the assistant explained:**

- **FRED (`retail_sales_fred.csv`):** The date is the first day of the reference month (a label, not a transaction date). The value is in **millions of USD** -- total estimated US retail sales for that month, seasonally adjusted. So `2010-01-01, 302325` = January 2010 US retail sales = **$302.325 billion**. FRED derives this from the Census Bureau's Advance Retail Sales survey, seasonally adjusted via X-13.

- **Walmart (`walmart_revenue.csv`):** The date is the **last day of Walmart's fiscal quarter** (not a calendar quarter). The value is in **raw USD**. So `2010-01-31, 113594000000` = **$113.594 billion** revenue for the quarter ending January 31, 2010.

- **Walmart's fiscal calendar** is offset: Q4 = Nov-Jan (ends Jan 31), Q1 = Feb-Apr (ends Apr 30), Q2 = May-Jul (ends Jul 31), Q3 = Aug-Oct (ends Oct 31). This means January is Walmart's highest-revenue quarter because it captures Black Friday and Christmas (November and December).

**What I verified:** I cross-checked the alignment by summing FRED's three months for Walmart's Q4 FY2010 (Nov 2009 + Dec 2009 + Jan 2010 ≈ $897B) and computed Walmart's implied share (~12.7%) -- which is plausible for the world's largest retailer. This confirmed the units and the fiscal-calendar mapping were correct before building the feature table.

**Key takeaway for the model:** When aggregating FRED monthly data to match Walmart quarters, the correct grouping is Nov-Jan, Feb-Apr, May-Jul, Aug-Oct -- not standard calendar quarters (Jan-Mar, Apr-Jun, Jul-Sep, Oct-Dec).

---

### Prompt 1
> "I've attached the pdf of my assessment. What should be my steps in defining a solid `naive_baseline`"

**Context:** First prompt. I attached the take-home PDF and asked for a structured approach to the naive baseline before writing any code.

**What the assistant got right:** It correctly identified that the naive baseline should be a seasonal one (same quarter last year), not a simple moving average, and highlighted that the baseline needs to be evaluated out-of-sample. It also flagged drift as an optional enhancement.

---

### Prompt 2
> "I see that there is around 3% MAPE for most of the years baseline. So I want you to add this 3% to my `naive_baseline` value"

**Context:** After running the first baseline loop, I saw ~3% MAPE and misread this as a correction to add to the forecast.

**Where I pushed back:** The assistant correctly clarified that adding a fixed 3% to the forecast would not reduce MAPE -- MAPE measures the error magnitude, not a directional bias. Adding a constant to the forecast can make things worse if the errors are not consistently positive. I accepted this correction and moved on.

---

### Prompt 3
> "Create a df of features with the following columns and this should align with my target variable which is rev['value']"

**Context:** I asked the assistant to build a merged feature table -- quarterly FRED aggregates, lagged revenue, YoY growth -- aligned with Walmart's quarter-end dates.

**What I verified:** I checked the index alignment manually (Walmart uses month-end dates; FRED quarterly timestamps are month-start). The assistant used `dt.to_period('Q').dt.to_timestamp()` to normalize both -- I confirmed the merged row counts matched expectations.

---

### Prompt 4
> "I am getting an empty df"

**Context:** The feature merge returned zero rows.

**Root cause (I caught it):** The assistant's initial code merged FRED quarterly timestamps (period-start: e.g. `2010-01-01`) directly against Walmart revenue dates (quarter-end: e.g. `2010-01-31`). The join key never matched. I identified the mismatch and asked for a period-normalized key on both sides. The fix worked.

**Lesson:** Always verify merge key types and date format before assuming a join is correct.

---

### Prompt 5
> "Use this code as reference to train a model which predicts my quarterly revenue"

**Context:** I provided reference code and asked the assistant to build an XGBoost model (Model A: revenue lags only; Model B: revenue lags + FRED).

**What the assistant got right:** It set up a clean train/test split (train <= 2020, test 2021+) and printed MAPE for both models.

---

### Prompt 6
> "This is my output: MAPE: 14.32% ... Naive Baseline MAPE: 2.65% ... Signal improvement over baseline: -11.66 pp. Now I want you to analyse and modify my features list as it contains lags and std of retail which is an exogenous feature. I want lags, std, YoY of the revenue itself in the features list"

**Context:** XGBoost was performing far worse than the naive baseline. I diagnosed that the feature list included same-quarter retail aggregates (potential look-ahead) and pushed the assistant to rebuild the features using only lagged and revenue-derived columns.

**Where I pushed back:** The assistant had initially included `retail_sales` (current quarter) as a feature alongside `rev_lag_1q`. I flagged that for a forecast of quarter Q, current-quarter retail is not known at forecast time. I enforced the lag rule explicitly.

---

### Prompt 7
> "Basically I have to improve my prediction signal using FRED data and using naive_baseline is a good reference to validate this improvement"

**Context:** Reframing the exercise goal clearly after the XGBoost detour -- the task is to show FRED adds value over the baseline, not just to build any model.

**What this achieved:** The assistant shifted focus to approaches that use FRED as an additive or corrective signal on top of the naive baseline (error correction, adjusted drift), rather than trying to replace it entirely with a tree model.

---

### Prompt 8
> "Naive Baseline MAPE: 2.65 ... Model A: 12.43 ... Model B: 13.22 ... I think tree model is not the right one for this as my `naive_baseline` which is a simple math equation is giving the best accuracy. What do you think and suggest other methods through which I can improve my signal by using FRED."

**Context:** Confirmed that XGBoost was the wrong tool for this small quarterly dataset (~65 rows). Asked for simpler, more interpretable FRED-based approaches.

**What the assistant got right:** It correctly diagnosed the problem -- too few training rows for gradient boosting, strong seasonal baseline, and likely overfitting on lags. It proposed OLS-based approaches: direct YoY regression, error correction on the naive residual, and drift adjustment.

---

### Prompt 9
> "Write code for each approach in a different cell."

**Context:** Asked the assistant to implement Granger causality, OLS regression, error correction, and adjusted drift as separate notebook cells.

**What I verified:** I checked that each approach used `retail_yoy_lag1` (not `retail_yoy`) to avoid look-ahead, and that the expanding-window loops did not accidentally include future data in training.

---

### Prompt 10
> "Explain this: [Granger Causality output -- all lags p > 0.23, not significant]"

**Context:** Granger test results showed no statistically significant predictive relationship between FRED YoY and Walmart revenue YoY at any lag (p = 0.88, 0.23, 0.37, 0.68 for lags 1-4).

**What I pushed back on:** The assistant's first explanation framed this as "FRED weakly leads at lag 2" because p=0.23 was the lowest. I corrected this -- p=0.23 is well above 0.05 and cannot be interpreted as a meaningful lead. None of the lags are significant. The assistant revised its interpretation to match the data.

---

### Prompt 11
> "[Final comparison table: Naive 2.51, Error Correction 2.55, Adjusted Drift 2.55, XGBoost 12.43+] These are my results. How do I improve my MAPE?"

**Context:** None of the initial FRED approaches beat the 2.51% naive MAPE. I asked what to try next.

**What the assistant suggested:** It proposed exploring regime awareness (exclude COVID from training), market-share dynamics (Walmart revenue as a proportion of FRED), and short rolling windows to capture the stabilizing post-COVID relationship.

---

### Prompt 12
> "I want to effectively use FRED data. I see FRED data as a pie out of which Walmart is trying to capture as much as possible. Now if my Walmart revenue has dipped but FRED has remained the same then there are other factors at play. I want to check the proportionality of Walmart data to FRED data. Write code to visualise this. I need a plot in a new cell. Don't combine the plots in one cell."

**Context:** My own insight about treating the Walmart/FRED ratio as a "market share" proxy. I asked for three plots: the raw ratio over time, YoY growth comparison, and a divergence bar chart.

**What I verified:** I confirmed the FRED units are an index (~300k-650k), not millions of dollars, so the ratio is not a true market share percentage -- it is a relative-scale ratio. The assistant labelled it "share %" which I flagged as misleading; we kept the visualization but treated it as a structural indicator rather than a literal share figure.

---

### Prompt 13
> "I see that trends are disrupted post-COVID (2020). In pre-COVID, FRED YoY growth was proportional to WMT revenue making it easier to predict. During COVID (2020-2022) the relation changed, more diverged. Post-COVID the curves are matching, divergence is lower and getting stabilised. Validate these observations and suggest what and how I can use this to improve my prediction signal."

**Context:** My own analysis of the divergence chart. I asked the assistant to validate my observation and suggest regime-based improvements.

**What the assistant got right:** It validated all three observations correctly (pre-COVID proportional, COVID disrupted, post-COVID re-converging) and proposed: (1) excluding COVID quarters from correction model training, (2) a short rolling window to weight recent stable history more, (3) a market-share mean-reversion signal.

---

### Prompt 14
> "Yes please."

**Context:** I confirmed I wanted all three strategies coded up.

---

### Prompt 15
> "[Final results table: Naive 2.51/2.75, Regime-Aware 3.43/3.32, Market Share 2.47/2.78, Short-Window 3.08/2.74] Market share and Short Window performed slightly better than naive_bsln. Suggest more data analysis which can draw insights to improve these two strategies."

**Context:** The FRED strategies were only marginally better post-2023. I asked for deeper analysis: optimal multipliers, window size sweep, and per-quarter breakdown.

**What the assistant got right:** It proposed grid searches on the multiplier and window parameters, and a per-quarter win-rate analysis. I accepted this but noted (in my own review) that running parameter optimization on post-2023 test data introduces look-ahead risk -- the best parameters are partly fitted to the test set.

---

### Prompt 16
> "[Window sweep results: Window 19Q = 2.07% best vs naive 2.75%] ... Since in the short-window strategy we are not using FRED data, I can replace the initial Naive calc with this strategy now. Or are we using FRED data in this strategy already? Also [Combined model output: post-COVID MAPE 2.18%, improvement 0.57 pp]. What does these outputs say?"

**Context:** Two questions at once: (a) does the short-window model actually use FRED? (b) how to read the combined model output.

**Where I caught an error:** I had misread the short-window strategy code as a pure revenue autoregression. The assistant clarified -- and I verified in the code -- that the short-window model is an **error correction** on the naive baseline using `retail_yoy_lag1` as a predictor of the naive residual. So FRED is being used. I confirmed this by tracing through Cell 22 in the notebook.

**On the combined model:** The post-COVID 2.18% MAPE with 0.57 pp improvement sounds strong, but both the share multiplier (0.0, effectively turning off share reversion) and the 19Q window were found by optimizing on post-2023 data. The combined model is mostly the short-window model with an inert share component. I flagged this in the memo.

---

## Reflection (< 200 words)

**What the assistant got right:**
GitHub Copilot was fast and accurate for boilerplate code -- feature engineering, expanding-window loops, OLS and XGBoost setup. It correctly warned against shuffled k-fold for time series and consistently used lagged FRED features rather than same-quarter values once I set that expectation. Its regime-analysis suggestions (COVID exclusion, rolling windows) matched what the data showed.

**Where I pushed back:**
Three places: (1) Prompt 2 -- it would have added a fixed 3% "correction" to forecasts, which is incorrect; I corrected it. (2) Prompt 10 -- it initially soft-pedalled a non-significant Granger result; I corrected the interpretation to "not significant at any lag." (3) Prompt 12 -- it labelled the Walmart/FRED ratio as a true "market share percentage"; I flagged the units issue (FRED is an index, not dollars).

**How I checked its output:**
I verified merge key alignment manually (date format mismatch caught in Prompt 4), traced look-ahead exposure in feature definitions (Prompt 6), confirmed the short-window model uses FRED by reading Cell 22 line by line (Prompt 16), and cross-checked all MAPE numbers against printed notebook outputs before including them in the memo.
