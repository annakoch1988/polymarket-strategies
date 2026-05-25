# Trading Strategy Proposals for 5-Minute Crypto Resolution Markets on Polymarket

**Classification:** Implementation Blueprint  
**Date:** May 2026  
**Scope:** Binary / multi-outcome contracts resolving on crypto price thresholds within 5-minute windows

---

## 0. Preliminaries: Why 5-Minute Crypto Markets Are Structurally Different

A 5-minute crypto Polymarket contract is not a long-horizon prediction market. It is a **micro-duration derivative** whose payoff depends on whether a spot price $S_T$ crosses a strike $K$ at resolution time $T$, where $T - t_0 = 5$ minutes. This has profound implications:

1. **The "true probability" is approximately computable.** Unlike political markets, the underlying (BTC, ETH spot price) is observable in real-time with sub-second latency from CEX order books. You can compute $P(S_T > K \mid S_t, \sigma_t, \tau)$ using a diffusion approximation.
2. **Latency arbitrage is the dominant edge source.** The Polymarket order book updates slower than Binance/Coinbase spot. A price move on Binance at $t$ may not be reflected on Polymarket until $t + \delta$ where $\delta \in [2, 30]$ seconds.
3. **Liquidity is thin and stale.** Market makers on 5-min contracts often quote wide spreads and update infrequently. This creates both opportunity (stale quotes) and risk (adverse selection, slippage).
4. **The market is effectively a discretized binary option.** Pricing theory from options markets applies, but with bounded $[0,1]$ probability space and discrete resolution.

### Notation

| Symbol | Meaning |
|--------|---------|
| $S_t$ | Spot crypto price at time $t$ (from CEX) |
| $K$ | Strike/threshold of the Polymarket contract |
| $p_t$ | Polymarket YES share price (implied probability) at time $t$ |
| $\tau$ | Time to resolution (seconds remaining) |
| $\sigma_t$ | Realized volatility estimate over recent window |
| $\Delta_t = S_t - K$ | Moneyness in spot-price space |
| $h$ | Trading horizon (seconds before resolution at which you enter) |

------

## Strategy 1: Spot-Led Latency Arbitrage (SLA)

### 1.1 Classification

**Type:** Heuristic rule-based system  
**Edge source:** Information latency between CEX spot feeds and Polymarket order book  
**Complexity:** Low-Medium  
**Capital efficiency:** High (short holding periods, high turnover)

### 1.2 Rationale

The core insight is that Polymarket 5-minute markets are **price-takers** relative to the CEX spot market. When BTC moves 0.3% on Binance in 10 seconds, the fair probability of "BTC > $K$ at resolution" changes materially, but Polymarket market makers may lag by $\delta$ seconds. During this window, the Polymarket price is stale and exploitable.

**Why this works structurally:**
- CEX crypto markets have far more liquidity and faster participants than Polymarket
- Polymarket MM algorithms must balance inventory risk across many markets and cannot update as fast as a dedicated CEX HFT system
- The 5-minute resolution window means even a small spot move can shift the "true" probability by 5-20 percentage points

### 1.3 Signal Construction

At each decision time $t$, compute the **fair probability** $\hat{\pi}_t$ using a Black-Scholes-style binary call approximation:

$$\hat{\pi}_t = \Phi\left(\frac{\ln(S_t / K) + (\mu - \sigma_t^2/2)\tau}{\sigma_t \sqrt{\tau}}\right)$$

where:
- $\Phi$ is the standard normal CDF
- $\mu$ is the short-term drift (can be set to 0 for a pure diffusion assumption over 5 minutes)
- $\sigma_t$ is estimated from a rolling window of 1-minute returns on the CEX
- $\tau$ is seconds remaining to resolution

The **mispricing signal** is:

$$\epsilon_t = \hat{\pi}_t - p_t$$

**Decision rule:**
- If $\epsilon_t > \theta_{entry}$ AND $p_t^{ask}$ is available at a price $< \hat{\pi}_t - c$ (where $c$ is estimated cost), BUY YES at ask.
- If $\epsilon_t < -\theta_{entry}$ AND $p_t^{bid}$ is available at a price $> \hat{\pi}_t + c$, SELL YES (or BUY NO) at bid.

### 1.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| Entry threshold | $\theta_{entry}$ | Grid search on historical data, optimizing for Sharpe ratio net of costs | 0.03 - 0.10 |
| Volatility window | $w_\sigma$ | Rolling window length for $\sigma_t$ estimation; test 30s, 60s, 120s, 300s | 30 - 300 seconds |
| Drift parameter | $\mu$ | Can be set to 0 or estimated from EMA of 1-min returns; test sensitivity | 0 or small |
| Cost estimate | $c$ | Empirical spread + fee estimate; measure from historical order book data | 0.01 - 0.04 |
| Minimum time to resolution | $\tau_{min}$ | Don't trade if $\tau < \tau_{min}$ (too illiquid); test 30s, 60s, 120s | 30 - 120 seconds |
| Maximum position size | $q_{max}$ | Risk management; test impact on PnL | 50 - 500 shares |

### 1.5 Concrete Example

**Setup:** BTC spot = $68,450. Polymarket contract: "Will BTC be above $68,500 at 14:05 UTC?" (resolution in 3 minutes).

