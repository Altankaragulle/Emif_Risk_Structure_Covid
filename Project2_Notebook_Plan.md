# Project 2 — Detailed Notebook Plan

**Title:** Has the structure of risk in financial markets changed since COVID-19?

**Authoring rules baked in:**
1. **Scope discipline (Concern #1).** Three core layers are mandatory; two are optional extensions cut first if time runs short.
2. **Literature integration (Concern #2).** Every empirical part has a literature anchor. Citations live where the methodology choice happens, not only in section 2.
3. **Diagnostic discipline (Concern #3).** Every estimated model is followed, in the same part, by a fixed sequence of diagnostic checks before any downstream step uses its output.

**Layer prioritization (locked):**
- **Mandatory core:** Levels (univariate GARCH), Co-movement (DCC-GARCH), Regimes (structural break tests on parameters)
- **Ambitious extension:** Risk Factors (PCA on standardized residuals)
- **Cut first if needed:** Risk Transmission (VAR on conditional volatilities)

---

## The Universal Diagnostic Protocol — applied after every model

This is the non-negotiable rule that fixes Concern #3. Every time a model is estimated, the *same* block of diagnostics runs immediately after, in the same notebook cell or in the cell directly following. If any check fails, you flag it and address it before any downstream step uses the model's output.

**For univariate GARCH models, the protocol is:**

1. **Convergence check.** Optimizer status `success=True`. If not, retry with different starting values; if still fails, document and try a simpler specification.
2. **Parameter validity.** ω > 0, α ≥ 0, β ≥ 0, and α + β < 1 (covariance stationarity). For GJR: also α + γ/2 + β < 1. Print the persistence α + β explicitly.
3. **Standardized residuals construction.** zₜ = εₜ / σₜ. These should be approximately i.i.d. with zero mean and unit variance.
4. **Ljung-Box on standardized residuals zₜ** — tests whether the *mean* equation captured all autocorrelation. Lags 5, 10, 20. Should not reject.
5. **Ljung-Box on squared standardized residuals zₜ²** — tests whether the *variance* equation captured all conditional heteroskedasticity. Lags 5, 10, 20. Should not reject. **This is the single most important diagnostic.**
6. **ARCH-LM test on residuals.** Confirms no remaining ARCH effects.
7. **Distribution of zₜ.** Jarque-Bera on standardized residuals. If using Student-t innovations, the degrees-of-freedom estimate ν should be reasonable (typically 4–15 for financial data).
8. **Visual check.** Q-Q plot of zₜ against the assumed innovation distribution.

**For DCC-GARCH, add:**
9. DCC parameters: a ≥ 0, b ≥ 0, a + b < 1 (correlation stationarity).
10. Conditional correlation paths fall in [-1, 1] (mechanical, but check).
11. Long-run correlation matrix Q̄ is positive definite.

**For structural break tests, add:**
12. Pre-period and post-period each have enough observations (rule of thumb: at least 10× the number of parameters per subsample).

If any check fails, the notebook prints a clear `[DIAGNOSTIC FAIL]` warning that you cannot miss when re-running.

A reusable function `run_garch_diagnostics(model_result, asset_name)` is built once in Part 0 and called everywhere. **Do not estimate any GARCH or DCC model without immediately calling this function.** This is the single rule that prevents the cascading-error problem from compounding through five layers.

---

## The 6 Selected Assets (locked)

| Asset | Risk role | Variable type |
|---|---|---|
| S&P 500 | US equity growth risk | Log-return |
| Eurostoxx 50 | European equity diversification | Log-return |
| US IG Bonds | Credit safety / 60-40 anchor | Log-return |
| US HY Bonds | Credit risk | Log-return |
| Gold | Alternative safe haven | Log-return |
| Oil futures | Real-economy commodity factor | Log-return |

Note: yields (`US T 10-year`, `German Gov 10-year`) are excluded from the core 6 to keep all variables on a comparable return scale. They appear in robustness only.

---

## Frequency choice (locked, justified once)

**Daily returns** are the working frequency for the entire project. Justification:
- GARCH and DCC are estimated on daily data in every Ielpo paper and every standard reference.
- Post-COVID sample on daily data ≈ 1,500 obs per asset, which is the minimum for credible DCC estimation.
- The professor's class materials use daily frequency throughout.

A short markdown cell at the start of Part 1 makes this explicit, with a citation to Bollerslev (1986) and Engle (2002).

---

## COVID Breakpoint (locked, justified)

**Cutoff date: 24 February 2020** (the first major equity sell-off week, S&P 500 −11% over the next 10 trading days).

Justification provided in the notebook:
- **Economic:** WHO declared global health emergency on 30 January 2020; US cases accelerated mid-February; the structural shock to risk markets begins late February.
- **Statistical:** A Quandt-Andrews supremum-Wald break test on a univariate GARCH for S&P 500 returns over a candidate window. The estimated break date confirms (or doesn't) the chosen point. **You must run this test and report the result.** If the data picks a different date (e.g. early March), use that instead and update the narrative.

**Sample split:**
- Pre-COVID: 1990-01-02 to 2020-02-24 (≈ 7,500 daily obs)
- Post-COVID: 2020-02-25 to 2026-04-24 (≈ 1,580 daily obs)

The COVID crash period (Feb–April 2020) is included in the post-COVID sample, not excluded. Excluding it would bias toward "no change" — exactly the wrong direction for the research question.

---

## Notebook Structure — 13 parts

The notebook has thirteen parts. Each part has:
- A short markdown header explaining the question being answered
- The code (cleanly commented)
- A markdown interpretation in your own voice (3–6 sentences, "we found X, which means Y for portfolio management")

Total expected length: 60–80 cells. About one third markdown, two thirds code.

---

### Part 0 — Setup and Helper Functions

**Goal:** One-time boilerplate. Imports, plotting style, and the diagnostic function used everywhere.

**Cells:**
- Imports (numpy, pandas, matplotlib, seaborn, scipy.stats, statsmodels, arch).
- Plotting defaults (figure size, grid, color cycle — keep it scientific, no fancy themes).
- The function `run_garch_diagnostics(result, asset_name, period_label)` that performs steps 1–8 above and returns a one-row pandas DataFrame summarizing pass/fail status.
- The function `summary_stats(returns)` returning mean (annualized), volatility (annualized), skewness, excess kurtosis, JB statistic, JB p-value, min, max, ADF p-value.

**Why this matters:** A pre-built diagnostic function means you cannot accidentally skip diagnostics. You either call it or you don't — and a missing call is visible at code review.

---

### Part 1 — Data Loading and Sample Construction

**Goal:** Load `Data.xlsx`, build the 6-asset return panel, document everything.

**Cells:**
- Read `Data.xlsx` with `pandas.read_excel`. The first column is the date.
- Print missing-value count per series and date range per series. Document any forward-filling logic.
- Subset to the 6 selected assets.
- Compute daily log-returns: `r_t = log(P_t / P_{t-1})`.
- Drop the first row (NaN from the differencing).
- Define `pre_covid` and `post_covid` masks using the locked breakpoint.
- Print observation counts per period per asset.

**Markdown to write:**
- Why these 6 assets (asset selection table with one-line rationale per asset).
- Why daily frequency.
- Why log-returns (additive over time, standard in GARCH literature).
- Why the COVID breakpoint.
- Sample split summary.

**Literature anchor:** Cite Bollerslev (1986) for GARCH on daily data and Engle (2002) for DCC.

---

### Part 2 — Stylized Facts and Pre/Post Comparison (Descriptive)

**Goal:** Establish that the data exhibits the three stylized facts (volatility clustering, fat tails, leverage) and provide the headline pre/post numbers that motivate the rest of the analysis.

**Cells:**
- Time series plot of all 6 return series (one chart per asset, 6 subplots).
- `summary_stats` table for each asset, full sample.
- Same table split by pre-COVID and post-COVID.
- Annualized volatility comparison bar chart (pre vs post).
- Static Pearson correlation matrices: full sample, pre-COVID, post-COVID. **Side-by-side heatmap.**
- A "correlation change" matrix: post − pre. Highlight the equity-bond flip and credit convergence.
- ACF of returns and squared returns (one chart per asset, two panels each). Stylized facts: ACF of returns ≈ 0; ACF of squared returns large and persistent → volatility clustering.
- Jarque-Bera test on each asset, full sample.

**Markdown to write:**
- Three findings explicitly stated:
  1. The equity-bond correlation flip (S&P500 / US IG: −0.12 → +0.17).
  2. Credit convergence (S&P500 / US HY: 0.24 → 0.57).
  3. The volatility paradox (equity vol falls slightly post-COVID, even though crisis was extreme).
- Explicit statement that all 6 series reject normality (JB) and exhibit volatility clustering (ACF of r²).
- This motivates GARCH for the next part.

**Literature anchor:** Cont (2001) for stylized facts. Baur & Lucey (2010) for the gold/equity hypothesis. Cite the institutional/practitioner observation of the equity-bond flip post-2020.

---

### Part 3 — Univariate GARCH (Layer 1: Risk Levels)

**Goal:** Estimate volatility models for each asset, pre and post separately, and test whether parameters changed.

**Sub-structure:**

#### 3.1 Model selection rationale

Markdown explaining:
- We use **GJR-GARCH(1,1)** as the base specification because it captures the leverage effect (Stylized Fact #3 from Class 9).
- We use **Student-t innovations** rather than Gaussian because the JB tests in Part 2 reject normality, and because Chorro, Guégan & Ielpo (2012) show that the conditional distribution matters more than the volatility specification.
- The mean equation is constant (no AR term unless the data demands it — test with Ljung-Box on raw returns).

The full specification:
```
r_t = μ + ε_t
ε_t = σ_t z_t,    z_t ~ Student-t(ν)
σ²_t = ω + α ε²_{t-1} + γ I[ε_{t-1} < 0] ε²_{t-1} + β σ²_{t-1}
```

#### 3.2 Estimation, full sample

For each of the 6 assets:
- Estimate GJR-GARCH(1,1) with Student-t innovations using the `arch` package.
- Print parameter estimates with standard errors.
- **Run the diagnostic protocol immediately.**
- Plot conditional volatility σₜ over time.

#### 3.3 Estimation, pre-COVID sample

Same as 3.2 but on the pre-COVID subsample only. Store all six models in a dictionary `garch_pre`.

#### 3.4 Estimation, post-COVID sample

Same as 3.2 but on the post-COVID subsample only. Store in `garch_post`.

#### 3.5 Parameter comparison table

Build a table with one row per asset, columns for each parameter (ω, α, γ, β, ν) shown pre and post side-by-side, plus the t-statistic for the difference (using delta-method standard errors or, simpler, just report whether the post estimate is outside the pre 95% CI).

Also compute and report:
- **Persistence** (α + β + γ/2): how long shocks last.
- **Unconditional volatility** σ̄ = √(ω / (1 − α − β − γ/2)): the long-run risk level.
- **Half-life of volatility shocks** = log(0.5) / log(α + β + γ/2): how many days for shocks to decay by half.

#### 3.6 Interpretation

Markdown of 8–12 sentences answering: did volatility levels change? Did persistence change? Did the leverage effect strengthen or weaken? Which assets show the biggest parameter shifts? What does this mean for risk management?

**Literature anchor:** Glosten, Jagannathan & Runkle (1993) for GJR. Bollerslev (1987) for Student-t GARCH. **Chorro, Guégan & Ielpo (2012, 2018)** for the leverage-vs-distribution argument — cite this prominently and connect it to your model choice.

---

### Part 4 — DCC-GARCH (Layer 2: Risk Co-movement)

**Goal:** Test whether the *correlation* structure changed, beyond what static correlations show.

**Sub-structure:**

#### 4.1 Why DCC and what it adds

Markdown explaining:
- Static correlations are biased when computed on volatility-conditional samples (Forbes & Rigobon 2002).
- DCC separates the volatility dynamics (univariate GARCH per asset) from the correlation dynamics, which is estimated in a second step.
- The DCC equation: Q_t = (1−a−b) Q̄ + a (z_{t-1} z'_{t-1}) + b Q_{t-1}
  - Then the correlation matrix Γ_t = diag(Q_t)^{-1/2} Q_t diag(Q_t)^{-1/2}
- The parameters a (reactivity) and b (persistence) are the correlation-space analogs of α and β.

#### 4.2 Estimation, full sample

Estimate a 6-variable DCC-GARCH using the `mgarch` package or via two-step manual estimation (univariate GARCHs from Part 3 + DCC second step in pure numpy).

If the `mgarch` package is unstable on 6 series, fall back to **DCC on 4 assets**: S&P500, US IG, Gold, Oil. Document the choice. The 60-40 dynamic and the gold safe-haven hypothesis are both still answered with these 4.

Run the DCC diagnostic protocol.

#### 4.3 Estimation, pre and post separately

Estimate DCC on pre-COVID and post-COVID samples.

Compare:
- a_pre vs a_post (correlation reactivity)
- b_pre vs b_post (correlation persistence)
- Q̄_pre vs Q̄_post (long-run correlation matrix)

#### 4.4 Time-varying correlation paths

Plot the conditional correlation between key pairs over time, with a vertical line at the COVID breakpoint:
- S&P500 / US IG (the equity-bond flip)
- S&P500 / US HY (credit convergence)
- S&P500 / Gold (safe-haven test)
- Oil / S&P500 (commodity-equity link)

#### 4.5 Diversification metrics

For each pair, compute the average pre and post conditional correlation. Report a 2×2 "diversification benefit" matrix:
- Average correlation pre vs post
- Frequency of correlation > 0.5 pre vs post (correlation breakdown indicator)

#### 4.6 Interpretation

Markdown answering: did the correlation structure change? Where most? What does this mean for the 60-40 portfolio? Does gold still diversify equities?

**Literature anchor:** Engle (2002) for DCC. Engle & Sheppard (2001) for DCC testing. Forbes & Rigobon (2002) for the conditioning bias. Baur & Lucey (2010) for gold-as-safe-haven, with explicit comparison: "their finding holds pre-COVID; we test whether it survives."

---

### Part 5 — Structural Break Tests (Layer 3: Risk Regimes)

**Goal:** Move from "pre vs post comparison looks different" to "we formally reject parameter constancy at the COVID date."

**Sub-structure:**

#### 5.1 The Quandt-Andrews / Bai-Perron framework

Markdown explaining: a Chow test asks whether parameters changed at a *known* date. A Quandt-Andrews supremum-Wald test asks whether parameters changed at an *unknown* date in a candidate window — giving the statistically optimal break date.

#### 5.2 Test on the conditional volatility series

For S&P500, US IG, Gold, Oil — apply a Bai-Perron multiple breakpoint test on the conditional volatility series σₜ from Part 3. The test should detect a break around late February / early March 2020.

#### 5.3 Test on parameter constancy directly

For each asset, run a Hansen (1992) parameter constancy test on the GARCH parameters. This is what Chorro, Guégan & Ielpo (2014) use for the leverage parameter γ.

If the Hansen test is too complex to implement, a credible substitute is the *likelihood ratio test* comparing:
- Restricted model: GARCH parameters constant across pre and post
- Unrestricted model: separate GARCH parameters for pre and post
The test statistic is 2(LL_unrestricted − LL_restricted) ~ χ²(k), where k is the number of free parameters.

#### 5.4 Test on DCC parameters

LR test on whether (a, b, Q̄) changed pre to post. This is the cleanest direct answer to the research question.

#### 5.5 Interpretation

For each layer (volatility, leverage, correlation), state whether the structural break is statistically significant and at what level.

**Literature anchor:** Andrews (1993) for sup-Wald. Bai & Perron (1998, 2003) for multiple breaks. Hansen (1992) for parameter constancy. Chorro, Guégan & Ielpo (2014) for the application to GARCH.

---

### Part 6 — PCA on Standardized Residuals (Layer 4: Risk Factors) [EXTENSION]

**Goal:** Test whether risk became more concentrated in fewer factors post-COVID — the "everything moves together" hypothesis.

**Sub-structure:**

#### 6.1 Why PCA on GARCH residuals (not on returns)

Markdown explaining: PCA on raw returns mixes volatility differences with correlation differences. PCA on standardized residuals zₜ = εₜ/σₜ isolates the correlation structure cleanly.

#### 6.2 PCA pre, PCA post

For both subsamples, compute zₜ for all 6 assets, build the standardized-residual matrix, and run PCA.

Report:
- Variance explained by PC1, PC2, PC3 (pre vs post).
- Loadings of PC1 (which assets dominate the first factor?).
- The change in PC1 explanatory power post − pre.

#### 6.3 Interpretation

If PC1 explains, say, 35% pre and 50% post, you have a clean factor-concentration result. The first factor became dominant.

**Literature anchor:** Connor (1995) for factor models. The recent literature on "risk-on/risk-off" market regimes.

**Cut policy:** This is the first thing cut if Parts 3–5 take longer than expected.

---

### Part 7 — VAR on Conditional Volatilities (Layer 5: Risk Transmission) [EXTENSION]

**Goal:** Test whether volatility *spillovers* across asset classes changed direction or intensity post-COVID.

**Sub-structure:**

#### 7.1 Build the conditional volatility panel

Take σₜ from Part 3 for the 6 assets. Stack into a 6-column daily matrix.

#### 7.2 Estimate VAR pre and post

For each subsample:
- Lag selection via AIC/BIC.
- VAR estimation.
- Diagnostic protocol (residual autocorrelation, stability of companion matrix).
- Granger causality tests for all pairs.
- Forecast Error Variance Decomposition (FEVD) at horizons 1, 5, 22 days.

#### 7.3 Spillover index

Compute Diebold-Yilmaz (2009) spillover index pre and post. A higher post-COVID index = more spillover = volatility transmits faster across asset classes.

#### 7.4 Interpretation

Did the direction of spillovers change? Did total spillover increase?

**Literature anchor:** Diebold & Yilmaz (2009, 2012) for the spillover index.

**Cut policy:** First to cut. Parts 3–5 alone are sufficient for an A-grade answer.

---

### Part 8 — Robustness Checks

**Goal:** Show your results are not artifacts of arbitrary choices.

**Sub-structure:**

#### 8.1 Alternative breakpoint date

Re-estimate Part 3–4 with the breakpoint at 11 March 2020 (WHO pandemic declaration). Show the headline parameter changes are similar.

#### 8.2 Alternative GARCH specification

Re-estimate Part 3 with EGARCH instead of GJR-GARCH. Confirm leverage effect direction is consistent.

#### 8.3 Alternative innovation distribution

Re-estimate Part 3 with skewed Student-t. Compare AIC/BIC — does the skewness parameter add explanatory power?

#### 8.4 Excluding the COVID crash period

Re-estimate Part 3–4 with the post-COVID sample starting in 1 May 2020 (after the worst crash days). Check whether structural-change conclusions survive when the most extreme period is removed. **This is the strongest robustness check** — if conclusions hold without the crash period, the structural change is genuine, not a crash artifact.

---

### Part 9 — Portfolio Implications (the Ielpo grading lever)

**Goal:** Translate every statistical result into a portfolio-management implication. This is what separates an A from an A+.

**Sub-structure:**

#### 9.1 The 60-40 portfolio test

Compute the conditional correlation between S&P500 and US IG bonds in pre and post. Show that the diversification benefit of the 60-40 portfolio is mathematically smaller post-COVID. Quantify the increase in portfolio variance under the post-COVID correlation regime.

#### 9.2 Gold as a diversifier

Repeat 9.1 with gold replacing or complementing IG bonds. Comment on whether gold absorbs the diversification role bonds lost.

#### 9.3 Risk monitoring frequency

If DCC parameter `a` (reactivity) increased post-COVID, the half-life of correlation shocks shortened. Quantify what this means for rebalancing: a portfolio rebalanced quarterly pre-COVID may now need monthly rebalancing to maintain the same correlation accuracy.

#### 9.4 VaR comparison

Compute 1-day 99% Value-at-Risk for an equal-weight portfolio of the 6 assets, pre and post, using the GARCH-t conditional volatility. Report the average VaR and the maximum VaR. Show whether tail risk increased.

**Literature anchor:** This section uses the Ielpo papers prominently — his entire research agenda is about converting econometric results into portfolio decisions.

---

### Part 10 — Conclusion and Limitations

**Goal:** A clear yes/no/partially answer with three sentences per layer.

**Structure:**
- The clear answer (one paragraph): "yes, the structure of risk changed in three of the five dimensions we tested. Specifically..."
- Layer-by-layer summary table: For Levels, Co-movement, Factors, Transmission, Regimes — what changed, by how much, statistical significance.
- Limitations: known weaknesses of the methodology.
- Suggestions for future work.

---

### Part 11 — Reproducibility Footer

A short final cell with:
- The package versions (`numpy.__version__`, etc.).
- The seed used for any stochastic procedure.
- A line that says "all results in the report are reproduced from this notebook by running cells in order from a fresh kernel."

---

## How the Three Concerns Are Addressed Throughout

### Concern #1 — Scope creep

**Mitigation built into the structure:** Parts 3, 4, 5 are mandatory (the core three layers). Parts 6 and 7 are explicitly tagged `[EXTENSION]`. If by the time you finish Part 5 you have less than two weeks before the deadline, **delete Parts 6 and 7 entirely** and route their findings (if any) into Part 9 as side comments. Do not let extensions cannibalize the writing time for the report.

A working budget rule:
- Parts 0–2: 3 days (data, descriptive)
- Part 3: 4 days (the heart of the project — do not rush)
- Part 4: 4 days
- Part 5: 2 days
- Parts 6–7 (if pursued): 4 days
- Part 8: 2 days
- Part 9: 2 days
- Part 10: 1 day
- Report writing: at least 7 days, ideally 10
- Buffer: 3 days for things going wrong

That's roughly 32–35 working days. With a 31 May deadline and starting now, this is tight but doable. **The single biggest mistake would be spending three weeks on Part 6 because PCA is fun, and then having no time for the report.** Pre-commit now to the cut policy.

### Concern #2 — Literature integration

**Mitigation built into the structure:** Every part has an explicit "Literature anchor" line. The literature review section of the report (Section 2) is built by harvesting these anchors and writing the three-sentence treatment for each: what the paper studied, what it found, why it motivates your choice.

The literature review is therefore not a separate writing exercise at the end. It is assembled from notes taken throughout the notebook work.

Required citations (about 12 papers — comfortable for a 2.5–3 page review):
- **GARCH foundations:** Bollerslev (1986), Bollerslev (1987), Engle (1982).
- **Asymmetric volatility:** Glosten, Jagannathan & Runkle (1993), Nelson (1991), Ding, Granger & Engle (1993).
- **Multivariate / DCC:** Engle (2002), Engle & Sheppard (2001).
- **Ielpo's research:** Chorro, Guégan & Ielpo (2012), Chorro, Guégan & Ielpo (2014, leverage paper). **These two are non-negotiable.**
- **Stylized facts:** Cont (2001).
- **Crisis/correlation breakdown:** Forbes & Rigobon (2002), Baur & Lucey (2010), Diebold & Yilmaz (2012).
- **Structural breaks:** Andrews (1993), Bai & Perron (1998), Hansen (1992).

When writing the report, each empirical paragraph in Section 5 (Empirical Results) should reference at least one paper. "Consistent with Engle (2002), our DCC parameters..." not "our DCC parameters..."

### Concern #3 — Diagnostic discipline

**Mitigation built into the structure:** The `run_garch_diagnostics` function in Part 0 is the engineering solution. Every model estimation is followed by a function call. The function prints either `[ALL CHECKS PASS]` or `[DIAGNOSTIC FAIL]` with the specific failed check. You cannot accidentally skip diagnostics — the only way to skip is to manually delete the function call, which is a visible action.

A second protective layer: at the end of Part 3, before moving to Part 4, build a single summary table:

| Asset | Period | Convergence | LB(z) | LB(z²) | ν estimate | α+β+γ/2 |
|---|---|---|---|---|---|---|
| S&P500 | Pre | ✓ | ✓ | ✓ | 7.2 | 0.978 |
| S&P500 | Post | ✓ | ✓ | ✓ | 6.5 | 0.985 |
| ... | | | | | | |

This table is included in the appendix of the report. It tells the professor at one glance that you ran the diagnostics. Without it, he assumes you didn't.

A third protective layer for Part 4: DCC inherits univariate GARCH residuals. If a univariate diagnostic failed in Part 3, the DCC model in Part 4 is contaminated. The notebook should include an explicit gate: `assert all_garch_diagnostics_passed`, raising an error if any pre-step failed. This is the single line of code that prevents the cascading error.

---

## Final Pre-Coding Checklist

Before opening Cursor / VS Code:
- [ ] You have agreement from your group on the 6-asset selection.
- [ ] You have agreement on the COVID breakpoint date.
- [ ] You have agreed on the cut policy (Parts 6, 7 are optional).
- [ ] You have decided who writes which sections of the report.
- [ ] You have set a hard internal deadline 5 days before 31 May for "all empirical work done, report writing only."

When you open the notebook:
- [ ] Part 0 first. Do not skip ahead even if it feels boring.
- [ ] Run cells in order from a fresh kernel every time you re-open the notebook for a new session.
- [ ] After each estimation, look at the diagnostic output. Do not commit code that prints `[DIAGNOSTIC FAIL]` and proceed regardless.
- [ ] Write the markdown interpretation immediately after each estimation, while the result is fresh. **Do not leave interpretation for "later" — later is when you forget what you found.**

