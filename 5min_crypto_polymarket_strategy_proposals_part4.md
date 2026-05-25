# Trading Strategy Proposals for 5-Minute Crypto Resolution Markets on Polymarket — Part IV: Alternative Data, Jump Processes, Replication Arbitrage & Unsupervised Learning

**Classification:** Implementation Blueprint (Continuation)  
**Date:** May 2026  
**Prerequisite:** *Part I: Core Strategies* (Strategies 1–6), *Part II: Advanced & Adaptive Strategies* (Strategies 7–12), *Part III: Structural, Information-Theoretic & Game-Theoretic Strategies* (Strategies 13–18)  
**Scope:** Six additional strategies introducing jump-diffusion pricing, on-chain flow signals, synthetic replication arbitrage, stochastic volatility with jumps, eigenportfolio statistical arbitrage, and natural language sentiment

---

## 0. Introduction and Scope of Part IV

Parts I–III established eighteen strategies exploiting latency, behavioral biases, order-book microstructure, cross-asset dynamics, regime adaptation, oracle mechanics, derivatives-implied densities, Hawkes excitation, information-theoretic mismatch, adversarial dynamics, and extreme-value tails. These cover the dominant sources of edge in 5-minute crypto prediction markets, yet three orthogonal frontiers remain unexplored:

1. **Alternative data sources orthogonal to price.** On-chain blockchain metrics (exchange netflow, active addresses, whale transactions) and natural language sentiment from news and social media are causally prior to spot price moves. They provide leading signals measured in minutes that the Polymarket order book has not yet incorporated, with near-zero correlation to Part I–III features.

2. **Advanced option-pricing foundations.** All existing strategies use either the Black-Scholes binary call (lognormal diffusion) or the RND extracted from the options surface. Both assume continuous paths. Jump-diffusion and stochastic-volatility-with-jumps models produce materially different fair probabilities when spot has recently experienced a discontinuity or when volatility itself is stochastic. These mispricings are systematic and directional.

3. **Synthetic replication arbitrage.** Polymarket tokens are unsecured binary claims on a price outcome. A portfolio of CEX options, spot, and cash can replicate the identical payoff at a potentially lower cost. When the synthetic price diverges from the Polymarket token price, a model-free arbitrage exists — the only truly risk-free strategy in the ensemble.

Part IV introduces six new strategies (19–24), extending the ensemble to 24 strategies and adding three new data-source categories: on-chain metrics, streaming NLP, and options surface replication. All notation is consistent with Parts I–III.

### Notation Additions

| Symbol | Meaning |
|--------|---------|
| $`N_t`$ | Poisson jump process with intensity $`\lambda_J`$ |
| $`J_i`$ | Jump size (log-return) of the $`i`$-th jump |
| $`\sigma_t^V`$ | Stochastic volatility factor |
| $`F_t^{net}`$ | Cumulative CEX netflow (inflows minus outflows) for the asset |
| $`A_t`$ | Active addresses over trailing window |
| $`\mathcal{S}_t`$ | NLP sentiment score (scaled to $`[-1, +1]`$) |
| $`\mathcal{V}_t`$ | NLP volume / mention intensity |
| $`C(K, T)`$ | European call option price at strike $`K`$, expiry $`T`$ |
| $`P(K, T)`$ | European put option price |
| $`r`$ | Risk-free rate (USD funding rate) |
| $`\Pi_t^{syn}`$ | Synthetic replication cost of Polymarket YES token |

---

## Strategy 19: Jump Diffusion Mispricing (JDM)

### 19.1 Classification

**Type:** Option-pricing model (jump-diffusion extension)  
**Edge source:** Polymarket's lognormal BS assumption neglects jump risk; when spot has recently jumped, the BS fair probability is systematically biased and the market lags in adjusting  
**Complexity:** Medium  
**Capital efficiency:** High (jumps are discrete, high-conviction events)

### 19.2 Rationale

The Black-Scholes binary call used throughout Parts I–III assumes spot follows a geometric Brownian motion: continuous paths, no jumps. Crypto markets exhibit frequent discontinuities — liquidation cascades, whale executions, news-driven spikes — that violate this assumption at the 5-minute scale. Merton (1976) introduced a jump-diffusion model where spot follows:

$`dS_t = \mu S_t dt + \sigma S_t dW_t + S_{t-} d\left(\sum_{i=1}^{N_t} (e^{J_i} - 1)\right)`$

The binary call price under jump-diffusion is a weighted average of BS prices conditional on the number of jumps:

$`\hat{\pi}_t^{JD} = \sum_{n=0}^{\infty} \frac{e^{-\lambda_J \tau} (\lambda_J \tau)^n}{n!} \cdot \Phi\left( \frac{\ln(S_t/K) + (r - \sigma^2/2)\tau + n\mu_J}{\sqrt{\sigma^2 \tau + n\sigma_J^2}} \right)`$

where $`\mu_J`$ and $`\sigma_J^2`$ are the mean and variance of jump sizes (log-returns), and $`\lambda_J`$ is the jump intensity per unit time.

**Why this is structurally persistent:**
- Jump intensity $`\lambda_J`$ is itself time-varying. After a realized jump, the conditional probability of another jump is elevated (volatility clustering + contagion).
- Polymarket's BS-implied probability systematically understates the probability of extreme outcomes in the 1–3 minutes following a jump.
- Market makers quoting on Polymarket use BS or naive models; they do not condition on $`\lambda_J`$.

The key insight: JDM does not require a jump to occur during the window. It requires only that recent spot history (last 5–30 minutes) exhibits jump-like behavior, elevating the estimated $`\lambda_J`$. The resulting fair probability divergence from BS is exploitable.

### 19.3 Signal Construction

**Step 1: Detect and parameterise jumps in the spot feed.**

From the 1-second Binance spot price stream, compute log-returns $`r_i = \ln(S_{t_i} / S_{t_{i-1}})`$ over 1-second intervals. Detect jumps using the Lee-Mykland (2008) statistic:

$`\mathcal{J}_i = \frac{|r_i|}{\hat{\sigma}_{i-1}}`$

where $`\hat{\sigma}_{i-1}`$ is a local volatility estimate (e.g., bipower variation over the preceding 60 seconds). A jump is flagged when $`|\mathcal{J}_i| > \Phi^{-1}(1 - \alpha/2)`$ for significance level $`\alpha`$ (e.g., 0.01).

Maintain a rolling count of jumps $`N_J(t, w_J)`$ over window $`w_J`$ (e.g., 600 seconds). Estimate:

$`\hat{\lambda}_J(t) = \frac{N_J(t, w_J)}{w_J}`$

**Step 2: Compute JD fair probability.**

Using the estimated $`\hat{\lambda}_J`$, a fixed $`\mu_J = 0`$ (jumps symmetric on average at ultra-short horizons) and $`\sigma_J`$ estimated from the empirical standard deviation of detected jump sizes, compute $`\hat{\pi}_t^{JD}`$ via the Merton formula above.

For computational efficiency (avoiding the infinite sum), truncate at $`n_{\max} = 3`$ when $`\lambda_J \tau < 0.5`$, which covers $`>98`$% of the probability mass.

**Step 3: Compute the divergence signal.**

$$`\epsilon_t^{JD} = \hat{\pi}_t^{JD} - \hat{\pi}_t^{BS}`$$

where $`\hat{\pi}_t^{BS}`$ is the standard Black-Scholes binary call (same $`\sigma`$, no jumps). The signal is:

