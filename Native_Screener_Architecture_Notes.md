# Native Screener Architecture — Consolidated Design Reference

> **Status: PLANNED — Not to be built until all pre-requisites are met.**
> **Created: 2026-08-04 | Source: Collaborative discussion (GLM + DeepSeek + User)**

---

## Pre-Requisites (must be completed before starting)

1. Backtest date failure — resolved and verified
2. Universe finalized — hard cap at 1,000–1,200 stocks (not more)
3. Exclusion list finalized — merged into `universe_excludes.json`
4. Renko scoring points finalized and backtested
5. Renko hard gate (3+ consecutive red bricks / descending lower-lows) finalized and backtested
6. Sync gating implemented — only universe stocks get parquet files
7. One-time cleanup of non-universe parquets
8. Any other v4.9/v4.10 engine fixes completed

---

## 1. Objective

Replace the Chartink/Playwright pipeline with a native Parquet-based screener that:
- Classifies all universe stocks (1,000–1,200) into 4 emergent phases
- Emits the same CSV contract the dashboard already consumes
- Runs on full Apollo indicator set (VPT, Renko, multi-TF RSI) that Chartink cannot compute
- Is self-contained — no external scraper dependencies, no auth failures, no rate limits

---

## 2. What Gets Built / Removed

### New Files
| File | Purpose |
|---|---|
| `nse_engine/features.py` | Per-bar normalized feature frames (rolling percentile / z-score vs own history), cached to Parquet |
| `nse_engine/momentum_analysis.py` | Step 1 empirical discovery: winner/loser profiling, effect sizes, weight derivation |
| `nse_engine/native_screener.py` | Reads feature Parquets, computes Recovery + Momentum scores, emits CSVs |
| `nse_engine/universe_regime.py` | Market-cap / trend / volatility regime overlays |
| `tombstone.json` (repo root) | Logs symbols dropped from universe — survivorship tracking |

### Removed (only after 2-week parallel-run validation)
- `chartink_pipeline/` (fetcher, auth, watchlist_generator, run_pipeline)
- Playwright dependency

---

## 3. Architecture: Three Layers

### Layer 1: Per-Stock Normalized Features
Every feature is a **rolling percentile** or **z-score** of that stock's own trailing history.
- Window: trailing 252-day (1 year) rolling window — **never expanding or centered** (prevents future leakage)
- This means "RSI at 12th percentile" = genuinely depressed *for this specific stock*, not just below a fixed threshold
- Handles stock diversity (large/mid/small cap) with one scorer — no per-bucket parameter tuning

### Layer 2: Regime Overlays
Smooth market-level adjustments:
- NIFTY z-score / percentile (not binary above/below 50-DMA — binary creates discontinuities)
- Cross-sectional median ATR percentile as volatility regime signal (VIX-free, zero new dependencies)
- Pull NIFTY via yfinance (^NSEI) into repo like any symbol

### Layer 3: Cross-Sectional Ranking (highest-value upgrade)
Per-day ranking of each feature against all stocks trading that day.
- Auto-adapts to market regime: when the whole market is oversold, per-stock percentiles all read "low," but cross-sectional ranking still surfaces the relatively strongest
- **Must be computed and included in Step 1's empirical analysis from the start** — not bolted on later. If you derive weights without it, then add it, your weights become stale and need re-derivation.
- Date alignment: forward-fill for historical analysis; in live screener, only rank stocks with fresh bars (<=3 trading days old) — halts/suspensions drop out naturally

---

## 4. Concrete Feature Set (~22 features)

All computable from D/4H/W OHLCV via existing `indicators.py` + `renko.py`.

### Pillar 1: Price Structure
| Feature | Computation |
|---|---|
| Distance from 200d range high (percentile) | (close - 200d low) / (200d high - 200d low), rolling percentile |
| Distance from 200d range low (percentile) | Same, inverted |
| 5-day ROC percentile | Rolling percentile of 5-day return |
| 10-day ROC percentile | Rolling percentile of 10-day return |
| 20-day ROC percentile | Rolling percentile of 20-day return |
| Gap vs 20-DMA (percentile) | (close - SMA20) / SMA20, rolling percentile |
| Gap vs 50-DMA (percentile) | (close - SMA50) / SMA50, rolling percentile |
| 50-200 DMA gap | (SMA50 - SMA200) / SMA200, rolling percentile |

