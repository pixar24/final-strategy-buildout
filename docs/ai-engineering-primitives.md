# AI Engineering Primitives for Lending
## Mapping Pandhi's Structural Primitives to a 25-Service AI-First Lending Platform

---

## 1. WHY THIS DOCUMENT EXISTS

Pandhi's 4 structural primitives -- **State, Context, Policy, Feedback** -- were articulated for voice AI agent platforms. They answer: *what are the irreducible engineering abstractions that make AI agents reliable in production?*

Lending is not customer service. It is a **regulated, multi-party, multi-year lifecycle** where:
- A single decision commits capital for 12-60 months
- Regulatory violations carry criminal liability (RBI Act Section 46)
- Borrowers are thin-file, vernacular-first, financially vulnerable
- Context compounds -- the 10,000th loan benefits from 9,999 prior traces
- Post-disbursal monitoring matters more than origination accuracy

The base 4 primitives remain necessary. But lending demands **5 extensions**: DPI/ULI data orchestration, segment intelligence, regulatory compliance as architecture, post-disbursal lifecycle, and context compounding. This document maps all 25 services to both the base primitives and lending extensions.

---

## 2. THE BASE 4 PRIMITIVES -- LENDING INSTANTIATION

### 2.1 PRIMITIVE 1: STATE

**Definition:** Immutable, tamper-evident, auditable record of every action, decision, and state transition. State is ground truth. If it's not in state, it didn't happen.

**Why lending intensifies this:** RBI Digital Lending Guidelines require full audit trails for every credit decision. DPDP Act mandates consent records per data element. A platform that loses state is non-compliant.

**4 State substrates:**

```
  AUDIT TRAIL ───> DECISION TRACES ───> WORKFLOW STATE ───> CONSENT
       │                 │                     │                │
  PostgreSQL         Neo4j graph          Temporal           DPDP Act
  append-only        subgraphs per        durable 14-state   per-element
  SHA-256 chain      decision w/          loan machine,      per-borrower
  every action       full lineage         survives failures  consent ledger
```

- **Audit Trail:** Append-only, partitioned by month, tamper-evident hash chain. Schema: `{event_id, timestamp, actor, action, entity, payload_hash, prev_hash, lender_id, compliance_tags[]}`. Every API call, agent action, policy evaluation, MCP invocation logged.
- **Decision Traces:** Neo4j subgraphs: `(:DecisionTrace)-[:USED_DATA]->(:DataSource)`, `-[:APPLIED_POLICY]->(:Policy {version})`, `-[:PRODUCED_SCORE]->(:CreditScore)`, `-[:HAD_OUTCOME]->(:LoanOutcome)`. Single Cypher query across millions of traces.
- **Temporal Durability:** 14-state loan workflow (CREATED→KYC→DATA_FETCH→DECISIONING→APPROVED/REFERRED/REJECTED→PRODUCT_DESIGN→LENDER_MATCH→OFFER→ACCEPT→DISBURSE→ACTIVE→CLOSED/GRADUATED/WRITTEN_OFF). Each transition emits Kafka event + audit entry + graph node.
- **Consent Ledger:** Per-borrower, per-data-element. No fetch without valid consent. Right to erasure cascades across all stores.

**Services that OWN state:**

| Service | State Responsibility |
|---------|---------------------|
| `audit-trail` | Immutable hash-chain event log; every service writes here |
| `context-graph` | Decision Traces as Neo4j subgraphs; context compounding substrate |
| `los-engine` | Temporal workflow state for every loan lifecycle |
| `compliance-reporter` | Consent ledger; DPDP Act per-element tracking |

---

### 2.2 PRIMITIVE 2: CONTEXT

**Definition:** Assembled, scoped, governed information available at the moment of action. Context is not raw data -- it is resolved, validated, permissioned, and shaped for the specific actor, segment, channel, and jurisdiction.

**Why lending intensifies this:** A dairy farmer in Gujarat applying via voice in Gujarati needs context from 8+ sources (bureau, AA, cooperative, land, weather, device, satellite, PMFBY). Each has different freshness, reliability, and consent requirements. The agent receives *exactly* the context its policy scope permits -- no more (DPDP minimization), no less (decision quality).

**5-layer Context Graph:**

