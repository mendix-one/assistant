# Knowledge Graphs & Reasoning for Resource Planning

## 1. Why Knowledge Graphs?

Resource planning involves **domain knowledge** that pure ML models cannot learn from data alone:

- Equipment X is **qualified** for process Y but **not** process Z
- Engineer A has **certification** for 3nm EUV but certification **expires** in 60 days
- Product family DDR5 **requires** equipment set {litho, etch, dep, test} in **specific sequence**
- Customer SLA **mandates** maximum 12-week lead time with **penalty** clauses
- Fab line 17 **cannot** run both HBM and standard DRAM **simultaneously** due to contamination risk

These are **relational, rule-based, and evolving constraints** — perfectly suited for knowledge graphs (KGs).

## 2. Knowledge Graph Architecture

### 2.1 Ontology Design for DS Division

```
Core Ontology Classes:

Resource
  |-- Equipment
  |     |-- LithoTool, EtchTool, DepTool, TestTool, PackageTool
  |-- HumanResource
  |     |-- Engineer, Technician, Operator
  |-- Material
        |-- Wafer, Chemical, Gas, Photomask

Product
  |-- MemoryProduct (DRAM, NAND, HBM)
  |-- LogicProduct (Exynos, ISOCELL, DDI)
  |-- FoundryProduct

Process
  |-- ProcessFlow (sequence of steps)
  |-- ProcessStep (individual operation)
  |-- ProcessRecipe (parameters for a step)

Facility
  |-- Site, Fab, Line, Bay, Cleanroom

Constraint
  |-- QualificationConstraint
  |-- CapacityConstraint
  |-- TemporalConstraint
  |-- ComplianceConstraint

Customer
  |-- Account, SLA, Contract
```

### 2.2 Key Relationships (Properties)

```
Equipment  --qualifiedFor-->     ProcessStep
Equipment  --locatedIn-->        Bay
Equipment  --hasCapacity-->      CapacitySpec
Engineer   --certifiedFor-->     ProcessStep
Engineer   --memberOf-->         Team
Product    --requiresFlow-->     ProcessFlow
ProcessFlow --hasStep-->         ProcessStep  (with ordering)
ProcessStep --usesEquipment-->   EquipmentClass
ProcessStep --requiresSkill-->   SkillCategory
Facility   --supportsNode-->     TechnologyNode
Customer   --hasSLA-->           SLA
SLA        --maxLeadTime-->      Duration
SLA        --penaltyRate-->      Currency
Constraint --appliesTo-->        (Resource | Product | Facility)
Constraint --validDuring-->      TimePeriod
```

### 2.3 Example Knowledge Graph (Partial)

```
[ASML NXE:3800 #3] --qualifiedFor--> [3nm EUV Litho Step]
                    --locatedIn-->    [Hwaseong Fab S8 Line15 Bay-A]
                    --hasCapacity-->  {wph: 120, availability: 0.92}
                    --requiresMaint-> [PM every 720 hours]

[3nm EUV Litho Step] --partOf-->     [3nm GAP ProcessFlow]
                     --precedesStep-> [3nm Etch Step]
                     --requiresSkill->[EUV Operation Cert Level 3]

[Engineer Kim]       --certifiedFor-> [3nm EUV Litho Step]
                     --certExpiry-->  "2026-06-15"
                     --memberOf-->    [Team Alpha]
                     --locatedAt-->   [Hwaseong]

[HBM4 Product]      --requiresFlow-> [3nm GAP ProcessFlow]
                     --orderedBy-->   [Customer A]
                     --hasPriority--> 1

[Customer A SLA]     --maxLeadTime--> "12 weeks"
                     --penaltyRate--> "$50K/week/lot"
```

## 3. Reasoning Capabilities

### 3.1 Constraint Validation via Graph Queries

**"Can lot X be scheduled on equipment Y?"**

```cypher
// Neo4j Cypher query
MATCH (lot:Lot {id: 'LOT-2847'})-[:forProduct]->(prod:Product)
      -[:requiresFlow]->(flow:ProcessFlow)-[:hasStep]->(step:ProcessStep)
      -[:usesEquipmentClass]->(eqClass:EquipmentClass)
MATCH (eq:Equipment {id: 'EUV-3'})-[:qualifiedFor]->(step)
MATCH (eq)-[:locatedIn]->(bay:Bay)-[:partOf]->(line:Line)
WHERE eq.status = 'available'
  AND NOT exists((eq)-[:underMaintenance]->())
  AND step.name = lot.currentStep
RETURN eq.id, step.name, eq.capacity
```

**"Which engineers can operate this equipment and are available next week?"**

```cypher
MATCH (eq:Equipment {id: 'EUV-3'})-[:requiresSkill]->(skill:Skill)
MATCH (eng:Engineer)-[:certifiedFor]->(skill)
WHERE eng.certExpiry > date('2026-03-12')
  AND eng.location = eq.location
  AND NOT exists((eng)-[:assignedTo]->(:Task {week: 'W11-2026'}))
RETURN eng.name, eng.team, eng.certExpiry
```

### 3.2 Impact Analysis via Graph Traversal

**"If equipment EUV-3 goes down, what is impacted?"**

```cypher
MATCH (eq:Equipment {id: 'EUV-3'})-[:qualifiedFor]->(step:ProcessStep)
      <-[:hasStep]-(flow:ProcessFlow)<-[:requiresFlow]-(prod:Product)
      <-[:forProduct]-(lot:Lot)-[:orderedBy]->(cust:Customer)
      -[:hasSLA]->(sla:SLA)
RETURN prod.name, lot.id, cust.name, sla.maxLeadTime, lot.deadline,
       duration.between(date(), lot.deadline) as daysRemaining
ORDER BY daysRemaining ASC
```