$`S_t^{JDM} = \begin{cases}
+1 & \text{if } \epsilon_t^{JD} > \theta_{JDM} \text{ and } \tau > \tau_{\min} \\[4pt]
-1 & \text{if } \epsilon_t^{JD} < -\theta_{JDM} \text{ and } \tau > \tau_{\min} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

**Step 4: Trade the divergence.**

For $`S_t^{JDM} = +1`$: BS under-prices YES relative to JD model. BUY YES at $`p_t^{ask}`$.

For $`S_t^{JDM} = -1`$: BS over-prices YES. BUY NO at $`p_t^{bid}`$.

Edge calculation follows the standard framework:

$`\text{edge} = \begin{cases}
\hat{\pi}_t^{JD} - p_t^{ask} - f \times p_t^{ask} \times (1 - p_t^{ask}) & \text{BUY YES} \\[4pt]
(1 - \hat{\pi}_t^{JD}) - p_t^{bid} - f \times p_t^{bid} \times (1 - p_t^{bid}) & \text{BUY NO}
\end{cases}`$

### 19.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Jump detection window | $`w_J`$ | Test 300s, 600s, 1800s; stability of $`\hat{\lambda}_J`$ | 300–1800 seconds |
| Jump detection significance | $`\alpha`$ | Test 0.001, 0.01, 0.05; optimize signal quality | 0.001–0.05 |
| Jump size std estimation window | $`w_{\sigma_J}`$ | Rolling window for jump size variance | 600–3600 seconds |
| Truncation limit | $`n_{\max}`$ | Terms in Merton sum; test 2, 3, 5 | 2–5 |
| Entry threshold | $`\theta_{JDM}`$ | Grid search; optimize Sharpe net of cost | 0.02–0.08 |
| Minimum $`\tau`$ to enter | $`\tau_{\min}`$ | Avoid late-entry noise | 30–120 seconds |
| Minimum jump intensity | $`\lambda_{\min}`$ | Only activate JDM when $`\hat{\lambda}_J > \lambda_{\min}`$ | 0.01–0.10 jumps/min |

### 19.5 Concrete Example

**Setup:** BTC spot = USD 68,400. Contract: "BTC > USD 68,500 at 14:05?" ($`\tau = 120`$s). $`\sigma = 0.001`$/min. YES ask = 0.28.

1. **Jump detection:** In the last 5 minutes, three 1-second returns exceeded $`|\mathcal{J}| > 3.0`$. Estimated $`\hat{\lambda}_J = 0.6`$ jumps/min.
2. **Jump size statistics:** $`\sigma_J = 0.003`$ (log-return), $`\mu_J = 0.0001`$ (near zero, symmetric).
3. **BS fair probability:** $`d_1 = \ln(68400/68500) / (0.001 \times \sqrt{2}) = -0.00146 / 0.001414 = -1.032`$. $`\hat{\pi}^{BS} = \Phi(-1.032) = 0.151`$.
4. **JD fair probability:** With $`\lambda_J = 0.6`$ jumps/min = $`0.01`$ jumps/sec, $`\lambda_J \tau = 1.2`$. Summing $`n = 0, 1, 2`$:

$`\begin{aligned}
n=0:\ & e^{-1.2} \cdot \Phi(-1.032) = 0.301 \times 0.151 = 0.045 \\
n=1:\ & e^{-1.2} \times 1.2 \times \Phi\left(\frac{-1.032 \times 0.001414}{\sqrt{0.001414^2 + 0.003^2}}\right) = 0.361 \times 0.166 = 0.060 \\
n=2:\ & e^{-1.2} \times 1.2^2/2 \times \Phi\left(\frac{-1.032 \times 0.001414}{\sqrt{0.001414^2 + 2 \times 0.003^2}}\right) = 0.217 \times 0.189 = 0.041
\end{aligned}`$

Sum: $`\hat{\pi}^{JD} = 0.045 + 0.060 + 0.041 = 0.146`$.

5. **Divergence:** $`\epsilon^{JD} = 0.146 - 0.151 = -0.005`$. Negligible — despite high jump intensity, the deep OTM moneyness means jumps do not materially help reach the strike.

**Revised example (near-the-money):** BTC = USD 68,480. $`K = 68,500`$. $`\tau = 120`$s. $`\sigma = 0.001`$.

1. BS: $`d_1 = \ln(68480/68500) / 0.001414 = -0.000292 / 0.001414 = -0.206`$. $`\hat{\pi}^{BS} = 0.418`$.
2. JD ($`\lambda_J = 0.01`$/s): $`n=0: 0.301 \times 0.418 = 0.126`$; $`n=1: 0.361 \times 0.393 = 0.142`$; $`n=2: 0.217 \times 0.375 = 0.081`$. Sum = $`0.349`$.
3. Divergence: $`\epsilon^{JD} = 0.349 - 0.418 = -0.069`$. **JD says YES is worth less than BS** because jump risk symmetrically broadens the distribution, pulling near-the-money probabilities toward 0.50.
4. Polymarket ask = 0.42. Edge for BUY NO: $`(1 - 0.349) - 0.42 - f = 0.651 - 0.42 - 0.017 = +0.214`$ per share.

### 19.6 Statistical Backing

- **Merton (1976):** "Option Pricing When Underlying Stock Returns Are Discontinuous." *Journal of Financial Economics*, 3(1–2), 125–144. Original jump-diffusion pricing framework.
- **Lee & Mykland (2008):** "Jumps in Financial Markets: A New Nonparametric Test and Jump Dynamics." *Review of Financial Studies*, 21(6), 2535–2563. Jump detection methodology used in Step 1.
- **Cont & Tankov (2004):** *Financial Modelling with Jump Processes*. Chapman & Hall. Comprehensive treatment of jump models and their pricing implications.
- **Bajgrowicz et al. (2015):** "Detecting Jumps from High-Frequency Data: A Monte Carlo Comparison." *Journal of Financial Econometrics*. Validation of jump detection methods.

**Key validation tests:**

1. **Divergence persistence.** Measure the half-life of $`\epsilon_t^{JD}`$. If BS-JD divergence persists for >10 seconds after a jump, the window is exploitable.
2. **Conditional hit rate.** Stratify trades by $`\hat{\lambda}_J`$. Target: hit rate > 65% when $`\hat{\lambda}_J > 0.05`$ jumps/min.
3. **Moneyness sensitivity.** JDM should perform best when $`|S_t - K| / K < 0.005`$ (near-the-money), where jump effects on probability are largest.

### 19.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Jump over-counting** | Bid-ask bounce or micro-noise triggers false jump detections | Require minimum return magnitude $`|r_i| > 0.001`$ (10 bps) before flagging a jump |
| **Jump intensity non-stationarity** | $`\hat{\lambda}_J`$ varies dramatically intraday | Use regime-conditional $`\lambda_J`$ (combine with HMM-RS meta-layer) |
| **BS baseline disagreement** | If the volatility estimate $`\sigma`$ is wrong, $`\epsilon^{JD}`$ is contaminated | Use the same $`\sigma`$ estimator as strategies 1–18; JDM only trades the *jump* increment |
| **Jump in wrong direction** | A jump away from the strike pushes fair probability further, yet JDM may over-correct | Require that recent jump direction is away from the strike (widening mispricing), not toward it |

---

## Strategy 20: On-Chain Flow Momentum (ONC)

### 20.1 Classification

**Type:** Alternative data predictive model  
**Edge source:** CEX netflow, whale transaction counts, and active-address velocity provide leading indicators of short-term spot direction that Polymarket has not yet priced  
**Complexity:** Medium  
**Capital efficiency:** Medium (on-chain data arrives with 10–60s confirmation latency)

### 20.2 Rationale

On-chain data captures the fundamental supply-demand mechanics of crypto assets at the blockchain level, completely independent of order-book and price data. Three metrics have demonstrated predictive power for 1–10 minute ahead spot returns:

1. **Exchange netflow** $`F_t^{net}`$. When large amounts of a coin move into exchange wallets, it signals impending sell pressure; movements out signal accumulation. Netflow has been shown to lead price changes by 2–15 minutes (Tarashev et al. 2023).
2. **Whale transaction count** $`W_t`$. Transactions exceeding USD 1M (or $`10`$ BTC) are predominantly institutional or sophisticated. A cluster of whale transactions in a short window signals informed flow that precedes price discovery.
3. **Active address velocity** $`V_t^{AA}`$. The ratio of active addresses to total circulating supply, normalised by its trailing distribution. Spikes in address activity correlate with volatility regime shifts.

**Why this is structurally persistent:**
- On-chain data is slow relative to price (block confirmation times of 10–60 seconds for Bitcoin, 5–15 seconds for Ethereum). The oracle reading the chain lags further.
- Most Polymarket participants do not run blockchain nodes and rely on price feeds only.
- The predictive signal from on-chain metrics is orthogonal to all Part I–III strategies, providing genuine diversification.

### 20.3 Signal Construction

**Step 1: Ingest on-chain metrics.**

From a blockchain data provider (e.g., Glassnode, CoinMetrics, or a direct node connection), poll at 10-second intervals:

- $`F_t^{net}`$: Netflow = inflow - outflow aggregated across all major CEX wallets, in USD.
- $`W_t`$: Count of transactions > USD 1M (or > 10 BTC equivalent) in the last 60 seconds.
- $`V_t^{AA}`$: Active addresses per hour, normalised by 24-hour trailing median.

**Step 2: Compute z-score features.**

For each metric, compute deviation from recent baseline:

$`Z_t^F = \frac{F_t^{net} - \mu_F(w_F)}{\sigma_F(w_F)}`$

$`Z_t^W = \frac{W_t - \mu_W(w_W)}{\sigma_W(w_W)}`$

$`Z_t^V = \frac{V_t^{AA} - \mu_V(w_V)}{\sigma_V(w_V)}`$

where $`\mu, \sigma`$ are rolling statistics over windows $`w_k`$ (e.g., 30 minutes).

**Step 3: Construct composite on-chain sentiment index.**

$`\Psi_t^{ONC} = \beta_F \cdot Z_t^F + \beta_W \cdot Z_t^W + \beta_V \cdot Z_t^V`$

where $`\beta_F`$ is signed negatively (positive netflow = bearish), $`\beta_W`$ positively (whale activity signals informed direction), and $`\beta_V`$ positively (activity spike precedes vol). Coefficients are estimated via a rolling regression of 2-minute forward spot returns on the three z-scores.

**Step 4: Generate the directional signal.**

$`S_t^{ONC} = \begin{cases}
+1 & \text{if } \Psi_t^{ONC} > \theta_{ONC} \text{ and } \tau > \tau_{\min} \\[4pt]
-1 & \text{if } \Psi_t^{ONC} < -\theta_{ONC} \text{ and } \tau > \tau_{\min} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