```
  LAKEHOUSE ──> DATA GRAPH ──> KNOWLEDGE ──> GOVERNANCE ──> ONTOLOGY
       │            │          GRAPH            │             │
  Versioned     Column-level   28 terms +    7 policies    Entity
  metadata +    lineage over   7-partition   14 rules/reg  resolution
  time travel   136 ULI srcs   RAG service   data flow     across 8
  per element                                governance    systems
```

1. **Context Lakehouse** -- versioned metadata with time travel; every element has source, timestamp, freshness, confidence, consent_ref
2. **Data Graph** -- column-level lineage across 136 ULI sources with full provenance
3. **Knowledge Graph** -- 28 domain terms + 7-partition RAG: product rules, regulatory, segment economics, verbatim compliance, operational, conversation patterns, decision history
4. **Governance Graph** -- 7 policies, 14 compliance rules/regulation. Enforces DPDP purpose limitation, data minimization, PSL constraints
5. **Active Ontology** -- entity resolution across 8 systems. Jaro-Winkler > 0.85 for names, exact match for Aadhaar/PAN

**Policy-scoped context compilation** -- context compiled per-request based on: segment (dairy→cooperative+weather; gig→platform earnings), channel (voice→summarized; dashboard→tabular), actor role (underwriter→full; voice agent→subset), regulatory scope (DPDP consent bounds), freshness requirements (per-source minimums).

**Services that OWN context:**

| Service | Context Responsibility |
|---------|----------------------|
| `context-graph` | 5-layer graph: Lakehouse + Data + Knowledge + Governance + Ontology |
| `knowledge-svc` | 7-partition RAG; verbatim compliance store; segment ontology |
| `data-quality-engine` | Entity resolution, cross-validation, confidence scoring, freshness |
| `uli-orchestrator` | Real-time context assembly from 136 ULI services per segment |

---

### 2.3 PRIMITIVE 3: POLICY

**Definition:** Rules, constraints, and boundaries governing what agents and services can and cannot do. Policy bridges business intent and system behavior.

**Why lending intensifies this:** In customer service, a policy violation means a bad response. In lending, it means regulatory non-compliance, capital loss, or borrower harm. Policy is the expression of RBI regulations, lender risk appetite, segment economics, and fairness requirements.

**3-layer enforcement:**

```
  STRUCTURAL (hard, never overridden)
  ──────────────────────────────────────────────────────────────
  KYC before data fetch │ Consent before ULI call │ KFS before
  offer │ Cooling-off before disburse │ PII encrypted │ RLS
  per lender │ Audit on every transition
  Implementation: code-level in Temporal workflows, not config.

  BEHAVIORAL (configurable per segment/lender)
  ──────────────────────────────────────────────────────────────
  Sentinel BRE: 14 operators, 3 templates (auto_approve,
  refer_to_human, auto_reject). Per-segment YAML: scoring
  weights, source requirements, pricing, product constraints,
  moratorium triggers, graduation criteria. Versioned,
  shadow-testable, champion/challenger. CRO approves promotion.

  SUPERVISOR (meta-policies on agent behavior)
  ──────────────────────────────────────────────────────────────
  Authority boundaries: Voice Agent presents offers, cannot
  modify terms. Decision Agent scores, cannot disburse.
  Monitoring Agent triggers moratorium, cannot write off.

  Progressive Trust Engine (L0-L4 with decay):
    L0: Fully supervised    L1: Human-in-loop
    L2: Supervised autonomy L3: Full autonomy
    L4: Self-improving (within bounds)
  Trust decays on accuracy drop -> CRO must re-approve.
```

**Services that OWN policy:**

| Service | Policy Responsibility |
|---------|---------------------|
| `sentinel-core` | BRE 14 operators; 3 templates; scoring; fraud thresholds |
| `compliance-reporter` | RBI rules, DPDP, PSL classification, fair lending, FREE-AI |
| `auth-svc` | RBAC per lender per role; API key scoping; borrower auth |
| `api-gateway` | Rate limiting, TLS enforcement, routing policies |

---

### 2.4 PRIMITIVE 4: FEEDBACK

**Definition:** The closed loop from production outcomes back to system improvement. Without feedback, every model drifts, every policy becomes stale, every context becomes outdated.