1. Current spot moves to $68,520 on Binance (a +$70 move in 15 seconds).
2. Compute $\hat{\pi}_t$: $\sigma_t = 0.001$ (per minute), $\tau = 180$ s. $\ln(68520/68500) = 0.000292$. 
   $\hat{\pi}_t = \Phi(0.000292 / (0.001 \times \sqrt{3})) \approx \Phi(0.169) \approx 0.567$
3. Polymarket still shows YES at bid=0.42, ask=0.46 (implied prob ~0.44).
4. Signal: $\epsilon_t = 0.567 - 0.44 = 0.127$. This exceeds $\theta_{entry} = 0.05$.
5. **Action:** BUY YES at ask = $0.46. Expected value per share = $0.567 - 0.46 - 0.02 = +0.087$.
6. Hold to resolution. If BTC stays above $68,500, share resolves to $1.00. Profit = $0.54 per share.

### 1.6 Statistical Backing

The strategy relies on the **efficient market hypothesis violation due to latency**. Academic work on cross-venue latency arbitrage (e.g., Budish, Cramton & Shim 2015; Baron et al. 2019 on HFT) shows that price discrepancies between faster and slower venues persist for short windows. The key statistic to validate is the **autocorrelation of the mispricing signal** $\epsilon_t$: if it decays within $\delta$ seconds, the window exists. Measure the half-life of $\epsilon_t$ autocorrelation; if $t_{1/2} > 3$ seconds, the strategy is potentially viable.

### 1.7 Failure Modes and Mitigations

| Risk | Mitigation |
|------|-----------|
| Spot reversal before resolution | Only enter when $\tau$ is small relative to spot distance from strike |
| Adverse selection (MM pulls liquidity) | Use limit orders with stale-quote detection |
| Polymarket API latency | Co-locate infrastructure; use WebSocket feeds |
| Volatility regime change | Adaptive $\sigma_t$ estimation with exponential weighting |

---

## Strategy 2: Probability Mean-Reversion After Overreaction (PMR)

### 2.1 Classification

**Type:** Statistical model (time-series + distributional)  
**Edge source:** Behavioral overreaction by retail traders in thin prediction markets  
**Complexity:** Medium  
**Capital efficiency:** Medium (requires holding positions for 1-4 minutes)

### 2.2 Rationale

Retail participants on Polymarket tend to **overreact to salient information**. When a large spot candle appears (e.g., BTC drops 0.5% in 30 seconds), traders rush to update their probability estimates, often overshooting the fair value. This creates a short-term mean-reversion opportunity.

**Why this works:**
- Prediction markets with retail-heavy participation exhibit documented overreaction (Page 2012; Camerer et al. on prospect theory in markets)
- The 5-minute window means traders are reacting emotionally to noise, not fundamentals
- The "true" probability can be continuously computed from the spot price, making overreaction measurable in real-time
- Mean-reversion in probability space is analogous to short-term reversal in equity markets (Lehmann 1990; Lo & MacKinlay 1990)

### 2.3 Statistical Model

Define the **overreaction indicator** as the rate of change of the implied probability:

$$\dot{p}_t = \frac{p_t - p_{t-w}}{w}$$

where $w$ is a lookback window (e.g., 15 seconds).

Define the **fair probability rate of change**:

$$\dot{\hat{\pi}}_t = \frac{\hat{\pi}_t - \hat{\pi}_{t-w}}{w}$$

The **overreaction magnitude** is:

$$O_t = \dot{p}_t - \dot{\hat{\pi}}_t$$

When $O_t$ exceeds a threshold (in standard deviations, estimated from a rolling window), we expect mean-reversion.

**Signal:** $S_t = -\text{sign}(O_t) \cdot f(|O_t|)$

where $f$ is a monotonically increasing function (e.g., clipped linear or sigmoid).

**Model for reversion dynamics:** Fit an Ornstein-Uhlenbeck (OU) process to the mispricing $\epsilon_t = p_t - \hat{\pi}_t$:

$$d\epsilon_t = -\kappa \epsilon_t dt + \nu dW_t$$

The mean-reversion speed $\kappa$ and noise $\nu$ are estimated from historical data. The expected reversion over horizon $h$ is:

$$E[\epsilon_{t+h} | \epsilon_t] = \epsilon_t e^{-\kappa h}$$

Trade when $E[\text{reversion profit}] > \text{cost}$:

$$|\epsilon_t| (1 - e^{-\kappa h}) > c$$

### 2.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| Lookback window for rate of change | $w$ | Test 10s, 15s, 30s, 60s; optimize for signal IC (information coefficient) | 10 - 60 seconds |
| OU reversion speed | $\kappa$ | MLE estimation on $\epsilon_t$ time series per market | 0.1 - 2.0 per minute |
| OU noise | $\nu$ | Joint MLE with $\kappa$ | Data-dependent |
| Overreaction threshold (z-score) | $z_\theta$ | Grid search on standardized $O_t$; optimize Sharpe | 1.5 - 3.0 |
| Minimum $\tau$ to enter | $\tau_{min}$ | Same as Strategy 1 | 60 - 180 seconds |
| Position holding time | $h$ | Optimize; may be dynamic based on $\kappa$ | 30 - 180 seconds |
| Rolling window for $\sigma_t, \kappa$ estimation | $W_{est}$ | Test 5min, 15min, 60min rolling windows | 5 - 60 minutes |

