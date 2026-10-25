# DQN-Guided Destruction and MILP Repair for the Vehicle Routing Problem with Time Windows

This project presents a **hybrid optimization framework** for the **Vehicle Routing Problem with Time Windows (VRPTW)**. A Deep Q-Network (DQN) selects the destruction operator of a Large Neighborhood Search (LNS), and **Gurobi** (MILP) repairs the destroyed neighborhood under a short time limit.

> This is an **analytical and exploratory study**. Its aim is not to match state-of-the-art VRPTW solvers, but to examine how a learned selector, an exact solver, and heuristics interact.

## Project Overview
- **Initial solution:** distance-based insertion heuristic (Solomon I1 specialization) followed by a short plain LNS (100 iterations).
- **Learned destruction:** a DQN observes the search state and chooses *which operator* to apply (not which customers).
- **MILP repair:** Gurobi reconstructs the removed customers within a per-call time limit.
- **Optional local refinement:** route fusion, 2-opt and Or-opt applied after the search.

## Search Loop
1. Observe the current solution (7-dimensional state).
2. DQN selects a destruction operator.
3. The operator removes 24 customers (K_DESTROY_RATIO = 0.24).
4. Gurobi repairs the partial solution; greedy insertion is the fallback if no solution is returned.
5. Reward = relative distance improvement after repair.

Episodes have 15 steps; 10 chained episodes (150 steps) are run per Solomon instance, with frozen weights and ε = 0.

## Destruction Portfolio
| Action | Information used |
|--------|------------------|
| Spatial | Customer locations |
| Temporal | Customer time windows |
| Cost-based | Contribution to route distance |
| Random | Random selection |

Tested portfolios: **Full** (4 actions), **Merged3** (spatiotemporal, cost-based, random) and **Merged2** (spatiotemporal, random).

## Mathematical Model
- **Objective:** minimize total travel distance, f(S) = Σ d_ij (vehicle penalty β = 0 in all main experiments).
- **Constraints:** each customer served once, vehicle capacity, time windows with waiting allowed, return to depot.
- **Reported metric:** distance deviation from the published Solomon reference, Δ = 100 × (f(S) − f_ref) / f_ref.

> Solomon references minimize the number of vehicles first, then distance. Deviations are therefore **descriptive**, not a ranking against methods with a different objective.

## Experimental Setup
- **Training:** 80 synthetic instances (100 customers; clustered, random, mixed), 20 episodes each (1,600 episodes, 24,000 transitions).
- **Evaluation:** the 56 Solomon instances (C1, C2, R1, R2, RC1, RC2), frozen network weights.
- **Calibration:** 18-run Taguchi L18 design on 7 parameters, on synthetic instances only.
- **Solver:** Gurobi, 2 s per repair call (4 s, Threads = 1, Seed = 0 in the final pipeline).
- **DQN:** 2 hidden layers of 64 units, Adam (lr 1e-3), γ = 0.95, replay buffer 5,000, batch size 32.

## Key Results

### Destruction Portfolio Comparison (56 Solomon instances, no local refinement)
| Portfolio | Actions | Mean Δ (%) | Min Δ (%) | Max Δ (%) | Recorded time (s) |
|-----------|---------|-----------|-----------|-----------|-------------------|
| Full      | 4 | 8.225 | −5.25 | 26.619 | 36,581.1 |
| Merged3   | 3 | 9.235 | −5.23 | 25.189 | 35,881.2 |
| Merged2   | 2 | **7.554** | −5.23 | 25.481 | 34,854.7 |

### Complete Pipeline (Merged2 + fixed solver settings + local refinement)
| Group | Instances | Mean Δ (%) | Reference vehicles | Obtained vehicles |
|-------|-----------|-----------|--------------------|-------------------|
| C1  | 9  | 16.04 | 90  | 98  |
| C2  | 8  | 3.74  | 24  | 28  |
| R1  | 12 | 7.13  | 143 | 179 |
| R2  | 11 | 0.10  | 30  | 46  |
| RC1 | 8  | 10.77 | 92  | 119 |
| RC2 | 8  | 2.02  | 26  | 40  |
| **All** | **56** | **6.49** | **405** | **510** |

- C101 reaches the reference exactly (828.94, 10 vehicles).
- 510 vehicles vs. 405 in the references (+25.93%), a consequence of the distance-only objective (β = 0).
- Example: R211 has Δ = −10.50% but uses 4 vehicles instead of 2, so it is **not** an improvement under the vehicles-first convention.

### Development Configurations (not a controlled ablation)
| Config | Main change | Mean Δ (%) |
|--------|-------------|-----------|
| C1 | Manual parameters, no I1, full portfolio | 12.90 |
| C2 | Fewer evaluation episodes | 9.85 |
| C3 | Taguchi-calibrated hyperparameters | 8.91 |
| C4 | I1 construction, larger training budget | 8.22 |

### Taguchi Calibration (L18, response relative to plain LNS)
- **LNS_ITER** dominates (88.2% of variance), followed by MAX_STEPS (3.8%) and EPS_DECAY (3.7%).
- Retained configuration (Run 18): LNS_ITER = 100, K_DESTROY_RATIO = 0.24, GUROBI_TLIMIT = 2, GUROBI_GAP = 0.05, MAX_STEPS = 15, EPS_DECAY = 0.99, VEHICLE_PENALTY = 0.
- The smaller-the-better S/N criterion rewards small improvement magnitudes on this signed response (Run 18 selected, although Run 2 had the largest raw improvement). This is reported as a methodological finding, and Run 18 is treated as the *evaluated* configuration, not an optimal one.

## Insights
- **Merged2** obtains the lowest mean deviation among the three portfolios, but the comparison changes both portfolio size and composition, so it does not show that fewer actions alone helps.
- **Route fusion** is almost inert (1 merge over 56 instances); the recorded gain comes from 2-opt/Or-opt.
- **R2, C2 and RC2** (long time windows) are close to the references; **C1 and RC1** show the largest deviations.
- The 6.49% result is a **separate execution** that also fixes solver seed/threads and doubles the repair time limit, so its gain cannot be attributed to local search alone.


## Conclusion
This project investigates a hybrid approach to the Vehicle Routing Problem with Time Windows, combining DQN-based destruction-operator selection with MILP repair. Gurobi reconstructs the destroyed neighborhood within a limited time budget, while an insertion heuristic followed by a short non-learned LNS provides the initial solution. The agent learns from the distance change observed after each destroy-and-repair cycle, and learning is restricted to operator selection.

On the 56 Solomon instances, the exploratory portfolio comparison yields mean distance deviations of 8.22% (four actions), 9.23% (three actions) and 7.55% (two actions). A separate complete-pipeline execution, with fixed solver settings, a doubled repair time limit and local refinement, yields 6.49%, using 510 vehicles against 405 in the references. These results are preliminary: they do not isolate the contribution of the DQN, and they cannot be compared like for like with references that prioritize fleet size.

**Future work:** investigate when learned destruction-operator selection improves MILP-based repair, and extend the framework to other time-critical transportation problems, such as healthcare and emergency transportation scheduling.

## Requirements
- Python 3.12
- PyTorch, NumPy
- `gurobipy` 13.0.2 (`pip install gurobipy`). The pip version includes a size-limited license; a full or academic license is needed for larger repair models.
