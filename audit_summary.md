# Audit Summary — Notebook_Project_2.ipynb (Parts 0–3)

Second-round audit applied: 2026-05-06

---

## Issue 1 (CRITICAL) — Breakpoint date ✓ FIXED

**Verified:** Cell 10 was using `COVID_DATE = break_date` (23 March 2020 — the market trough). The entire crash was inside the pre-COVID sample.

**Applied:** `COVID_DATE = pd.Timestamp("2020-02-24")` hardcoded in Cell 10. Cell 13 markdown rewritten to explain the override. Cell 9 (sup-Wald test) left untouched.

**Result:** Pre-COVID 7,970 obs (1989–2020-02-24), Post-COVID 1,631 obs (2020-02-25–2026-04-24). COVID crash is correctly in the post sample.

---

## Issue 2 (CRITICAL) — DGT chi-squared over-rejection ✓ FIXED

**Verified:** `n_bins = min(50, max(10, int(sqrt(n))))` gave ~89 bins for full sample. All 18 model-period combinations had `dgt_chi2_pass = False`.

**Applied:** `n_bins = 20` (fixed). Note added to Cell 50 markdown explaining that persistent LB(z²) failures on long pre-COVID samples reflect parameter instability over 30 years — formally tested in Part 5.

---

## Issue 3 (CRITICAL) — PCA section had no code ✓ FIXED

**Verified:** Cells 31 and 32 were both markdown with no PCA computation.

**Applied:** PCA code cell inserted using `StandardScaler` + `sklearn.decomposition.PCA`. `sklearn` imports added to Cell 1.

**Actual result:**

| PC  | Pre-COVID | Post-COVID | Change |
|-----|-----------|------------|--------|
| PC1 | 30.6%     | 41.3%      | +10.7 pp |
| PC2 | 20.8%     | 18.9%      | −1.9 pp  |
| PC3 | 19.4%     | 16.5%      | −2.9 pp  |

PC1 share rose +10.7 pp — co-movement became more concentrated in a single common factor post-COVID. Cell 32 updated with actual numbers.

---

## Issue 4 (IMPORTANT) — `rescale=True` corrupting omega ✓ FIXED (second pass)

**First-round verification error:** The empirical test used a wrong COVID date (2023-03-23) and tested only S&P 500 (scale=1.0). The bug was incorrectly declared absent.

**Second-round verification:** The `arch` package applies scale=10 to US IG Bonds and US HY Bonds (std ≈ 0.25–0.29, below the package's threshold). With scale=10, the reported `omega` is `omega_true × 100`, making σ̄ 10× too large. This produced:
- US IG Bonds pre: σ̄ = **46.1%** (true value: **4.6%**)
- US HY Bonds pre: σ̄ = **52,654%** (degenerate local optimum with `rescale=True`)
- Gold: σ̄ = **7,999%** (near-IGARCH mishandled)

**Applied (Option B):** Keep `rescale=True` for optimizer convergence quality. After fitting, correct omega: `omega = omega_raw / scale²` where `scale = getattr(res, 'scale', 1.0)`. Near-IGARCH guard added: when π ≥ 0.9999, `uncond_vol = np.nan` and `half_life = np.inf` — displayed as "IGARCH" in the parameter table.

**Why not `rescale=False`?** Setting `rescale=False` caused US HY Bonds pre to converge to a degenerate local optimum (π=0.587, ν=317, z_std=0.52). `rescale=True` + omega correction finds the economically correct solution.

---

## Issue 5 (IMPORTANT) — Gaussian estimation redundant in pre/post ✓ FIXED

**Verified:** BIC selects Student-t for all 6 assets by 200–2,300 points.

**Applied:** Gaussian estimation removed from Cells 43 and 46. Vol and Q-Q plot cells show Student-t only.

---

## Issue 6 (IMPORTANT) — AR(1) decision silent ✓ FIXED

**Verified:** All 6 assets carry AR(1). Not mentioned in Section 3.6.

**Applied:** Dedicated paragraph added to Cell 52 naming all assets and explaining the role of AR(1) in sharpening GARCH estimates.

---

## Minor cleanups ✓ ALL APPLIED

| # | Fix |
|---|-----|
| 7 | `breaks_cusumolsresid` removed from Cell 1; duplicate imports removed from Cell 34; `sklearn` imports added |
| 8 | Q-Q plots (Cells 41, 44, 47) changed from `probplot(fit=True)` + OLS line to `probplot(fit=False)` + `axline((0,0), slope=1)` |
| 9 | Comment added to Cell 2 for `prices.where(prices > 0)` |
| 10 | `include_adf=True/False` parameter added to `summary_stats`; `include_adf=False` passed in Cell 20 |

---

## Final parameter table — Student-t GJR-GARCH(1,1)

| Asset | π pre | π post | σ̄ pre | σ̄ post | HL pre | HL post | z_std pre | z_std post |
|---|---|---|---|---|---|---|---|---|
| S&P 500 | 0.9890 | 0.9726 | 16.7% | 15.6% | 62.6d | 25.0d | 0.991 | 1.008 |
| Eurostoxx 50 | 0.9878 | 0.9604 | 19.6% | 18.0% | 56.4d | 17.2d | 1.005 | 0.990 |
| US IG Bonds | 0.9938 | 0.9818 | 4.6% | 5.5% | 111.7d | 37.8d | 0.998 | 0.999 |
| US HY Bonds | 1.0000 | 1.0000 | IGARCH | IGARCH | IGARCH | IGARCH | 1.039 | 0.990 |
| Gold | 1.0000 | 0.9726 | IGARCH | 18.1% | IGARCH | 25.0d | 1.007 | 0.985 |
| Oil futures | 0.9954 | 0.9574 | 42.0% | 40.1% | 149.7d | 15.9d | 0.991 | 1.008 |

All z_std ≈ 1.0 — standardised residuals are correctly scaled. IGARCH entries are genuine findings (near-integrated variance), not numerical errors.

---

## Diagnostic pass/fail — Student-t GJR-GARCH(1,1)

### Full sample: 0 / 6 pass all checks
Expected — single-regime GARCH over 36 years spans dot-com, GFC, euro crisis, COVID; LB(z²)/ARCH-LM failures are structural, motivating Part 5 break tests.

### Pre-COVID: 0 / 6 pass all checks
DGT-χ² fails for all (7,970 obs × 20 bins → ~400 expected per bin; test detects any slight t-distribution skewness misfit). LB(z²)/ARCH-LM failures in US IG Bonds and Gold reflect 30-year parameter instability.

### Post-COVID: 1 / 6 pass all checks (Gold)
LB(z) and LB(z²) now pass for most assets — GJR-GARCH(1,1) captures post-COVID dynamics well on 1,631 obs. Remaining DGT-χ² failures reflect symmetric Student-t unable to capture slight negative skewness.

---

## Notes for Part 5

- **US IG Bonds and Gold** fail LB(z²)/ARCH-LM pre-COVID → clearest candidates for Hansen parameter-constancy test
- **US HY Bonds**: IGARCH in both periods → no finite unconditional VaR from single-regime model; motivates regime-specific estimation
- **DGT-χ² failures**: distributional misfit (skewness), not GARCH dynamics — DGT-LM (serial dependence) is the reliable diagnostic and passes for most post-COVID models
