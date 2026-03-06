# Meta-Heuristic Optimization for Resource Planning

## 1. Overview

Meta-heuristic algorithms are **general-purpose optimization methods** that find near-optimal solutions for NP-hard scheduling and allocation problems where exact methods are computationally infeasible.

They are the **most mature AI technique** for production scheduling — proven in industry for 20+ years — and serve as both practical solvers and **baselines for comparing newer RL/GNN approaches**.

## 2. Algorithm Comparison

### 2.1 Core Algorithms

| Algorithm | Inspiration | Mechanism | Strengths | Weaknesses |
|-----------|------------|-----------|-----------|------------|
| **Genetic Algorithm (GA)** | Biological evolution | Population of solutions evolves via crossover, mutation, selection | Global search, handles multi-objective, parallelizable | Slow convergence, parameter tuning (pop size, rates) |
| **Simulated Annealing (SA)** | Metal cooling | Single solution, accepts worse solutions with decreasing probability | Escapes local optima, simple implementation | Slower than GA for large problems, cooling schedule tuning |
| **Particle Swarm Optimization (PSO)** | Bird flocking | Population of particles moves through solution space guided by personal and global best | Fast convergence, few parameters | Can get stuck in local optima, less effective for discrete |
| **Ant Colony Optimization (ACO)** | Ant foraging | Solutions built incrementally, pheromone trails guide future construction | Natural for sequencing/routing problems | Memory-intensive, slow for large instances |
| **Tabu Search (TS)** | Memory-based | Neighborhood search with tabu list preventing revisits | Best for single-objective, fast local search | Limited global exploration, list size tuning |

### 2.2 Performance Comparison for Scheduling

```
Problem: Flexible Job Shop Scheduling (100 jobs, 20 machines, 5 operations/job)

Algorithm      Makespan    Compute Time    Solution Quality    Consistency
                (hours)     (seconds)       (vs optimal)       (std dev)
------------------------------------------------------------------------
GA              142.3        45              97.2%              2.1%
SA              140.8        32              97.8%              1.8%
PSO             144.1        28              96.5%              3.2%
ACO             143.5        52              96.8%              2.5%
Tabu Search     141.2        18              97.6%              1.5%
Hybrid GA+SA    139.5        55              98.5%              1.2%   <-- best

Note: Performance varies by problem instance. No single algorithm dominates all cases.
```

## 3. Application to DS Division Problems

### 3.1 GA for Multi-Objective Capacity Allocation

**Problem:** Allocate fab capacity across product families to maximize revenue while meeting all customer commitments and minimizing changeovers.

```
Chromosome Encoding:
  [DDR5:35%, HBM:25%, NAND:20%, Foundry:15%, Exynos:5%]
   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   Percentage allocation of total fab capacity per product

  + Schedule sequence per product:
  [LOT-A, LOT-C, LOT-B, LOT-E, LOT-D, ...]

Fitness Function (multi-objective):
  f1 = maximize(total_revenue)
  f2 = minimize(total_tardiness)
  f3 = minimize(total_changeovers)

  Pareto front: Find non-dominated solutions across all three objectives

GA Operators:
  Selection:  Tournament selection (size 5)
  Crossover:  Partially Mapped Crossover (PMX) for lot sequence
              Blend crossover for capacity percentages
  Mutation:   Swap mutation for sequence (5% rate)
              Gaussian perturbation for capacity (3% rate)
  Population: 200 individuals
  Generations: 500
```

**NSGA-II for Multi-Objective:**

```
Generation 0:                     Generation 500:
Revenue ↑                         Revenue ↑
  |  *  *   *                       |     * * * *    <- Pareto front
  | *    * *   *                    |    *  *  * *
  |*  *    *  *  *                  |   *    *   *
  | *   *   *                       |  *     *
  |   *  *  *   *                   | *
  +---------> Tardiness ↓           +---------> Tardiness ↓

  Scattered, random                 Converged to Pareto-optimal trade-offs

  Decision maker picks preferred point on Pareto front
```

### 3.2 SA for Equipment Maintenance Scheduling

**Problem:** Schedule preventive maintenance windows across equipment to minimize production disruption while meeting maintenance requirements.

```
Solution Representation:
  maintenance_schedule = {
      "EUV-1": {"week": 12, "shift": "Night", "duration": 48},
      "EUV-2": {"week": 14, "shift": "Night", "duration": 48},
      "Etch-A": {"week": 11, "shift": "Day", "duration": 24},
      ...
  }

Objective: minimize(production_loss + risk_penalty)
  production_loss = sum(lost_wafers * wafer_value for each maintenance)
  risk_penalty = sum(overdue_penalty for each delayed maintenance)

Neighbor Generation:
  - Shift a maintenance window by +/- 1 week
  - Swap maintenance slots between two equipment
  - Change shift (Day <-> Night)

Cooling Schedule:
  T_initial = 1000
  T_final = 0.1
  alpha = 0.995  (geometric cooling)
  iterations_per_temperature = 100
```

### 3.3 PSO for Workforce Allocation

**Problem:** Assign engineers to projects/fab lines optimizing skill match, workload balance, and project priority coverage.