**Why lending intensifies this:** A lending decision takes 12-60 months to fully validate. The feedback signal is delayed, sparse, and noisy. A month-18 default must propagate back to scoring model, policy parameters, segment config, and monitoring triggers.

**SRAL loop + evaluation + canary + RL signals:**

```
  SENSE ──> REASON ──> ACT ──> LEARN
    │          │         │        │
  Channel    LLM via   MCP tool  Outcome -> RL signal ->
  input +    Constel-  execution model retrain -> policy
  ULI data + lation +  gated by  update -> shadow ->
  graph mem  context   policy    promote
```

- **Eval Suite:** tau-bench (task completion), pass^k (statistical confidence), 6 scorers (accuracy, latency, compliance, empathy, completion, satisfaction). Regression gating: new must beat incumbent on ALL scorers.
- **7-Stage Canary:** 1%→5%→10%→25%(manual gate)→50%→75%→100%. Auto-rollback on accuracy regression, latency spike, compliance violation, complaint rate increase.
- **RL Signal Collection:** Kafka `rl.signal` topic. Signals: EMI_COLLECTED, PRODUCTION_VERIFIED, WEATHER_MORATORIUM_TRIGGERED, MORATORIUM_CURE, DEFAULT, GRADUATION. Weekly batch: collect→join→compute reward (0.6×verifiable + 0.3×partial + 0.1×proxy)→retrain→evaluate→shadow→promote.
- **Explorer:** AI-powered failure clustering across agent interactions -- ASR errors by dialect, intent misclassification patterns, policy edge cases, segment-specific journey drop-offs. PSI-based drift detection.
- **Monitoring-Agent as Continuous Feedback:** 7 data stream watchers generate real-time signals. Every intervention outcome becomes training data. "Moratorium on heat wave: 80% cure rate" feeds back into policy calibration.

**Services that OWN feedback:**

| Service | Feedback Responsibility |
|---------|------------------------|
| `ml-platform` | MLflow registry, Feast feature store, training pipelines, A/B eval |
| `observability` | Prometheus+Grafana+Loki+Jaeger; PSI drift; Explorer clustering |
| `monitoring-agent` | Continuous outcome signals from 7 data stream watchers |
| `agent-runtime` | SRAL loop execution; conversation-level feedback collection |

---

## 3. LENDING-SPECIFIC EXTENSIONS

The base 4 primitives are necessary but insufficient. Five extensions address the structural differences between lending and customer service AI.

### 3.1 EXTENSION 1: DPI/ULI DATA ORCHESTRATION

**Why base primitives don't cover this:** Pandhi's Context assumes data is *available*. In Indian lending, the challenge is *fetching* from 136 ULI services with wildly different latencies, availability, and consent requirements. This is an orchestration problem, not a context problem.

```
  136 ULI SERVICES ──> PARALLEL FETCH ──> ENRICHED PROFILE
                            │
              ┌─────────────┼──────────────────┐
              │             │                  │
        CONSENT       CIRCUIT BREAKERS   SEGMENT-AWARE
        VALIDATION    per source (open   SOURCE MAPPING
        per source    after 5 failures,  DAIRY: bureau+AA+coop+land+sat
        before fetch  half-open @30s)    GIG: bureau+AA+platform+device
                                         VENDOR: bureau+AA+UPI+ONDC+device
```

Per-source: timeout (3s default, configurable per segment), retry (1× exponential backoff), HTTP/2 multiplexing.

**Graceful degradation protocol:**
- 1/5 sources fail → proceed with minor confidence penalty
- 2/5 sources fail → reduce max loan amount + increase interest rate + flag for human review
- 3/5 sources fail → route to REFER queue (human must review)
- Required sources fail (e.g., bureau for salaried) → cannot proceed, retry with borrower notification
- NEVER hard-reject on data unavailability alone

**Data quality pipeline (post-fetch):**
- Entity resolution: match borrower across sources (Jaro-Winkler > 0.85)
- Cross-source validation: income (bank) vs income (declared/cooperative), delta > 30% flagged
- Freshness scoring: FRESH (<30d)=1.0, RECENT (30-90d)=0.8, STALE (90-180d)=0.5, VERY_STALE (>180d)=0.2
- Confidence score: `0.3×(sources/required) + 0.3×avg_freshness + 0.2×cross_validation + 0.2×(1-anomaly_rate)`