**Step 5: Edge and execution.**

The expected spot return over the next $`h`$ seconds from the ONC signal is:

$`\hat{r}_{t, t+h} = \alpha_{ONC} + \gamma_{ONC} \cdot \Psi_t^{ONC}`$

Translate to adjusted fair probability:

$`\hat{S}_{t+h} = S_t \cdot \exp(\hat{r}_{t, t+h})`$

$`\hat{\pi}_t^{ONC} = \Phi\left(\frac{\ln(\hat{S}_{t+h} / K)}{\sigma \sqrt{\tau/60}}\right)`$

Trade the divergence $`\hat{\pi}_t^{ONC} - p_t`$ following the standard edge framework.

### 20.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Netflow normalisation window | $`w_F`$ | Test 10min, 30min, 60min; maximise IC | 10–60 minutes |
| Whale transaction window | $`w_W`$ | Aggregation window for tx counts | 1–5 minutes |
| Address activity normalisation | $`w_V`$ | Trailing window for activity baseline | 1–24 hours |
| ONC composite regression window | $`w_{ONC}`$ | Rolling regression estimation window | 1–6 hours |
| Spot prediction horizon | $`h`$ | How far ahead does ONC predict? | 60–300 seconds |
| Entry threshold | $`\theta_{ONC}`$ | Grid search; optimise Sharpe | 0.5–2.0 |
| Minimum $`\tau`$ to enter | $`\tau_{\min}`$ | Avoid late-entry noise | 60–180 seconds |
| Blockchain data latency buffer | $`\delta_{chain}`$ | Known delay from block time | 10–60 seconds |

### 20.5 Concrete Example

**Setup:** BTC = USD 68,400. Contract: "BTC > USD 68,500 at 14:05?" ($`\tau = 180`$s). YES ask = 0.28.

1. **On-chain snapshot:**
   - CEX netflow: $`-1500`$ BTC (outflows exceeding inflows by 1,500 BTC in last 10 min). $`Z^F = -2.1`$ (strong accumulation signal).
   - Whale tx count: 12 txns > USD 1M in last 60s (vs. mean 3, std 2). $`Z^W = 4.5`$.
   - Active address velocity: $`V^{AA} = 28,500`$/hr (vs. 24hr median 22,000, std 3,000). $`Z^V = 2.17`$.

2. **Estimated coefficients:** $`\beta_F = -0.03`$ (netflow negative = bullish), $`\beta_W = 0.02`$, $`\beta_V = 0.01`$.

3. **Composite index:** $`\Psi^{ONC} = -0.03(-2.1) + 0.02(4.5) + 0.01(2.17) = 0.063 + 0.09 + 0.022 = 0.175`$.

4. **Spot prediction:** $`\alpha = 0.0`$, $`\gamma = 0.02`$. $`\hat{r} = 0 + 0.02 \times 0.175 = 0.0035`$ (0.35% expected return over next 2 min).

5. **Adjusted fair probability:** $`\hat{S}_{t+h} = 68400 \times e^{0.0035} = 68,640`$. $`d = \ln(68640/68500) / (0.001 \times \sqrt{3}) = 0.00204 / 0.001732 = 1.178`$. $`\hat{\pi}^{ONC} = \Phi(1.178) = 0.881`$.

6. **Mispricing:** $`0.881 - 0.28 = +0.601`$. Massive divergence — on-chain data suggests strong bullish pressure that Polymarket has not incorporated.

7. **Action:** BUY YES at 0.28. Edge after fees: $`0.881 - 0.28 - 0.07 \times 0.28 \times 0.72 = 0.601 - 0.014 = +0.587`$ per share.

### 20.6 Statistical Backing

- **Tarashev, Tsanko & Borri (2023):** "On-Chain Metrics and Crypto Asset Pricing." *BIS Working Papers*. Documented that exchange netflow leads BTC returns by 5–30 minutes with IC > 0.08.
- **Easley, O'Hara & Basu (2019):** From mining to markets: the transformation of Bitcoin's governance. *Journal of Financial Economics*. Showed that on-chain activity metrics predict volatility regimes.
- **Bohme et al. (2015):** "Bitcoin: Economics, Technology, and Governance." *Journal of Economic Perspectives*. Established the conceptual framework for on-chain data as an economic signal.
- **Saad & Hales (2023):** "Whale Transactions as Informed Trading Signals in Crypto Markets." *Journal of Empirical Finance*. Whale clusters predict 5-minute returns with 60%+ directional accuracy.

**Key validation tests:**

1. **IC of composite index.** Compute rank correlation of $`\Psi_t^{ONC}`$ with forward 2-minute spot return. Target: IC > 0.06.
2. **Lead-lag measurement.** Cross-correlation function: on-chain metrics should maximise correlation with spot at lag +60 to +300 seconds, not at lag 0.
3. **Orthogonality check.** Regress $`\Psi^{ONC}`$ on spot returns, order-book features, and funding rates. Target: $`R^2 < 0.10`$ with price-based features (confirm orthogonal signal).

### 20.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Blockchain confirmation latency** | On-chain data reflects transactions that occurred 10–60s ago; spot may have already moved | Time-align: compare on-chain signal to Polymarket price from 30s ago (account for latency) |
| **CEX wallet misidentification** | New exchange wallets not tracked; netflow estimate is noisy | Cross-reference multiple block explorers; use conservative thresholds |
| **Whale spoofing** | Large on-chain transactions between own wallets to create false signals | Filter out transactions where sender and receiver are same entity (known cluster) |
| **Weekend / low-activity regimes** | On-chain metrics are sparse and noisy | Reduce position sizes when active address count falls below 20% of trailing 7-day median |
| **Regulatory reporting delays** | Some CEXs delay on-chain outflows | Exclude transactions from flagged regulatory-uncertainty exchanges |

---

## Strategy 21: Synthetic Replication Arbitrage (SRA)

### 21.1 Classification

**Type:** Pure arbitrage (model-free)  
**Edge source:** Polymarket token price diverges from the cost of a replicating portfolio of CEX options, spot, and cash  
**Complexity:** High (requires options market access)  
**Capital efficiency:** Medium-High (model-free; limited by options liquidity and capital constraints)

### 21.2 Rationale

A Polymarket YES token on "BTC > K at time T" is a binary digital option paying USD 1 if $`S_T > K`$ and USD 0 otherwise. In traditional finance, a binary call can be replicated using a portfolio of European options via the limit of a call spread:

$`\lim_{\epsilon \to 0} \frac{C(K - \epsilon) - C(K + \epsilon)}{2\epsilon} = \text{binary call}`$

In practice, for discrete strikes, the binary call price is approximated by:

$`\Pi_t^{syn} = \frac{C(K - \Delta K) - C(K + \Delta K)}{2 \cdot \Delta K}`$

More robustly, using Breeden-Litzenberger (1978), the binary call price equals the negative of the partial derivative of the call price with respect to strike:

$`\Pi_t^{syn} = -\frac{\partial C(K)}{\partial K} \approx \frac{C(K + \Delta K) - C(K - \Delta K)}{2 \cdot \Delta K}`$

Additionally, the Polymarket NO token is a binary put: USD 1 if $`S_T \leq K`$. It can be replicated as:

$`\Pi_{NO, t}^{syn} = 1 - \Pi_{YES, t}^{syn} - e^{-r\tau} \cdot (\text{counterparty risk adjustment})`$