### 2.5 Concrete Example

**Setup:** ETH spot = $3,420. Contract: "Will ETH be above $3,400 at 15:10 UTC?" (3 minutes to resolution).

1. ETH drops sharply from $3,425 to $3,405 in 20 seconds.
2. Polymarket YES price drops from 0.72 to 0.45 in 25 seconds (rate = -0.011/sec).
3. Fair probability drops from 0.70 to 0.52 (rate = -0.007/sec).
4. Overreaction: $O_t = -0.011 - (-0.007) = -0.004$/sec. Over 25s, cumulative overreaction = -0.10.
5. Mispricing: $\epsilon_t = 0.45 - 0.52 = -0.07$. Z-score of $O_t$ (rolling 15-min std = 0.002): $z = 2.0$.
6. Signal exceeds $z_\theta = 1.5$. **Action:** BUY YES at $0.45.
7. Expected reversion: $\kappa = 0.5$/min, $h = 2$ min. $E[\text{reversion}] = 0.07 \times (1 - e^{-0.5 \times 2}) = 0.07 \times 0.632 = 0.044$.
8. Expected value = $0.044 - 0.02 (\text{cost}) = +0.024$ per share.

### 2.6 Statistical Validation Protocol

1. **Information Coefficient (IC):** Compute rank correlation between signal $S_t$ and forward return $r_{t,h}$. Target: $|IC| > 0.03$ consistently.
2. **Hit Rate:** Fraction of trades with positive PnL. Target: > 55%.
3. **Autocorrelation of $\epsilon_t$:** Verify OU process fit. Test Ljung-Box on residuals.
4. **Walk-forward validation:** Estimate $\kappa, \nu$ on trailing 2 hours; predict next 10 minutes; roll forward.
5. **Multiple testing correction:** If testing 10+ parameter combinations, apply Bonferroni or BH-FDR.

### 2.7 Failure Modes

- **Under-reaction regime:** In trending markets, what looks like overreaction may be rational updating. Filter by regime (use ADX or trend strength).
- **Liquidity vacuum:** After large moves, the order book may empty. Must have stale-quote detection.
- **Parameter instability:** $\kappa$ may vary across volatility regimes. Use regime-conditional estimation.

---

## Strategy 3: Multi-Outcome Probability Mass Redistribution (MPR)

### 3.1 Classification

**Type:** Statistical model (probability distribution fitting)  
**Edge source:** Inconsistent pricing across mutually exclusive outcomes  
**Complexity:** Medium-High  
**Capital efficiency:** Medium (requires capital across multiple outcomes)

### 3.2 Rationale

Many 5-minute crypto Polymarket markets have **multiple outcomes** (e.g., price ranges):
- "BTC < $68,000"
- "BTC $68,000 - $68,200"  
- "BTC $68,200 - $68,400"
- "BTC $68,400 - $68,600"
- "BTC > $68,600"

The sum of all YES prices must equal 1.0 (ignoring the vig/overround). However, in practice:
- **Sum may exceed or fall below 1.0** due to asynchronous quoting
- **Individual outcomes may be mispriced relative to a fitted distribution** (e.g., lognormal)

This creates two sub-strategies: (a) arbitrage the sum deviation, and (b) exploit shape mispricing.

### 3.3 Statistical Model

**Step 1: Fit a parametric distribution to the implied probabilities.**

Given outcomes with boundaries $[K_i, K_{i+1}]$ and implied probabilities $\{p_i\}$, fit a distribution $F(S_T | S_t, \sigma, \mu)$ such that:

$$\hat{p}_i = F(K_{i+1}) - F(K_i)$$

Minimize the KL-divergence or sum of squared errors:

$$\min_{\sigma, \mu} \sum_i (p_i - \hat{p}_i)^2$$

This gives you the market-implied distribution. Then compare to your own estimate using CEX data:

$$\tilde{p}_i = \tilde{F}(K_{i+1} | S_t, \hat{\sigma}_{CEX}, \hat{\mu}_{CEX}) - \tilde{F}(K_i | S_t, \hat{\sigma}_{CEX}, \hat{\mu}_{CEX})$$

**Step 2: Identify mispriced outcomes.**

For each outcome $i$, compute:

$$\delta_i = \tilde{p}_i - p_i$$

**Step 3: Construct a portfolio.**

- Buy outcomes with $\delta_i > \theta$
- Sell outcomes with $\delta_i < -\theta$
- Ensure the portfolio is approximately delta-neutral (the sum of positions ~ 0 in probability space)

**Arbitrage sub-strategy:** If $\sum p_i > 1.0 + c_{total}$, sell all outcomes proportionally. If $\sum p_i < 1.0 - c_{total}$, buy all outcomes.

### 3.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| Mispricing threshold | $\theta$ | Grid search on $\delta_i$; optimize risk-adjusted return | 0.02 - 0.08 |
| Distribution family | -- | Compare lognormal, normal, skew-normal, Student-t fits to spot returns | Data-dependent |
| Volatility estimation method | -- | Compare realized vol, EWMA, GARCH(1,1) on 1-min returns | Data-dependent |
| Portfolio rebalance frequency | $f_{rebal}$ | Test 5s, 10s, 30s | 5 - 30 seconds |
| Maximum single-outcome exposure | $q_{max}$ | Risk limit; test impact on drawdown | 100 - 1000 shares |
| Overround tolerance | $\gamma$ | Max $\sum p_i$ deviation from 1.0 before forcing rebalance | 0.01 - 0.05 |

