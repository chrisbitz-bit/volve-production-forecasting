# Volve Production Forecasting: Decline Curve Analysis vs Machine Learning

Probabilistic oil rate forecasting (P10/P50/P90) using Equinor's public **Volve** field dataset (Norwegian North Sea, production 2008 to 2016).

This study addresses two questions:

1. **Can machine learning models outperform Arps decline curve analysis (DCA) for 12-month oil rate forecasting?**
2. **Where does machine learning provide measurable value in production engineering workflows?**

All models are evaluated under an identical out-of-sample protocol: each forecast uses only data available before its forecast origin and is scored against subsequently observed production.

![Test-year forecasts for wells F-11 and F-12](figures/02_where_ml_failed_F11_F12.png)

## Data

- Source: Equinor Volve daily production data, 15,634 daily well records (Sep 2007 to Dec 2016).
- Scope: 5 oil producers (15/9-F-1 C, F-11, F-12, F-14, F-15 D); 2 water injectors (F-4, F-5) used as pressure-support inputs in Part 2.
- Preprocessing: daily records aggregated to weekly averages; oil rate normalized by on-stream hours; weeks with fewer than 48 producing hours excluded from the target; rate step-changes (restarts, workovers) detected automatically and encoded as features.

## Part 1: 12-month rate forecasting

Three methods, each producing P10/P50/P90 quantiles:

| Method | Description |
| --- | --- |
| Arps decline curve analysis | Hyperbolic decline fitted in log space from the most recent peak or detected restart; bootstrap resampling for uncertainty |
| Quantile gradient boosting | `HistGradientBoostingRegressor` with quantile loss; inputs: lagged log-rates, water cut and its trend, choke opening, wellhead pressure, time on stream, time since last restart, forecast horizon |
| Temporal Fusion Transformer (TFT) | Implemented in PyTorch: variable selection networks, gated residual networks, LSTM encoder–decoder, interpretable multi-head attention, quantile loss; 3-seed ensemble |

Prediction intervals were calibrated by split-conformal adjustment on a validation year (Oct 2014 to Sep 2015) and evaluated on a held-out test year (Oct 2015 to Sep 2016).

**Held-out test year (5 wells, 52 weeks)**

| Method | Weekly MAPE | Cumulative volume error | P10–P90 coverage, calibrated (nominal 80%) |
| --- | --- | --- | --- |
| Arps DCA | 26.7% | -18.1% | 58% |
| Quantile gradient boosting | 40.4% | +22.9% | 74% |
| Temporal Fusion Transformer | 44.3% | +39.3% | 65% |

**Rolling-origin backtest.** All models were re-trained and re-evaluated from 25 quarterly forecast origins (Oct 2009 to Sep 2015). Neither ML model outperformed Arps DCA with statistical significance: the block-bootstrap 95% confidence interval of the paired difference in mean absolute log error includes zero for both. All three methods significantly outperformed a persistence (flat-rate) baseline.

**Interpretation.** The largest ML errors coincided with rapid water breakthrough. On F-11 and F-12, water cut increased toward 80–90% during the test year. Arps DCA extrapolated the established decline, whereas both ML models projected a rate plateau that did not materialize, likely reflecting rate recoveries after earlier restarts in the training data.

![Rolling-origin backtest across 25 forecast origins](figures/03_rolling_backtest_25_dates.png)

## Part 2: Applications where ML adds value

All results below use the same rolling-origin, out-of-sample protocol.

| Application | ML performance | Reference method |
| --- | --- | --- |
| 4-week oil volume forecast (198 forecasts) | Median absolute error 5.7%; outperforms Arps in 68% of paired forecasts | Arps DCA 9.0%; persistence 7.6% |
| Virtual flow meter (VFM), liquid rate | WAPE 17.0%; 21.5% on a well excluded from training | Choke equation 41.3% (37.1% on the excluded well) |
| 52-week water-cut forecast | Mean absolute error 7.3 percentage points | Logit trend 11.3; persistence 9.2 |
| 12-month volume, Arps–GB ensemble (equal-weight average) | 15.5% mean absolute volume error | Arps DCA 17.8% (difference not statistically significant) |

Additional components:

- **Physics-informed constraints:** monotonic constraints ensure forecasts are non-increasing with horizon and with water-cut trend.
- **Anomaly detection:** VFM residual thresholds combined with an isolation forest over 7,609 well-days of sensor data.
- **Model interpretability:** permutation importance, partial dependence and SHAP values.
- **MLOps simulation:** monthly retraining (Jan 2015 to Jul 2016) with a versioned model registry, input data-drift monitoring and a forecast-error alert threshold of ±20%.

![Virtual flow meter performance](figures/05_virtual_flow_meter.png)

## Limitations

- **Sample size.** Five producing wells limit statistical power; differences among the 12-month forecasting methods are not significant.
- **VFM scope.** The VFM is reliable for total liquid rate only. Oil-rate estimates underperformed a stale well test, because no sensor measures water cut directly.
- **Well-test frequency.** With monthly well tests, holding the last test rate (7.2% WAPE) outperforms the recalibrated VFM (9.0%). The VFM becomes advantageous when test intervals extend to 3–6 months.
- **Decline exponent.** The fitted Arps b-factor reached a bound on 4 of 5 wells: approximately 0 (exponential decline) on three wells with high water cut, and 1.0 (harmonic) on two wells with a restart less than 13 weeks before the forecast origin.

## Repository structure

```
notebooks/
  volve_production_forecast_ML_Part1.ipynb   # data preparation, Arps DCA, gradient boosting, TFT, backtests
  volve_production_forecast_ml_part2.ipynb   # hybrid models, short-horizon, water cut, VFM, anomalies, MLOps
figures/                                      # key result figures
requirements.txt
```

## Reproducing the results

1. Download `Volve production data.xlsx` from [Equinor's Volve data sharing page](https://www.equinor.com/energy/volve-data-sharing) (free registration required) and place it in a Google Drive folder.
2. Open either notebook in Google Colab and set `PROJECT` and `SRC` in the paths cell.
3. Run all cells. Part 1 takes about 30 minutes on CPU; TFT training and the rolling backtest account for most of the runtime.

The dataset is not redistributed in this repository.

## Data acknowledgement

Data provided by Equinor and the Volve licence partners under the Equinor Open Data Licence.

## Author

Joseph Lebbie, Petroleum Engineer (MS, University of Houston) and AI/ML researcher.
LinkedIn: [linkedin.com/in/joseph-lebbie](https://www.linkedin.com/in/joseph-lebbie/)