```
Particle Representation:
  position = [eng1_project, eng2_project, ..., engN_project]
  velocity = [delta_1, delta_2, ..., delta_N]

  Each dimension maps an engineer to a project/fab assignment

Fitness: maximize(
    w1 * skill_match_score +      # Engineer skills match project needs
    w2 * priority_coverage +       # High-priority projects are fully staffed
    w3 * workload_balance -        # Even distribution across engineers
    w4 * relocation_cost           # Minimize site transfers
)

PSO Update:
  v_i(t+1) = w*v_i(t)
             + c1*r1*(pbest_i - x_i(t))    # cognitive (personal best)
             + c2*r2*(gbest - x_i(t))       # social (global best)
  x_i(t+1) = x_i(t) + v_i(t+1)

  Discrete adaptation: Round continuous position to nearest valid assignment
```

## 4. Hybrid Approaches

### 4.1 GA + Local Search (Memetic Algorithm)

```
for generation in range(max_gen):
    # GA global search
    parents = tournament_selection(population)
    offspring = crossover(parents)
    offspring = mutate(offspring)

    # Local search refinement (on top offspring)
    for individual in top_k(offspring, k=20):
        individual = local_search(individual, method="tabu", steps=100)

    population = survivor_selection(population + offspring)
```

### 4.2 Meta-Heuristic + RL Hybrid

```
Phase 1: Meta-heuristic generates initial schedule
  - GA finds a good base schedule (offline, takes minutes)
  - This becomes the starting point

Phase 2: RL agent handles real-time adjustments
  - When disruptions occur (breakdown, rush order, yield drop)
  - RL agent adjusts the base schedule in real-time
  - Constrained to stay within boundaries of Phase 1 solution

Benefit: Best of both worlds
  - GA: Strong global optimization, no training data needed
  - RL: Fast real-time adaptation, handles stochastic events
```

### 4.3 Meta-Heuristic + Digital Twin

```
Digital Twin Simulation <--> Meta-Heuristic Solver

Loop:
  1. Meta-heuristic proposes candidate schedule
  2. Digital twin simulates the schedule under stochastic conditions
  3. Simulation results become the fitness evaluation
  4. Meta-heuristic evolves based on simulation feedback
  5. Repeat until convergence

Advantage: Fitness is evaluated in a realistic simulation,
           not a simplified mathematical model
```

## 5. When to Use Meta-Heuristics vs Other AI

| Criterion | Meta-Heuristic | Deep RL | Exact (LP/MIP) |
|-----------|---------------|---------|----------------|
| Problem size | Medium-Large | Large | Small-Medium |
| Solution quality | Near-optimal (95-99%) | Near-optimal | Optimal (guaranteed) |
| Compute time | Minutes-hours | Seconds (after training) | Hours-days |
| Training data needed | None | Lots (simulation) | None |
| Real-time adaptivity | Low (re-run needed) | High | Very low |
| Interpretability | Medium (solution visible) | Low (black box) | High |
| Implementation effort | Low-Medium | High | Medium |
| Dynamic environments | Poor (static optimization) | Excellent | Poor |
| Multi-objective | Good (NSGA-II) | Emerging | Limited |

**Decision Guide:**

```
Is the problem static (plan once, execute)?
  YES --> Meta-heuristic or Exact method
  NO  --> RL for real-time adaptation

Is the problem small enough for exact methods? (< ~1000 variables)
  YES --> LP/MIP solver (Gurobi, CPLEX)
  NO  --> Meta-heuristic or RL

Do you need multi-objective Pareto analysis?
  YES --> NSGA-II (GA-based)
  NO  --> Any method

Is training data/simulation available?
  NO  --> Meta-heuristic (no training needed)
  YES --> Consider RL for superior real-time performance

Is this a one-time strategic decision?
  YES --> Meta-heuristic + Digital Twin validation
  NO  --> RL for ongoing operational scheduling
```

## 6. Implementation with Python

### Key Libraries

| Library | Purpose | License |
|---------|---------|---------|
| **DEAP** | GA, PSO, GP — flexible evolutionary computation | LGPL |
| **pymoo** | Multi-objective optimization (NSGA-II, NSGA-III) | Apache 2.0 |
| **OR-Tools** | Google's constraint programming + meta-heuristics | Apache 2.0 |
| **scipy.optimize** | Simulated annealing (dual_annealing) | BSD |
| **jMetalPy** | Multi-objective meta-heuristics | MIT |
| **Optuna** | Hyperparameter optimization (TPE, CMA-ES) | MIT |

---

## Sources

- [Metaheuristics for Multi-Objective Scheduling in Industry 4.0 and 5.0 - Survey](https://www.frontiersin.org/journals/industrial-engineering/articles/10.3389/fieng.2025.1540022/full)
- [Comparative Study of Meta-Heuristic Algorithms](https://arxiv.org/pdf/1407.4863)
- [Comparison of SA and GA for Integrated Process Routing and Scheduling](https://ijisae.org/index.php/IJISAE/article/view/940)
- [Hybrid Evolution Strategies-SA for Job Shop Scheduling](https://www.sciencedirect.com/science/article/abs/pii/S095219762400174X)
- [SA Metaheuristic for Hybrid Flow Shop Scheduling](https://www.sciencedirect.com/science/article/pii/S2666912924000096)
- [Metaheuristics in Optimization - INFORMS](https://www.informs.org/Publications/OR-MS-Tomorrow/Metaheuristics-in-Optimization-Algorithmic-Perspective)