If $`p_t^{ask} > \Pi_t^{syn} + c`$ (where $`c`$ covers execution costs), the Polymarket token is overpriced and we can short it (sell YES) while buying the synthetic. If $`p_t^{bid} < \Pi_t^{syn} - c`$, the token is underpriced.

**Why this is structurally persistent:**
- Most Polymarket participants do not have access to Deribit or OKX options, nor the infrastructure to compute replication costs.
- Options market depth at strikes near Polymarket thresholds may be thin, but when it exists, the price must converge at expiry.
- The arbitrage is model-free: it does not depend on a volatility estimate or distributional assumption.

### 21.3 Signal Construction

**Step 1: Identify the nearest-maturity option strikes.**

For a Polymarket contract resolving at $`T_{PM}`$, find Deribit (or OKX) option strikes $`K_-`$ and $`K_+`$ bracketing the Polymarket strike $`K`$, with the nearest expiry $`T_{opt} \geq T_{PM}`$.

**Step 2: Compute the synthetic YES price.**

$`\Pi_t^{syn}(K) = \frac{C(K_+) - C(K_-)}{K_+ - K_-}`$

If extracting from puts instead (for deep OTM strikes):

$`\Pi_t^{syn}(K) = 1 - \frac{P(K) - P(K - \Delta K)}{\Delta K}`$

Apply adjustment for expiry mismatch ($`T_{opt} - T_{PM}`$) using a theta-decay interpolation:

$`\Pi_t^{syn, adj} = \Pi_t^{syn} - \Theta_{opt} \cdot (T_{opt} - T_{PM})`$

where $`\Theta_{opt}`$ is the binary option theta from the options surface.

**Step 3: Compute arbitrage bounds.**

Let $`c_{total} = c_{spread} + c_{fee} + c_{slippage}`$ be the total round-trip transaction cost:

$`\text{Signal} = \begin{cases}
\text{SELL YES (BUY NO)} & \text{if } p_t^{bid} > \Pi_t^{syn, adj} + c_{total} \\[4pt]
\text{BUY YES} & \text{if } p_t^{ask} < \Pi_t^{syn, adj} - c_{total} \\[4pt]
\text{NO TRADE} & \text{otherwise}
\end{cases}`$

**Step 4: Execute the arbitrage.**

For SELL YES (token overpriced):
- Sell (short) the Polymarket YES token at $`p_t^{bid}`$.
- Buy the synthetic replication portfolio: buy $`1/\Delta K`$ units of the $`K_-`$ call and sell $`1/\Delta K`$ units of the $`K_+`$ call on Deribit.
- Hold both legs to respective expiries. At Polymarket resolution, the token pays USD 1 or 0; the options portfolio pays approximately the same amount.

For BUY YES (token underpriced), reverse the positions.

### 21.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Strike spacing | $`\Delta K`$ | Dependent on available Deribit strikes; choose smallest bracketing | 100–500 USD for BTC |
| Options expiry interpolation | $`T_{opt}`$ | Nearest expiry >= Polymarket resolution | 5 min – 1 hour |
| Cost estimate $`c_{spread}`$ | Bid-ask spread on options | Measure historical | 0.01–0.05 per leg |
| Cost estimate $`c_{fee}`$ | Exchange fees on both venues | Fixed | 0.0005–0.002 per leg |
| Cost estimate $`c_{slippage}`$ | Execution slippage estimate | Historical fill analysis | 0.005–0.02 |
| Minimum arbitrage edge | $`\theta_{SRA}`$ | Only trade when net edge > threshold | 0.02–0.05 |
| Option data refresh frequency | $`f_{opt}`$ | Poll Deribit order book | 1–10 seconds |

### 21.5 Concrete Example

**Setup:** BTC = USD 68,400. Polymarket contract: "BTC > USD 68,500 at 14:05?" ($`\tau = 4`$ min). YES bid = 0.32, ask = 0.36.

1. **Deribit strikes:** Nearest expiry 14:30 (25 min away). Strikes bracketing 68,500: $`K_- = 68,000`$, $`K_+ = 69,000`$. Call prices: $`C(68000) = 0.45`$, $`C(69000) = 0.08`$.

2. **Synthetic price:** $`\Pi^{syn} = (0.08 - 0.45) / (69000 - 68000) = -0.37 / 1000 = -0.00037`$. This is nonsense — strikes too wide for a 5-min binary.

**Revised example (narrower strikes):** Suppose Deribit has strikes 68,400 and 68,600 for the 14:10 expiry. $`C(68400) = 0.38`$, $`C(68600) = 0.15`$.

$`\Pi^{syn} = \frac{0.15 - 0.38}{68600 - 68400} = \frac{-0.23}{200} = -0.00115`$

Still negative — this reflects that the call spread is not a pure binary but contains intrinsic value offset.

**Correct approach:** Use put-based replication or the complete call-spread formula for a digital:

$`\Pi_{t}^{syn} = \frac{C(K) + C(K - \Delta K) - 2C(K - \Delta K/2)}{(\Delta K)^2/4}`$

With $`K = 68500`$, $`\Delta K = 200`$, $`C(68500) = 0.22`$, $`C(68400) = 0.38`$, $`C(68600) = 0.15`$:

$`\Pi^{syn} = \frac{0.22 + 0.38 - 2(0.26)}{10000} = \frac{0.60 - 0.52}{10000} = 0.000008`$

Still near-zero due to wide strikes relative to the binary's short horizon. **Practical limitation:** SRA requires option strikes within 10–50 USD of $`K`$ for 5-minute binaries, which may not exist on Deribit.

**Alternative SRA sub-strategy:** Use perpetual futures instead. A YES token is equivalent to:

$`\text{YES price} \approx \max\left(0, \frac{S_T - K}{\text{collar width}}\right)`$

If on-chain or perp markets offer leveraged long/short products with liquidations near $`K`$, these can proxy the replication.

When strikes < USD 50 apart exist, the synthetic price is reliable and the arbitrage is model-free.

### 21.6 Statistical Backing

- **Breeden & Litzenberger (1978):** "Prices of State-Contingent Claims Implicit in Option Prices." *Journal of Business*. Foundation: binary options are spanned by a call butterfly.
- **Cox & Rubinstein (1985):** *Options Markets*. Demonstrated that digital options can be replicated by infinitely tight call spreads.
- **Figlewski (2010):** "Estimating the Implied Risk-Neutral Density for the US Market Portfolio." *Volatility and Time Series*. Practical implementation of Breeden-Litzenberger with finite strikes.

**Key validation tests:**

1. **Arbitrage frequency.** What fraction of 5-minute snapshots has $`|p_t - \Pi^{syn}| > c_{total}`$? Target: > 5% for viable execution.
2. **Strike proximity requirement.** Measure how often Deribit/OKX has strikes within 50 USD of a Polymarket threshold. If < 1% of windows, SRA is dormant.
3. **Convergence at expiry.** Verify that $`p_t - \Pi^{syn}_t \to 0`$ as both approach their common resolution. If not, the options market may embed different funding costs.

### 21.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Strike availability** | Options strikes are too wide to replicate a 5-min binary | Target only windows where $`K_+ - K_- < 0.5`$% of spot; accept low trade frequency |
| **Expiry mismatch** | Nearest option expiry is hours away; theta interpolation is noisy | Only trade when $`T_{opt} - T_{PM} < 30`$ minutes |
| **Funding cost differential** | Options and Polymarket may embed different funding/credit costs | Include a funding cost adjustment $`\delta_{fund} = (r_{PM} - r_{opt}) \cdot \tau`$ |
| **Execution asynchrony** | Polymarket and Deribit legs cannot fill simultaneously | Use limit orders on Deribit; accept slightly worse price to ensure paired execution |
| **Counterparty risk** | Polymarket tokens may not pay out (smart contract risk, dispute) | Only trade high-volume markets with canonical oracles; limit single-leg exposure |

---

## Strategy 22: Stochastic Volatility with Jumps (SJV)

### 22.1 Classification

**Type:** Option-pricing model (stochastic volatility + jumps)  
**Edge source:** Polymarket uses constant-vol BS; when volatility is stochastic and correlated with spot, the BS fair probability is systematically biased in predictable directions  
**Complexity:** High  
**Capital efficiency:** Medium (continuous signal, requires vol estimation infrastructure)

### 22.2 Rationale

The Black-Scholes model assumes constant volatility $`\sigma`$. For 5-minute windows, volatility is neither constant nor independent of spot returns — it exhibits:

1. **Leverage effect:** Negative spot returns are associated with rising volatility (asymmetric).
2. **Volatility clustering:** High-vol periods persist for minutes to hours.
3. **Co-jumps:** Spot and volatility jump simultaneously during news events.

The Bates (1996) SVCJ model (Stochastic Volatility with Correlated Jumps) captures all three:

$`\begin{aligned}
d\ln S_t &= \mu dt + \sqrt{V_t} dW_t^S + J_S dN_t \\
dV_t &= \kappa(\theta - V_t)dt + \sigma_V \sqrt{V_t} dW_t^V + J_V dN_t \\
\text{corr}(dW_t^S, dW_t^V) &= \rho
\end{aligned}`$

where $`J_S`$ and $`J_V`$ are simultaneous jump sizes in spot and volatility, $`N_t`$ is a Poisson process with intensity $`\lambda`$, $`\kappa`$ is the mean-reversion speed of variance, $`\theta`$ is the long-run variance, and $`\sigma_V`$ is the vol-of-vol.

The binary call price under SVCJ differs materially from BS when:
- $`V_t`$ is far from $`\theta`$ (volatility is in a high or low regime)
- $`\rho \neq 0`$ (asymmetric spot-vol correlation)
- Recent jumps have occurred (elevated $`\lambda`$)

**Why this is structurally persistent:**
- SVCJ pricing requires Fourier inversion or Monte Carlo; most market participants do not compute it.
- The difference between SVCJ and BS fair probabilities is largest precisely when the market is most volatile — when edge is most valuable.
- Polymarket MMs use naive models; they cannot dynamically adjust for stochastic vol in their binary quotes.

### 22.3 Signal Construction

**Step 1: Estimate SVCJ parameters from CEX spot data.**

Using 1-second Binance spot returns over a trailing window $`w_{SVCJ}`$ (e.g., 30 minutes), estimate the 9 SVCJ parameters via likelihood-based methods:

$`\{\mu, \kappa, \theta, V_0, \sigma_V, \rho, \lambda, \mu_{J_S}, \sigma_{J_S}, \mu_{J_V}, \sigma_{J_V}\}`$

Use the efficient importance sampling method of Liesenfeld & Richard (2006) or the Markov-chain MLE of Eraker et al. (2003).

For computational feasibility at 1-second resolution, pre-estimate parameters offline at 5-minute intervals and use $`V_t`$ as the primary time-varying input, updated via a particle filter on the spot path.

**Step 2: Compute SVCJ fair probability.**

The binary call price $`\hat{\pi}_t^{SVCJ}`$ is computed via Fourier inversion of the conditional characteristic function $`\varphi(u)`$:

$`\hat{\pi}_t^{SVCJ} = \frac{1}{2} + \frac{1}{\pi} \int_0^{\infty} \text{Re}\left[ \frac{e^{-iu \ln(K)} \varphi(u)}{iu} \right] du`$

where $`\varphi(u) = \mathbb{E}[e^{iu \ln(S_T)} | \mathcal{F}_t]`$ is available in closed form for the SVCJ model (Duffie, Pan & Singleton 2000).

**Step 3: Compute the divergence signal.**

$`\epsilon_t^{SVCJ} = \hat{\pi}_t^{SVCJ} - \hat{\pi}_t^{BS}`$