### 3.5 Concrete Example

**Setup:** BTC spot = $68,350. Five outcome brackets, 2 minutes to resolution.

| Outcome | Boundaries | Polymarket $p_i$ | Model $\tilde{p}_i$ | $\delta_i$ |
|---------|-----------|-------------------|---------------------|-----------|
| A | < $68,200 | 0.15 | 0.12 | -0.03 |
| B | $68,200-$68,300 | 0.22 | 0.20 | -0.02 |
| C | $68,300-$68,400 | 0.28 | 0.34 | +0.06 |
| D | $68,400-$68,500 | 0.20 | 0.22 | +0.02 |
| E | > $68,500 | 0.18 | 0.12 | -0.06 |
| **Sum** | | **1.03** | **1.00** | |

Observations:
1. Sum = 1.03 -> slight overround (vig or stale quotes).
2. Outcome C is underpriced by 0.06 (model says 0.34, market says 0.28).
3. Outcome E is overpriced by 0.06.

**Action:**
- BUY C at 0.28 (ask). Expected edge = 0.06 - cost ~ +0.04.
- SELL E at 0.18 (bid). Expected edge = 0.06 - cost ~ +0.04.
- The sum deviation (0.03) suggests you could also short the basket for +0.03 if cost allows.

### 3.6 Statistical Backing

The sum-to-one constraint is a fundamental identity in probability theory. Violations indicate **pricing inefficiency**. Research on multi-outcome prediction markets (e.g., Page 2012 on the "favourite-longshot bias"; Wolfers & Zitzewitz 2004) shows that extreme outcomes tend to be overpriced (longshot bias) and middle outcomes underpriced. This is consistent with prospect theory: overweighting of small probabilities.

The distribution-fitting approach is analogous to **implied volatility surface fitting** in options markets. Deviations from the fitted surface are exploited by volatility arbitrageurs (Carr & Madan 1998).

### 3.7 Failure Modes

- **Fat tails:** If returns have fatter tails than the fitted distribution, extreme outcomes will be systematically underpriced by the model. Use Student-t or mixture distributions.
- **Liquidity fragmentation:** Each outcome has less liquidity than a binary market. Slippage can destroy edge.
- **Correlated mispricing:** If all outcomes move together (e.g., during a flash crash), the model's distributional assumption may break down.

---

## Strategy 4: Volume-Spike Informed Trading (VIT)

### 4.1 Classification

**Type:** Heuristic with statistical filtering  
**Edge source:** Informed traders revealing information through unusual volume  
**Complexity:** Medium  
**Capital efficiency:** Medium-High

### 4.2 Rationale

In prediction markets, **informed traders** (those with better models, faster data, or inside knowledge) leave footprints in the volume and order flow. A sudden spike in volume on a 5-minute contract -- especially one that is "out of the money" and thus has low baseline activity -- often signals that someone has information not yet reflected in the price.

**Why this works:**
- Kyle (1985) and Glosten & Milgrom (1985) models predict that informed traders trade more aggressively when their information advantage is large
- In 5-minute markets, "information" is often just a better real-time spot feed or a proprietary volatility model
- Volume spikes are observable in the Polymarket trade feed (WebSocket)
- Following informed flow is a well-documented profitable strategy in traditional markets (Chordia, Roll & Subrahmanyam 2002)

### 4.3 Signal Construction

**Step 1: Compute baseline volume.**

$$\bar{V}_t = \text{EMA}(V, \lambda)$$

where $V$ is trade volume per second and $\lambda$ is the EMA decay (e.g., 60-second half-life).

**Step 2: Detect volume spike.**

$$Z_t^V = \frac{V_t - \bar{V}_t}{\sigma_V}$$

where $\sigma_V$ is the rolling standard deviation of volume.

A spike is detected when $Z_t^V > z_{spike}$.

**Step 3: Determine direction of informed flow.**

Compute the **order flow imbalance** during the spike:

$$OFI_t = \sum_{i \in \text{spike}} \text{sign}(trade_i) \cdot \text{size}_i$$

where $\text{sign}(trade_i)$ is +1 if the trade was at the ask (buyer-initiated) and -1 if at the bid (seller-initiated). Use the Lee-Ready algorithm or Polymarket's native trade-side indicator.

**Step 4: Generate signal.**

If $Z_t^V > z_{spike}$ AND $OFI_t > \theta_{ofi}$: BUY YES  
If $Z_t^V > z_{spike}$ AND $OFI_t < -\theta_{ofi}$: BUY NO (SELL YES)

**Step 5: Statistical filter (reduce false positives).**

Only act if the volume spike is **consistent with the spot price direction**:

- If buying YES, require $S_t > S_{t-w}$ (spot is moving toward the strike)
- If buying NO, require $S_t < S_{t-w}$ (spot is moving away from the strike)

This cross-validation reduces false signals from noise traders.