**Owning service:** `uli-orchestrator` | **Supporting:** `identity-svc`, `bankconnect-svc`, `deviceconnect-svc`, `data-quality-engine`

---

### 3.2 EXTENSION 2: SEGMENT INTELLIGENCE

**Why base primitives don't cover this:** Pandhi's framework is segment-agnostic. Lending has 38 borrower segments with fundamentally different economics, data sources, risk profiles, products, and monitoring triggers. Segment intelligence is a *meta-primitive* that determines how all 4 base primitives behave.

```yaml
  # Each segment YAML defines:
  scoring:
    model: DairyScore_v1
    features: {cooperative_production: 0.25, bank_stability: 0.25,
               land_ownership: 0.15, bureau: 0.15, device: 0.10, coop_years: 0.10}
  data_sources: [bureau, AA, cooperative, land, satellite, PMFBY]
  product_type: SEASONAL_DYNAMIC
  flush_months: [Oct-Feb]  |  lean_months: [Apr-Jun]
  moratorium_triggers: [{WEATHER_HEAT: 38C, 1mo}, {PRODUCTION_DROP: 40%, 1mo}]
  graduation: {min_months: 12, on_time: 0.90, multiplier: 2.0x}
  insurance: PASHU_BIMA  |  languages: [gu, hi]  |  channels: [voice, whatsapp]
```

Segment determines: **State**→which trace fields captured, **Context**→which ULI sources fetched + knowledge partitions, **Policy**→which Sentinel rules + thresholds + products, **Feedback**→which watchers active + RL signals collected.

**Owning services:** `product-engine`, `sentinel-core` | **Supporting:** All 25 services read segment config.

---

### 3.3 EXTENSION 3: REGULATORY COMPLIANCE AS ARCHITECTURE

**Why base primitives don't cover this:** Pandhi's Policy handles behavioral guardrails. Lending compliance is *structural architecture*. RBI regulations dictate database schemas, API ordering, report formats, and audit requirements. Compliance is woven into every service, not layered on top.

| Regulation | Architecture Impact | Enforcing Service |
|------------|-------------------|-------------------|
| RBI Digital Lending (Sep 2022) | KFS generation, cooling-off enforcement, grievance redressal, audit trail | `product-engine`, `los-engine`, `audit-trail` |
| DPDP Act (2023) | Per-element consent ledger, purpose limitation, data minimization, right to erasure | `compliance-reporter`, `context-graph`, `uli-orchestrator` |
| PSL Classification | Auto-classify at origination, quarterly reporting, gap optimization in routing | `sentinel-core`, `compliance-reporter`, `prism-matcher` |
| RBI CLA (Jan 2026) | Co-lending templates, split disbursement/repayment, consolidated reporting | `prism-matcher`, `los-engine`, `compliance-reporter` |
| FREE-AI | No protected attributes in scoring, statistical parity, SHAP explainability, bias reports | `sentinel-core`, `ml-platform`, `compliance-reporter` |

**Owning service:** `compliance-reporter` | **Enforcing:** `los-engine`, `sentinel-core`, `context-graph`, `audit-trail`

---

### 3.4 EXTENSION 4: POST-DISBURSAL LIFECYCLE

**Why base primitives don't cover this:** Pandhi's framework ends at task completion. In lending, origination is the beginning. A 36-month loan requires continuous monitoring, proactive intervention, collections, and graduation. Post-disbursal is where most defaults are prevented or caused.

```
  7 WATCHERS ───> TRIGGER EVALUATOR ───> INTERVENTIONS ───> GRADUATION
       │                │                      │                 │
  Cooperative      Lookup policy          AUTO_MORATORIUM    Pre-approve
  Weather (IMD)    trigger rules          SCHEDULE_ADJUST    larger loan
  Bank (AA)        Evaluate threshold     EMERGENCY_LOAN     at better
  Platform (gig)   Determine action       INSURANCE_CLAIM    terms after
  EPFO/GST/EMI     Execute or route       GENTLE_REMINDER    min_months +
                   Log + emit rl.signal   HUMAN_ESCALATION   on_time rate
```