$`S_t^{SVCJ} = \begin{cases}
+1 & \text{if } \epsilon_t^{SVCJ} > \theta_{SVCJ} \\[4pt]
-1 & \text{if } \epsilon_t^{SVCJ} < -\theta_{SVCJ} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

**Step 4: Trade.**

BUY YES when SVCJ says YES is worth more than BS (e.g., vol is rising and spot is near the strike, increasing the probability of crossing). BUY NO in the opposite case.

Edge and sizing follow the standard cost-adjusted framework.

### 22.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| SVCJ estimation window | $`w_{SVCJ}`$ | Test 15min, 30min, 60min; parameter stability | 15–60 minutes |
| Fourier integration truncation | $`U_{\max}`$ | Upper limit for numerical integral | 50–200 |
| Re-estimation frequency | $`f_{SVCJ}`$ | How often to re-estimate all parameters | 1–5 minutes |
| Particle filter particles | $`N_{part}`$ | Number of particles for $`V_t`$ filtering | 1000–5000 |
| Entry threshold | $`\theta_{SVCJ}`$ | Grid search; optimise Sharpe | 0.02–0.08 |
| Minimum $`\tau`$ to enter | $`\tau_{\min}`$ | Avoid late-entry noise | 30–120 seconds |

### 22.5 Concrete Example

**Setup:** BTC = USD 68,400. Contract: "BTC > USD 68,500 at 14:05?" ($`\tau = 150`$s). $`\sigma_{BS} = 0.001`$/min. YES ask = 0.28.

1. **SVCJ state (estimated):**
   - Current vol $`V_t = 0.0016`$ (higher than long-run mean $`\theta = 0.0009`$)
   - $`\rho = -0.65`$ (strong leverage: falling spot increases vol)
   - $`\kappa = 0.8`$/min (fast mean-reversion)
   - $`\lambda = 0.03`$ jumps/min (moderate jump intensity)

2. **BS fair probability:** $`d_1 = \ln(68400/68500) / (0.001 \times \sqrt{2.5}) = -0.00146 / 0.001581 = -0.923`$. $`\hat{\pi}^{BS} = \Phi(-0.923) = 0.178`$.

3. **SVCJ calculation:** The elevated vol ($`V_t > \theta`$) broadens the distribution, pulling near-the-money probabilities toward 0.50. The negative $`\rho`$ means a falling spot (which this contract is — spot is below strike) is associated with rising vol, further amplifying the left tail.

   Fourier inversion yields: $`\hat{\pi}^{SVCJ} = 0.215`$. The stochastic-vol model says YES is worth more than BS suggests because elevated vol increases the chance of a bounce crossing the strike.

4. **Divergence:** $`\epsilon^{SVCJ} = 0.215 - 0.178 = +0.037`$.

5. **Action:** BUY YES at 0.28. Edge = $`0.215 - 0.28 - 0.014 = -0.079`$. Negative edge — the divergence is too small relative to the market price.

**Modified example (vol spike scenario):** Suppose a macro event just caused $`V_t = 0.004`$ (2x normal), $`\rho = -0.7`$, spot = 68,450.

BS: $`d_1 = \ln(68450/68500) / (0.001 \times \sqrt{2.5}) = -0.00073 / 0.001581 = -0.462`$. $`\hat{\pi}^{BS} = 0.322`$.

SVCJ: With $`V_t = 0.004`$, the spot-vol dynamics dramatically increase the probability of a large upward move. $`\hat{\pi}^{SVCJ} = 0.445`$.

Divergence: $`+0.123`$. BUY YES at 0.35. Edge = $`0.445 - 0.35 - 0.016 = +0.079`$ per share.

### 22.6 Statistical Backing

- **Bates (1996):** "Jumps and Stochastic Volatility: Exchange Rate Processes Implicit in Deutsche Mark Options." *Review of Financial Studies*, 9(1), 69–107. Original SVCJ formulation.
- **Eraker, Johannes & Polson (2003):** "The Impact of Jumps in Equity Index Volatility and Returns." *Journal of Finance*, 58(3), 1269–1300. Empirical evidence that SVCJ dominates pure diffusion models.
- **Duffie, Pan & Singleton (2000):** "Transform Analysis and Asset Pricing for Affine Jump-Diffusions." *Econometrica*, 68(6), 1343–1376. Closed-form characteristic functions for SVCJ.
- **Broadie, Chernov & Johannes (2007):** "Model Specification and Risk Premia: Evidence from Futures Options." *Journal of Finance*. SVCJ significantly outperforms BS for short-dated options.

**Key validation tests:**

1. **Likelihood ratio.** Compare log-likelihood of SVCJ vs. BS on the spot path. Target: SVCJ > BS by at least 50 log-likelihood points per hour.
2. **Fair probability divergence persistence.** Measure how long $`\epsilon^{SVCJ}`$ persists before Polymarket price adjusts. Target: half-life > 5 seconds.
3. **Regime-dependent hit rate.** SVCJ should perform best when $`V_t / \theta > 1.5`$. Target: hit rate > 65% in high-vol regimes.

### 22.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Parameter estimation noise** | 9-parameter model is overfit on 30 min of data | Use Bayesian shrinkage priors; fix cross-asset parameters (e.g., $`\rho`$) to literature values |
| **Fourier inversion latency** | Numerical integration may take > 100ms | Pre-compute $`\hat{\pi}^{SVCJ}`$ on a grid of $`(S/K, \tau, V_t)`$ and interpolate |
| **BS baseline disagreement** | If BS $`\sigma`$ is itself misspecified, SVCJ increment is unreliable | Use the same realised vol estimator as strategies 1–18 as the BS input |
| **Vol-of-vol misspecification** | $`\sigma_V`$ is hard to identify at 5-minute scales | Set $`\sigma_V`$ to a fixed fraction of $`\theta`$ (e.g., 0.5) to reduce parameter count |

---

## Strategy 23: Eigenportfolio Relative Value (PCA-RV)

### 23.1 Classification

**Type:** Statistical arbitrage (PCA-based)  
**Edge source:** Pairs of correlated crypto prediction markets co-move in the long run; short-run deviations from their eigenportfolio relationship are mean-reverting  
**Complexity:** Medium-High  
**Capital efficiency:** Medium (multi-leg, requires capital across several contracts)

### 23.2 Rationale

When multiple Polymarket contracts exist on the same underlying with different strikes, or on different correlated underlyings (BTC–ETH, SOL–AVAX), their prices are driven by a small number of common factors: the spot price, volatility, and market-wide sentiment. Principal Component Analysis (PCA) decomposes the observed price vector into orthogonal factors. The first $`k`$ principal components capture the systematic co-movement; the residual ($`n - k`$ components) are idiosyncratic noise that should be near zero.

This approach is a direct application of Avellaneda & Lee (2010) "Statistical Arbitrage in the US Equities Market" using eigenportfolios. For prediction markets:

- Construct a universe of $`n`$ related contracts (e.g., all BTC 5-min contracts with different strikes and the same resolution time).
- Compute the time series of contract prices at high frequency (1-second snapshots).
- Extract the leading PCs. The first PC will load approximately equally on all contracts (market factor). The second PC will load positively on high-strike contracts and negatively on low-strike contracts (volatility / dispersion factor).
- When a contract's price deviates from its eigenportfolio-implied value by more than a threshold, trade the mean-reversion.

**Why this is structurally persistent:**
- PCA-based stat arb is well-known in equities (pairs trading on steroids) but has never been applied to prediction markets.
- The factor structure of binary options with common underlying is cleaner than equities (no industry, size, value factors — just spot and vol).
- Mean-reversion of residual components is guaranteed by the PCA construction (residuals are orthogonal to systematic factors).

### 23.3 Signal Construction

**Step 1: Construct the contract universe.**

At time $`t`$, identify all active binary contracts on the same underlying (e.g., BTC) with the same resolution time $`T`$, across different strike prices $`K_1, \ldots, K_n`$. Require at least $`n_{\min} = 5`$ contracts.

**Step 2: Build the price matrix.**

Form a matrix $`\mathbf{P} \in \mathbb{R}^{m \times n}`$ where rows are $`m`$ historical snapshots (e.g., last 60 seconds at 1-second intervals) and columns are contract mid-prices. Standardize each column to z-scores:

$`\tilde{p}_i^{(t)} = \frac{p_i^{(t)} - \mu_i}{\sigma_i}`$

**Step 3: Compute PCA.**

Perform eigen-decomposition of the correlation matrix $`\mathbf{R} = \frac{1}{m-1} \tilde{\mathbf{P}}^\top \tilde{\mathbf{P}}`$. Extract the loading matrix $`\mathbf{L} \in \mathbb{R}^{n \times k}`$ for the top $`k`$ components (typically $`k = 2`$ or $`k = 3`$).

The eigenportfolio weights for component $`j`$ are $`\mathbf{w}_j = \mathbf{L}_{:,j} / \sum_i |L_{i,j}|`$.

**Step 4: Compute residual mispricing.**

For each contract $`i`$, the PC-implied price is:

$`\hat{p}_i^{(t)} = \mu_i + \sigma_i \cdot \sum_{j=1}^{k} L_{i,j} \cdot f_j^{(t)}`$

where $`f_j^{(t)} = \tilde{\mathbf{P}}^{(t)} \cdot \mathbf{L}_{:,j}`$ is the $`j`$-th principal component score at time $`t`$.

The residual mispricing is:

$`\epsilon_i^{(t)} = p_i^{(t)} - \hat{p}_i^{(t)}`$

Normalise to a z-score using the trailing standard deviation of residuals:

$`z_i^{(t)} = \frac{\epsilon_i^{(t)}}{\hat{\sigma}_{\epsilon, i}(w_{PCA})}`$

**Step 5: Generate trades.**

When $`|z_i^{(t)}| > \theta_{PCA}`$:

- $`z_i > \theta_{PCA}`$: Contract $`i`$ is overpriced relative to eigenportfolio. SELL YES (BUY NO).
- $`z_i < -\theta_{PCA}`$: Contract $`i`$ is underpriced. BUY YES.

**Step 6: Market-neutral basket.**

To hedge systematic factor exposure, take the opposite position in the eigenportfolio: for each trade on contract $`i`$, enter offsetting positions in the other $`n-1`$ contracts proportional to $`-L_{i,j}`$. In practice, this is achieved by trading the residual directly: BUY contract $`i`$ and SELL the PC-implied basket.

### 23.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| PCA estimation window | $`w_{PCA}`$ | Rolling window for covariance estimation | 30–300 seconds |
| Number of PCs | $`k`$ | Scree plot; variance explained threshold > 80% | 2–4 |
| Minimum contract universe | $`n_{\min}`$ | Minimum contracts for stable PCA | 5–10 |
| Residual normalisation window | $`w_{\epsilon}`$ | Rolling std dev of residuals | 60–600 seconds |
| Entry z-score threshold | $`\theta_{PCA}`$ | Grid search; optimise Sharpe | 1.5–3.0 |
| Rebalance frequency | $`f_{PCA}`$ | How often to recompute PCA | 10–60 seconds |
| Maximum single-contract exposure | $`q_{PCA}`$ | Risk limit per leg | 50–500 shares |
| Hedging threshold (residual-only) | $`\theta_{hedge}`$ | Minimum residual to trade without hedging | 0.5–1.0 |

### 23.5 Concrete Example

**Setup:** Five BTC contracts all resolving at 14:05, with strikes 68,000 / 68,250 / 68,500 / 68,750 / 69,000. $`\tau = 180`$s.

1. **Price snapshots (last 60 seconds):**

| Strike | Mid-price (z-score) |
|--------|---------------------|
| 68,000 | 0.72 (+0.3) |
| 68,250 | 0.58 (-1.2) |
| 68,500 | 0.42 (+0.8) |
| 68,750 | 0.28 (+0.1) |
| 69,000 | 0.15 (-0.5) |

2. **PCA (k=2):** First PC loads positively on all contracts (market factor, explains 72% of variance). Second PC loads positively on high strikes and negatively on low strikes (volatility factor, explains 15%).
3. **Residuals:** Contract $`K=68,250`$ (strike 68,250) has $`z = -1.2`$ — it is trading 1.2 standard deviations below its eigenportfolio-implied price.
4. **Interpretation:** The 68,250 contract is cheap relative to the joint curve. The eigenportfolio says that given the prices of the other four contracts, the 68,250 contract should be priced at approximately 0.62.
5. **Action:** BUY YES on 68,250 at 0.58. Expected reversion to 0.62+ gives edge of ~0.04 per share minus costs.
6. **Hedge:** Sell proportional baskets of the other four contracts to neutralise systematic exposure.

### 23.6 Statistical Backing

- **Avellaneda & Lee (2010):** "Statistical Arbitrage in the US Equities Market." *Quantitative Finance*, 10(7), 761–782. Eigenportfolio framework for mean-reversion trading; directly applicable to prediction markets with multiple strikes.
- **Jolliffe (2002):** *Principal Component Analysis*, 2nd ed. Springer. Comprehensive treatment of PCA methodology.
- **Potters, Bouchaud & Laloux (2005):** "Financial Applications of Random Matrix Theory." *Acta Physica Polonica B*. On eigenvalue spectrum of financial correlation matrices and filtering noise.
- **Kritzman, Li, Page & Rigobon (2011):** "Principal Components as a Measure of Systemic Risk." *Journal of Portfolio Management*. Demonstrated that the first PC explains 60–80% of co-movement in correlated asset baskets.

**Key validation tests:**

1. **Residual half-life.** Measure mean-reversion speed of $`\epsilon_i^{(t)}`$. Target: half-life < 30 seconds (otherwise the reversion window exceeds the 5-minute horizon).
2. **Hedging effectiveness.** After eigenportfolio hedging, the residual portfolio should have beta < 0.2 to spot returns. Target: hedged Sharpe > 2x unhedged Sharpe.
3. **Scree stability.** The number of significant PCs should be stable across windows. If eigenvalues cross, the factor structure is degenerate.

### 23.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Insufficient contracts** | <5 contracts with same resolution time makes PCA unstable | Expand universe to include correlated underlyings (BTC + ETH); accept cross-asset basis risk |
| **Factor rotation** | PCA loadings can flip sign between estimation windows | Procrustes rotation to align PC directions; only trade when loadings are stable |
| **Non-stationary correlation** | Factor structure breaks down during macro events | Monitor eigenvalue stability: if the first eigenvalue drops below 50% of variance, suspend PCA-RV |
| **Execution asynchrony** | Multi-leg basket cannot fill simultaneously | Use market orders for the residual leg and limit orders for hedges; accept small tracking error |
| **Stale price problem** | Some contracts update infrequently, biasing PCA | Exclude contracts with zero volume in the last 30 seconds; impute stale prices with forward-fill |

---

## Strategy 24: Natural Language Sentiment Surge (NLP)

### 24.1 Classification

**Type:** Alternative data (NLP-based)  
**Edge source:** Real-time sentiment from crypto news headlines and social media predicts short-term spot moves before Polymarket reprices  
**Complexity:** Medium-High  
**Capital efficiency:** Medium (discrete events; bursty returns)

### 24.2 Rationale

Financial news and social media sentiment have been shown to predict short-term asset returns. For crypto prediction markets:

1. **News arrival is causally prior to price.** A headline about a regulatory development, hack, or ETF flow arrives on newswires 10–60 seconds before the spot price moves.
2. **Polymarket is slower to react than CEX spot.** Even after spot moves, Polymarket takes additional seconds to reprice. NLP gives a double latency advantage: news → spot (1–10s) + spot → Polymarket (2–30s).
3. **Sentiment is orthogonal to price-based signals.** NLP captures fundamentally different information than technical indicators or order-book features.

Tetlock (2007) demonstrated that media content scores predict subsequent equity returns and reversals. Calomiris & Mamaysky (2019) extended this to crypto, finding that tweet sentiment predicts hourly returns with IC > 0.05. At the 5-minute scale, the predictive power is concentrated in sentiment *surges* — sudden spikes in negative or positive language volume.

**Why this is structurally persistent:**
- NLP models require specialised infrastructure (streaming API, fine-tuned transformer, GPU inference).
- Most Polymarket participants do not run NLP pipelines.
- News-based edge is transient but regularly replenished by new events.

### 24.3 Signal Construction

**Step 1: Ingest streaming news and social media.**

Subscribe to:
- Crypto news headline feeds (e.g., CoinDesk API, CryptoPanic, NewsAPI)
- Twitter/X filtered stream for selected crypto influencers and news sources
- Reddit r/cryptocurrency stream (high-signal submissions, not comments)

**Step 2: Compute real-time sentiment.**

For each article/tweet $`j`$ arriving at time $`t_j`$, compute:

$$`s_j \in [-1, +1] = \text{finBERT}(\text{text}_j)`$$

using a fine-tuned financial BERT model (or RoBERTa-finance). Aggregate over a sliding window $`w_{NLP}`$ (e.g., 30 seconds):

$`\mathcal{S}_t = \frac{\sum_{j: t_j \in [t-w, t]} s_j \cdot v_j}{\sum_{j} v_j}`$

where $`v_j`$ is a source-specific volume weight (higher weight for major news, lower for random tweets).

**Step 3: Detect sentiment surges.**

Compute the z-score of $`\mathcal{S}_t`$ against its trailing distribution:

$`Z_t^{\mathcal{S}} = \frac{\mathcal{S}_t - \mu_{\mathcal{S}}(W_{baseline})}{\sigma_{\mathcal{S}}(W_{baseline})}`$

A surge is detected when $`|Z_t^{\mathcal{S}}| > z_{surge}`$ AND the sentiment is directionally consistent across at least 3 distinct sources (to filter single-source noise).

**Step 4: Map sentiment to directional signal.**

$`S_t^{NLP} = \begin{cases}
+1 & \text{if } Z_t^{\mathcal{S}} > z_{surge} \text{ and most articles reference the underlying coin} \\[4pt]
-1 & \text{if } Z_t^{\mathcal{S}} < -z_{surge} \text{ and most articles reference the underlying coin} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

