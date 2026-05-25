# Range Oscillation Momentum & Time Decay Effect — Supplementary Strategy Proposals for 5-Minute Crypto Resolution Markets

**Classification:** Implementation Blueprint (Supplement)  
**Date:** May 2026  
**Prerequisites:** Part I: Core Strategies (Strategies 1–6), Part II (Strategies 7–12), Part III (Strategies 13–18)  
**Scope:** Two additional strategies exploiting intra-window price range dynamics and probability decay acceleration. ROM is a *new* strategy concept distinct from Strategy 11 (Resolution Oracle Mechanism Exploitation); TDE expands Strategy 5 (Time-to-Resolution Theta Decay Exploitation) into a fully specified, directly implementable signal.

---

## 0. Preliminaries

### 0.1 Notation

| Symbol | Meaning |
|--------|---------|
| $`S_t`$ | Spot crypto price (from Binance) at time $`t`$ |
| $`t_0`$ | Window open time (5-minute boundary) |
| $`T`$ | Window close / resolution time ($`T = t_0 + 300`$) |
| $`\tau = T - t`$ | Seconds remaining to resolution |
| $`p_t`$ | Polymarket YES share mid-price at time $`t`$ |
| $`p_t^{ask}`$, $`p_t^{bid}`$ | Polymarket best ask / best bid for given token |
| $`\hat{\pi}_t`$ | Model fair probability (diffusion-based binary call) |
| $`K`$ | Strike / threshold (window open price in codebase) |
| $`\sigma_t`$ | Rolling realized volatility per minute |
| $`\Phi`$, $`\phi`$ | Standard normal CDF / PDF |

### 0.2 Fair Probability Reference

The fair probability $`\hat{\pi}_t`$ is computed via the Black-Scholes binary call approximation implemented in `polyfon/pricing/fair_probability.py`:

$`\hat{\pi}_t = \Phi(d_1)`$

$`d_1 = \frac{\ln(S_t / K)}{\sigma_t \sqrt{\tau/60}}`$

where drift $`\mu = 0`$ (pure diffusion over 5 minutes). This is the baseline against which both ROM and TDE signals are measured.

---

## Strategy A: Range Oscillation Momentum (ROM)

### A.1 Classification

**Type:** Statistical range-break model  
**Edge source:** Price breakouts from intra-window equilibrium ranges signal directional momentum; Polymarket MMs lag the CEX in repricing after range violations  
**Complexity:** Low-Medium  
**Capital efficiency:** High (short holding periods, clear entry/exit points)

### A.2 Rationale

Within a 5-minute window, spot price exhibits noise-driven oscillations around a local equilibrium. Bid-ask bounce and uninformed retail flow create a "noise band" — a price range where neither buyers nor sellers have conviction. When informed flow or a momentum cascade pushes price beyond this band, the breakout is a high-quality signal: it confirms that the initial directional pressure is genuine, not a random walk step.

**Why this works structurally:**

1. **Consolidation before continuation.** Markets alternate between trending and ranging phases. A tight 30–90 second consolidation (narrow range, low volatility) is a classic continuation pattern. When price breaks out of that range, the initial move is validated, and the fair probability $`\hat{\pi}_t`$ shifts sharply.
2. **Polymarket MM latency.** Polymarket market makers update their quotes based on the spot mid-price, not the breakout signal. After a range breakout, the Polymarket YES/NO prices often remain anchored to the pre-breakout level for 5–20 seconds, creating a latency window analogous to SLA.
3. **Range as a noise filter.** Unlike WDM (which compares current spot to the opening price), ROM conditions on the *intra-window path*. A $`+0.15\%`$ move from open means something different if the price ranged tightly around open vs. oscillated wildly. ROM only enters when the price has *proven* direction by breaking a tested boundary.
4. **Statistical persistence at 5-minute horizons.** Crypto spot returns exhibit short-horizon momentum (Jegadeesh & Titman 1993; Fama & French 2012) with half-lives of 1–5 minutes. A breakout in the first 2–3 minutes of a window has meaningful continuation probability.

### A.3 Signal Construction

**Step 1: Track the intra-window range.**

