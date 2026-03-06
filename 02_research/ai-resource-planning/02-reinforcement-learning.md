# Reinforcement Learning for Scheduling

## 1. Why RL for Scheduling?

Traditional scheduling methods (heuristic rules, LP, CP) struggle with:
- **Dynamic environments** where jobs arrive continuously
- **Stochastic elements** (equipment breakdowns, yield variation, rush orders)
- **Reentrant flows** common in semiconductor manufacturing (wafers revisit same tools)
- **Multi-objective optimization** (minimize makespan + maximize utilization + meet deadlines)

RL learns **adaptive policies** that respond to real-time state changes, something static optimization cannot do.

## 2. RL Formulation for Resource Scheduling

### 2.1 Markov Decision Process (MDP)

| MDP Component | Resource Planning Mapping |
|---------------|--------------------------|
| **State (s)** | Current resource allocations, equipment status, WIP levels, queue lengths, time remaining on active jobs |
| **Action (a)** | Assign resource X to task Y on equipment Z at time T |
| **Reward (r)** | Composite: -penalty(tardiness) + bonus(utilization) - cost(changeover) - penalty(constraint_violation) |
| **Transition (P)** | Stochastic: processing times vary, equipment may fail, new jobs arrive |
| **Policy (pi)** | Learned mapping from state to optimal action |

### 2.2 State Representation for DS Division

```python
state = {
    # Resource states
    "equipment_status": {
        "EUV_scanner_1": {"status": "running", "job": "lot_2847",
                          "remaining_hours": 4.2, "fab_line": "Line17"},
        "EUV_scanner_2": {"status": "idle", "last_maintenance": "2026-03-01"},
        ...
    },
    # Queue states (per equipment class)
    "queues": {
        "EUV_litho": [
            {"lot": "lot_2848", "product": "HBM4", "priority": 1,
             "deadline": "2026-03-15", "wait_hours": 2.5},
            {"lot": "lot_1052", "product": "foundry_X", "priority": 2,
             "deadline": "2026-03-20", "wait_hours": 8.0},
        ],
        ...
    },
    # Global metrics
    "overall_utilization": 0.87,
    "constraint_violations": 2,
    "time_step": 1440,  # minutes since start
}
```

### 2.3 Action Space

```python
actions = {
    "assign":     (resource_id, task_id, start_time),
    "preempt":    (resource_id, current_task, replacement_task),
    "defer":      (task_id, defer_until),
    "reroute":    (task_id, from_equipment, to_equipment),
    "maintenance": (equipment_id, start_time, duration),
}
```

### 2.4 Reward Function Design

```python
def reward(state, action, next_state):
    r = 0.0

    # Positive rewards
    r += w1 * tasks_completed(next_state)          # throughput
    r += w2 * utilization_improvement(state, next_state)  # efficiency

    # Negative penalties
    r -= w3 * tardiness_penalty(next_state)         # deadline miss
    r -= w4 * constraint_violations(next_state)     # hard constraint breach
    r -= w5 * changeover_cost(action)               # setup time
    r -= w6 * idle_cost(next_state)                 # equipment idle

    # Weights (tunable per business priority)
    # w1=1.0, w2=0.3, w3=2.0, w4=5.0, w5=0.5, w6=0.2
    return r
```

## 3. Deep RL Architectures

### 3.1 Single-Agent DRL

Best for: scheduling a single fab line or equipment group.

```
                    State Encoder
                    (Equipment status, queues, WIP)
                         |
                    +----v----+
                    |  Neural |
                    | Network |
                    | (DQN or |
                    |  PPO)   |
                    +----+----+
                         |
                    Action Selection
                    (Assign lot X to tool Y)
                         |
                    Environment Step
                    (Fab simulation or real system)
                         |
                    Reward + Next State
```

**Algorithms:**

| Algorithm | Type | Best For |
|-----------|------|----------|
| **DQN** (Deep Q-Network) | Value-based | Discrete action spaces, single equipment type |
| **PPO** (Proximal Policy Optimization) | Policy gradient | Continuous/large action spaces, robust training |
| **SAC** (Soft Actor-Critic) | Off-policy, entropy-regularized | Continuous control, exploration-exploitation balance |

### 3.2 Multi-Agent RL (MARL)

Best for: multi-fab, multi-product scheduling where different agents manage different resources.

```
+-------------+    +-------------+    +-------------+
| Agent 1:    |    | Agent 2:    |    | Agent 3:    |
| EUV Litho   |    | Etch Bay    |    | Test Lab    |
| scheduler   |    | scheduler   |    | scheduler   |
+------+------+    +------+------+    +------+------+
       |                  |                  |
       +--------+---------+--------+---------+
                |                  |
         Shared State         Communication
         (Global WIP,         (Coordination
          utilization)         protocol)
                |                  |
         +------v------------------v------+
         |        Fab Environment          |
         |  (Shared simulation / real fab) |
         +---------------------------------+
```

**Key MARL approaches for manufacturing:**

| Approach | Description | Advantage |
|----------|-------------|-----------|
| **Independent DQN** | Each agent learns independently, treats others as environment | Simple, scalable |
| **QMIX** | Monotonic value decomposition across agents | Cooperative, handles partial observability |
| **MAPPO** | Multi-agent PPO with shared critic | Stable training, good for cooperative tasks |
| **Heterogeneous GNN + MARL** | Graph encodes machine-operation relationships | Captures spatial structure of fab layout |

### 3.3 Graph Neural Networks (GNN) for Scheduling

GNNs naturally model the **relational structure** of scheduling problems:

```
Job-Shop Scheduling as a Graph:

    [Op1] ---depends_on---> [Op2] ---depends_on---> [Op3]     (Job 1)
      |                       |                       |
   runs_on               runs_on                  runs_on
      |                       |                       |
   [Machine A]            [Machine B]             [Machine A]  (Resources)
      |                       |                       |
   runs_on               runs_on                  runs_on
      |                       |                       |
    [Op4] ---depends_on---> [Op5]                              (Job 2)

Node features: processing time, priority, current queue position
Edge features: dependency type, transport time
```

**Architecture: Heterogeneous GNN + PPO**

```
Input Graph                  GNN Layers              Policy Head
(operations,         +----> Message Passing     +--> Action probabilities
 machines,           |      (3-5 layers)        |    (which op to schedule
 jobs as nodes)  ----+      Attention-based  ----+    on which machine)
                     |      aggregation         |
                     +----> Node embeddings ----+--> Value estimate
                            (128-256 dim)            (state value for
                                                      critic network)
```

**Why GNN for semiconductor scheduling:**
- Naturally encodes **reentrant flows** (same machine appears multiple times)
- Handles **variable-size problems** (different numbers of lots, machines)
- Captures **spatial relationships** between equipment in fab layout
- Generalizes to **unseen problem instances** (new products, new machines)

## 4. Training Strategy

### 4.1 Simulation-Based Training

RL agents cannot be trained directly on a live fab. Use simulation:

```
Training Pipeline:

1. Build fab simulator
   - Model equipment processing times (stochastic)
   - Model breakdowns (Weibull distribution)
   - Model job arrivals (from historical data / demand forecast)
   - Model constraint rules (from knowledge graph)

2. Train RL agent in simulation
   - Millions of episodes
   - Curriculum learning: start simple, increase complexity
   - Domain randomization: vary parameters to improve robustness

3. Validate in simulation
   - Compare vs heuristic baselines (FIFO, SPT, EDD, critical ratio)
   - Compare vs meta-heuristic solutions (GA, SA)
   - Stress test with edge cases

4. Deploy to real system
   - Shadow mode first (recommend but don't execute)
   - Gradual rollout with human override
   - Continuous monitoring + retraining
```

### 4.2 Baseline Comparisons

| Method | Type | Expected Performance |
|--------|------|---------------------|
| FIFO (First In First Out) | Rule | Baseline, fair but suboptimal |
| SPT (Shortest Processing Time) | Rule | Good for throughput, ignores priorities |
| EDD (Earliest Due Date) | Rule | Good for tardiness, ignores utilization |
| Critical Ratio | Rule | Balanced, but fixed heuristic |
| Genetic Algorithm | Meta-heuristic | Strong offline solutions, slow for real-time |
| **DRL Agent** | **Learned** | **Best real-time adaptivity, near-optimal** |
| **MARL** | **Learned** | **Best for multi-resource coordination** |

## 5. Application to DS Division

### 5.1 EUV Lithography Scheduling (High-Value Bottleneck)

```
Problem:
- EUV scanners cost $150M+ each
- Utilization must be >95%
- Multiple products compete for limited EUV capacity
- Changeovers between products cost 4-8 hours

RL Agent Design:
- State: Scanner status, lot queue, product mix, priority levels
- Action: Which lot to process next, whether to batch similar lots
- Reward: Maximize throughput, minimize changeovers, meet priority deadlines
- Architecture: PPO with attention mechanism over lot queue

Expected Impact:
- 2-5% utilization improvement = $20-50M annual value per scanner
```

### 5.2 Multi-Fab Coordination

```
Problem:
- 4+ fabs worldwide (Hwaseong, Pyeongtaek, Austin, Xi'an)
- Some products can be made at multiple fabs
- Need to balance load while respecting qualification constraints

MARL Design:
- Agent per fab, shared global state
- Actions: Accept/reject/redirect lots between fabs
- Communication: Agents share utilization and queue state
- Architecture: MAPPO with centralized critic

Expected Impact:
- Reduce inter-fab imbalance from 15% to <5%
- Decrease total cycle time by 8-12%
```

## 6. Challenges & Mitigations

| Challenge | Mitigation |
|-----------|------------|
| Simulation-to-reality gap | Domain randomization, conservative policies, human oversight |
| Reward function design | Multi-objective with tunable weights, stakeholder-aligned KPIs |
| Training instability | Curriculum learning, PPO (more stable than DQN for large spaces) |
| Explainability | Attention visualization, policy distillation to interpretable rules |
| Action space explosion | Hierarchical RL (high-level: which product, low-level: which tool) |
| Safety constraints | Constrained RL (CPO), hard-coded constraint masks on actions |

---

## Sources

- [Deep RL for Job Scheduling - Algorithm-Level Review](https://arxiv.org/html/2501.01007v1)
- [Multi-Agent RL for Flexible Shop Scheduling - Survey](https://www.frontiersin.org/journals/industrial-engineering/articles/10.3389/fieng.2025.1611512/full)
- [Multi-Agent RL for Resource Allocation - Survey](https://link.springer.com/article/10.1007/s10462-025-11340-5)
- [GNN for Job Shop Scheduling - Survey](https://www.sciencedirect.com/science/article/pii/S0305054824003861)
- [Solving IPPS via GNN-Based DRL](https://arxiv.org/html/2409.00968v1)
- [Fast and Robust Resource-Constrained Scheduling with GNN](https://ojs.aaai.org/index.php/ICAPS/article/download/27244/27017/31313)
- [Heterogeneous GNN Scheduling for Multi-Product FJSP](https://www.mdpi.com/2076-3417/15/10/5648)