### Pillar 2: RSI / Oscillators
| Feature | Computation |
|---|---|
| RSI21 percentile (daily) | Rolling percentile of RSI(21) |
| RSI21 percentile (weekly) | Rolling percentile of weekly RSI(21) |
| Stoch K percentile | Rolling percentile of Stochastic %K(14,3) |
| Stoch D percentile | Rolling percentile of Stochastic %D(14,3) |
| RSI36 level | Raw RSI(36) — used as-is or percentile |
| RSI56 level | Raw RSI(56) — used as-is or percentile |
| ADX percentile | Rolling percentile of ADX(14) |
| +/-DI alignment | Binary: +DI > -DI = 1, else 0 |

### Pillar 3: Volume
| Feature | Computation |
|---|---|
| Volume z-score (60d) | (volume - 60d mean) / 60d std |
| Up-day / Down-day volume ratio | Mean volume on up-days / mean volume on down-days (20d lookback) |
| VPT slope divergence (5 vs 10-bar) | Difference between 5-bar and 10-bar VPT linear regression slopes |
| Volume surge vs 20d | Current volume / 20d average volume |

### Pillar 4: Volatility
| Feature | Computation |
|---|---|
| ATR percentile (120d) | Rolling percentile of ATR(14) |
| Volatility compression | ATR / median(ATR, 120d) — values <1 indicate compression |

### Pillar 5: Trend / Higher Timeframe
| Feature | Computation |
|---|---|
| Renko trend alignment | Binary, reuse `renko.py` swing machinery |
| 4H RSI lead vs daily RSI | 4H RSI(21) - daily RSI(21), rolling percentile |
| Weekly RSI percentile | Rolling percentile of weekly RSI(21) |

### Cross-Sectional Additions (per day, across all stocks)
| Feature | Computation |
|---|---|
| RSI cross-sectional rank | Today's RSI percentile rank across all stocks trading today |
| Volume cross-sectional rank | Today's volume percentile rank across all stocks trading today |
| ROC cross-sectional rank | Today's 10d ROC percentile rank across all stocks trading today |

---

## 5. Empirical Discovery (Step 1 — Build First)

### Success Definition
- **Winner**: >=15% gain within 20 trading days, max drawdown < 8% from entry
- **Non-winner**: Everything else
- Time split: **Train on 2024–2025, validate on 2026** — never tune on the same window you profile

### Analysis Method
1. Compute all ~22 features at every bar, for every stock in the universe
2. For each bar, look forward 20 days to label winner/non-winner
3. Profile winners vs non-winners using **Cohen's d** (for normally-distributed features) and **Mann-Whitney U** (for non-normal)
4. Derive scoring weights **proportional to effect sizes** — larger effect = more weight
5. Compute **pairwise correlation matrix** on the winner/loser population — for any |rho| > 0.7, keep the higher-effect-size feature, drop the weaker (prevents double-counting)
6. Compute **feature stability** (autocorrelation: how much today's value correlates with yesterday's) — low-autocorrelation features add noise; apply stability penalty or smooth (e.g., 3-day average for binary features like Renko trend)

### Survivorship Mitigation
- `tombstone.json` logs every symbol dropped from universe: symbol, date removed, reason (exclusion list, delisted, user removed, sector change)
- Step 1 can profile former-universe stocks via tombstone + their existing parquet history
- Accepted limitation: pre-universe history (stocks never added) is unavailable — document this

---

## 6. Scoring Formula

**Weighted linear combination of normalized features** → 0–100 per stock.

- Consistent with Apollo's existing weighted-sum convention (interpretable, debuggable)
- Weights derived from Step 1 effect sizes — not hand-tuned
- No per-bucket weights — the phase is emergent from feature values, not a separate classifier

### Validation Gate
After deriving weights, run a cluster check: do Recovery-phase stocks and Bottoming-Out-phase stocks separate cleanly on the feature set alone?
- **If yes**: proceed — phases are truly emergent
- **If no**: add one disambiguating rule: MA structure intact -> Recovery; MA structure broken but RSI inflecting -> Bottoming Out

---

## 7. Phase Classification (Emergent, Not Bucketed)

