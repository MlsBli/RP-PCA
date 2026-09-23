# RP-PCA Asset Pricing

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/[YOUR-GITHUB-USERNAME]/[REPO-NAME]/blob/main/rp_pca_asset_pricing.ipynb)

A Python (Google Colab) implementation of **Risk-Premium PCA (RP-PCA)** from:

> Lettau, M. & Pelger, M. (2020). *Factors That Fit the Time Series and Cross-Section of Stock Returns.* The Review of Financial Studies, 33(5), 2274–2325.
>
> Lettau, M. & Pelger, M. (2020). *Estimating Latent Asset-Pricing Factors.* Journal of Econometrics, 218(1), 1–31.

RP-PCA generalizes PCA by adding a penalty on cross-sectional pricing errors. Instead of eigen-decomposing the return covariance matrix, it decomposes

```
V = X'(I_T + γ·11'/T)X / T
```

where `γ` controls the weight on mean returns (`γ = -1` recovers standard PCA of the covariance matrix). This tilts the extracted factors toward the cross-section of expected returns, making the estimator far better at detecting "weak" factors with low variance but high Sharpe ratios.

## What the notebook does

1. **Data update (Section 1)** — downloads anomaly-sorted decile portfolio returns and firm characteristics from [Open Source Asset Pricing](https://www.openassetpricing.com/data/) and converts them to parquet in your Google Drive. Run once (and after each new data release).
2. **Data loading & signal selection (Section 2)** — lazily loads the data with Polars and selects the anomaly signals used as test assets (~180 signals; extreme deciles 1 and 10 by default).
3. **RP-PCA factor analysis (Section 3)** — fits RP-PCA with K = 5 factors by default (an Ahn–Horenstein (2013) eigenvalue-ratio diagnostic with a Marchenko–Pastur sanity check runs in the background and can be enabled to choose K automatically), and evaluates in-sample and rolling out-of-sample Sharpe ratios, pricing-error RMSEs, and unexplained variance.
4. **Portfolio construction (Section 4)** — maps the estimated SDF weights from characteristic-decile portfolio space to individual NYSE/NASDAQ stocks using a permno→ticker crosswalk, current market caps from yfinance, and decile re-assignment on current characteristic values.

## Quick start

1. Open the notebook in Google Colab (badge above).
2. Edit the **Configuration** cell (data path, γ, K, sample period, decile mode, etc.). All tunable parameters live there.
3. First run only: run **Section 1** to download the data into your Google Drive (~a few GB; requires authorizing Colab to access your Drive).
4. Run Sections 2–4 in order. Section 3 takes a few minutes (rolling out-of-sample loop); Section 4.2 (market-cap fetching) can take a while due to yfinance rate limits and is cached to Drive so it only needs to run once.

## Data

The data files are **not** included in this repository (they are several GB). The notebook pulls them from public sources:

| Data | Source |
|---|---|
| Anomaly decile portfolio returns (`PredictorAltPorts_DecilesVW`) | [Open Source Asset Pricing](https://www.openassetpricing.com/data/) (Chen & Zimmermann) |
| Firm-level characteristics (`signed_predictors_dl_wide`) | [Open Source Asset Pricing](https://www.openassetpricing.com/data/) |
| permno → ticker crosswalk | [Wenzhi Ding's standardized security codes](https://github.com/Wenzhi-Ding/Std_Security_Code) |
| Market caps & recent prices | [yfinance](https://github.com/ranaroussi/yfinance) |
| NASDAQ / NYSE ticker lists | [nasdaqtrader.com](https://www.nasdaqtrader.com/) |

Everything is stored in the Drive folder set by `DRIVE_DATA_DIR` in the Configuration cell.

## Configuration highlights

| Parameter | Default | Meaning |
|---|---|---|
| `GAMMA` | 10 | Risk-premium weight γ (paper's recommendation; `-1` = standard PCA) |
| `K_MANUAL` / `USE_AUTO_K` | K = 5, manual | Number of factors; set `USE_AUTO_K = True` to let the eigenvalue-ratio test choose instead |
| `DECILE_MODE` | `extreme` | Deciles 1 & 10 only (paper benchmark) or all 10 |
| `OOS_WINDOW` | 240 | Rolling out-of-sample window (months) |
| `START_DATE` | 1960-01-01 | Sample start |

## Implementation notes & caveats

- The estimator follows the paper's normalization conventions: sign normalization (factors positive on average), variance normalization (loadings scaled by √eigenvalues), and QR orthogonalization of factors. All are toggleable in the Configuration cell.
- Asset-space SDF weights are constructed as `FW[:, :k] @ w_sdf`, which reproduces the SDF exactly (`X @ w == SDF`) under any normalization setting.
- The eigenvalue-ratio diagnostic tends to suggest K = 1 when a dominant (market-like) factor is present, which is why K is set manually by default. Inspect the scree plot when choosing K (the paper's benchmark analyses use K ≈ 3–6).
- Reported Sharpe ratios are annualized from monthly (×√12); RMSEs are monthly.
- Mapping decile-portfolio weights to individual stocks assigns stocks to deciles by the 10%/90% quantiles of the *current* cross-section of each characteristic — an approximation of the original NYSE-breakpoint sorts used to build the portfolios.
- Missing portfolio returns: columns with >20% missing observations are dropped; remaining gaps are zero-filled to keep a balanced panel.
- This is an independent replication for research/educational purposes, not affiliated with the paper's authors. **Nothing here is investment advice.**

## Requirements

Designed for Google Colab (uses `google.colab` for Drive mounting/auth). See [requirements.txt](requirements.txt) for the package list; in Colab, everything except `polars`, `yfinance`, and `gdown` is preinstalled.

## References

- Lettau, M., & Pelger, M. (2020). Factors that fit the time series and cross-section of stock returns. *The Review of Financial Studies*, 33(5), 2274–2325.
- Lettau, M., & Pelger, M. (2020). Estimating latent asset-pricing factors. *Journal of Econometrics*, 218(1), 1–31.
- Ahn, S. C., & Horenstein, A. R. (2013). Eigenvalue ratio test for the number of factors. *Econometrica*, 81(3), 1203–1227.
- Chen, A. Y., & Zimmermann, T. (2022). Open source cross-sectional asset pricing. *Critical Finance Review*, 11(2), 207–264.
