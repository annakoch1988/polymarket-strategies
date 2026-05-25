# 5-Minute Crypto Polymarket Strategy Proposals

This repository documents **24 core trading strategies** and **2 supplementary strategies** for 5-minute crypto binary resolution markets on Polymarket. All strategies are grounded in peer-reviewed research from financial economics, market microstructure, statistical learning, and alternative data analysis.

> **Polyfon** — a Python quantitative trading simulator — implements these strategies in production-grade code with WebSocket data collection, SQLite persistence, and replay-based backtesting. See https://github.com/annakoch1988/polyfon for the full project.

## Documents

| File | Content |
|------|---------|
| `5min_crypto_polymarket_strategy_proposals.md` | **Part I** — Core strategies 1–6: latency, behavioral, distributional, volume, theta, relative value |
| `5min_crypto_polymarket_strategy_proposals_part2.md` | **Part II** — Advanced strategies 7–12: microstructure, toxicity, cross-asset, regime-switching, oracle, MM inventory |
| `5min_crypto_polymarket_strategy_proposals_part3.md` | **Part III** — Structural strategies 13–18: funding rates, RND, Hawkes, KL divergence, adversarial RL, EVT |
| `5min_crypto_polymarket_strategy_proposals_part4.md` | **Part IV** — Alternative data strategies 19–24: jump-diffusion, on-chain, replication arbitrage, SVCJ, eigenportfolio, NLP |
| `range_momentum_and_theta_decay.md` | Supplementary: Range Oscillation Momentum (ROM) + Time Decay Effect expansion |
| `window_delta_momentum_wdm.md` | Supplementary: Window Delta Momentum — minimal-data baseline using spot displacement at T-10s |

## Strategy Overview

### Part I — Core (Strategies 1–6)

| # | Code | Name | Edge |
|---|------|------|------|
| 1 | SLA | Spot-Led Latency Arbitrage | CEX spot moves before Polymarket reprices |
| 2 | PMR | Probability Mean-Reversion After Overreaction | Retail overreacts to salient moves; prices revert |
| 3 | MPR | Multi-Outcome Probability Mass Redistribution | Inconsistent pricing across mutually exclusive outcomes |
| 4 | VIT | Volume-Spike Informed Trading | Unusual volume reveals informed flow |
| 5 | TDE | Time-to-Resolution Theta Decay Exploitation | Probability converges non-linearly as expiry approaches |
| 6 | CRV | Cross-Contract Relative Value | Mispricing across parallel strikes on the same underlying |

### Part II — Advanced (Strategies 7–12)

| # | Code | Name | Edge |
|---|------|------|------|
| 7 | OBI | Order Book Microstructure Imbalance | L2 depth predicts short-term price direction |
| 8 | VPIN-X | CEX Toxicity as Leading Volatility Indicator | VPIN spikes on Binance precede volatility regime shifts |
| 9 | CLL | Cross-Asset Correlation Lead-Lag | BTC–ETH lead-lag dynamics predict lagging asset moves |
| 10 | HMM-RS | Hidden Markov Model Regime-Switching | Meta-layer: regime-adaptive parameter selection |
| 11 | ROM | Resolution Oracle Mechanism Exploitation | Oracle construction creates exploitable price anchors |
| 12 | MIP | Market-Maker Inventory Pressure | MM inventory rebalancing creates predictable quote drift |

### Part III — Structural (Strategies 13–18)

| # | Code | Name | Edge |
|---|------|------|------|
| 13 | PFR | Perpetual Funding Rate Sentiment Arbitrage | Funding extremes predict spot reversals |
| 14 | RND | Risk-Neutral Density Recovery from Options | Option-implied density differs from Polymarket prices |
| 15 | HPE | Hawkes Process Microstructure Excitation | Trade clustering predicts imminent price jumps |
| 16 | KLD | Kullback-Leibler Divergence Regime Mismatch | Market-implied distribution diverges from empirical reality |
| 17 | ARL | Adversarial RL Agent Exploitation | Opponent RL agents have stale policies after regime shifts |
| 18 | EVT | Extreme-Value Tail Harvesting | BS underprices extreme moves; EVT gives correct tail probabilities |

### Part IV — Alternative Data (Strategies 19–24)

| # | Code | Name | Edge |
|---|------|------|------|
| 19 | JDM | Jump Diffusion Mispricing | BS neglects jump risk; Merton (1976) corrects fair probability |
| 20 | ONC | On-Chain Flow Momentum | CEX netflow, whale txns, active addresses lead spot |
| 21 | SRA | Synthetic Replication Arbitrage | Breeden-Litzenberger replication of binary via options |
| 22 | SJV | Stochastic Volatility with Jumps | Bates (1996) SVCJ: stochastic vol + correlated jumps |
| 23 | PCA-RV | Eigenportfolio Relative Value | PCA-based stat arb across related strikes (Avellaneda & Lee 2010) |
| 24 | NLP | Natural Language Sentiment Surge | Real-time finBERT sentiment from news/social media |

### Supplementary

| Code | Name | File | Edge |
|------|------|------|------|
| WDM | Window Delta Momentum | `window_delta_momentum_wdm.md` | Spot displacement from open at T-10s is near-decisive |
| ROM | Range Oscillation Momentum | `range_momentum_and_theta_decay.md` | Intra-window range breakouts signal momentum |
| TDE | Time Decay Effect | `range_momentum_and_theta_decay.md` | Theta-accelerated probability convergence (detailed expansion of Strategy 5) |

## Data Sources Used Across Strategies

- **CEX spot price** (Binance/Coinbase) — all strategies
- **Polymarket L2 order book** — OBI, MIP, SLA, TDE, ARL
- **Polymarket trade feed** — VIT, HPE, OBI
- **Perpetual futures funding rates** — PFR
- **CEX options surface** (Deribit/OKX) — RND, SRA, SJV
- **Blockchain on-chain data** — ONC
- **News / social media NLP** — NLP
- **CEX trade-level data** — VPIN-X, HPE, SJV

## References

The 24 core + 3 supplementary strategies reference over 50 peer-reviewed papers including: Black & Scholes (1973), Merton (1976), Breeden & Litzenberger (1978), Kyle (1985), Jegadeesh & Titman (1993), Bates (1996), Avellaneda & Lee (2010), Easley, Lopez de Prado & O'Hara (2012), Tetlock (2007), and others. Full citations are in each strategy document.
