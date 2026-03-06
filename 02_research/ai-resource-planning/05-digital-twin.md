# Digital Twin & Simulation for Resource Planning

## 1. What is a Digital Twin?

A digital twin is a **real-time virtual replica** of physical manufacturing systems (fabs, equipment, processes) that mirrors actual operational state using sensor data, MES feeds, and IoT inputs.

For resource planning, digital twins serve as **risk-free testing environments** for scheduling decisions, capacity changes, and what-if scenarios.

```
Physical Fab                              Digital Twin
+---------------------------+             +---------------------------+
| Real equipment            |  real-time  | Virtual equipment         |
| Real WIP                  |  data sync  | Virtual WIP               |
| Real sensor data          |------------>| Simulated physics         |
| Real operator actions     |             | AI-driven predictions     |
+---------------------------+             +---------------------------+
                                                    |
                                          +---------v---------+
                                          | What-if scenarios  |
                                          | Optimization       |
                                          | Predictive alerts  |
                                          +-------------------+
```

## 2. Digital Twin Architecture for DS Division

### 2.1 Layered Architecture

```
Layer 4: Decision & Optimization
+----------------------------------------------------------+
| What-if scenario engine | RL-based scheduler | Alert system|
+----------------------------------------------------------+
                          |
Layer 3: AI/Analytics
+----------------------------------------------------------+
| Demand forecast | Yield prediction | Predictive maintenance|
| Bottleneck detection | Capacity optimization              |
+----------------------------------------------------------+
                          |
Layer 2: Digital Twin Model
+----------------------------------------------------------+
| Discrete Event Simulation (DES) Engine                    |
| - Equipment models (processing, breakdown, maintenance)   |
| - Process flow models (reentrant, multi-step)             |
| - Transport models (AMHS, inter-bay)                      |
| - Operator models (shifts, skills, availability)          |
+----------------------------------------------------------+
                          |
Layer 1: Data Integration
+----------------------------------------------------------+
| MES data | Equipment sensors | ERP | IoT | SCADA          |
| Real-time sync via Kafka / MQTT                            |
+----------------------------------------------------------+
                          |
Physical Layer
+----------------------------------------------------------+
| Hwaseong Fab | Pyeongtaek Fab | Austin Fab | Xi'an Fab    |
+----------------------------------------------------------+
```

### 2.2 Simulation Model Components

| Component | Model Type | Key Parameters |
|-----------|-----------|----------------|
| **Equipment** | Stochastic processing | Processing time (mean, variance), MTBF, MTTR, qualification matrix |
| **Process Flow** | Directed graph with reentry | Step sequence, branching rules, rework probability |
| **Transport** | Queue + travel time | AMHS speed, buffer sizes, priority rules |
| **Operators** | Shift schedule + skill matrix | Availability windows, certification status |
| **Batching** | Batch formation rules | Min/max batch size, compatible products |
| **Yield** | Probabilistic per step | Base yield, learning curve, defect density |
| **Demand** | Stochastic arrivals | From ML forecast model (P10, P50, P90) |

## 3. What-If Scenario Simulation

### 3.1 Scenario Types and Twin Usage

| Scenario Category | Example | Twin Simulation Approach |
|-------------------|---------|--------------------------|
| **Equipment change** | "Add one more EUV scanner" | Clone twin, add equipment entity, run 6-month simulation |
| **Demand shock** | "HBM demand increases 40%" | Inject increased arrival rate, observe bottleneck emergence |
| **Process change** | "3nm yield improves from 78% to 85%" | Modify yield parameters, measure throughput and cycle time |
| **Scheduling policy** | "Switch from FIFO to RL-based dispatching" | Swap dispatch logic, compare KPIs across 10K simulation runs |
| **Maintenance strategy** | "Move from calendar-based to predictive maintenance" | Swap maintenance model, compare uptime and disruption risk |
| **Workforce change** | "Reduce night shift by 20%" | Modify operator model, assess impact on throughput |

### 3.2 Simulation Workflow

```
1. Define Scenario
   - Parameter changes vs baseline
   - Time horizon (weeks, months, quarters)
   - Number of replications (for statistical confidence)

2. Configure Digital Twin
   - Clone baseline twin state
   - Apply parameter modifications
   - Set random seeds for reproducibility

3. Execute Simulation
   - Run discrete event simulation
   - Collect metrics at configured intervals
   - Store results per replication

4. Analyze Results
   - Aggregate across replications (mean, CI)
   - Compare vs baseline scenario
   - Identify statistically significant differences

5. Feed Results to MDA
   - Load scenario results into FactCapacity, FactDemand
   - scenario_id = new scenario identifier
   - Available for OLAP drill-down, slice, comparison

6. Decision
   - Present results in Kibana dashboard
   - Stakeholder review
   - Approve / modify / reject scenario
```

### 3.3 Example: EUV Capacity Expansion Scenario

```
Baseline:
  - Hwaseong Fab S8, Line 15: 2 EUV scanners
  - Current utilization: 97% (bottleneck)
  - Throughput: 45K wafers/month for 3nm

Scenario: Add 3rd EUV scanner (available Q3-2026)

Simulation Results (1000 replications, 12-month horizon):

Metric                   Baseline        With 3rd EUV     Delta
--------------------------------------------------------------
EUV utilization           97.2%           78.5%           -18.7%
Line throughput (wpm)     45,200          58,800          +30.1%
Avg cycle time (days)     42.3            35.1            -17.0%
On-time delivery          82%             96%             +14%
WIP level (lots)          2,340           1,890           -19.2%

Bottleneck shift:
  Before: EUV Litho (97.2%)
  After:  Etch Bay (91.5%) -- new bottleneck!

Financial Impact:
  CapEx: $180M (scanner + installation)
  Revenue uplift: $420M/year (from increased throughput)
  Payback: 5.1 months

Recommendation: APPROVE. ROI is strong. Plan etch capacity
                expansion for Phase 2 to avoid new bottleneck.
```