### 4.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| EMA half-life for baseline volume | $\lambda$ | Test 30s, 60s, 120s, 300s; choose where spike detection is most predictive | 30 - 300 seconds |
| Volume spike z-score threshold | $z_{spike}$ | Grid search; optimize precision (fraction of spikes followed by favorable price move) | 2.0 - 5.0 |
| OFI threshold | $\theta_{ofi}$ | Grid search on net order flow during spike window | Data-dependent |
| Spot consistency window | $w$ | How far back to check spot direction; test 5s, 15s, 30s | 5 - 30 seconds |
| Post-spike holding time | $h$ | How long to hold after entering; optimize PnL curve | 30 - 240 seconds |
| Minimum spike volume (absolute) | $V_{min}$ | Filter out tiny spikes; test 10, 50, 100, 500 shares in single burst | 50 - 500 shares |

### 4.5 Concrete Example

**Setup:** Contract: "Will BTC > $68,500 at 16:00?" 4 minutes to resolution. BTC spot = $68,380. YES price = 0.30.

1. Suddenly, 200 shares of YES are bought in 3 seconds (baseline volume is ~10 shares/minute).
2. $Z_t^V = (200/3 - 0.167) / 0.3 \approx 222$. Massive spike.
3. OFI: +200 shares (all buyer-initiated at ask).
4. Spot check: BTC just moved from $68,380 to $68,420 in the same 3 seconds.
5. **Signal:** Volume spike + buyer flow + spot confirmation -> BUY YES.
6. **Action:** Buy YES at current ask = 0.34. Fair probability given spot move: ~0.40.
7. Expected edge: $0.40 - 0.34 - 0.02 = +0.04$ per share.

### 4.6 Statistical Backing

The key statistic is the **conditional probability of a favorable price move given a volume spike**:

$$P(\Delta p > 0 \mid Z^V > z_{spike}, OFI > 0)$$

Measure this from historical data. If this probability significantly exceeds 0.5 (e.g., > 0.58), the strategy has edge. Also measure the **average return conditional on a spike** vs. unconditional average return.

Academic support: Easley, Lopez de Prado & O'Hara (2012) on Volume-Synchronized Probability of Informed Trading (VPIN) shows that volume imbalance predicts future price moves.

### 4.7 Failure Modes

- **Noise trader spikes:** Large retail orders that are not informed. The spot-consistency filter mitigates this.
- **Front-running:** Other bots detect the same spike and trade ahead of you. Latency matters.
- **Post-spike reversal:** If the spike was a market-order that temporarily moved the price, it may revert. Use limit orders and patience.

---

## Strategy 5: Time-to-Resolution Theta Decay Exploitation (TDE)

### 5.1 Classification

**Type:** Statistical model (options-pricing analog)  
**Edge source:** Non-linear probability convergence as $\tau \to 0$  
**Complexity:** Medium  
**Capital efficiency:** High (systematic, repeatable)

### 5.2 Rationale

As $\tau$ approaches 0, the probability of a binary outcome converges to either 0 or 1. The **rate of convergence** (analogous to theta in options) is non-linear and depends on moneyness $\Delta_t = S_t - K$:

$$\frac{\partial \hat{\pi}}{\partial \tau} \propto \frac{\Delta_t}{\sigma_t \tau^{3/2}} \exp\left(-\frac{\Delta_t^2}{2\sigma_t^2 \tau}\right)$$

This means:
- **Near-the-money contracts** experience rapid probability swings in the last 60 seconds
- **Deep in/out-of-the-money contracts** are already near 0 or 1, and move slowly
- Market makers may **under-hedge** the theta risk near expiration, creating systematic mispricing

**Why this works:**
- Analogous to gamma scalping in options: the last few minutes of a binary option have the highest gamma
- Market makers on Polymarket may not dynamically hedge with the same sophistication as options MMs
- You can **predict the direction** of probability convergence based on current moneyness

### 5.3 Statistical Model

For each contract, compute the **theta exposure**:

$$\Theta_t = \frac{\partial \hat{\pi}_t}{\partial \tau} = -\frac{\phi(d_1)}{\sigma_t \sqrt{\tau}} \cdot \frac{d_1}{2\tau}$$

where $d_1 = \frac{\ln(S_t/K) + (\mu - \sigma_t^2/2)\tau}{\sigma_t \sqrt{\tau}}$ and $\phi$ is the standard normal PDF.

**Strategy:**
- When $|\Theta_t|$ is large (high theta, i.e., contract is near-the-money and close to resolution), the Polymarket price should be moving rapidly.
- If the current $p_t$ has not caught up with the predicted movement (i.e., $|\hat{\pi}_t - p_t| > \theta_{entry}$), enter a position in the direction of convergence.

**Position sizing:** Scale by $\Theta_t$. The expected profit from theta convergence over a small window $dt$ is approximately $\Theta_t \cdot dt$.

### 5.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| Entry window (seconds before resolution) | $\tau_{entry}$ | Test entering at 120s, 90s, 60s, 30s before resolution | 30 - 120 seconds |
| Minimum moneyness for entry | $|\Delta|_{min}$ | Only trade when spot is near strike; test +/- 0.1%, +/- 0.3% | 0.05% - 0.5% |
| Theta threshold | $\Theta_{min}$ | Minimum absolute theta to trigger; grid search | Data-dependent |
| Exit rule | -- | Hold to resolution, or exit at $\tau_{exit}$; test | 0 - 30 seconds before resolution |
| Volatility model | -- | Compare realized vol, Parkinson/GK estimator, EWMA | Data-dependent |