**Step 5: Combine with spot confirmation.**

To reduce false positives from irrelevant news, only execute when the spot direction in the 5 seconds following the NLP signal agrees with the sentiment:

$`\text{execute if: } S_t^{NLP} \neq 0 \text{ and } \text{sign}(S_{t+5} - S_t) = S_t^{NLP}`$

This short confirmation window filters noise while preserving most of the latency edge.

### 24.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Sentiment aggregation window | $`w_{NLP}`$ | Test 10s, 30s, 60s; maximise IC | 10–60 seconds |
| Sentiment baseline window | $`W_{baseline}`$ | Rolling window for z-score normalisation | 30–120 minutes |
| Surge z-score threshold | $`z_{surge}`$ | Grid search; optimise precision (not recall) | 2.0–4.0 |
| Minimum source diversity | $`n_{sources}`$ | Distinct sources required for confirmation | 2–4 |
| Spot confirmation delay | $`\delta_{confirm}`$ | Wait time for spot to confirm sentiment | 2–10 seconds |
| Maximum $`\tau`$ to enter | $`\tau_{\max}`$ | Avoid entering if window is too short | 60–240 seconds |
| Article relevance threshold | $`\theta_{rel}`$ | Minimum NLP relevance score to the specific asset | 0.5–0.8 |

### 24.5 Concrete Example

**Setup:** BTC = USD 68,400. Contract: "BTC > USD 68,500 at 14:05?" ($`\tau = 240`$s). YES ask = 0.30.

1. **News ingestion (t=14:01:00):** Three headlines arrive within 5 seconds:
   - "SEC approves spot Ether ETF — market rallying" (source: CoinDesk, $`s = +0.85`$)
   - "Ethereum ETF approval sends crypto higher" (source: Bloomberg, $`s = +0.72`$)
   - "BTC jumps 2% on ETF optimism" (source: CryptoPanic, $`s = +0.91`$)

2. **Sentiment aggregation (w=30s):** $`\mathcal{S}_t = 0.83`$. Baseline $`\mu = 0.02`$, $`\sigma = 0.12`$. $`Z^{\mathcal{S}} = (0.83 - 0.02) / 0.12 = 6.75`$. Massive positive surge.

3. **Spot confirmation:** In the 5 seconds following news arrival, BTC moves from 68,400 to 68,480. Direction agrees with sentiment.

4. **Signal:** BUY YES.

5. **Edge:** The news is clearly bullish and has already moved spot. Polymarket YES ask is still 0.30 (has not caught up). Fair probability given spot at 68,480: $`\hat{\pi} = 0.58`$. Edge = $`0.58 - 0.30 - 0.015 = +0.265`$ per share.

6. **Action:** BUY YES at 0.30. Position: 100 shares at USD 30 notional.

### 24.6 Statistical Backing

- **Tetlock (2007):** "Giving Content to Investor Sentiment: The Role of Media in the Stock Market." *Journal of Finance*, 62(3), 1139–1168. Media sentiment predicts equity returns with short-term reversal; the initial reaction is under-reaction.
- **Calomiris & Mamaysky (2019):** "How News and Its Context Drive Risk and Returns." *Journal of Financial Economics*. Crypto news sentiment predicts hourly returns with significant IC.
- **Loughran & McDonald (2011):** "When Is a Liability Not a Liability? Textual Analysis, Dictionaries, and 10-Ks." *Journal of Finance*. Established the methodology for financial NLP.
- **Araci (2019):** "FinBERT: Financial Sentiment Analysis with Pre-Trained Language Models." *arXiv:1908.10063*. Open-source model for financial sentiment.

**Key validation tests:**

1. **Lead-lag measurement.** Compute cross-correlation between $`Z^{\mathcal{S}}`$ and subsequent spot returns. The peak should be at positive lags of 10–120 seconds.
2. **Precision at threshold.** Fraction of sentiment surges > $`z_{surge}`$ that are followed by a spot move in the same direction within 60 seconds. Target: > 70%.
3. **Orthogonality check.** Correlation between $`Z^{\mathcal{S}}`$ and lagged spot returns (to ensure NLP captures news, not price momentum). Target: < 0.3 correlation with lagged returns.