Every intervention outcome becomes a training signal. The monitoring-agent watches 7 streams per active loan:

| Watcher | Source | Frequency | Segment Applicability |
|---------|--------|-----------|----------------------|
| CooperativeWatcher | Dairy/agri coop API | Daily | Agriculture segments |
| WeatherWatcher | IMD API per pincode | Hourly | All agri + outdoor segments |
| BankAccountWatcher | Account Aggregator | Weekly | All segments |
| PlatformWatcher | Gig platform API | Daily | Gig workers |
| EPFOWatcher | EPFO employment | Quarterly | Salaried, gig |
| GSTWatcher | GSTN volume | Monthly | MSME, kirana, vendor |
| RepaymentWatcher | Internal EMI tracker | On due date | All segments |

Graduation is the ultimate feedback: a borrower who graduates validates the entire origination + monitoring chain. Graduation graph edge `(:Loan)-[:GRADUATED_TO]->(:Loan)` captures the full credit trajectory.

**Owning service:** `monitoring-agent` | **Supporting:** `notification-svc`, `product-engine`, `context-graph`, `los-engine`

---

### 3.5 EXTENSION 5: CONTEXT COMPOUNDING

**Why base primitives don't cover this:** Pandhi's Context is per-interaction. Lending requires *compounding*: 10,000 decision traces with features, policies, outcomes, interventions, and graduations form a knowledge substrate that improves every subsequent decision.

**Compounding mechanisms:**
1. **Model Improvement** -- RL signals from outcomes retrain segment models weekly. Gini improves monotonically with volume.
2. **Policy Calibration** -- threshold tuning from observed default/graduation rates. "Lower auto-approve from 650→620 for dairy; 500 traces show safe at 620."
3. **Segment Discovery** -- clustering traces reveals sub-segments. "Dairy + 2 animals + coop >5yr = distinct profile → create DAIRY_ESTABLISHED."
4. **Intervention Optimization** -- which interventions work for which triggers? "Moratorium cures 80% weather-DPD but only 40% income-drop-DPD."
5. **Fraud Evolution** -- graph analysis detects new patterns. "Shared reference + same device + sequential apps = coordinated ring."

Each trace is a Neo4j subgraph with full lineage:

```
  DECISION TRACE GRAPH STRUCTURE
  ═══════════════════════════════════════════════════════════════

  (:DecisionTrace {loan_id, timestamp, segment, lender})
    -[:USED_DATA]->(:DataSource {type, freshness, confidence})
    -[:APPLIED_POLICY]->(:Policy {version, segment, status})
    -[:PRODUCED_SCORE]->(:CreditScore {value, grade, SHAP[]})
    -[:HAD_OUTCOME]->(:Outcome {dpd, cured, graduated, wrote_off})
    -[:TRIGGERED_INTERVENTION]->(:Intervention {type, result})
    -[:SIMILAR_TO {cosine}]->(:DecisionTrace)

  Trace accumulation over time:
  ┌─────────┬──────────────────────────────────────────────────┐
  │ Volume  │ Emergent Capability                              │
  ├─────────┼──────────────────────────────────────────────────┤
  │ 100     │ Basic scoring model calibration                  │
  │ 1,000   │ Segment-level default rate estimation            │
  │ 5,000   │ Sub-segment discovery via clustering             │
  │ 10,000  │ Actuarial-grade risk tables (emergent)           │
  │ 50,000  │ Intervention effectiveness by trigger×segment    │
  │ 100,000 │ Cross-lender fraud ring detection at scale       │
  └─────────┴──────────────────────────────────────────────────┘
```

**Owning service:** `context-graph` | **Supporting:** `ml-platform`, `sentinel-core`, `fraud-engine`, `observability`

---

## 4. PRIMITIVE INTERACTION GRAPH