### 5.5 Concrete Example

**Setup:** BTC = 68,502 USD. Strike = 68,500 USD. $\tau$ = 60 seconds. $\sigma_t = 0.001$/min.

1. $d_1 = \ln(68502/68500) / (0.001 \times 1) = 0.0000292 / 0.001 = 0.0292$
2. $\hat{\pi} = \Phi(0.0292) = 0.512$
3. Theta: $\Theta_t \approx -\phi(0.0292) \times 0.0292 / (2 \times 1) = -0.399 \times 0.0146 = -0.0058$/sec (probability is converging toward 0.5 as we approach resolution with spot barely above strike)
4. If Polymarket shows $p_t = 0.55$, the market is overestimating the probability.
5. **Action:** SELL YES at $0.55. Fair value is $0.512. Edge = $0.038 - cost.

### 5.6 Statistical Backing

The key statistic is the **correlation between predicted theta direction and actual price change** in the final $\tau$ seconds. Measure:

$$\text{corr}(\text{sign}(\Theta_t), \text{sign}(\Delta p_{t \to T}))$$

If significantly positive (> 0.55), the theta model predicts convergence direction better than chance.

Also validate the Black-Scholes binary approximation: compute the **calibration error** (Brier score) of $\hat{\pi}_t$ against realized outcomes. If Brier score of the model is lower than the market's, the model has informational value.

### 5.7 Failure Modes

- **Last-second spot jumps:** A sudden spot move in the final 10 seconds can flip the outcome. Mitigate by exiting early (at $\tau = 15$s).
- **Pin risk:** If $S_T \approx K$ exactly, resolution may be disputed or depend on the specific price feed Polymarket uses. Avoid trading when $|\Delta_t| < \epsilon_{pin}$.
- **Illiquidity at expiry:** Market makers may pull quotes in the final seconds. Use limit orders.

---

## Strategy 6: Cross-Contract Relative Value (CRV)

### 6.1 Classification

**Type:** Statistical model (cointegration / pairs trading analog)  
**Edge source:** Correlated contracts priced inconsistently  
**Complexity:** Medium-High  
**Capital efficiency:** Medium

### 6.2 Rationale

Polymarket often lists **parallel contracts** for the same crypto asset with different strikes:
- "BTC > $68,000 at 14:05"
- "BTC > $68,500 at 14:05"
- "BTC > $69,000 at 14:05"

These are effectively a **strip of binary calls** on the same underlying. Their prices must satisfy monotonicity:

$$p(K_1) \geq p(K_2) \geq p(K_3) \text{ when } K_1 < K_2 < K_3$$

And they must be consistent with a valid probability distribution (no butterfly arbitrage). Violations of these constraints are pure arbitrage. Even near-violations represent relative value opportunities.

### 6.3 Statistical Model

**Constraint 1 -- Monotonicity:** If $p(K_i) < p(K_{i+1})$ for $K_i < K_{i+1}$, buy $K_i$ and sell $K_{i+1}$. This is a pure arbitrage (always profitable regardless of outcome).

**Constraint 2 -- Convexity (no butterfly arbitrage):** For three strikes $K_1 < K_2 < K_3$ with $K_2 = (K_1 + K_3)/2$:

$$p(K_2) \leq \frac{p(K_1) + p(K_3)}{2}$$

If violated, sell $K_2$ and buy a 50/50 basket of $K_1$ and $K_3$.

**Constraint 3 -- Smooth implied distribution:** Fit a smooth CDF $\hat{F}(K)$ to all available strike prices. Outliers (strikes where $p_i$ deviates significantly from $\hat{F}(K_i)$) are relative-value trades.

$$\text{RV signal}_i = p_i - \hat{F}(K_i)$$

### 6.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|-----------------|----------------|
| Deviation threshold for RV trade | $\theta_{RV}$ | Grid search; optimize Sharpe | 0.02 - 0.06 |
| Distribution fitting method | -- | Compare isotonic regression, kernel density, parametric fits | Data-dependent |
| Rebalance frequency | $f$ | Test 10s, 30s, 60s | 10 - 60 seconds |
| Maximum leg count | $n_{max}$ | Max number of simultaneous positions | 3 - 10 |
| Position sizing method | -- | Equal weight, risk-parity, or Kelly fraction | Test all |

### 6.5 Concrete Example

**Setup:** Three BTC contracts resolving in 4 minutes. Spot = $68,300.

| Strike | Polymarket $p$ | Model $\hat{F}(K)$ | Deviation |
|--------|----------------|---------------------|-----------|
| $68,000 | 0.72 | 0.70 | +0.02 |
| $68,300 | 0.52 | 0.50 | +0.02 |
| $68,600 | 0.35 | 0.30 | +0.05 |
| $68,900 | 0.15 | 0.14 | +0.01 |

The $68,600 strike is overpriced by 0.05 relative to the fitted distribution. **Action:** SELL YES on the $68,600 contract. Hedge by BUYing YES on the $68,300 contract (partially offsets directional risk).

### 6.6 Statistical Backing