| Phase | Emerges From | Action |
|---|---|---|
| **Momentum** | High RSI percentile + volume z-score + price near range top + MA alignment + Renko bullish | Emit to Momentum Candidates table |
| **Recovery** | Low RSI percentile + rising volume + near range bottom + MA intact + Renko turning | Emit to Recovery Candidates table |
| **Breaking Down** | Deteriorating MA + declining up-day volume + support breaks + Renko bearish | Exclude (or show in "Avoid" table for dashboard) |
| **Bottoming Out** | Very low RSI, no volume confirmation, new lows, Renko bearish but decelerating | Exclude (or show in "Avoid" table for dashboard) |

**Open question**: Should Breaking Down and Bottoming Out be completely excluded, or shown in a separate "Avoid" table on the dashboard? The "Avoid" list has informational value — user should decide.

---

## 8. Output & Dashboard Integration

### CSV Contract (no dashboard breakage)
- Write to `nse_output/chartink_apollo_ranked_*.csv` (same prefix the dashboard's `_render_scan_intelligence()` already globs)
- The dashboard reads these files as-is — zero dashboard code changes needed
- Resolve the `nse_output/` vs `chartink_output/` path mismatch in the screener, not the dashboard

### Rank Cutoffs (not just thresholds)
- Top-N Recovery + Top-N Momentum per day — prevents table flooding during market-wide bounces
- Can also require both score-threshold AND cross-sectional percentile (e.g., score >= 70 AND cross-sectional RSI rank <= 10th percentile)

### Parallel-Run Safety Net
- Run native screener alongside Chartink pipeline for 2 weeks
- Compare Recovery tables on overlapping symbols
- **Only remove `chartink_pipeline/` after tables converge on overlapping symbols**

---

## 9. Feature Caching & Performance

### Cache Strategy
- Feature frames cached as Parquet: `apollo_data/features/<SYMBOL>.parquet`
- Computed once per symbol during sync, stored alongside price data
- **Cache invalidation on sync**: recompute last 60 bars of each feature frame, overwrite the tail
- Rolling windows only shift at the current edge, so this is correct and cheap
- Full universe scan becomes a fast read-only operation — estimate <10 seconds for 1,000–1,200 stocks

### Performance Target
- Daily full-universe scan: <10 seconds (read cached feature parquets + rank)
- Single-stock feature computation: <50ms (vectorized pandas)
- Sync with feature update: ~30 seconds for 1,200 stocks (recompute 60-bar tail only)

---

## 10. Implementation Roadmap

| Step | Task | Key Decisions | Depends On |
|---|---|---|---|
| 1 | Build `features.py` — feature-frame builder + Parquet cache | 252-day rolling window, no future leakage | Clean parquet data for all universe stocks |
| 2 | Build `momentum_analysis.py` — empirical discovery | Winner definition, time split, effect sizes, correlation gate, stability analysis | Step 1 |
| 3 | Derive + review weights | Manual review of effect sizes, correlation pruning | Step 2 |
| 4 | Build Recovery scorer | Weighted linear combination, emergent phase check | Step 3 |
| 5 | Build Momentum scorer | Same formula, different feature emphasis | Step 3 |
| 6 | Volume analytics module | Surge, up/down ratio, PVD | Step 1 |
| 7 | CSV output + dashboard path fix | chartink_ prefix, top-N cutoffs | Steps 4–6 |
| 8 | Regime overlays | NIFTY z-score, cross-sectional ATR, cap overlay | Step 1 |
| 9 | Cross-sectional rank layer | Per-day ranking, freshness gate (<=3 days), tombstone-aware | Step 1 (computed from start, activated here) |
| 10 | Parallel-run validation | 2-week gate, compare with Chartink output | Steps 1–9 |
| 11 | Remove Chartink pipeline | Delete `chartink_pipeline/`, Playwright dependency | Step 10 passes |

---

## 11. What This Does NOT Change

- The core Apollo scoring engine (21 signals + 8 Renko) — unchanged for backtesting and trade execution
- Backtest studio, live dashboard, trade engine — untouched
- This is a **screening layer** that feeds better candidates to the existing engine
- No new dependencies beyond what's already installed (pandas, numpy, pyarrow)

---

## 12. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Survivorship bias in Step 1 | `tombstone.json` tracks dropped stocks; profile former-universe stocks; document pre-universe limitation |
| Cross-sectional date alignment (halts, IPOs, suspensions) | Forward-fill for analysis; live screener requires fresh bars (<=3 trading days) — stale stocks drop out |
| Feature correlation inflating weights | Correlation matrix in Step 1; for |rho| > 0.7, keep higher-effect-size feature, drop weaker |
| Emergent phase overlap (Recovery vs Bottoming) | Cluster analysis in Step 2; add MA-structure disambiguating rule if they don't separate |
| Binary features adding noise (Renko, DI alignment) | Compute feature stability (autocorrelation); smooth low-stability features with 3-day rolling mean |
| Universe changes after weights are derived | Re-run Step 1 if universe changes materially; weights are tied to the universe they were trained on |
| Future leakage in rolling windows | Use trailing windows only (never expanding or centered); validate with out-of-sample period |

---

## 13. Design Decisions Record

| Decision | Source | Rationale |
|---|---|---|
| Universe hard cap: 1,000–1,200 stocks | User | Curated quality over breadth; manageable daily scan time |
| Rolling percentile (not z-score) as primary normalization | Original doc + GLM | More intuitive ("12th percentile" vs "-1.3 sigma"); robust to non-normal distributions |
| Cross-sectional rank included in Step 1, not deferred | GLM | Weights must be derived from the full feature set; adding it later requires full re-analysis |
| Weighted linear combination (not rank-sum, not model) | DeepSeek | Consistent with Apollo convention; interpretable; debuggable; model re-ranking is a future optional upgrade |
| tombstone.json for survivorship tracking | DeepSeek | Simple, practical, near-zero overhead; enables profiling former universe stocks |
| Freshness gate (<=3 days) for live cross-sectional ranking | DeepSeek | Halts/suspensions drop out naturally; avoids ranking against stale data |
| Forward-fill for historical analysis date alignment | DeepSeek | Pragmatic; the analysis has the full history to fill from |
| Feature stability (autocorrelation) analysis in Step 1 | GLM | Binary/low-autocorrelation features add score noise; need smoothing or penalty |
| chartink_ prefix on output CSVs | DeepSeek | Zero dashboard breakage; resolve path mismatch in screener, not dashboard |
| 2-week parallel-run before Chartink removal | DeepSeek + GLM | De-risks the Playwright dependency removal; data-driven go/no-go decision |
| Phase classification emergent, not explicit | Original doc | Avoids per-bucket discontinuity and parameter explosion; validated empirically in Step 2 |
| Rank cutoffs (top-N) over threshold-only | Original doc + GLM | Prevents table flooding in market-wide bounces; curated output at any market state |
| NIFTY via yfinance (^NSEI) for regime overlay | Original doc | Single new data source; stored in repo like any symbol |
| VIX-free: cross-sectional median ATR for volatility regime | Original doc | India VIX hard to source reliably; cross-sectional ATR achieves same effect, zero new dependencies |
| PVD (Price-Volume Divergence) as headline feature | GLM | Already computed (VPT in indicators.py); strong momentum-quality signal Chartink cannot produce |
| Feature Parquet cached alongside price Parquet | DeepSeek | Daily scan becomes read-only; <10 seconds for full universe |
| 60-bar tail recompute on sync | DeepSeek + GLM | Rolling windows only shift at current edge; correct and cheap |

---

## 14. How This Improves Accuracy & Reliability

| What | Before (Current) | After (Native Screener) | Impact |
|---|---|---|---|
| Signal thresholds | Fixed raw numbers (RSI < 30) | Rolling percentile of stock's own history | Fewer false positives — "oversold" means *actually* at this stock's extreme |
| Market context | None | Cross-sectional ranking + NIFTY regime overlay | Reliable candidate lists in both bull and bear markets |
| Indicator coverage | Limited to what Chartink exposes | Full access to VPT, Renko, 4H/W RSI, divergence | Stronger signals that Chartink literally cannot compute |
| Weight derivation | Hand-tuned intuition | Empirical effect sizes from historical data | Weights reflect what actually predicted winners |
| External dependency | Chartink + Playwright (fragile, rate-limited, auth failures) | Self-contained computation | No scraping failures, no rate limits, no TOS risk |
| Universe coverage | NIFTY-500 (Chartink limitation) | Full 1,000–1,200 curated stocks | No blind spots in small/mid-cap space |
| Future leakage risk | N/A (no normalization) | Explicitly handled: trailing windows only | Screener outputs are as honest as backtest results |
| Phase classification | Per-bucket with explicit rules | Emergent from normalized features | No discontinuity at bucket boundaries; reproducible |