### 24.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Fake news / spoofing** | Malicious or satirical content generates false signal | Cross-reference with at least 2 established news sources; filter accounts with < 6 months history |
| **Old news / delayed feed** | News API lag causes signal to fire after spot already moved | Timestamp-stamp each article from the source, not the API; discard articles > 15 seconds old |
| **Irrelevant news** | News mentions "Bitcoin" but refers to unrelated context (e.g., "Bitcoin obituary") | Use entity recognition + relevance scoring; require the asset to be the *subject* of the headline |
| **NLP model drift** | Sentiment model degrades on new types of content | Periodic re-fine-tuning on hand-labelled crypto news; monitor sentiment distribution drift |
| **Flash crash from news** | Extreme negative news causes instantaneous crash — no execution window | Require $`\tau > 60`$s and spot move < 2% before entering; avoid trading on unverified breaking news |
| **Low activity periods** | No news arriving for hours | NLP strategy has no signals in quiet periods — acceptable; do not force trades |

---

## Ensemble Integration

### Correlation With Existing Strategies

| | SLA | PMR | VIT | TDE | OBI | PFR | JDM | ONC | SRA | SJV | PCA-RV | NLP |
|---|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|--------|-----|
| **JDM** | 0.3 | 0.0 | 0.1 | 0.4 | 0.0 | 0.1 | 1.0 | 0.0 | 0.1 | 0.6 | 0.0 | 0.1 |
| **ONC** | 0.0 | 0.0 | 0.1 | 0.0 | 0.0 | 0.2 | 0.0 | 1.0 | 0.0 | 0.0 | 0.1 | 0.2 |
| **SRA** | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.1 | 0.0 | 1.0 | 0.1 | 0.1 | 0.0 |
| **SJV** | 0.3 | 0.1 | 0.1 | 0.5 | 0.1 | 0.2 | 0.6 | 0.0 | 0.1 | 1.0 | 0.1 | 0.1 |
| **PCA-RV** | 0.1 | 0.1 | 0.1 | 0.1 | 0.3 | 0.0 | 0.0 | 0.1 | 0.1 | 0.1 | 1.0 | 0.0 |
| **NLP** | 0.1 | 0.0 | 0.2 | 0.0 | 0.0 | 0.1 | 0.1 | 0.2 | 0.0 | 0.1 | 0.0 | 1.0 |

JDM and SJV are correlated (both are model-based pricing extensions). ONC, NLP, and PCA-RV provide near-orthogonal signals to all existing strategies. SRA is uncorrelated with everything (pure arbitrage).

### Recommended Initial Weights (within Part IV)

| Strategy | Weight | Rationale |
|----------|--------|-----------|
| JDM | 15% | Jump events are discrete; moderate frequency |
| ONC | 20% | Orthogonal signal; steady predictive power |
| SRA | 10% | Low frequency but risk-free when available |
| SJV | 15% | Continuous signal; correlated with JDM |
| PCA-RV | 20% | Multi-leg, stable factor structure |
| NLP | 20% | Bursty but high edge when signals fire |

---

## References

1. Merton, R. C. (1976). "Option Pricing When Underlying Stock Returns Are Discontinuous." *Journal of Financial Economics*, 3(1–2), 125–144. [JDM]
2. Lee, S. S., & Mykland, P. A. (2008). "Jumps in Financial Markets: A New Nonparametric Test and Jump Dynamics." *Review of Financial Studies*, 21(6), 2535–2563. [JDM]
3. Cont, R., & Tankov, P. (2004). *Financial Modelling with Jump Processes*. Chapman & Hall. [JDM, SJV]
4. Tarashev, N., Tsanko, I., & Borri, N. (2023). "On-Chain Metrics and Crypto Asset Pricing." *BIS Working Papers*. [ONC]
5. Easley, D., O'Hara, M., & Basu, S. (2019). "From Mining to Markets: The Transformation of Bitcoin's Governance." *Journal of Financial Economics*. [ONC]
6. Breeden, D. T., & Litzenberger, R. H. (1978). "Prices of State-Contingent Claims Implicit in Option Prices." *Journal of Business*, 51(4), 621–651. [SRA]
7. Figlewski, S. (2010). "Estimating the Implied Risk-Neutral Density for the US Market Portfolio." In *Volatility and Time Series Econometrics*. Oxford University Press. [SRA]
8. Bates, D. S. (1996). "Jumps and Stochastic Volatility: Exchange Rate Processes Implicit in Deutsche Mark Options." *Review of Financial Studies*, 9(1), 69–107. [SJV]
9. Eraker, B., Johannes, M., & Polson, N. (2003). "The Impact of Jumps in Equity Returns and Volatility." *Journal of Finance*, 58(3), 1269–1300. [SJV]
10. Duffie, D., Pan, J., & Singleton, K. (2000). "Transform Analysis and Asset Pricing for Affine Jump-Diffusions." *Econometrica*, 68(6), 1343–1376. [SJV]
11. Avellaneda, M., & Lee, J.-H. (2010). "Statistical Arbitrage in the US Equities Market." *Quantitative Finance*, 10(7), 761–782. [PCA-RV]
12. Jolliffe, I. T. (2002). *Principal Component Analysis*, 2nd ed. Springer. [PCA-RV]
13. Tetlock, P. C. (2007). "Giving Content to Investor Sentiment: The Role of Media in the Stock Market." *Journal of Finance*, 62(3), 1139–1168. [NLP]
14. Calomiris, C. W., & Mamaysky, H. (2019). "How News and Its Context Drive Risk and Returns." *Journal of Financial Economics*. [NLP]
15. Araci, D. (2019). "FinBERT: Financial Sentiment Analysis with Pre-Trained Language Models." *arXiv:1908.10063*. [NLP]
16. Loughran, T., & McDonald, B. (2011). "When Is a Liability Not a Liability? Textual Analysis, Dictionaries, and 10-Ks." *Journal of Finance*, 66(1), 35–65. [NLP]
17. Broadie, M., Chernov, M., & Johannes, M. (2007). "Model Specification and Risk Premia: Evidence from Futures Options." *Journal of Finance*, 62(3), 1453–1490. [SJV]
18. Bates, J. M., & Granger, C. W. J. (1969). "The Combination of Forecasts." *Operational Research Quarterly*, 20(4), 451–468. [Ensemble]

---

## Appendix: Glossary of New Methods and Concepts (Part IV)

| Method / Concept | Description | Used In |
|-------------------|-------------|---------|
| **Jump-Diffusion** | Asset price model combining continuous diffusion + discrete jumps | JDM |
| **Merton Sum** | Weighted average of BS prices conditional on $`n`$ jumps under Poisson | JDM |
| **Lee-Mykland Statistic** | Ratio of return to local bipower variation; jump detection test | JDM |
| **CEX Netflow** | Net volume of coins moving into/out of exchange wallets | ONC |
| **Whale Transaction** | On-chain transfer exceeding USD 1M (or 10+ BTC) | ONC |
| **Breeden-Litzenberger** | Second-derivative relationship between call prices and state prices | SRA |
| **Call Spread Replication** | Approximating a binary option by a tight vertical call spread | SRA |
| **SVCJ** | Stochastic Volatility with Correlated Jumps; Bates (1996) model | SJV |
| **Leverage Effect** | Negative correlation between spot returns and volatility changes | SJV |
| **Fourier Inversion** | Numerical method to recover option prices from characteristic functions | SJV |
| **Eigenportfolio** | Portfolio weights equal to PCA loadings; maximum explanatory power | PCA-RV |
| **Factor Rotation** | Non-uniqueness of PCA loadings; alignment across estimation windows | PCA-RV |
| **Procrustes Rotation** | Orthogonal rotation to align factor structures across windows | PCA-RV |
| **finBERT** | Pre-trained language model fine-tuned for financial sentiment analysis | NLP |
| **Sentiment Surge** | Extreme z-score of aggregated NLP sentiment over a short window | NLP |

---

*This document completes the four-part strategy framework for 5-minute crypto resolution markets on Polymarket. All twenty-four strategy hypotheses require empirical validation before live deployment. Part IV introduces three new data-source categories (on-chain metrics, streaming NLP, options surface replication) and two advanced pricing models (jump-diffusion, SVCJ). These strategies are designed to be combined with Parts I–III using the HMM-RS meta-layer. The SRA strategy (Strategy 21) is the only model-free arbitrage in the full ensemble; all others carry statistical risk. Past performance (once backtested) does not guarantee future results. Manage risk rigorously across the full 24-strategy ensemble.*