Test for **arbitrage frequency**: what fraction of snapshots violate monotonicity or convexity? Even if violations are rare (1-5% of snapshots), they represent riskless profit when they occur. For the RV strategy, measure the **half-life of deviation mean-reversion**: how quickly does $p_i - \hat{F}(K_i)$ revert to 0? If $t_{1/2} < 60$ seconds, the strategy is viable within the 5-minute window.

### 6.7 Failure Modes

- **Execution risk:** Multi-leg trades may not fill simultaneously. Use atomic batch orders if the API supports them.
- **Liquidity mismatch:** Some strikes are much less liquid than others. Weight positions by inverse spread.
- **Resolution disputes:** Different contracts may use different price feeds. Verify resolution oracle consistency.

---

## 7. Unified Implementation Architecture

### 7.1 Data Pipeline

```
CEX WebSocket (Binance/Coinbase) --> Spot Price Engine --> Fair Probability Calculator
                                                           |
Polymarket WebSocket --> Order Book Engine --> Signal Generator --> Risk Manager --> Order Router
                              |                                        |
                    Trade Feed Engine --> Volume/OFI Analyzer ---------+
```

### 7.2 Core Components

| Component | Responsibility | Latency Target |
|-----------|---------------|----------------|
| Spot price ingest | Real-time BTC/ETH from 2+ CEXs | < 10ms |
| Fair probability engine | Compute $\hat{\pi}_t$ for all active contracts | < 5ms per contract |
| Polymarket order book ingest | L2 book + trades via WebSocket | < 50ms |
| Signal generator | Run all 6 strategies, produce signals | < 20ms |
| Risk manager | Position limits, exposure checks, drawdown circuit breaker | < 5ms |
| Order router | Smart order routing, limit/market decisions | < 10ms |

**Total system latency target:** < 100ms from spot move to order placement.

### 7.3 Parameter Discovery Pipeline

```
1. Raw data collection (2-4 weeks of tick data from Polymarket + CEX)
   |
2. Feature engineering (compute all signals for all parameter combinations)
   |
3. Walk-forward backtesting (expanding window, no lookahead)
   |
4. Parameter selection (maximize Sharpe, subject to: max drawdown < 15%, hit rate > 52%)
   |
5. Out-of-sample testing (hold-out 30% of data, never used in selection)
   |
6. Paper trading (1-2 weeks live, no real money)
   |
7. Live deployment (small size, scale up based on realized Sharpe)
```

### 7.4 Key Performance Metrics to Track

| Metric | Formula | Target |
|--------|---------|--------|
| Sharpe Ratio (per trade) | $\mu_r / \sigma_r$ (annualized) | > 2.0 |
| Hit Rate | $`\#\{r_i > 0\} / N`$ | > 55% |
| Profit Factor | $\sum r_i^+ / \lvert\sum r_i^-\rvert$ | > 1.5 |
| Max Drawdown | $\max_t (peak_t - trough_t)$ | < 15% of capital |
| Average Trade PnL | $\bar{r}$ net of costs | > 0.01 per share |
| Information Coefficient | $\text{corr}(S_t, r_{t,h})$ | > 0.03 |
| Turnover | Trades per hour | > 10 |

---

## 8. Strategy Selection and Portfolio Construction

### 8.1 Correlation Structure

Not all strategies should be run simultaneously at full size. Their correlation structure matters:

|       | SLA  | PMR  | MPR  | VIT  | TDE  | CRV  |
|-------|------|------|------|------|------|------|
| **SLA** | 1.0  | -0.2 | 0.1  | 0.3  | 0.4  | 0.1  |
| **PMR** |      | 1.0  | 0.0  | -0.1 | -0.3 | 0.0  |
| **MPR** |      |      | 1.0  | 0.2  | 0.3  | 0.7  |
| **VIT** |      |      |      | 1.0  | 0.2  | 0.1  |
| **TDE** |      |      |      |      | 1.0  | 0.2  |
| **CRV** |      |      |      |      |      | 1.0  |

*(Estimated qualitative correlations; measure empirically in backtest)*

### 8.2 Recommended Portfolio Allocation

Using inverse-volatility weighting as a starting point:

| Strategy | Weight | Rationale |
|----------|--------|-----------|
| SLA | 30% | Highest Sharpe, lowest latency risk, but capacity-constrained |
| PMR | 20% | Good diversifier, negative correlation with SLA |
| MPR | 15% | Complex but systematic; requires multi-outcome markets |
| VIT | 15% | Simple and robust; good as a confirmation signal |
| TDE | 10% | Works only near resolution; bursty returns |
| CRV | 10% | Requires multiple strikes; lower frequency |

### 8.3 Ensemble Signal

For any given contract, multiple strategies may fire simultaneously. Combine signals:

$$S_{ensemble} = \sum_i w_i \cdot S_i \cdot \mathbb{1}[\text{strategy } i \text{ is active}]$$

Enter when $|S_{ensemble}| > \theta_{ensemble}$. The ensemble threshold $\theta_{ensemble}$ should be discovered empirically to maximize the combined Sharpe ratio.

---

## 9. Risk Management Framework

### 9.1 Position Limits

- **Per-contract:** Max 5% of portfolio in any single contract
- **Per-strategy:** Max 40% of portfolio in any single strategy
- **Total exposure:** Max 80% of capital deployed (20% cash buffer)