```
PRIMITIVE INTERACTION GRAPH
═══════════════════════════════════════════════════════════════════════

                     ┌───────────────────────┐
                     │  SEGMENT INTELLIGENCE  │
                     │  (Extension 2)         │
                     │  38 segments define    │
                     │  HOW all primitives    │
                     │  behave per borrower   │
                     └───────────┬───────────┘
                                 │ configures
              ┌──────────────────┼──────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
  ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐
  │    CONTEXT      │ │     POLICY      │ │   DPI/ULI DATA   │
  │  (Primitive 2)  │ │  (Primitive 3)  │ │  ORCHESTRATION   │
  │  5-layer graph  │◄│  3-layer        │ │  (Extension 1)   │
  │  assembles what │ │  enforcement    │ │  136 svc fetch   │
  │  agents know    │ │  constrains     │ │  circuit breakers│
  └────────┬────────┘ └────────┬────────┘ └────────┬─────────┘
           │ provides          │ gates              │ feeds
           ▼                   ▼                    ▼
  ┌────────────────────────────────────────────────────────────┐
  │                    STATE (Primitive 1)                      │
  │  Audit trail + Decision Traces + Temporal + Consent        │
  │  Every action from Context, Policy, ULI recorded here      │
  └───────────────────────────┬────────────────────────────────┘
                              │ outcomes feed
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  FEEDBACK (Primitive 4)                   │
  │  SRAL loop + eval suite + canary pipeline + RL signals   │
  └────────┬───────────────┬─────────────────┬───────────────┘
           ▼               ▼                 ▼
  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐
  │   CONTEXT    │ │  REGULATORY  │ │  POST-DISBURSAL    │
  │  COMPOUNDING │ │  COMPLIANCE  │ │  LIFECYCLE         │
  │ (Extension 5)│ │ (Extension 3)│ │ (Extension 4)      │
  │ 10K traces → │ │ RBI/DPDP/PSL│ │ 7 watchers +       │
  │ institutional│ │ woven into   │ │ interventions +    │
  │ knowledge    │ │ every service│ │ graduation         │
  └──────────────┘ └──────────────┘ └────────────────────┘
```

---

## 5. FULL SERVICE-TO-PRIMITIVE MAPPING TABLE

**P** = Primary owner | **C** = Contributor | **-** = Not directly involved

### 5.1 Core Services (18)

```
                        BASE PRIMITIVES          LENDING EXTENSIONS
                    ┌────┬────┬────┬────┐  ┌────┬────┬────┬────┬────┐
Service             │Stat│Ctxt│Plcy│Fdbk│  │DPI │Segm│Regl│Post│Cmpd│
────────────────────┼────┼────┼────┼────┤  ├────┼────┼────┼────┼────┤
uli-orchestrator    │ C  │ C  │ C  │ C  │  │ P  │ C  │ C  │  - │  - │
data-quality-engine │ C  │ P  │  - │ C  │  │ C  │ C  │  - │  - │ C  │
bankconnect-svc     │ C  │ C  │  - │ C  │  │ C  │ C  │  - │ C  │ C  │
deviceconnect-svc   │ C  │ C  │  - │ C  │  │ C  │ C  │  - │  - │ C  │
identity-svc        │ C  │ C  │ C  │  - │  │ C  │  - │ C  │  - │  - │
────────────────────┼────┼────┼────┼────┤  ├────┼────┼────┼────┼────┤
sentinel-core       │ C  │ C  │ P  │ C  │  │  - │ P  │ C  │  - │ C  │
product-engine      │ C  │ C  │ C  │  - │  │  - │ P  │ C  │ C  │  - │
fraud-engine        │ C  │ C  │ C  │ C  │  │  - │ C  │  - │  - │ C  │
────────────────────┼────┼────┼────┼────┤  ├────┼────┼────┼────┼────┤
prism-matcher       │ C  │ C  │ C  │  - │  │  - │ C  │ C  │  - │  - │
los-engine          │ P  │ C  │ C  │  - │  │  - │ C  │ C  │ C  │  - │
journey-studio      │ C  │ C  │  - │ C  │  │  - │ C  │  - │  - │  - │
notification-svc    │ C  │  - │  - │  - │  │  - │ C  │  - │ C  │  - │
────────────────────┼────┼────┼────┼────┤  ├────┼────┼────┼────┼────┤
agent-runtime       │ C  │ C  │ C  │ P  │  │  - │ C  │  - │  - │  - │
voice-pipeline      │ C  │  - │  - │ C  │  │  - │ C  │  - │  - │  - │
constellation       │  - │  - │  - │ C  │  │  - │  - │  - │  - │  - │
knowledge-svc       │  - │ P  │  - │  - │  │  - │ C  │ C  │  - │ C  │
context-graph       │ P  │ P  │ C  │ C  │  │  - │ C  │ C  │ C  │ P  │
monitoring-agent    │ C  │ C  │ C  │ P  │  │  - │ C  │  - │ P  │ C  │
────────────────────┴────┴────┴────┴────┘  └────┴────┴────┴────┴────┘
```

