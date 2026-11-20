# Multi-Regional Supply Chain Strategic Simulator

A discrete-event simulation and Monte Carlo decision-support engine built for **SUTD Supply Chain Management (40.260)**. This project models the multi-continental chemical drum network from **Responsive Learning Technologies' (RLT) *Supply Chain Game*** (Chopra & Afèche), allowing players to stress-test network topology, capacity expansion, and $(s, Q)$ reorder policies across Years 3–4 (Days 730–1,460) before committing capital in-game.

---

## 🌍 Scenario & Network Mechanics

The simulation models a 5-region supply chain over a 730-day live horizon ($t \in [730, 1460)$):
* **Mainland Regions:** Calopeia (initial factory & warehouse), Sorange, Tyran, and Entworpe.
* **Offshore Region:** Fardo, subject to higher freight rates and doubled transit lead times (14 days by truck vs. 7 days mainland; 2 days by mail vs. 1 day mainland).
* **Capital & Lead-Time Constraints:** Opening a new factory requires $500,000 CAPEX, capacity upgrades cost $50,000 per drum/day with a **90-day construction lag**, and new regional warehouses cost $100,000 with a **60-day build lag**.

---

## 📐 Mathematical & Discrete-Event Architecture

### 1. Dual-Mode Stochastic Demand Engine (`helper.py`)
The simulator supports both historical Excel demand replay (`Demand for each destination region.xlsx`) and parametric Monte Carlo demand generation fitted to each region's empirical distribution:

| Region | Demand Pattern | Mathematical Model ($t \ge 640$) |
| :--- | :--- | :--- |
| **Calopeia** | Trigonometric seasonal curve | $\mu_t = 39.19 - 30.89\cos(2\pi t / 365)$, $D_t \sim \mathcal{N}(\mu_t, (0.2\mu_t)^2)$ |
| **Sorange** | Linear growth trend | $\mu_t = \max(0, 0.15t - 97.45)$, $D_t \sim \mathcal{N}(\mu_t, (0.2\mu_t)^2)$ |
| **Tyran** | Stationary stochastic | $D_t \sim \mathcal{N}(19.70, 16.76^2)$ |
| **Fardo** | Offshore high-variance | $D_t \sim \mathcal{N}(15.70, 16.77^2)$ |
| **Entworpe** | Lumpy distributor reorders | $D_t = 250 \cdot X_t$, $X_t \sim \text{Bernoulli}(0.044)$ |

During the final 30-day product phase-out ($t \in [1430, 1460]$), regional demand scales down linearly via an end-of-life multiplier:

$$\lambda_{\text{EOL}}(t) = \max\left(0, \frac{1460 - t}{30}\right)$$

### 2. Daily Discrete-Event Execution Loop (`functions.py`)
Each simulated day executes six sequential state transitions:
1. **Cash Compounding:** Positive cash balances earn 10% annual interest compounded daily ($r_d = 0.10 / 365$).
2. **Scheduled Policy & CAPEX Events:** Processes time-stamped strategic actions (`upgrade_capacity`, `set_fulfillment`, and `set_reorder_policy`).
3. **Least-Cost Customer Fulfillment:** Active warehouses assigned to serve a customer region are sorted by unit mail shipping cost, fulfilling demand greedily at $1,450/drum.
4. **Transit & Construction Pipeline:** Decrements warehouse build timers and updates in-transit batch arrivals.
5. **Priority-Scheduled Factory Production:** An idle factory triggers a production batch of size $Q$ for a warehouse if and only if four feasibility criteria hold:
   * **Inventory Position Trigger:** $I_{\text{warehouse}} + I_{\text{transit}} \le \text{ROP}$
   * **Positive Order Quantity:** $Q > 0$
   * **Upfront Liquidity Constraint:** $\text{Balance} \ge C_{\text{setup}} + c_{\text{unit}}Q + C_{\text{freight}}(Q)$, where truck freight scales in discrete 200-drum truckloads ($\lceil Q / 200 \rceil \times C_{\text{truck}}$)
   * **Factory Availability & Priority Tie-Breaking:** Selects among eligible warehouses by highest assigned priority.
6. **Holding Cost Accrual:** Deducts $100 / 365 per drum/day across all on-hand warehouse inventory and in-transit stock.

---

## 🛠️ Tech Stack & Analytics Output

* **Python 3.12**, **Pandas**, **Matplotlib**
* **Monte Carlo Policy Evaluation (`run_multiple_simulations`):** Executes $N$-run stochastic simulations to quantify expected terminal cash, downside risk (min balance), and upside potential across competing ROP/OQ schedules.
* **4-Tier Diagnostic Dashboard (`plot_results`):** Plots daily cash balance trajectories, regional warehouse inventory sawtooth cycles, factory production utilization, and horizontal Gantt bars for regional stockout intervals.