At each tick time $`t \in [t_0, T]`$, maintain running extrema since window open:

$`H_t = \max\{S_u : u \in [t_0, t]\}`$

$`L_t = \min\{S_u : u \in [t_0, t]\}`$

$`R_t = H_t - L_t`$

**Step 2: Detect breakouts with a lag window.**

To avoid triggering on a single tick that immediately reverts (whipsaw), introduce a confirmation delay $`w`$ (typically 3–5 seconds). The breakout magnitude is measured relative to the extreme *as of $`t - w`$*:

$`B_t^+ = \max\!\left(0,\; \frac{S_t - H_{t-w}}{H_{t-w}}\right)`$

$`B_t^- = \max\!\left(0,\; \frac{L_{t-w} - S_t}{L_{t-w}}\right)`$

A breakout is confirmed when price crosses the historical extreme and stays beyond it for at least $`w`$ seconds.

**Step 3: Generate the entry signal.**

$`S_t^{ROM} = \begin{cases}
+1 & \text{if } B_t^+ > \theta_{entry} \text{and } \tau_{\min} < \tau < \tau_{\max} \text{and} R_t > R_{\min} \\[4pt]
-1 & \text{if } B_t^- > \theta_{entry} \text{and } \tau_{\min} < \tau < \tau_{\max} \text{and } R_t > R_{\min} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

- $`S_t^{ROM} = +1`$: **BUY YES** (upside breakout — momentum is bullish)
- $`S_t^{ROM} = -1`$: **BUY NO** (downside breakout — momentum is bearish)  
- $`R_{\min}`$: Minimum range width in absolute terms (e.g., 0.03% of spot). Avoids entering on noise-level micro-range breakouts.
- $`\tau_{\min}`$: Don't enter too early (need range to form; typically 30–60s).
- $`\tau_{\max}`$: Don't enter too late (insufficient time for momentum to play out; typically 15–30s before close).

**Step 4: Confidence model.**

The confidence $`C_t \in [0, 1]`$ is a product of three factors:

$`C_t = \min\!\left(\frac{B_t}{\theta_{sat}},\; 1\right) \;\times\; f_{range}\!\left(\frac{R_t}{\sigma_t}\right) \;\times\; g_{timing}(\tau)`$

where:

$`f_{range}(x) = \min\!\left(\frac{R_{ref}}{x},\; 1\right)`$

Rewards *narrow* ranges ($`R_t`$ small relative to volatility). A tight consolidation before breakout is a stronger signal than a breakout from a wide, noisy range.

$`g_{timing}(\tau) = \min\!\left(\frac{\tau}{\tau_{optimal}},\; 1\right)`$

Rewards more remaining time for momentum to materialize. $`\tau_{optimal}`$ is typically 120–180s.

**Step 5: Compute expected edge.**

For a BUY YES at $`p_t^{ask}`$ with confidence $`C_t`$ and fee rate $`f`$:

$`\text{edge} = C_t \times (1 - p_t^{ask}) - f \times p_t^{ask} \times (1 - p_t^{ask}) - \theta_{edge}`$

Where $`\theta_{edge}`$ is a minimum edge floor (e.g., 0.01). Symmetric for BUY NO. The intuition: confidence quantifies how likely the breakout direction correctly predicts the resolution outcome; this is discounted by fees.

### A.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Entry threshold (breakout %) | $`\theta_{entry}`$ | Grid search on historical windows; maximize Sharpe net of cost | 0.0005–0.003 (0.05%–0.30%) |
| Saturation threshold | $`\theta_{sat}`$ | Breakout magnitude where confidence saturates; optimize precision-recall curve | 0.001–0.005 |
| Confirmation delay | $`w`$ | Test 1s, 3s, 5s, 10s; minimize false positive breakout rate | 3–7 seconds |
| Minimum range width (absolute) | $`R_{\min}`$ | Filter breakouts below noise floor; measure typical spread noise amplitude | 0.02%–0.10% of spot |
| Range quality reference | $`R_{ref}`$ | Range width at which $`f_{range} = 1`$; set from empirical range/vol ratio | $`0.2\sigma_t`$–$`0.5\sigma_t`$ |
| Optimal timing reference | $`\tau_{optimal}`$ | Maximum $`\tau`$ for full timing confidence; test sensitivity | 120–180 seconds |
| Minimum $`\tau`$ to enter | $`\tau_{\min}`$ | Range needs time to form; test 30s, 60s, 90s | 30–90 seconds |
| Maximum $`\tau`$ to enter | $`\tau_{\max}`$ | Don't trade with insufficient remaining momentum time | Set at 15–30s before close |
| Minimum edge floor | $`\theta_{edge}`$ | Filter trades below minimum expected edge | 0.005–0.02 |
| Position size | $`q_{\max}`$ | USD to spend (market orders) or shares (limit orders) | 50–500 USD |

### A.5 Concrete Example

**Setup:** BTC spot opens window at USD 68,400. Contract: "Will BTC be above USD 68,400 at 14:05 UTC?" ($`\tau = 300`$s). $`\sigma_t = 0.0008`$/min.

| Time ($`\tau`$) | Spot ($`S_t`$) | $`H_t`$ | $`L_t`$ | $`R_t`$ |
|------------|----------|----------|----------|--------|
| 300s | 68,400 | 68,400 | 68,400 | 0 |
| 270s | 68,420 | 68,420 | 68,400 | 20 |
| 240s | 68,410 | 68,420 | 68,400 | 20 |
| 210s | 68,415 | 68,420 | 68,400 | 20 |
| 180s | 68,418 | 68,420 | 68,400 | 20 |
| 150s | 68,412 | 68,420 | 68,400 | 20 |
| 120s | 68,408 | 68,420 | 68,400 | 20 |
| 90s | 68,435 | **68,435** | 68,400 | 35 |

At $`\tau = 90`$s, spot breaks above the running high of 68,420 (set 180 seconds ago and re-tested 5 times).

1. **Breakout magnitude:** $`B_t^+ = (68435 - 68420) / 68420 = 0.000219`$ (0.022%). Exceeds $`\theta_{entry} = 0.0002`$.
2. **Range quality:** $`R_t = 35`$ USD. $`\sigma_t \approx 54.7`$ USD/min. Range is tight: $`f_{range} = \min(0.3\sigma_t/35,\; 1) \approx 0.47`$ — moderate consolidation before breakout.
3. **Timing:** $`\tau = 90`$s. $`g_{timing}(90) = \min(90/120,\; 1) = 0.75`$.
4. **Confidence:** $`C_t = \min(0.000219/0.002,\; 1) \times 0.47 \times 0.75 = 0.11 \times 0.47 \times 0.75 = 0.039`$.
5. **Polymarket:** YES ask = 0.52. Market is still near even because MMs haven't processed the breakout.
6. **Edge:** $`0.039 \times (1 - 0.52) - 0.07 \times 0.52 \times 0.48 = 0.019 - 0.017 = +0.002`$ per share. Marginal — tighten $`\theta_{entry}`$ or increase $`q_{\max}`$ for practical sizing.
7. **Action:** BUY YES at ask 0.52. Position 50 shares at USD 26.00 notional.

### A.6 Statistical Backing

The ROM strategy is grounded in three well-established empirical regularities:

1. **Short-horizon momentum.** Jegadeesh & Titman (1993, 2001) documented that momentum profits are strongest at 1–6 month horizons in equities, but the same mechanisms — delayed information diffusion and trend-following behavior — operate at intraday scales. Crypto markets amplify these effects due to higher retail participation and leverage-driven cascades (Bianchi et al. 2022).
2. **Support and resistance as self-fulfilling prophecies.** Osler (2000, 2003) demonstrated that limit order clustering at round numbers and recent extremes creates genuine price barriers. When broken, these levels accelerate price moves as stop-losses trigger and breakout traders enter. The intra-window range acts as a miniature support/resistance zone.
3. **Consolidation-breakout patterns.** The "volatility contraction → expansion" cycle (Minervini 2012; O'Neil 2009) manifests at all time frames. Narrowing range + low volatility predicts an imminent expansion. ROM captures this at the 5-minute scale by rewarding tighter ranges ($`f_{range}`$).

**Key validation tests:**

1. **Conditional hit rate.** Fraction of breakouts (above $`\theta_{entry}`$) that result in the breakout direction matching the final resolution outcome. Target: > 60% (vs. 50% baseline).
2. **Mean reversion of false breakouts.** Of breakouts that reverse before window close, measure the distribution of time-to-reversal. If the median reversal time > 30 seconds, there is a viable exit window.
3. **Range quality signal.** Compare hit rates for tight-range breakouts ($`R_t / \sigma_t < 0.5`$) vs. wide-range breakouts ($`R_t / \sigma_t > 1.5`$). A significant difference validates the $`f_{range}`$ weighting.
4. **Sharpe ratio sensitivity to $`\tau`$.** Plot PnL per trade as a function of entry $`\tau`$. Expect optimal $`\tau`$ in 60–150 second range, with declining performance near window close (noise dominates).

### A.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Whipsaw / false breakout** | Price breaks the range high by $`\theta_{entry}`$ then immediately reverses | The confirmation delay $`w`$ filters single-tick spikes. Increase $`w`$ in high-noise regimes. |
| **Range never forms** | Price trends monotonically from open with no consolidation | ROM naturally degrades to no-trade in trending windows. This is acceptable — not every window must produce a signal. |
| **Wide, noisy range** | $`R_t`$ is large relative to $`\sigma_t`$, making any "breakout" just continuation of random walk | $`f_{range}`$ scales confidence down. Below a critical ratio, ROM produces no signal. |
| **Late-window breakout** | Breakout occurs at $`\tau < \tau_{\max}`$ with insufficient time for momentum to mature | Hard filter: no entry when $`\tau < \tau_{\max}`$. |
| **Slippage after breakout** | Polymarket MMs widen spreads or pull quotes after a sharp CEX move | Use limit orders ($`order\_class = \text{limit}`$) posted slightly inside the best ask/bid to capture the latency window before MM reprices. |
| **Multi-breakout whipsaw** | Price breaks up, reverses, breaks down — repeated breakouts in one window | Only enter on the *first* confirmed breakout per window. Subsequent breakouts within the same window are lower-quality (range integrity destroyed). |
| **Volatility regime mismatch** | In low-vol windows, $`\theta_{entry}`$ is never reached; in high-vol windows, breakouts are noise | Dynamically scale $`\theta_{entry}`$ proportional to $`\sigma_t`$. |

### A.8 Implementation Notes

ROM is stateful per window: $`H_t`$, $`L_t`$, $`R_t`$, and a `breakout_fired` flag must be tracked individually. At window open, state is initialized: $`H_{t_0} = L_{t_0} = S_{t_0}`$. At tick $`t`$, update $`H_t`$, $`L_t`$ and check breakout conditions.

For dry-run replay at a single evaluation time $`t_{eval} = T - 10\text{s}`$, the strategy must reconstruct the range from all spot prices between $`t_0`$ and $`t_{eval}`$. This requires querying `SpotPrice` rows within $`[t_0, t_{eval}]`$ to compute $`H_{eval}`$ and $`L_{eval}`$ before evaluating the breakout signal.

Context additions needed: none — ROM uses `spot_price`, `window_open_price`, `tau_seconds`, and state variables tracked internally.

---

## Strategy B: Time Decay Effect (TDE)

### B.1 Classification

**Type:** Statistical options-theoretic model  
**Edge source:** Accelerating probability convergence as $`\tau \to 0`$ creates predictable near-resolution drift; Polymarket MMs systematically under-hedge theta risk  
**Complexity:** Medium  
**Capital efficiency:** High (systematic, repeatable, highest Sharpe near resolution)

### B.2 Rationale

The binary call fair probability $`\hat{\pi}_t = \Phi(d_1)`$ is a function of $`\tau`$. As time runs out, $`\hat{\pi}_t`$ converges to either 0 or 1 — deterministically. The *rate* of this convergence, analogous to theta in options pricing, follows a predictable, non-linear profile:

- **Far from expiry (**$`\tau \gg 0`$**):** Probability drifts slowly. A 0.2% spot move might shift $`\hat{\pi}`$ by 0.02.
- **Near expiry (**$`\tau \to 0`$**):** The same 0.2% spot move can shift $`\hat{\pi}`$ by 0.15 or more. The payoff function steepens into a step.

This creates a structural opportunity: in the final 60–120 seconds, the deterministic component of the probability shift (time decay) often outpaces the market's ability to reprice. Polymarket market makers, constrained by wide quoting obligations and inventory risk, do not dynamically hedge theta with the granularity that options market makers do on Deribit or CME. The result is systematic, predictable mispricing near resolution.

**Why this works:**

1. **Theta is computable in closed form.** Unlike other edge sources that require statistical estimation (order flow toxicity, regime detection), theta is a direct analytical derivative of the Black-Scholes binary formula. No fitting or calibration is needed beyond the volatility estimate $`\sigma_t`$.
2. **The mispricing is self-correcting.** When $`|\Theta_t|`$ is large, the fair probability is accelerating toward 0 or 1. The market price $`p_t`$ *will* follow — it must, because at $`\tau = 0`$ there is no uncertainty. TDE enters before the market catches up.
3. **TDE complements SLA, not duplicates it.** SLA enters when $`|\hat{\pi}_t - p_t| > \theta`$ — a *level* condition. TDE enters when $`\partial\hat{\pi}_t/\partial\tau`$ is large *and* the level mispricing exists — a *rate-of-change* catalyst. An SLA signal can fire 3 minutes before close and remain valid; a TDE signal only fires in the final 30–90 seconds when the convergence rate peaks.
4. **Risk is well-bounded.** The maximum holding period is $`\tau`$ seconds. Unlike momentum or mean-reversion strategies, TDE's exit is the resolution itself — there is no exit timing decision.

### B.3 Signal Construction

**Step 1: Compute the theta (rate of probability change).**

From the fair probability formula $`\hat{\pi}_t = \Phi(d_1)`$ with $`d_1 = \ln(S_t/K) / (\sigma_t \sqrt{\tau/60})`$, differentiate with respect to $`\tau`$ (in seconds):

$`\frac{\partial d_1}{\partial \tau} = -\frac{1}{2} \cdot \frac{\ln(S_t / K) \cdot \sqrt{60}}{\sigma_t \cdot \tau^{3/2}}`$

$`\Theta_t = \frac{\partial \hat{\pi}_t}{\partial \tau} = \phi(d_1) \cdot \frac{\partial d_1}{\partial \tau}`$

A computationally simpler approximation, valid when $`|\ln(S_t/K)| \gg \sigma_t^2 \tau / 2`$ (the moneyness term dominates the drift term):

$`\Theta_t \approx -\frac{\phi(d_1) \cdot d_1}{2\tau}`$

This yields units of probability change per second. **Sign convention:** $`\Theta_t`$ is the derivative of $`\hat{\pi}_t`$ with respect to *remaining* time $`\tau`$. When $`S_t > K`$, $`\hat{\pi}_t \to 1`$ as $`\tau \to 0`$, so $`\hat{\pi}_t`$ *increases* as $`\tau`$ *decreases*, meaning $`\Theta_t < 0`$. Conversely, when $`S_t < K`$, $`\Theta_t > 0`$. Therefore:

- $`\Theta_t < 0`$: probability is converging **UP** toward 1 → BUY YES direction
- $`\Theta_t > 0`$: probability is converging **DOWN** toward 0 → BUY NO direction

**Step 2: Define the entry condition.**

The core idea: theta predicts the direction of probability drift. If the market price $`p_t`$ has not converged to the fair probability $`\hat{\pi}_t`$, and theta indicates further divergence is imminent, enter in the direction of the drift.

$`\epsilon_t = \hat{\pi}_t - p_t`$

Entry when:

$`|\epsilon_t| > \theta_{entry} \quad \text{and} \quad \text{sign}(\Theta_t) = \text{sign}(\epsilon_t) \quad \text{and} \quad \tau < \tau_{\max}`$

This double condition is critical: we require both that the market is *already* mispriced (level check) and that the mispricing is *expected to grow* (theta direction check). A mispricing that theta is pushing *against* (e.g., $`\hat{\pi}_t > p_t`$ but $`\Theta_t < 0`$) is a *converging* mispricing — gamma risk, not theta edge.

**Step 3: Signal mapping.**

$`S_t^{TDE} = \begin{cases}
+1 & \text{if} \epsilon_t > \theta_{entry} \text{and} \Theta_t > 0 \text{and} \tau < \tau_{\max} \\[4pt]
-1 & \text{if} \epsilon_t < -\theta_{entry} \text{and} \Theta_t < 0 \text{and} \tau < \tau_{\max} \\[4pt]
0 & \text{otherwise}
\end{cases}`$

- $`S_t^{TDE} = +1`$: **BUY YES** — fair probability is above market AND still climbing
- $`S_t^{TDE} = -1`$: **BUY NO** — fair probability is below market AND still falling

**Step 4: Confidence model.**

TDE confidence is driven by theta magnitude — larger theta means faster convergence, which means more certainty in the directional prediction:

$`C_t = \min\!\left(\frac{|\epsilon_t|}{\epsilon_{sat}},\; 1\right) \;\times\; \min\!\left(\frac{|\Theta_t|}{\Theta_{sat}},\; 1\right)`$

where $`\epsilon_{sat}`$ (e.g., 0.15) and $`\Theta_{sat}`$ (e.g., 0.005 probability/sec) are saturation thresholds.

**Step 5: Compute expected edge.**

For BUY YES at $`p_t^{ask}`$:

$`\text{edge} = \hat{\pi}_t - p_t^{ask} - f \times p_t^{ask} \times (1 - p_t^{ask}) - \theta_{edge}`$

Unlike ROM where the edge is confidence-scaled, TDE's edge is computed directly from the fair probability $`\hat{\pi}_t`$, which is expected to be a more accurate estimate than $`p_t`$ near resolution.

### B.4 Empirically Discoverable Parameters

| Parameter | Symbol | Discovery Method | Expected Range |
|-----------|--------|------------------|----------------|
| Entry threshold (mispricing level) | $`\theta_{entry}`$ | Grid search; optimize Sharpe net of cost | 0.03–0.10 |
| Maximum $`\tau`$ to enter | $`\tau_{\max}`$ | Test entering at 120s, 90s, 60s, 30s; find knee in PnL curve | 30–120 seconds |
| Minimum $`\tau`$ to enter | $`\tau_{\min}`$ | Avoid illiquid final seconds; test 5s, 10s, 15s | 5–15 seconds |
| Saturation mispricing | $`\epsilon_{sat}`$ | Mispricing level where confidence saturates | 0.10–0.20 |
| Saturation theta | $`\Theta_{sat}`$ | Theta level where rate-confidence saturates | 0.002–0.01 prob/sec |
| Minimum edge floor | $`\theta_{edge}`$ | Filter trades below minimum expected edge | 0.005–0.02 |
| Volatility estimation window | $`w_\sigma`$ | Window for $`\sigma_t`$ estimation | 30–300 seconds |
| Position size | $`q_{\max}`$ | USD to spend or shares | 50–500 USD |

### B.5 Concrete Example

**Setup:** BTC spot = USD 68,580. Strike $`K`$ = USD 68,500 (window open price). $`\tau = 60`$s remaining. $`\sigma_t = 0.001`$/min.

**Step 1 — Fair probability:**

$`\tau_{\min} = 60/60 = 1.0`$ min. $`\sigma = 0.001 \times \sqrt{1.0} = 0.001`$.

$`d_1 = \ln(68580/68500) / 0.001 = 0.001168 / 0.001 = 1.168`$.

$`\hat{\pi}_t = \Phi(1.168) = 0.878`$.

The model says there is an 87.8% probability BTC finishes above strike. Polymarket YES ask = 0.72 — the market is 15.8 percentage points below fair value.

**Step 2 — Theta (approximation):**

$`\Theta_t \approx -\frac{\phi(1.168) \times 1.168}{2 \times 60}`$

$`\phi(1.168) = \frac{1}{\sqrt{2\pi}} e^{-1.168^2/2} = 0.399 \times e^{-0.682} = 0.399 \times 0.506 = 0.202`$.

$`\Theta_t \approx -0.202 \times 1.168 / 120 = -0.00197`$ prob/sec.

The negative theta means probability is drifting **upward** toward 1.0 as time runs out. Over the remaining 60 seconds, the expected probability gain is roughly $`60 \times 0.002 = 0.12`$, pushing $`\hat{\pi}`$ from 0.878 toward $`\approx 0.996`$.

**Step 3 — Double condition check:**

Market mispricing: $`\epsilon_t = \hat{\pi}_t - p_t = 0.878 - 0.72 = +0.158`$.

- Level check: $`|\epsilon_t| = 0.158 > \theta_{entry} = 0.05`$ ✓
- Direction check: $`\Theta_t = -0.00197 < 0`$ (probability climbing) and $`\epsilon_t > 0`$ (fair above market) — same direction ✓
- Timing check: $`\tau = 60 < \tau_{\max} = 90`$ ✓

**Double condition satisfied.** Signal: **BUY YES**.

**Step 4 — Confidence and edge:**

$`C_t = \min(0.158/0.15, 1) \times \min(0.00197/0.005, 1) = 1.0 \times 0.394 = 0.394`$.

$`\text{edge} = 0.878 - 0.72 - 0.07 \times 0.72 \times (1 - 0.72) - 0.01 = 0.158 - 0.014 - 0.01 = +0.134`$ USD per share.

**Action:** BUY YES at 0.72. Exposure: 100 shares at USD 72.00 notional. Expected profit: $`100 \times 0.134 = \text{USD } 13.40`$ net of fees.

### B.6 Statistical Backing

TDE is the most theoretically grounded strategy in the suite because it derives directly from the mathematics of the binary call formula:

1. **Breeden & Litzenberger (1978).** The binary call's delta ($`\partial \hat{\pi} / \partial S`$) and theta ($`\partial \hat{\pi} / \partial \tau`$) are exact analytical constructs. They are not statistical approximations; they are derivatives of a pricing formula whose inputs are observable.
2. **Black & Scholes (1973) and Merton (1973).** Theta decay is a fundamental component of option pricing. While prediction markets are not European options (they have no early exercise, no continuous hedging, and discrete payoffs), the convergence property at expiry is identical: the price must converge to the intrinsic value as $`\tau \to 0`$.
3. **Berg & Rietz (2006).** Documented that prediction market prices converge to outcomes with accelerating accuracy in the final minutes. This is the empirical counterpart to the analytical theta formula.
4. **Wolfers & Zitzewitz (2004).** Showed that prediction market inefficiencies are largest when the information structure is asymmetric. The theta-Greek is information that requires mathematical differentiation; most Polymarket participants do not compute it.

**Key validation tests:**

1. **Theta calibration.** Compare predicted $`\Delta\hat{\pi}`$(from $`\Theta_t \times \Delta\tau`$) to actual $`\Delta p`$ over $`\Delta\tau`$ intervals of 5–30 seconds. Target: direction agreement > 65%.
2. **Conditional hit rate.** Among TDE signals where the double condition fires: fraction that resolve in the predicted direction. Target: > 75% (much higher than SLA because the signal fires closer to resolution).
3. **Performance by moneyness.** Stratify by $`|\ln(S_t/K)|`$. TDE should perform best when spot is clearly on one side of the strike (0.15%–0.5% away) — enough moneyness for theta to be large, but not so much that the contract is already near 0 or 1.
4. **Theta threshold sensitivity.** Plot PnL per trade against minimum $`|\Theta_t|`$ required for entry. Expect optimal threshold where theta is large enough to overcome noise but not so restrictive that no signals fire.

### B.7 Failure Modes and Mitigations

| Risk | Mechanism | Mitigation |
|------|-----------|------------|
| **Last-second spot reversal** | Spot jumps across the strike in the final 10 seconds, flipping the outcome | Exit early at $`\tau = \tau_{\min}`$ (e.g., 15s). Do not hold through resolution. The edge is captured by the convergence *during* the window, not the final instantaneous outcome. |
| **Pin risk** | $`S_t \approx K`$ — moneyness is negligible, theta is near zero, and probability oscillates around 0.5 | Skip contracts where $`|\ln(S_t/K)| < \theta_{pin}`$ (e.g., 0.05%). TDE cannot work when there is no directional conviction. |
| **Volatility spike near resolution** | A sudden volatility expansion (news, large order) makes the BS model's sigma estimate obsolete | Monitor $`\sigma_t`$ for structural breaks. If $`\sigma_t`$ > 3× its trailing median, suspend TDE for that window. |
| **Illiquidity at expiry** | Polymarket MMs pull quotes in the final 10–30 seconds | Enter at $`\tau \geq 30`$s when liquidity is still present. Do not attempt fills in the final seconds. |
| **Theta sign reversal** | Spot crosses the strike after entry, flipping the sign of $`\Theta_t`$ | Re-evaluate signal on each tick. If theta sign flips against the position, exit immediately (the convergence direction has reversed). |
| **Competing strategies** | Multiple sophisticated participants also compute theta and front-run each other | The edge size per trade shrinks. In shadow mode, measure the latency between TDE signal generation and the Polymarket mid-price adjustment. If this latency drops below 2 seconds, the edge is competed away. |

---

## Appendix: Strategy Comparison

| | ROM | TDE |
|---|---|---|
| **Type** | Statistical range-break | Options-theoretic |
| **Edge source** | Range breakout momentum | Theta-accelerated probability convergence |
| **Holding period** | 30–180s | 15–60s |
| **Entry window (**$`\tau`$**)** | 30–180s before close | 15–90s before close |
| **Implementation complexity** | Low-Medium | Medium |
| **Data requirements** | CEX spot (already collected) | CEX spot + volatility (already available in Context) |
| **State requirements** | $`H_t, L_t`$ per window (trivial) | Stateless per tick |
| **Key risk** | Whipsaw / false breakout | Last-second spot reversal |
| **Synergy with existing strategies** | Complements WDM (WDM uses open price; ROM uses intra-window range). ROM's breakout signal can confirm or override WDM's delta signal | Complements SLA (SLA uses level mispricing; TDE adds rate-of-change catalyst). In the final 60 seconds, TDE should dominate SLA due to converging uncertainty |

---

## References

1. Jegadeesh, N., & Titman, S. (1993). "Returns to Buying Winners and Selling Losers: Implications for Stock Market Efficiency." *Journal of Finance*, 48(1), 65–91.
2. Jegadeesh, N., & Titman, S. (2001). "Profitability of Momentum Strategies: An Evaluation of Alternative Explanations." *Journal of Finance*, 56(2), 699–720.
3. Osler, C. L. (2000). "Support for Resistance: Technical Analysis and Intraday Exchange Rates." *FRB New York Economic Policy Review*, 6(2), 53–68.
4. Osler, C. L. (2003). "Currency Orders and Exchange Rate Dynamics: An Explanation for the Predictive Success of Technical Analysis." *Journal of Finance*, 58(5), 1791–1819.
5. Bianchi, D., Babiak, M., & Dickerson, A. (2022). "Trading Volume and Momentum in Cryptocurrency Markets." Working paper.
6. Black, F., & Scholes, M. (1973). "The Pricing of Options and Corporate Liabilities." *Journal of Political Economy*, 81(3), 637–654.
7. Merton, R. C. (1973). "Theory of Rational Option Pricing." *Bell Journal of Economics and Management Science*, 4(1), 141–183.
8. Breeden, D. T., & Litzenberger, R. H. (1978). "Prices of State-Contingent Claims Implicit in Option Prices." *Journal of Business*, 51(4), 621–651.
9. Berg, J. E., & Rietz, T. A. (2006). "The Iowa Electronic Markets: Stylized Facts and Open Issues." In *Information Efficiency in Financial and Betting Markets*, Cambridge University Press.
10. Wolfers, J., & Zitzewitz, E. (2004). "Prediction Markets." *Journal of Economic Perspectives*, 18(2), 107–126.

---

*Document prepared as a supplement to the 18-strategy implementation blueprint. Both strategies require empirical validation on historical Polymarket data before live deployment. ROM trading frequency is inherently lower than WDM or SLA due to the range-formation requirement; TDE frequency is concentrated in the final 60–90 seconds of each window.*