### 5.2 Cross-Cutting Services (7)

```
                        BASE PRIMITIVES          LENDING EXTENSIONS
                    ┌────┬────┬────┬────┐  ┌────┬────┬────┬────┬────┐
Service             │Stat│Ctxt│Plcy│Fdbk│  │DPI │Segm│Regl│Post│Cmpd│
────────────────────┼────┼────┼────┼────┤  ├────┼────┼────┼────┼────┤
api-gateway         │ C  │  - │ P  │  - │  │  - │  - │  - │  - │  - │
auth-svc            │ C  │  - │ P  │  - │  │  - │  - │ C  │  - │  - │
audit-trail         │ P  │  - │  - │ C  │  │  - │  - │ P  │  - │ C  │
compliance-reporter │ C  │ C  │ C  │  - │  │  - │  - │ P  │  - │  - │
ml-platform         │  - │  - │  - │ P  │  │  - │ C  │ C  │  - │ C  │
observability       │ C  │  - │  - │ P  │  │  - │  - │  - │ C  │ C  │
mcp-server          │ C  │ C  │ C  │  - │  │ C  │  - │  - │  - │  - │
────────────────────┴────┴────┴────┴────┘  └────┴────┴────┴────┴────┘
```

**Legend:** Stat=State | Ctxt=Context | Plcy=Policy | Fdbk=Feedback | DPI=DPI/ULI Orchestration | Segm=Segment Intelligence | Regl=Regulatory Compliance | Post=Post-Disbursal Lifecycle | Cmpd=Context Compounding

---

## 6. WHY LENDING REQUIRES EXTENSIONS BEYOND THE BASE 4

| Dimension | Customer Service AI | Lending AI | Extension Required |
|-----------|--------------------|-----------|--------------------|
| **Time horizon** | Minutes (one conversation) | 12-60 months (loan lifecycle) | Post-Disbursal Lifecycle |
| **Parties** | 2 (customer + agent) | 4+ (borrower, lender, regulator, insurer, co-lender) | Regulatory Compliance as Architecture |
| **Data sources** | 1-3 (CRM, KB, orders) | 136 (ULI, each with own API/consent/latency) | DPI/ULI Data Orchestration |
| **Decision cost** | Low (bad answer = retry) | High (capital loss, regulatory violation) | Context Compounding |
| **Heterogeneity** | Low (customers ~similar) | High (38 segments, different economics) | Segment Intelligence |
| **Feedback latency** | Immediate (CSAT) | 12-60 months (loan outcome) | Enhanced Feedback + RL infrastructure |
| **Compliance** | Guidelines (brand voice) | Laws (RBI Act, DPDP, PSL norms) | Compliance as architecture, not policy |
| **Lifecycle** | Ends at task completion | Begins at disbursement | Monitoring as first-class concern |

**The synthesis:** Pandhi's 4 primitives are the *foundation*. The 5 lending extensions are the *superstructure*. Without the foundation, extensions have nothing to build on. Without extensions, the foundation cannot support lending.

```
          ┌──────────────────────────────────────────────┐
          │        LENDING SUPERSTRUCTURE                 │
          │  ┌─────┐ ┌──────┐ ┌──────┐ ┌────┐ ┌──────┐  │
          │  │ DPI │ │Segmnt│ │ Regl │ │Post│ │Compnd│  │
          │  │ Orch│ │ Intl │ │ Comp │ │Disb│ │ Ctxt │  │
          │  └──┬──┘ └──┬───┘ └──┬───┘ └─┬──┘ └──┬───┘  │
          └─────┼───────┼───────┼───────┼───────┼────────┘
          ══════╪═══════╪═══════╪═══════╪═══════╪════════
          ┌─────┼───────┼───────┼───────┼───────┼────────┐
          │     ▼       ▼       ▼       ▼       ▼        │
          │   PANDHI'S 4 STRUCTURAL PRIMITIVES           │
          │  ┌───────┐ ┌───────┐ ┌──────┐ ┌────────┐    │
          │  │ STATE │ │CONTEXT│ │POLICY│ │FEEDBACK│    │
          │  └───────┘ └───────┘ └──────┘ └────────┘    │
          │        AI ENGINEERING FOUNDATION              │
          └──────────────────────────────────────────────┘
```