This traversal instantly reveals: which products, lots, and customers are affected, and which are at risk of SLA breach.

### 3.3 Rule-Based Inference

```
Rules encoded in the KG:

Rule 1: Contamination Prevention
  IF product.type = 'HBM' AND line.currentProduct.type = 'standard_DRAM'
  THEN changeover_required = TRUE AND changeover_hours = 48

Rule 2: Certification Expiry Warning
  IF engineer.certExpiry < today() + 30 days
  THEN alert(manager, "Recertification needed for {engineer} on {skill}")

Rule 3: Capacity Constraint Propagation
  IF equipment.utilization > 95% AND equipment.isBottleneck = TRUE
  THEN propagate_constraint_to(all products using this equipment)

Rule 4: SLA Risk Detection
  IF lot.estimatedCompletion > lot.deadline - sla.bufferDays
  THEN escalate(customer_team, lot, risk_level = "HIGH")
```

## 4. KG + LLM Integration

### 4.1 LLM as Natural Language Interface to KG

```
User Query (natural language):
  "What happens to Customer A deliveries if we take EUV-3 offline
   for maintenance next week?"

LLM Pipeline:
  1. Parse intent: impact analysis, equipment=EUV-3, time=next week
  2. Generate Cypher/SPARQL query against KG
  3. Execute query, get structured results
  4. LLM synthesizes answer with MDA context

LLM Response:
  "Taking EUV-3 offline in Week 11 impacts 3 lots for Customer A:
   - LOT-2847 (HBM4): Currently at EUV step, would be delayed 5 days.
     SLA risk: LOW (18 days of buffer remain)
   - LOT-2850 (HBM4): Queued for EUV, would be delayed 8 days.
     SLA risk: MEDIUM (10 days of buffer)
   - LOT-2853 (DDR5): Can be rerouted to EUV-5 (qualified, 60% utilized)
     SLA risk: NONE

   Recommendation: Reroute LOT-2853 to EUV-5, process LOT-2847 before
   maintenance window, defer LOT-2850 maintenance to Week 12 if possible."
```

### 4.2 LLM for KG Construction & Maintenance

```
Unstructured Source                LLM Extraction            KG Update
+---------------------+          +-----------------+        +----------+
| Equipment manual:   |          | Extract:        |        | CREATE   |
| "ASML NXE:3800 req- |  LLM    | Equipment type  |  API   | (:Equip  |
|  uires PM every 30  |-------->| PM interval     |------->|  {pm:720}|
|  days of operation"  |          | Condition       |        | )        |
+---------------------+          +-----------------+        +----------+

| Engineer email:      |          | Extract:        |        | UPDATE   |
| "Kim completed 3nm  |  LLM    | Person          |  API   | (:Eng    |
|  EUV cert on March 1"|-------->| Certification   |------->|  {cert:  |
|                      |          | Date            |        |  3nm_EUV}|
+---------------------+          +-----------------+        +----------+
```

## 5. KG + MDA Integration

```
Knowledge Graph                  MDA (OLAP Cube)
+-------------------+            +-------------------+
| Constraints &     |            | Measures &        |
| Rules             |            | Aggregations      |
|                   |            |                   |
| "EUV-3 qualified  |  Feeds    | "EUV utilization  |
|  for 3nm only"    |---------->|  by product: 95%" |
|                   |            |                   |
| "Kim certified    |  Feeds    | "Team Alpha       |
|  for 3nm EUV"     |---------->|  available hours:  |
|                   |            |  120h next week"  |
| "Customer A SLA:  |  Feeds    | "Customer A lots  |
|  12 week max"     |---------->|  on-time rate: 92%"|
+-------------------+            +-------------------+

KG provides: WHAT is allowed (constraints, rules, relationships)
MDA provides: HOW MUCH is happening (metrics, trends, comparisons)
Together:     WHAT SHOULD happen (informed decisions)
```

## 6. Technology Stack

| Component | Recommended | Alternative |
|-----------|-------------|-------------|
| Graph Database | Neo4j | Amazon Neptune, TigerGraph |
| Ontology Language | OWL 2 / RDFS | SHACL for validation |
| Query Language | Cypher (Neo4j) | SPARQL, Gremlin |
| LLM Integration | Claude API / Anthropic SDK | GPT-4, local LLM |
| KG Construction | LLM extraction + human validation | Manual + NER tools |
| Reasoning Engine | Neo4j GDS + custom rules | Apache Jena (OWL reasoner) |
| Integration with ES | Neo4j -> ETL -> Elasticsearch | Direct API bridge |

---

## Sources

- [Knowledge Graphs in Manufacturing - Survey](https://www.sciencedirect.com/science/article/pii/S0278612524001572)
- [Knowledge Graphs in Manufacturing and Production - Systematic Review](https://arxiv.org/pdf/2012.09049)
- [Knowledge Graphs: Connecting the Dots in Manufacturing Intelligence](https://mathco.com/article/knowledge-graphs-connecting-the-dots-in-manufacturing-intelligence/)
- [Knowledge Graph-Based Framework for Industry 5.0](https://www.mdpi.com/2076-3417/14/8/3398)
- [Role of Knowledge Graphs in Agentic AI Systems](https://zbrain.ai/knowledge-graphs-for-agentic-ai/)
- [From LLMs to Knowledge Graphs - Production Systems 2025](https://medium.com/@claudiubranzan/from-llms-to-knowledge-graphs-building-production-ready-graph-systems-in-2025-2b4aff1ec99a)