### 9.2 Circuit Breakers

| Trigger | Action |
|---------|--------|
| Daily loss > 3% of capital | Halt all new trades for 1 hour |
| Daily loss > 5% of capital | Halt all trading for the day |
| Single-strategy loss > 2% in 1 hour | Disable that strategy |
| API error rate > 5% | Switch to read-only mode |
| Spot price flash crash (>5% in 1 min) | Close all positions, halt for 30 min |

### 9.3 Model Risk

- **Recalibrate** all statistical parameters every 4 hours using trailing data
- **Monitor** model performance (IC, hit rate) in real-time; if IC drops below 0.01 for 2+ hours, reduce position sizes by 50%
- **A/B test** parameter changes: run new parameters on 10% of capital alongside old parameters

---

## 10. Implementation Roadmap

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| **Phase 1: Data Infrastructure** | Weeks 1-2 | CEX + Polymarket WebSocket ingest, tick database, order book reconstruction |
| **Phase 2: Signal Prototyping** | Weeks 3-4 | Implement all 6 strategies in Python; compute signals on historical data |
| **Phase 3: Backtesting** | Weeks 5-6 | Walk-forward backtest, parameter optimization, performance attribution |
| **Phase 4: Paper Trading** | Weeks 7-8 | Live paper trading with all strategies; monitor fills, latency, slippage |
| **Phase 5: Live Deployment** | Week 9+ | Go live with small capital; scale based on realized performance |

### Recommended Tech Stack

- **Language:** Python (strategy logic) + Rust/C++ (latency-sensitive components)
- **Database:** TimescaleDB or ClickHouse for tick data
- **Message Queue:** Redis or Kafka for inter-component communication
- **Monitoring:** Grafana + Prometheus for real-time dashboards
- **Deployment:** Kubernetes with co-located nodes near exchange servers

---

## Appendix A: Quick Reference -- Strategy Comparison Matrix

|       | SLA | PMR | MPR | VIT | TDE | CRV |
|-------|-----|-----|-----|-----|-----|-----|
| **Type** | Heuristic | Statistical | Statistical | Hybrid | Statistical | Statistical |
| **Edge Source** | Latency | Overreaction | Distribution misfit | Informed flow | Theta decay | Relative value |
| **Holding Period** | 5-60s | 30-180s | 30-120s | 30-240s | 15-60s | 30-120s |
| **Expected Sharpe** | 2.5-4.0 | 1.5-2.5 | 1.5-3.0 | 1.0-2.0 | 1.5-2.5 | 1.0-2.0 |
| **Capacity** | Low | Medium | Low | Medium | Medium | Low |
| **Implementation Difficulty** | Medium | Medium | High | Low-Medium | Medium | High |
| **Data Requirements** | CEX spot + PM book | PM book + trades | Multi-outcome PM | PM trades | PM book + CEX | Multi-strike PM |
| **Key Risk** | Latency | Regime change | Illiquidity | False signals | Pin risk | Execution risk |

## Appendix B: Key Academic References

1. Budish, Cramton & Shim (2015). "The High-Frequency Trading Arms Race: Frequent Batch Auctions as a Market Design Response." *Quarterly Journal of Economics*.
2. Kyle, A. (1985). "Continuous Auctions and Insider Trading." *Econometrica*.
3. Easley, Lopez de Prado & O'Hara (2012). "Flow Toxicity and Liquidity in a High-Frequency World." *Review of Financial Studies*.
4. Page (2012). "The Favorite-Longshot Bias in Parimutuel Markets." *Journal of Economic Literature*.
5. Wolfers & Zitzewitz (2004). "Prediction Markets." *Journal of Economic Perspectives*.
6. Lehmann, B. (1990). "Fads, Martingales, and Market Efficiency." *Quarterly Journal of Economics*.
7. Glosten & Milgrom (1985). "Bid, Ask and Transaction Prices in a Specialist Market with Heterogeneously Informed Traders." *Journal of Financial Economics*.
8. Carr & Madan (1998). "Towards a Theory of Volatility Trading." *Volatility: New Estimation Techniques for Pricing Derivatives*.

## Appendix C: Glossary of Empirical Discovery Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **Grid Search** | Systematically test all combinations of parameter values on a predefined grid | Small parameter spaces (2-4 params), quick initial exploration |
| **Walk-Forward Optimization** | Optimize on trailing window, test on next period, roll forward | All strategies; prevents overfitting |
| **MLE (Maximum Likelihood Estimation)** | Fit distributional parameters by maximizing likelihood of observed data | OU process, volatility models |
| **Bayesian Optimization** | Use Gaussian process surrogate to efficiently search parameter space | Large parameter spaces where grid search is infeasible |
| **Information Coefficient (IC)** | Rank correlation between signal and forward returns | Signal quality assessment |
| **Brier Score** | Mean squared error of probabilistic predictions vs binary outcomes | Calibrating probability models |
| **Ljung-Box Test** | Test for residual autocorrelation in time-series model | Validating OU process, ARMA fits |
| **Bonferroni / BH-FDR** | Multiple testing corrections | When testing many parameter combinations simultaneously |

---

*Document prepared as an implementation blueprint. All strategy hypotheses require empirical validation before live deployment. Past performance (once backtested) does not guarantee future results. Manage risk rigorously.*