---

## 7. IMPLEMENTATION PRIORITIES

Primitives should be implemented in this order:

1. **State** (Build 0) -- without immutable audit and workflow durability, nothing is trustworthy
2. **Policy** (Build 0-1) -- structural policies must exist before any agent acts
3. **DPI/ULI Orchestration** (Build 1) -- data availability unlocks everything downstream
4. **Context** (Build 1-2) -- context graph + knowledge service + entity resolution
5. **Segment Intelligence** (Build 2) -- segment configs drive all behavioral variation
6. **Regulatory Compliance** (Build 2-3) -- woven in, not bolted on
7. **Feedback** (Build 4-5) -- SRAL loop, eval suite, canary pipeline
8. **Post-Disbursal Lifecycle** (Build 5) -- watchers, interventions, graduation
9. **Context Compounding** (Build 5-6) -- emerges from trace accumulation; explicit optimization later

State and Policy are foundational -- they gate everything. Context Compounding is emergent -- it improves with volume. The middle primitives (3-8) are the engineering substance of the platform.

**Critical dependencies:**
- DPI/ULI Orchestration requires State (consent ledger) and Policy (consent validation) to exist first
- Segment Intelligence requires Context (knowledge graph loaded with segment ontology) and Policy (Sentinel templates)
- Post-Disbursal Lifecycle requires Feedback (RL signal infrastructure) and Context (graph memory for interventions)
- Context Compounding requires all other primitives -- it is the emergent property of the complete system

---

## 8. MEASURING PRIMITIVE HEALTH

| Primitive | Health Metric | Target | Alert |
|-----------|--------------|--------|-------|
| State | Audit log completeness | 100% | <99.9% |
| State | Hash chain integrity | 100% | <100% (critical) |
| Context | Assembly latency p95 | <500ms | >1s |
| Context | Entity resolution accuracy | >95% | <90% |
| Policy | Evaluation latency p95 | <200ms | >500ms |
| Policy | Shadow-production alignment | >95% | <90% |
| Feedback | Model Gini (per segment) | >0.45 | <0.35 |
| Feedback | Canary rollback rate | <10% | >25% |
| DPI/ULI | Fetch success rate (per source) | >95% | <85% |
| DPI/ULI | Overall fetch latency p95 | <5s | >8s |
| Segment | Config coverage (active segments) | 38/38 | <35/38 |
| Regulatory | PSL classification accuracy | 100% | <99% (critical) |
| Regulatory | Consent ledger completeness | 100% | <100% (critical) |
| Post-Disbursal | Intervention success (DPD prevented) | >70% | <50% |
| Post-Disbursal | Graduation rate (eligible borrowers) | >30% | <15% |
| Compounding | Gini improvement per 1000 traces | >0.01 | <0.005 |

---

## 9. CONCLUSION

Pandhi's 4 primitives -- State, Context, Policy, Feedback -- are the correct abstractions for AI agent systems. They are necessary for lending. But lending's regulated, multi-party, multi-year, high-stakes, heterogeneous nature demands 5 extensions that transform these primitives from a voice AI framework into a lending operating system.

The 25-service platform maps cleanly to this 4+5 framework. Every service has clear primitive affiliation. Every primitive has clear owning services. The extensions are not ad hoc -- they are structural consequences of lending's inherent complexity.

Build the foundation (State + Policy). Wire the data (DPI/ULI + Context). Configure the intelligence (Segment + Compliance). Close the loop (Feedback + Lifecycle + Compounding). The platform emerges from the primitives, not the other way around.

---

*Document version: 1.0 | Platform: AI-First Lending OS | Services: 18 core + 7 cross-cutting = 25*
*Primitives: 4 base (Pandhi) + 5 lending extensions = 9 total*
*Segments: 38 | ULI services: 136 | MCP tools: 49+ | Languages: 12*