## 4. Real-Time Digital Twin for Monitoring

### 4.1 Live State Synchronization

```
MES Events --> Kafka --> Twin State Update --> Dashboard

Events:
  lot_started(lot_id, equipment_id, timestamp)
  lot_completed(lot_id, equipment_id, timestamp, yield)
  equipment_down(equipment_id, reason, estimated_repair)
  new_lot_released(lot_id, product, priority, customer)
  maintenance_started(equipment_id, type, duration)
```

### 4.2 Predictive Capabilities

| Capability | Method | Lookahead |
|------------|--------|-----------|
| **WIP projection** | DES from current state forward | 1-4 weeks |
| **Completion date prediction** | Remaining steps x avg processing time | Per lot |
| **Bottleneck prediction** | Trend analysis on utilization trajectory | 2-6 weeks |
| **SLA risk detection** | Simulated completion vs deadline | Per customer order |
| **Maintenance prediction** | Equipment degradation model (sensor-based) | 1-4 weeks |

### 4.3 Closed-Loop Optimization

```
     +----------+      +-----------+      +----------+
     |  Sensors |      | Digital   |      | RL-based |
     |  + MES   |----->| Twin      |----->| Scheduler|
     |          |      | (current  |      | (optimal |
     +----------+      |  state)   |      |  action) |
          ^            +-----------+      +-----+----+
          |                                     |
          |            +-----------+            |
          +------------| Execution |<-----------+
                       | (apply    |
                       |  schedule)|
                       +-----------+

Loop frequency: Every 15 minutes to 1 hour
Human override: Always available, alerts on deviation
```

## 5. Integration with MDA and Elasticsearch

### 5.1 Data Flow

```
Digital Twin                    Elasticsearch              Kibana
Simulation Results              Indices                    Dashboards
+-------------------+          +-------------------+      +---------------+
| Scenario A output |  ETL     | capacity-plan     |      | Scenario      |
|   - throughput    |--------->|   scenario_id =   |----->| comparison    |
|   - utilization   |          |   "sim-euv-add"   |      | dashboard     |
|   - cycle time    |          +-------------------+      +---------------+
|   - WIP levels    |          | scenario-results  |      | What-if       |
+-------------------+  ETL     |   - sim metadata  |----->| explorer      |
| Scenario B output |--------->|   - confidence    |      |               |
|   ...             |          |     intervals     |      +---------------+
+-------------------+          +-------------------+
```

### 5.2 Scenario Results Index

```json
PUT scenario-results
{
  "mappings": {
    "properties": {
      "scenario_id":        { "type": "keyword" },
      "scenario_name":      { "type": "keyword" },
      "parent_scenario":    { "type": "keyword" },
      "simulation_run":     { "type": "integer" },
      "timestamp":          { "type": "date" },

      "facility_site":      { "type": "keyword" },
      "product_family":     { "type": "keyword" },
      "time_period":        { "type": "keyword" },

      "throughput_wpm":     { "type": "float" },
      "utilization_pct":    { "type": "float" },
      "cycle_time_days":    { "type": "float" },
      "wip_lots":           { "type": "integer" },
      "on_time_delivery":   { "type": "float" },
      "revenue_impact_usd": { "type": "float" },
      "capex_required_usd": { "type": "float" },

      "confidence_interval": {
        "properties": {
          "lower": { "type": "float" },
          "upper": { "type": "float" },
          "level": { "type": "float" }
        }
      }
    }
  }
}
```

## 6. Technology Stack

| Component | Recommended | Alternative |
|-----------|-------------|-------------|
| DES Engine | AnyLogic, Simio | SimPy (Python, OSS), Tecnomatix (Siemens) |
| Real-time data | Apache Kafka | MQTT, AWS Kinesis |
| State store | Redis (low-latency) | In-memory in simulation engine |
| Analytics store | Elasticsearch | ClickHouse, TimescaleDB |
| Visualization | Kibana + custom 3D | Grafana, Unity (3D twin) |
| AI integration | Python (PyTorch, Stable-Baselines3) | TensorFlow, Ray RLlib |

---

## Sources

- [Digital Twin Manufacturing Applications - Simio](https://www.simio.com/digital-twin-manufacturing-applications-benefits-and-industry-insights/)
- [Digital Twin Framework for Simulation and Optimization](https://www.sciencedirect.com/science/article/pii/S221282712101026X)
- [AI-Based Digital Twin Applications in Manufacturing - Review](https://www.mdpi.com/2079-9292/14/4/646)
- [Optimizing Manufacturing with Digital Twins and AI Agents](https://www.akira.ai/blog/digital-twins-simulations-with-ai-agents)
- [Digital Twins: Next Frontier of Factory Optimization - McKinsey](https://www.mckinsey.com/capabilities/operations/our-insights/digital-twins-the-next-frontier-of-factory-optimization)
- [Simulation-Based Digital Twin for Data-Driven Decision Optimization](https://www.sciencedirect.com/science/article/pii/S277266222500102X)
