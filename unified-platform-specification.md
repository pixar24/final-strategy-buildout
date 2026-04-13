# Unified AI-First Lending Platform: Production Specification
## Multi-Agent Orchestration for India's 300M Underserved Borrowers

---

# EXECUTIVE SUMMARY

This document synthesizes 19 repositories, 75,000+ lines of working code, 1,100+ tests, and strategic research into a single production specification for an AI-first lending platform. The platform applies **Nurix.ai-style multi-agent orchestration** -- central orchestrator, specialized domain agents, flexible routing, end-to-end visibility -- to the lending domain, targeting India's 300M underserved borrowers across 38 segments.

**Platform Identity:** Not a lending product. A **lending operating system** that turns multiple AI agents into one high-performing lending system aligned to lender workflows and optimized for borrower outcomes.

**Core Thesis:** Layer 2 (intelligence) recommends, Layer 1 (governed execution) decides. Context compounds -- the 10,000th loan benefits from 9,999 prior decision traces. Monitoring prevents defaults. Agents are the interface, not the intelligence.

**What Exists Today:**
- 6 production-grade codebases covering every platform layer
- 14-microservice voice AI platform (voiceai-agent-services-platform)
- 60 lending tools across 38 borrower segments (embeddedcredit-as-a-service-v2)
- Credit infrastructure decomposition: Sentinel BRE, LOS, Prism routing, BankConnect, RiskEngine (fb-product-l2)
- Context graph architecture: Lakehouse, Governance Graph, Decision Traces (contextgraphs-platform-fintech)
- Agent lifecycle: SRAL loop, eval/shadow/canary pipeline, autonomy levels (caas-casestudy)
- MCP integration layer: 80+ tools across 4 servers (fb-specifics, fb-product-l2, contextgraphs)

**What This Specification Delivers:**
- Unified service registry consolidating all repos into 18 core services + 7 cross-cutting services
- Multi-agent orchestration architecture with central orchestrator, 6 agent families, flexible routing
- 7-stage progressive build roadmap with verification gates and lender workflows enabled per stage
- Complete DPI/ULI integration design for 136 services
- First lender deployment playbook with pre-launch checklist
- Go-to-market phasing: autopilot (alongside) -> embedded (inside) -> platform (owns)

---

# PART I: PLATFORM POSITIONING

## 1.1 Market Reality

| Dimension | Data Point |
|-----------|------------|
| Addressable borrowers | 300M underserved (thin-file, informal income, vernacular-first) |
| Segments | 38 defined (agriculture: dairy, crop, FPO; micro-enterprise: kirana, vendor, SHG; gig: delivery, ride-hail; salaried NTC) |
| Lender pain | Cost-per-loan for <INR 2L ticket size makes manual origination unviable |
| Current infrastructure | FinBox (data layer only), traditional LOS (no AI), bureau-dependent scoring (excludes thin-file) |
| DPI opportunity | ULI (136 services), Account Aggregator (live), ONDC (growing), Aadhaar eKYC (universal) |
| Regulatory context | RBI Digital Lending (Sep 2022), RBI CLA (Jan 2026), DPDP Act (2023), FREE-AI framework, PSL classification |

## 1.2 Platform vs. Competitors

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         LENDING STACK POSITIONING                                │
│                                                                                  │
│  TRADITIONAL LOS          FINBOX STACK           THIS PLATFORM                   │
│  (Nucleus, Newgen)        (Data + Risk)          (AI-First OS)                   │
│                                                                                  │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────────────────────┐   │
│  │ Manual UW    │    │ BankConnect      │    │ MULTI-AGENT ORCHESTRATION     │   │
│  │ Form-based   │    │ DeviceConnect    │    │                               │   │
│  │ Branch ops   │    │ FinBox Score     │    │ ┌─ Voice Agent (borrower)     │   │
│  │ Paper trail  │    │ API-only         │    │ ├─ Decisioning Agent (risk)   │   │
│  │              │    │                  │    │ ├─ Monitoring Agent (post-disb)│   │
│  │ NO AI       │    │ DATA LAYER ONLY  │    │ ├─ Copilot Agent (lender ops) │   │
│  │ NO voice    │    │ No agent runtime │    │ ├─ Compliance Agent (audit)   │   │
│  │ NO segments │    │ No voice/NLU     │    │ └─ Context Agent (graph)      │   │
│  │ NO DPI      │    │ No orchestration │    │                               │   │
│  │              │    │ No lifecycle mgmt│    │ + 38 segment configs          │   │
│  │              │    │                  │    │ + Voice in 12 languages       │   │
│  │              │    │                  │    │ + Context graphs (compound)   │   │
│  │              │    │                  │    │ + Full agent lifecycle         │   │
│  │              │    │                  │    │ + DPI/ULI native              │   │
│  └──────────────┘    └──────────────────┘    └───────────────────────────────┘   │
│                                                                                  │
│  Sells to: large banks  Sells to: NBFCs+fintechs  Sells to: MFIs, SFBs, NBFCs, │
│  INR 5Cr+ deals         API integration            banks, LSPs, ONDC platforms   │
│  12-month deploy        2-week integration          2-week pilot to live          │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## 1.3 Nurix-Style Multi-Agent Orchestration Applied to Lending

Nurix.ai's core architecture -- central orchestrator, specialized agents, flexible routing, end-to-end visibility -- maps directly to lending:

| Nurix Concept | Lending Implementation | Source Repo |
|---------------|----------------------|-------------|
| **Central Orchestrator** | Loan lifecycle workflow engine (Temporal-based). Routes each loan through: data fetch -> decisioning -> product design -> lender matching -> disbursement -> monitoring. Dynamic task assignment based on segment, channel, and lender config. | voiceai-platform (agent runtime) + fb-product-l2 (LOS state machine) |
| **Specialized Agents** | Each agent owns a bounded domain: Voice Agent (borrower conversation), Decisioning Agent (Sentinel scoring), Data Agent (ULI fetch), Monitoring Agent (post-disbursal watch), Copilot Agent (lender operator assist), Compliance Agent (audit + reporting), Context Agent (graph queries). Narrow scope = higher accuracy. | ecaas-v2 (60 tools), caas-casestudy (18 agents), contextgraphs (15 agents) |
| **Flexible Routing** | Rules-based for compliance-gated steps (KYC must precede data fetch). LLM-based for borrower intent classification (voice/WhatsApp). Hybrid for decisioning (Sentinel policy = rules, risk assessment = ML). Sequential for loan origination, parallel for data fetch (ULI sources fetched simultaneously). | fb-product-l2 (Prism routing), voiceai-platform (constellation router) |
| **End-to-End Visibility** | Decision Traces as first-class objects in context graph. Every agent action logged to immutable audit trail. Orchestration-level analytics: which agent handled what, latency per step, error rates, borrower satisfaction per agent. SRAL feedback loop: every outcome trains next decision. | contextgraphs (Decision Traces), caas-casestudy (SRAL loop), voiceai-platform (trace service + explorer) |

**Nurix value props applied to lending:**

| Metric | Nurix Benchmark | Lending Target | How |
|--------|----------------|----------------|-----|
| Task accuracy | 90-99% | >95% credit decision accuracy (Gini > 0.45) | Specialized Sentinel agent with domain-specific scoring per segment |
| Cost to serve | -15% | -60% cost-per-loan vs. manual branch origination | Voice agent replaces field visits; auto-decisioning replaces manual UW |
| Containment | +20% | 85% borrower journeys completed without human handoff | Journey Studio with segment-optimized flows, empathy detection for escalation |
| Latency | -25% | <500ms decision, <1.5s voice response | Parallel ULI data fetch, cached policies, fast brain/slow brain execution |

## 1.4 ICP and Go-to-Market Phasing

**Phase 1 (Months 0-6): Autopilot Mode**
- ICP: MFIs and SFBs with existing LOS
- Value prop: AI copilot that sits alongside their existing system, augments underwriters
- Delivery: autopilot-in-enterprise pattern
- Revenue: SaaS per-seat for lender dashboard access

**Phase 2 (Months 6-12): Embedded Mode**
- ICP: NBFCs launching new segment lending (dairy, gig, ONDC)
- Value prop: Embedded lending stack via API/MCP + voice agents + Prism routing
- Delivery: Full platform deployment with lender-specific config
- Revenue: Per-loan origination fee + platform SaaS

**Phase 3 (Months 12-24): Platform Mode**
- ICP: Banks, large NBFCs, ONDC network participants
- Value prop: Multi-lender marketplace with context graphs, multi-agent orchestration, full lifecycle
- Delivery: Self-service lender onboarding (<2 weeks)
- Revenue: Per-loan + AUM-based + data services

---

# PART II: SYSTEM ARCHITECTURE -- UNIFIED PRODUCT MAP

## 2.1 Platform Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              UNIFIED PLATFORM ARCHITECTURE                                   │
│                                                                                              │
│  BORROWER CHANNELS          MULTI-AGENT ORCHESTRATOR           EXTERNAL SYSTEMS              │
│  ┌───────────────┐    ┌────────────────────────────────────┐   ┌─────────────────────────┐  │
│  │ Voice (PSTN/  │    │                                    │   │ DPI / ULI (136 svc)     │  │
│  │ VoIP/Exotel)  │───>│   CENTRAL ORCHESTRATOR (Temporal)  │<─>│ Account Aggregator      │  │
│  ├───────────────┤    │                                    │   │ Credit Bureaus           │  │
│  │ WhatsApp      │───>│   Routes tasks to specialized      │<─>│ UIDAI / eKYC            │  │
│  │ (BSP)         │    │   agents based on:                 │   │ DigiLocker              │  │
│  ├───────────────┤    │   - Loan lifecycle stage            │   │ eSign / eNACH           │  │
│  │ ONDC Partner  │───>│   - Borrower segment               │<─>│ NPCI (UPI/IMPS/NEFT)    │  │
│  │ SDK           │    │   - Channel type                   │   │ GSTN / EPFO             │  │
│  ├───────────────┤    │   - Lender policy                  │   ├─────────────────────────┤  │
│  │ Web/App       │───>│   - Compliance requirements         │   │ SEGMENT-SPECIFIC        │  │
│  ├───────────────┤    │                                    │   │ Dairy cooperatives       │  │
│  │ Lender        │    │  ┌──────────────────────────────┐  │   │ Mandi (eNAM)             │  │
│  │ Dashboard     │───>│  │ SPECIALIZED AGENT FAMILIES   │  │   │ ONDC marketplace         │  │
│  └───────────────┘    │  │                              │  │   │ Gig platforms (Swiggy/   │  │
│                       │  │ Voice Agent ──── Borrower    │  │   │   Zomato/Uber/Ola)       │  │
│                       │  │ Data Agent ───── ULI/DPI     │  │<─>│ Warehouse receipts       │  │
│                       │  │ Decision Agent ─ Sentinel    │  │   │ IMD Weather              │  │
│                       │  │ Monitor Agent ── Portfolio   │  │   │ ISRO Satellite (NDVI)    │  │
│                       │  │ Copilot Agent ── Lender Ops  │  │   ├─────────────────────────┤  │
│                       │  │ Compliance Agent ── Audit    │  │   │ LENDER SYSTEMS           │  │
│                       │  │ Context Agent ─── Graph      │  │<─>│ Core Banking (disburse)  │  │
│                       │  └──────────────────────────────┘  │   │ Insurance (Pashu Bima)   │  │
│                       └────────────────────────────────────┘   │ Co-lending partners      │  │
│                                                                └─────────────────────────┘  │
│                                                                                              │
│  PLATFORM CORE SERVICES                                                                      │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐        │
│  │ DATA LAYER       │ │ DECISION LAYER   │ │ DISTRIBUTION     │ │ INTELLIGENCE     │        │
│  │ (5 services)     │ │ (3 services)     │ │ (4 services)     │ │ (6 services)     │        │
│  │                  │ │                  │ │                  │ │                  │        │
│  │ uli-orchestrator │ │ sentinel-core    │ │ prism-matcher    │ │ agent-runtime    │        │
│  │ data-quality     │ │ product-engine   │ │ los-engine       │ │ voice-pipeline   │        │
│  │ bankconnect-svc  │ │ fraud-engine     │ │ journey-studio   │ │ constellation    │        │
│  │ deviceconnect    │ │                  │ │ notification-svc │ │ knowledge-svc    │        │
│  │ identity-svc     │ │                  │ │                  │ │ context-graph    │        │
│  │                  │ │                  │ │                  │ │ monitoring-agent │        │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘        │
│                                                                                              │
│  PLATFORM INFRASTRUCTURE                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐    │
│  │ K8s HA │ PostgreSQL (RLS) │ Redis Cluster │ Kafka │ Neo4j │ S3/MinIO │ Temporal     │    │
│  │ Vault  │ MLflow           │ Feast (features) │ Prometheus+Grafana+Loki+Jaeger       │    │
│  └──────────────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Service Registry -- 18 Core Services + 7 Cross-Cutting

Each service maps to one or more repo implementations. Consolidation column shows the merge path.

### Core Services

| # | Service | Domain | Consolidates From | DB | Queue |
|---|---------|--------|-------------------|-----|-------|
| S1 | `uli-orchestrator` | Parallel ULI fetch, consent mgmt, timeout handling | ecaas-v2/tools/identity + ecaas-v2/tools/data + Knowledge-Repo/ULI-spec | PostgreSQL | Kafka `uli.fetch.*` |
| S2 | `data-quality-engine` | Entity resolution, cross-source validation, confidence scoring | contextgraphs/Active-Ontology + fb-product-l2/shared/validation | PostgreSQL + Redis | Kafka `dq.enriched` |
| S3 | `bankconnect-svc` | Statement parsing, cluster intelligence, FBCS scoring, fraud flags | fb-product-l2/bankconnect (FBCS 0-900, Cluster TX, 30+ lender patterns) | PostgreSQL | Kafka `bc.analyzed` |
| S4 | `deviceconnect-svc` | SDK telemetry, behavioral features, FIS inclusion score | fb-product-l2/deviceconnect (FIS 0-900, 20+ features) | TimescaleDB | Kafka `dc.scored` |
| S5 | `identity-svc` | Aadhaar OTP, PAN, eKYC, AA consent, verification pipelines | ecaas-v2/tools/identity + fb-product-l2/shared/kyc | PostgreSQL | Kafka `id.verified` |
| S6 | `sentinel-core` | Policy engine, scoring, fraud detection, shadow/challenger, MCP | fb-product-l2/sentinel (BRE, 14 operators) + voiceai-platform/policy-engine (3-layer) + ecaas-v2/policy-engine (7+10 rules) | PostgreSQL + Redis | Kafka `sentinel.decision` |
| S7 | `product-engine` | 8 product types, dynamic schedule generation, moratorium rules | ecaas-v2/tools/product + fb-product-l2/journey_studio/product | PostgreSQL | Kafka `product.designed` |
| S8 | `fraud-engine` | Velocity checks, BankConnect cluster analysis, graph-based ring detection | fb-product-l2/sentinel/fraud + bankconnect/cluster-intelligence | PostgreSQL + Neo4j | Kafka `fraud.alert` |
| S9 | `prism-matcher` | Multi-lender matching, co-lending structures, PSL gap optimization, FLDG | fb-product-l2/prism (waterfall routing, multi-lender) | PostgreSQL | Kafka `prism.matched` |
| S10 | `los-engine` | Loan origination workflow, approvals, e-sign, disbursement, insurance | fb-product-l2/los (14-state machine) + contextgraphs (24-state machine) | PostgreSQL + Temporal | Kafka `los.event.*` |
| S11 | `journey-studio` | No-code journey builder, GenAI flow generation, A/B testing | fb-product-l2/journey_studio + voiceai-platform/workspace-service | PostgreSQL + Redis | Kafka `journey.analytics` |
| S12 | `notification-svc` | SMS, WhatsApp template, voice callback, email dispatch | ecaas-v2/gateway/whatsapp + voiceai-platform/voice-pipeline/telephony | Redis | Kafka `notify.*` |
| S13 | `agent-runtime` | Multi-agent orchestrator, conversation state, MCP client, tool execution | voiceai-platform/agent-runtime (16-step turn) + ecaas-v2/agent (60 tools) + caas-casestudy/SRAL | PostgreSQL + Redis | Kafka `agent.interaction` |
| S14 | `voice-pipeline` | STT (Whisper/Deepgram), TTS (ElevenLabs/Bhashini), VAD, telephony | voiceai-platform/voice-pipeline (Deepgram+ElevenLabs+Twilio) + ecaas-v2 (Cartesia+Deepgram) | Redis | Kafka `voice.stream` |
| S15 | `constellation` | Multi-model routing, failover, fast brain/slow brain parallel exec | voiceai-platform/constellation (multi-model router) | Redis | -- |
| S16 | `knowledge-svc` | 7-partition RAG, verbatim compliance store, lending domain knowledge | voiceai-platform/knowledge-service + contextgraphs/Knowledge-Graph (28 terms) | PostgreSQL + Vector DB | Kafka `knowledge.query` |
| S17 | `context-graph` | Context Lakehouse, Data Graph, Governance Graph, Decision Traces | contextgraphs-platform (full implementation) + voiceai-platform/agent-data-platform | Neo4j + PostgreSQL | Kafka `graph.ingest` |
| S18 | `monitoring-agent` | Continuous ULI data watching, trigger evaluation, intervention execution | ecaas-v2/tools/monitoring + caas-casestudy/monitoring-patterns | PostgreSQL + Redis | Kafka `monitor.trigger.*` |

### Cross-Cutting Services

| Service | Purpose | Consolidates From |
|---------|---------|-------------------|
| `api-gateway` | Kong/custom: auth, rate-limit, routing, TLS | voiceai-platform/gateway + fb-product-l2/gateway |
| `auth-svc` | JWT/OAuth2 lender, API keys partner, Aadhaar OTP borrower | fb-product-l2/shared/auth + lia-openclaw/security |
| `audit-trail` | Immutable append-only log of every decision, MCP call, intervention | contextgraphs/Decision-Traces + caas-casestudy/audit + fb-specifics/compliance |
| `compliance-reporter` | PSL auto-classification, RBI reports, bias detection, DPDP consent ledger | fb-product-l2/shared/compliance + caas-casestudy/FREE-AI |
| `ml-platform` | MLflow registry, feature store (Feast), training pipelines, A/B evaluation | fb-product-l2/riskengine/feedback + voiceai-platform/evaluation |
| `observability` | Prometheus + Grafana + Loki + Jaeger, drift detection (PSI) | voiceai-platform/trace-service + voiceai-platform/explorer |
| `mcp-server` | Unified MCP endpoint: all platform tools via JSON-RPC 2.0 | fb-product-l2/mcp-server (27 tools) + contextgraphs/mcp (5 tools) + fb-specifics/finbox-mcp (17 tools) = 49+ unified tools |

## 2.3 Multi-Agent Orchestration Design

### Agent Family Taxonomy

The platform consolidates 100+ agents across repos into 6 families, each with a central orchestrator:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    MULTI-AGENT ORCHESTRATION MAP                          │
│                                                                           │
│  CENTRAL ORCHESTRATOR (Temporal workflow engine)                           │
│  ├── Determines lifecycle stage                                           │
│  ├── Routes to appropriate agent family                                   │
│  ├── Manages handoffs between families                                    │
│  ├── Enforces compliance gates (KYC before data, consent before fetch)    │
│  └── Emits orchestration-level analytics                                  │
│                                                                           │
│  AGENT FAMILY 1: BORROWER INTERFACE                                       │
│  Source: voiceai-platform + ecaas-v2 + caas-casestudy                     │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Voice Agent ─── Gujarati/Hindi/Tamil/+ 9 languages                │   │
│  │   ├── NLU: intent classification (8 intents)                      │   │
│  │   ├── ASR: Whisper fine-tuned on lending vocab                    │   │
│  │   ├── TTS: Bhashini/ElevenLabs, segment-appropriate voice         │   │
│  │   └── Empathy: distress detection -> human escalation             │   │
│  │                                                                    │   │
│  │ WhatsApp Agent ── text + buttons + document + photo OCR           │   │
│  │ Web Agent ────── React form flows                                  │   │
│  │ Partner Agent ── ONDC/gig platform SDK embedded                   │   │
│  │ Returning Agent ── graph memory context for repeat borrowers      │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  AGENT FAMILY 2: DATA INTELLIGENCE                                        │
│  Source: fb-product-l2 + ecaas-v2 + contextgraphs                         │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ ULI Fetch Agent ─── parallel calls to 136 DPI services            │   │
│  │ BankConnect Agent ── statement analysis, FBCS 0-900               │   │
│  │ DeviceConnect Agent ── behavioral scoring, FIS 0-900              │   │
│  │ Identity Agent ───── Aadhaar eKYC, PAN, DigiLocker, AA consent   │   │
│  │ Data Quality Agent ── entity resolution, cross-validation         │   │
│  │ Enrichment Agent ─── segment-specific: coop data, mandi, weather │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  AGENT FAMILY 3: DECISIONING                                              │
│  Source: fb-product-l2/sentinel + voiceai-platform/policy + ecaas-v2      │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Scoring Agent ────── segment-specific ML models (Dairy, Vendor,   │   │
│  │                      Gig, SHG, Kirana, etc.)                      │   │
│  │ Policy Agent ─────── BRE evaluation, 14 operators, 3 templates   │   │
│  │ Fraud Agent ──────── velocity, cluster analysis, graph ring detect│   │
│  │ Product Design Agent ── 8 schedule generators, dynamic pricing    │   │
│  │ Explainability Agent ── SHAP values, human-readable reason codes  │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  AGENT FAMILY 4: DISTRIBUTION                                             │
│  Source: fb-product-l2/prism + fb-product-l2/los + fb-product-l2/journey  │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Lender Matching Agent ── PSL gap optimization, co-lending struct  │   │
│  │ LOS Agent ────────────── 14-state origination workflow            │   │
│  │ Disbursement Agent ───── UPI/NEFT/IMPS via lender bank API        │   │
│  │ Insurance Agent ──────── Pashu Bima, crop, health enrollment      │   │
│  │ Journey Agent ────────── A/B tested flows per segment/channel     │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  AGENT FAMILY 5: POST-DISBURSAL LIFECYCLE                                 │
│  Source: ecaas-v2/monitoring + caas-casestudy/SRAL + contextgraphs        │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Monitoring Agent ───── 7 data stream watchers (coop, weather,     │   │
│  │                        bank, platform, EPFO, GST, repayment)      │   │
│  │ Intervention Agent ─── auto-moratorium, schedule adjust,          │   │
│  │                        emergency microloan, insurance claim        │   │
│  │ Collections Agent ──── context-aware, empathetic, graduated       │   │
│  │ Graduation Agent ───── pre-approve better terms for good history  │   │
│  │ Graph Agent ──────────  Neo4j multi-hop queries, fraud ring,      │   │
│  │                        similar-borrower-default detection          │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  AGENT FAMILY 6: LENDER OPERATIONS                                        │
│  Source: autopilot-in-enterprise + ecaas-v2/dashboard + voiceai/studio    │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Copilot Agent ──────── NL queries via MCP ("What is dairy NPA?")  │   │
│  │ Policy Builder Agent ── no-code policy creation + shadow testing  │   │
│  │ Compliance Agent ───── PSL report, audit trail, bias detection    │   │
│  │ Portfolio Agent ────── real-time AUM, DPD, segment breakdown      │   │
│  │ Onboarding Agent ───── self-service lender config + go-live       │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### Agent Lifecycle (SRAL Pattern from caas-casestudy)

Every agent in every family follows the same lifecycle:

```
SENSE ──> REASON ──> ACT ──> LEARN
  │          │         │        │
  │          │         │        └─ Outcome feedback -> RL signal -> model update
  │          │         └────────── Tool execution (MCP), gated by policy engine
  │          └──────────────────── LLM inference (constellation-routed) + context graph
  └─────────────────────────────── Channel input + ULI data + graph memory

LIFECYCLE PIPELINE (per agent deployment):
  Development ─> Eval Suite ─> Shadow Mode ─> Canary (7-stage 1%->100%) ─> Production
       │              │             │                │                          │
       │         tau-bench      parallel run      auto-graduate            SRAL feedback
       │         pass^k         no state change   auto-rollback            loop active
       │         6 scorers      verify alignment  on regression
       │
  AUTONOMY LEVELS (per-bank override, from lia-openclaw trust engine):
    L0: Fully supervised (human approves every action)
    L1: Human-in-loop (agent recommends, human decides)
    L2: Supervised autonomy (agent acts, human reviews async)
    L3: Full autonomy (agent acts, human monitors)
    L4: Self-improving (agent adjusts own parameters within bounds)

  Trust decays: if agent accuracy drops below threshold for N days,
  autonomy level automatically decreases. CRO must re-approve promotion.
```

## 2.4 Data Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│ UNIFIED DATA ARCHITECTURE                                                │
│                                                                          │
│ PRIMARY STORES                                                           │
│ ┌────────────────────────────────────────────────────────────────────┐   │
│ │ PostgreSQL (partitioned by lender_id, row-level security)          │   │
│ │ ├── borrower_profiles (PII encrypted AES-256-GCM)                 │   │
│ │ ├── loan_applications (full lifecycle state machine)               │   │
│ │ ├── credit_decisions (score, grade, policy version, explain)       │   │
│ │ ├── loan_accounts (disbursed, active, closed, graduated)          │   │
│ │ ├── repayment_events (scheduled, actual, adjusted, moratorium)    │   │
│ │ ├── lender_profiles (Prism config, PSL targets, risk appetite)    │   │
│ │ ├── policy_definitions (versioned, shadow state, diff history)    │   │
│ │ ├── segment_configs (38 segments, YAML-sourced, versioned)        │   │
│ │ ├── journey_definitions (GenAI generated, A/B config)             │   │
│ │ ├── consent_ledger (DPDP Act: per data element per borrower)      │   │
│ │ └── audit_log (immutable, partitioned by month, tamper-evident)   │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│ CONTEXT GRAPH (from contextgraphs-platform-fintech)                      │
│ ┌────────────────────────────────────────────────────────────────────┐   │
│ │ Neo4j Causal Cluster                                               │   │
│ │ ├── Context Lakehouse (versioned metadata, time travel)            │   │
│ │ ├── Data Graph (column-level lineage across 136 ULI sources)      │   │
│ │ ├── Knowledge Graph (28 lending domain terms + segment ontology)   │   │
│ │ ├── Governance Graph (7 policies, 14 compliance rules per reg.)   │   │
│ │ ├── Active Ontology (entity resolution across 8 systems)           │   │
│ │ ├── Decision Traces (first-class: input -> reasoning -> output)   │   │
│ │ │                                                                  │   │
│ │ │ Graph entities:                                                  │   │
│ │ │ (:Borrower)-[:HAS_LOAN]->(:Loan)                               │   │
│ │ │ (:Borrower)-[:MEMBER_OF]->(:Cooperative)                        │   │
│ │ │ (:Borrower)-[:OWNS]->(:Land)                                    │   │
│ │ │ (:Loan)-[:DECIDED_BY {score,grade}]->(:Policy {version})       │   │
│ │ │ (:Loan)-[:HAS_REPAYMENT]->(:RepaymentEvent)                    │   │
│ │ │ (:Loan)-[:HAS_MORATORIUM {trigger}]->(:MoratoriumEvent)        │   │
│ │ │ (:Loan)-[:GRADUATED_TO]->(:Loan)                               │   │
│ │ │ (:Pincode)-[:HAD_WEATHER {severity}]->(:WeatherEvent)          │   │
│ │ └── Every event becomes a graph node. Context compounds.           │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│ CACHE / STATE                                                            │
│ ┌────────────────────────────────────────────────────────────────────┐   │
│ │ Redis Cluster (6-node, TLS)                                        │   │
│ │ ├── Agent conversation state + OTP sessions                        │   │
│ │ ├── Policy cache (hot policies for <5ms lookup)                   │   │
│ │ ├── Feature cache (pre-computed borrower features from Feast)     │   │
│ │ ├── Rate limiting (per-lender, per-API, per-borrower)             │   │
│ │ └── Distributed locks (idempotency for disbursement)              │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│ EVENT BUS                                                                │
│ ┌────────────────────────────────────────────────────────────────────┐   │
│ │ Kafka (3-broker, Avro schema registry, 7-day retention)            │   │
│ │ ├── uli.fetch.{requested|completed|failed|partial}                │   │
│ │ ├── dq.enriched (borrower profile ready for decisioning)          │   │
│ │ ├── sentinel.decision.{requested|completed|referred|rejected}     │   │
│ │ ├── product.designed / los.{created|signed|disbursed|activated}   │   │
│ │ ├── prism.matched / prism.co_lend_structured                      │   │
│ │ ├── agent.interaction.{voice|whatsapp|web|partner}                │   │
│ │ ├── monitor.trigger.{detected|intervened|resolved}                │   │
│ │ ├── graph.ingest (every state change -> graph node)               │   │
│ │ ├── audit.event (immutable, every action logged)                  │   │
│ │ └── rl.signal (reward signals for model retraining)               │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│ ML STORES                                                                │
│ ┌────────────────────────────────────────────────────────────────────┐   │
│ │ MLflow: model registry, experiment tracking, model versioning      │   │
│ │ Feast: feature store, pre-computed segment features                │   │
│ │ S3/MinIO: training datasets, model artifacts, voice recordings     │   │
│ └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│ SECURITY: AES-256-GCM at rest. TLS 1.3 in transit. Vault for keys.      │
│ Row-level security per lender. DPDP consent per data element.            │
│ PCI-grade scrubbing at event boundary (from caas-casestudy pattern).     │
└──────────────────────────────────────────────────────────────────────────┘
```

## 2.5 Loan Application State Machine

Every loan follows this state machine. Each transition emits a Kafka event, an audit log entry, and a graph memory node. This unifies the 24-state machine (contextgraphs), 14-state machine (fb-product-l2/los), and the ecaas-v2 lifecycle.

```
                          ┌─────────┐
                          │ CREATED │
                          └────┬────┘
                               │ borrower initiates (voice/WhatsApp/web/partner)
                               v
                          ┌─────────┐
                     ┌───>│KYC_CHECK│
                     │    └────┬────┘
                     │         │ pass ──────────────────> KYC_FAILED (terminal)
                     │         v
                     │    ┌──────────┐
                     │    │DATA_FETCH│  (parallel: ULI + AA + BankConnect + Device)
                     │    └────┬─────┘
                     │         │ enriched profile + confidence score
                     │         v
                     │    ┌───────────┐
                     │    │DECISIONING│  Sentinel AI + segment model + fraud check
                     │    └────┬──────┘
                     │    ┌────┴────────────────┐
                     │    v                     v
                     │ ┌────────┐         ┌──────────┐
                     │ │APPROVED│         │ REFERRED │──> human review queue
                     │ └───┬────┘         └────┬─────┘
                     │     │                   │ approve/reject
                     │     v                   v
                     │ ┌───────────────┐  ┌──────────┐
                     │ │PRODUCT_DESIGN │  │ REJECTED │ (terminal, with reason codes)
                     │ └───────┬───────┘  └──────────┘
                     │         │ dynamic schedule generated (8 types)
                     │         v
                     │ ┌──────────────────┐
                     │ │LENDER_MATCHING   │  Prism: PSL gap + risk + cost optimization
                     │ └────────┬─────────┘
                     │          │ lender selected (or co-lending structure built)
                     │          v
                     │ ┌─────────────────┐
                     │ │ OFFER_PRESENTED │  borrower sees terms via channel agent
                     │ └───────┬─────────┘
                     │    ┌────┴──────┐
                     │    v           v
                     │ ┌────────┐ ┌──────────┐
                     │ │ACCEPTED│ │ DECLINED │ (terminal)
                     │ └───┬────┘ └──────────┘
                     │     │ e-sign + insurance enrollment
                     │     v
                     │ ┌──────────┐
                     │ │DISBURSING│  LOS triggers UPI/NEFT via lender bank API
                     │ └────┬─────┘
                     │      │ funds confirmed
                     │      v
                     │ ┌──────────┐
                     │ │ ACTIVE   │<──── monitoring agent watches continuously
                     │ └────┬─────┘
                     │      │
                     │  ┌───┴──────────────────────────────────────┐
                     │  │ ACTIVE sub-states:                       │
                     │  │ ├── CURRENT (0 DPD)                      │
                     │  │ ├── MORATORIUM (weather/health trigger)   │
                     │  │ ├── ADJUSTED (schedule modified)          │
                     │  │ ├── DELINQUENT (DPD > 0)                 │
                     │  │ ├── RESTRUCTURED (human approved)         │
                     │  │ └── EMERGENCY_LOAN (micro-loan active)   │
                     │  └───┬──────────────────────────────────────┘
                     │      │
                     │  ┌───┴──────────┐
                     │  v              v
                     │ ┌──────┐   ┌─────────┐
                     │ │CLOSED│   │GRADUATED│──> new loan at better terms
                     │ │(paid)│   └─────────┘
                     │ └──────┘
                     │
                     │ (if write-off)
                     │      v
                     │ ┌──────────┐
                     └─│WRITTEN_OFF│ (terminal)
                       └──────────┘
```

## 2.6 MCP Protocol Layer -- Unified Tool Registry

The platform exposes all capabilities via MCP (Model Context Protocol), enabling any AI agent (internal or external) to interact with lending services.

```
UNIFIED MCP SERVER: lending-platform.mcp.internal:8443 (TLS, JSON-RPC 2.0)

TOOL CATEGORIES (consolidated from fb-product-l2 27 + contextgraphs 5 + fb-specifics 17 + voiceai 20):

DATA TOOLS (12):
  fetch_borrower_data         ── parallel ULI fetch with segment-aware source mapping
  analyze_bank_statement      ── BankConnect: FBCS score + cluster intelligence
  score_device_behavior       ── DeviceConnect: FIS score + fraud flags
  verify_identity             ── Aadhaar eKYC + PAN + DigiLocker
  fetch_aa_data               ── Account Aggregator consent + data
  resolve_entity              ── Active Ontology cross-system entity match
  compute_data_quality        ── Confidence score + freshness + anomaly detection
  fetch_cooperative_data      ── Dairy/agri cooperative production records
  fetch_weather_data          ── IMD pincode-level weather + satellite NDVI
  fetch_gst_data              ── GSTN business volume
  fetch_epfo_data             ── EPFO employment verification
  fetch_platform_data         ── Gig platform earnings API

DECISIONING TOOLS (8):
  trigger_credit_decision     ── Sentinel policy evaluation + segment model
  run_what_if                 ── Simulate policy changes on historical data
  fetch_model_performance     ── Gini, KS, AUC, approval rate, NPA
  detect_drift                ── PSI-based feature distribution monitoring
  check_fraud                 ── Velocity + cluster + graph ring detection
  design_product              ── Dynamic schedule from 8 generators
  explain_decision            ── SHAP values + human-readable reason codes
  generate_kfs               ── RBI-compliant Key Fact Statement

DISTRIBUTION TOOLS (7):
  match_lender                ── Prism: PSL gap + risk + cost optimization
  build_co_lending_structure  ── RBI CLA compliant split
  create_application          ── LOS: initiate workflow
  execute_esign               ── Aadhaar e-sign
  initiate_disbursement       ── UPI/NEFT via lender bank API
  enroll_insurance            ── Pashu Bima, crop, health
  check_application_status    ── Current state + next steps

MONITORING TOOLS (6):
  check_portfolio_health      ── AUM, DPD, NPA, segment breakdown
  trigger_intervention        ── Moratorium, schedule adjust, emergency
  check_graduation_eligibility ── Repayment history + improvement metrics
  fetch_borrower_context      ── Graph memory: full relationship subgraph
  detect_fraud_ring           ── Multi-hop graph query
  generate_compliance_report  ── PSL, audit, fair lending, DPDP

AGENT TOOLS (8):
  classify_intent             ── NLU: 8 lending intents
  generate_response           ── Constellation-routed LLM with lending context
  detect_empathy_escalation   ── Distress keywords + tonal analysis
  fetch_conversation_history  ── Cross-session memory
  log_interaction             ── Audit trail + graph memory
  evaluate_agent              ── tau-bench, pass^k scoring
  promote_agent               ── Shadow -> canary -> production lifecycle
  configure_journey           ── Journey Studio: segment + channel flow

PERMISSIONS (per-tool, per-role):
  AGENT_VOICE, AGENT_WHATSAPP, AGENT_WEB, AGENT_PARTNER   ── borrower-facing agents
  RISK_ANALYST, CRO, UNDERWRITER, COMPLIANCE                ── lender dashboard roles
  INTERNAL_COPILOT                                            ── lender AI copilot
  PLATFORM_ADMIN                                              ── system operations
```

---

# PART III: BUILD STAGES -- PROGRESSIVE, VERIFIED, LENDER-ALIGNED

Each build stage has: **components built**, **technical specifications**, **repo sources consolidated**, **verification gate** (must pass before next stage), and **lender workflow enabled** (what the lender client can do after this stage).

---

## BUILD 0: FOUNDATION INFRASTRUCTURE + CONTEXT LAYER (Weeks 1-4)

### 0.1 Components

| Component | Specification | Source |
|-----------|--------------|--------|
| **Kubernetes cluster** | 3-node HA control plane, autoscaling workers (CPU + GPU pools), namespaces: `platform`, `data`, `ml`, `monitoring`, `staging` | New infrastructure |
| **PostgreSQL** | Primary + 2 read replicas, row-level security, PII AES-256-GCM via pgcrypto, partitioned by `lender_id` | fb-product-l2/shared patterns |
| **Redis Cluster** | 6-node (3 primary + 3 replica), `allkeys-lru`, TLS | voiceai-platform config |
| **Kafka** | 3-broker, schema registry (Avro), topics pre-created per service registry, 7-day retention | caas-casestudy event patterns |
| **Neo4j** | Causal cluster (3 core), Context Graph schema from contextgraphs-platform, backup to S3 | contextgraphs-platform-fintech |
| **Temporal** | Workflow engine for durable loan lifecycle (addresses critical gap #2) | New -- non-negotiable |
| **Object Storage** | S3/MinIO: model artifacts, voice recordings, documents | Standard |
| **HashiCorp Vault** | Secrets, key rotation, database credential injection | Standard |
| **API Gateway** | Kong/custom: JWT, rate-limit per lender, TLS termination | voiceai-platform/gateway + fb-product-l2/gateway |
| **Observability** | Prometheus + Grafana + Loki + Jaeger + PagerDuty | voiceai-platform/trace-service patterns |
| **CI/CD** | GitHub Actions: lint, test, security scan (Trivy), build, deploy staging, deploy prod with approval | New -- addresses critical gap #6 |
| **Context Graph Bootstrap** | Deploy Context Lakehouse, Governance Graph (RBI rules), Active Ontology (entity resolution), Decision Trace infrastructure | contextgraphs-platform-fintech |

### 0.2 Security Baseline

```
SECURITY CONTROLS (BUILD 0)
────────────────────────────
1. Encryption at rest: AES-256-GCM for all PII (Aadhaar, PAN, bank accounts)
2. Encryption in transit: TLS 1.3 all inter-service + external
3. Secrets: HashiCorp Vault, zero secrets in code/config/env
4. AuthN: OAuth2 + JWT (lender), API keys (partner SDK), Aadhaar OTP (borrower)
5. AuthZ: RBAC per lender, per role (CRO, analyst, UW, ops, compliance)
6. Audit: every API call logged (caller identity, timestamp, payload hash)
7. Data isolation: row-level security in PostgreSQL, lender_id on every table
8. Consent: DPDP Act compliant per-data-element consent ledger
9. Network: service mesh (Istio) with mTLS between services
10. PCI scrubbing: at event boundary (from caas-casestudy pattern)
11. Container scanning: every build, block CRITICAL/HIGH CVEs
12. Penetration testing: pre-launch mandatory, quarterly ongoing
```

### 0.3 Verification Gate

| Criteria | Test |
|----------|------|
| All infrastructure healthy | `kubectl get nodes` all Ready, all services responding |
| PostgreSQL RLS enforced | Insert as lender_A, query as lender_B returns empty |
| Kafka end-to-end | Produce -> consume -> verify integrity on all topic patterns |
| Neo4j cluster healthy | `CALL dbms.cluster.overview()` shows 3 cores |
| Context Graph schema loaded | Governance Graph has RBI DLG + DPDP + PSL rules |
| Temporal accepting workflows | Submit test workflow, verify completion |
| Vault sealed and serving | `vault kv get secret/test` returns value |
| CI/CD green | Push to `main` -> build -> test -> deploy-staging completes |
| Security scan clean | Zero CRITICAL or HIGH CVEs in container images |

### 0.4 Lender Workflow Enabled

None yet. Foundation only. Lender technical team verifies: data isolation, encryption, audit trail, compliance posture.

---

## BUILD 1: DATA LAYER + DPI/ULI INTEGRATION (Weeks 5-12)

### 1.1 Services Built

**S1: `uli-orchestrator`** -- Single API for parallel ULI data fetch with consent management.

```
POST /v1/data/fetch
{
  "borrower_id": "uuid",
  "aadhaar_token": "encrypted",
  "aa_consent_handle": "string",
  "segment_hint": "DAIRY_FARMER",      // determines which sources to fetch
  "lender_id": "uuid",
  "request_id": "uuid"                  // idempotency key
}

Segment-Source Mapping (from ecaas-v2 38 segment configs):

  DAIRY_FARMER:  Credit bureau + Bank(AA) + Dairy coop + Land records [+ Satellite, Weather, PMFBY]
  STREET_VENDOR: Credit bureau + Bank(AA) + UPI history [+ DeviceConnect, ONDC orders]
  GIG_WORKER:    Credit bureau + Bank(AA) + Platform API [+ DeviceConnect, multi-platform]
  ASHA_WORKER:   Credit bureau + Bank(AA) + DBT records [+ DigiLocker]
  KIRANA_ONDC:   Credit bureau + Bank(AA) + GST + ONDC orders [+ DeviceConnect]
  SHG_MEMBER:    Credit bureau + Bank(AA) + SHG ledger [+ Meeting attendance]
  FPO_MEMBER:    Credit bureau + Bank(AA) + FPO records + Warehouse receipts + Mandi
  SALARIED_NTC:  Credit bureau + Bank(AA) + EPFO [+ DeviceConnect]
  ...38 total segment definitions
```

Technical specifications:
- Parallel HTTP/2 calls with per-source timeout (3s default, configurable per segment)
- Circuit breaker per source (open after 5 consecutive failures, half-open after 30s)
- Consent validation before any fetch (Aadhaar OTP token verified)
- Source config: YAML per segment defining required vs optional sources
- Retry: 1 retry with exponential backoff for transient failures
- Metrics: p50/p95/p99 latency per source, success rate, overall fetch time

**S2: `data-quality-engine`** -- Entity resolution, cross-validation, confidence scoring.

Processing pipeline (from contextgraphs Active Ontology + fb-product-l2 validation patterns):
1. **Entity Resolution**: Match borrower across sources (Jaro-Winkler > 0.85 for names, exact for Aadhaar/PAN)
2. **Cross-Source Validation**: Income (bank) vs Income (declared/cooperative), delta > 30% flagged
3. **Freshness Scoring**: FRESH (<30d) = 1.0, RECENT (30-90d) = 0.8, STALE (90-180d) = 0.5, VERY_STALE (>180d) = 0.2
4. **Anomaly Detection**: Statistical outlier detection + BankConnect cluster intelligence flags
5. **Confidence Scoring**: `conf = 0.3*(sources/required) + 0.3*avg_freshness + 0.2*cross_validation_rate + 0.2*(1-anomaly/checks)`
6. **Graceful Degradation**: Low confidence -> reduce max amount + increase rate + flag for human review. NEVER hard-reject on data unavailability.

**S3: `bankconnect-svc`** -- Statement analysis + cluster intelligence (from fb-product-l2/bankconnect).
- PDF/image OCR (Tabula + Tesseract), 40+ bank formats
- Transaction categorization: 15 categories (salary, UPI_credit, cooperative_credit, platform_payout, etc.)
- Income stability: stddev/mean over 6 months
- Cluster intelligence: circular transfers (A->B->C->A), round-trip, income inflation
- Fraud flags: `CIRCULAR_TRANSFER`, `ROUND_TRIP`, `INCOME_INFLATION`, `CASH_DEPOSIT_PATTERN`, `BOUNCE_FREQUENCY`
- FBCS 0-900 composite score

**S4: `deviceconnect-svc`** -- Behavioral scoring (from fb-product-l2/deviceconnect).
- Android/iOS SDK with explicit consent
- 5,000+ features: app_usage, location_stability, financial_app_usage, behavioral
- FIS (FinBox Inclusion Score) 0-1000 via XGBoost
- Privacy: raw data deleted after feature extraction, consent revocation within 24h

**S5: `identity-svc`** -- KYC + verification pipelines (from ecaas-v2/tools/identity).
- Aadhaar eKYC (UIDAI API, OTP mode for digital, biometric for assisted)
- PAN verification + PAN-Aadhaar seeding
- DigiLocker document fetch (income proof, PMFBY)
- Account Aggregator consent management (FI types: DEPOSIT, MUTUAL_FUNDS, INSURANCE, GST)

### 1.2 Verification Gate

| Criteria | Test |
|----------|------|
| ULI orchestrator fetches 5+ sources for dairy segment in <5s (p95) | Load test: 100 concurrent dairy fetches |
| Data quality produces confidence score with correct weighting | Unit test: 20 known profiles with expected confidence ranges |
| BankConnect parses SBI/HDFC/Kotak/BoB statements correctly | Integration test: 10 real (anonymized) statement PDFs per bank |
| DeviceConnect SDK collects and scores on Android test device | End-to-end: install SDK, consent, upload, verify FIS score returned |
| Identity service completes Aadhaar OTP flow in sandbox | Integration test: UIDAI sandbox, full OTP + eKYC |
| Entity resolution matches borrower across 3+ sources | Test: same person in bureau + bank + cooperative, verify match |
| Graceful degradation: 2/5 sources timeout, loan still processable | Test: inject timeouts, verify reduced amount + human review flag |
| All data encrypted in PostgreSQL (PII check) | Query pg_catalog, verify all PII columns use pgcrypto |

### 1.3 Lender Workflow Enabled

Lender risk team can: validate data quality for their geography/segment, review source-by-source data for test applications, compare ULI data richness against their existing bureau-only flow.

---

## BUILD 2: DECISIONING CORE (Weeks 9-16)

### 2.1 Services Built

**S6: `sentinel-core`** -- Unified policy engine consolidating:
- fb-product-l2/sentinel: BRE with 14 operators, 3 policy templates, champion/challenger
- voiceai-platform/policy-engine: 3-layer enforcement (structural + behavioral + supervisor)
- ecaas-v2/policy-engine: 7 structural + 10 behavioral rules

Policy definition format (YAML, segment-specific, versioned):

```yaml
policy:
  id: DAIRY_ANIMAL_PURCHASE_v1
  segment: DAIRY_FARMER
  version: 1
  status: SHADOW                    # SHADOW -> CANARY -> PRODUCTION

  scoring:
    model: DairyScore_v1            # MLflow model ID
    features:
      cooperative_production: {source: uli, weight: 0.25}
      bank_balance_stability: {source: bankconnect, weight: 0.25}
      land_ownership: {source: uli, weight: 0.15}
      bureau_score: {source: bureau, weight: 0.15}
      device_inclusion: {source: deviceconnect, weight: 0.10}
      cooperative_membership_years: {source: uli, weight: 0.10}

  decision_rules:
    auto_approve: {min_score: 600, max_fraud_flags: 0, min_confidence: 0.7}
    refer_to_human: {min_score: 400, or_fraud_flags: 1, or_confidence_below: 0.7}
    auto_reject: {max_score: 399, or_fraud_flags: [CIRCULAR_TRANSFER]}

  pricing:
    base_rate: 15.0
    adjustments: {A1: -2.0, A2: -1.0, B1: 0.0, B2: +2.0}

  product:
    type: SEASONAL_DYNAMIC
    amount_range: {min: 25000, max: 200000}
    insurance_required: PASHU_BIMA
    moratorium_triggers:
      - {type: WEATHER_HEAT, threshold_celsius: 38, duration_months: 1}
      - {type: PRODUCTION_DROP, threshold_percent: 40, duration_months: 1}
    graduation: {min_months: 12, min_on_time: 0.90, amount_multiplier: 2.0, rate_reduction: 2.0}
```

**S7: `product-engine`** -- 8 dynamic schedule generators (from ecaas-v2 product design):

| Type | Logic | Segment |
|------|-------|---------|
| SEASONAL_DYNAMIC | Scale EMI by flush/lean/transition months from cooperative calendar | Dairy, crop farmer |
| DAILY_MICRO | Daily deduction adjusted for earning pattern, skip if below threshold | Street vendor |
| CATCH_SALE_LINKED | Repayment triggered by mandi/warehouse sale event | FPO, warehouse |
| INCOME_CONTINGENT | Monthly = verified_income * share_rate (EPFO/platform verified) | Gig, salaried NTC |
| MORATORIUM_WITH_TRIGGER | Standard EMI + auto-pause on weather/health triggers | All agri segments |
| BRIDGE_ZERO_INTEREST | Principal-only, auto-deduct on DBT/insurance arrival | Emergency, bridge |
| STAGED_DISBURSEMENT | Tranche at milestones (medical: 30/40/30 at admission/procedure/discharge) | Medical, asset |
| PAYOUT_LINKED | Auto-deduct from platform settlement (e.g., 15% of each Swiggy payout) | Gig worker |

**S8: `fraud-engine`** -- Consolidated fraud detection (from fb-product-l2 + bankconnect):
- Velocity checks (max 2 applications per Aadhaar in 48h)
- BankConnect cluster analysis (circular transfers, round-trip, income inflation)
- Graph-based ring detection via Neo4j (shared reference persons, shared devices)

### 2.2 MCP Server Implementation

All decisioning tools exposed via MCP (JSON-RPC 2.0, port 8443 TLS):
- `trigger_credit_decision`: full policy evaluation + segment model + fraud check
- `run_what_if`: simulate policy changes on historical portfolio
- `fetch_model_performance`: Gini, KS, AUC, approval rate, NPA
- `detect_drift`: PSI-based feature distribution monitoring
- `explain_decision`: SHAP values + reason codes
- `generate_kfs`: RBI-compliant Key Fact Statement

### 2.3 Verification Gate

| Criteria | Test |
|----------|------|
| Sentinel evaluates dairy policy on test profile in <200ms (p95) | Load test: 500 concurrent evaluations |
| DairyScore produces score 0-900 with correct feature weights | Unit test: 20 known profiles with expected scores |
| Shadow mode runs parallel without affecting production | Enable shadow, verify results produced but state unchanged |
| Champion/challenger splits correctly | Route 10% to challenger, verify split |
| MCP server responds to all decisioning tools | Integration test: call each tool, verify schema |
| Fraud engine catches velocity abuse | Submit 3 apps same Aadhaar in 1 hour, verify 3rd held |
| Product engine generates valid seasonal schedule | Dairy schedule: sum(EMIs) = principal + interest, flush > lean |
| Risk-based pricing maps correctly | Score 780 -> 14%, score 580 -> 17% |
| Explainability produces reason codes | Request explanation for reject, verify codes present |
| KFS generation RBI-compliant | Generate KFS, verify all mandatory fields present |

### 2.4 Lender Workflow Enabled

Risk analyst uses policy builder (API/CLI at this stage) to: load template -> adjust weights -> shadow test -> review results. CRO asks copilot: "What is projected NPA if I lower moratorium trigger from 40C to 38C?"

---

## BUILD 3: DISTRIBUTION LAYER (Weeks 13-20)

### 3.1 Services Built

**S9: `prism-matcher`** -- Multi-lender matching (from fb-product-l2/prism):

```python
def match_lender(application, decision):
    eligible = filter_by_segment(lenders, application.segment)
    eligible = filter_by_geography(eligible, application.pincode)
    eligible = filter_by_ticket_size(eligible, decision.amount)
    eligible = filter_by_capital_available(eligible)

    scored = []
    for lender in eligible:
        score = (psl_gap_score(lender, application.psl_category)      # 0-40
               + approval_probability(lender, decision)                # 0-30
               + cost_to_borrower(lender, decision.rate)               # 0-20
               + fldg_availability(lender, application.segment))       # 0-10
        scored.append((lender, score))

    ranked = sorted(scored, key=lambda x: x[1], reverse=True)
    if ranked[0][0].co_lending and decision.amount > co_lend_threshold:
        return build_co_lending_structure(ranked[0], ranked[1], decision)
    return ranked[0]
```

**S10: `los-engine`** -- Temporal-based loan origination workflow:

```
LoanOriginationWorkflow (Temporal):
  1. CreateApplication(borrower, channel, segment)
  2. RunKYC(identity-svc)                          // verification pipeline
  3. FetchData(uli-orchestrator)                   // parallel ULI fetch
  4. RunDecisioning(sentinel-core)                 // policy evaluation
  5. IF APPROVED:
     a. DesignProduct(product-engine)               // dynamic schedule
     b. MatchLender(prism-matcher)                  // lender selection
     c. PresentOffer(agent-runtime -> channel)      // back to borrower
     d. IF ACCEPTED:
        i.   ExecuteESign(aadhaar_esign)
        ii.  EnrollInsurance(insurance_api)
        iii. InitiateDisbursement(lender_bank_api)
        iv.  ConfirmDisbursement(payment_status)
        v.   ActivateLoan(loan_account)             // state -> ACTIVE
        vi.  StartMonitoring(monitoring-agent)       // continuous watch begins
  6. IF REFERRED:
     a. RouteToHumanQueue(lender_dashboard)
     b. AwaitHumanDecision(timeout: 48h)
  7. IF REJECTED:
     a. GenerateRejectionNotice(explainability)
     b. NotifyBorrower(notification-svc)
```

**S11: `journey-studio`** -- No-code journey builder (from fb-product-l2/journey_studio + voiceai-platform/workspace):
- JSON schema for screens, fields, validations, branching, data source bindings
- GenAI journey generator (LLM takes segment + channel -> journey definition)
- A/B testing (borrower hash split, chi-squared significance)
- Conversion funnel: start -> KYC -> data -> offer -> accept -> disburse

**Lender Dashboard** (React + TypeScript, from ecaas-v2/frontend + voiceai-platform/agent-studio):

| Screen | Features |
|--------|----------|
| Portfolio Overview | AUM, active loans, segment breakdown, DPD buckets, NPA, origination chart |
| Application Queue | Pending, referred cases with AI assessment + recommendation |
| Policy Builder | Visual no-code editor: load template -> adjust -> shadow -> promote |
| Shadow Test Results | Side-by-side: production vs shadow approval rate, score distribution |
| AI Copilot | NL interface via MCP ("Show me dairy NPA trend", "What if expand to Rajasthan?") |
| Compliance Center | PSL report, audit trail search, bias report, decision explainability |
| Lender Config | Prism profile: segments, geography, ticket range, PSL targets, co-lending |
| User Management | RBAC: CRO, risk_analyst, underwriter, ops, compliance |

### 3.2 Verification Gate

| Criteria | Test |
|----------|------|
| Prism routes dairy app to highest PSL-gap bank | 10 dairy apps -> verify routing logic |
| Co-lending structure generated for FPO > 2L | 5L FPO app -> verify 80/20 split |
| Journey Studio generates dairy voice journey from prompt | "Dairy farmer, Gujarati, voice, animal purchase" -> valid JSON |
| LOS workflow end-to-end in staging (sandbox payments) | Submit -> KYC -> data -> decision -> offer -> accept -> mock disburse |
| Dashboard renders with test data | 100 test loans -> verify AUM, DPD, NPA correct |
| Policy builder: create -> shadow -> view results | Create test policy, run against 50 profiles |
| Copilot answers "What is dairy NPA?" via MCP | Submit query, verify MCP call + response |
| RBAC: analyst cannot promote without CRO approval | Attempt, verify rejected |
| PSL report auto-generates correctly | Generate report, verify counts match DB |
| Full audit trail for single application | Query, verify all 15+ events logged |

### 3.3 Lender Workflow Enabled

Full staging simulation: CRO reviews dashboard, risk analyst creates policies, underwriter processes referrals, compliance generates reports, operations monitors disbursements. **2-week shadow test with real (anonymized) data = trust-building before real capital.**

---

## BUILD 4: BORROWER INTERFACE AGENTS (Weeks 17-24)

### 4.1 Services Built

**S13: `agent-runtime`** -- Multi-agent orchestrator (consolidating voiceai-platform 16-step turn processor + ecaas-v2 60-tool agent + caas-casestudy SRAL):

```
AGENT ORCHESTRATOR
├── CHANNEL ADAPTERS
│   ├── Voice Adapter (Exotel/Twilio PSTN, toll-free, IVR bypass)
│   ├── WhatsApp Adapter (BSP: Gupshup/Twilio, buttons, lists, documents)
│   ├── Web Adapter (REST API)
│   └── Partner Adapter (ONDC/gig platform SDK)
│
├── NLU / INTENT ENGINE
│   ├── Language detection (12 Indian languages)
│   ├── ASR: Whisper large-v3 fine-tuned on Indian languages + lending vocab
│   ├── Intent: NEW_LOAN, CHECK_STATUS, PAYMENT_QUERY, SCHEDULE_CHANGE,
│   │          COMPLAINT, EMERGENCY, GRADUATION_INQUIRY, GENERAL_QUERY
│   ├── Entity: amount, tenure, segment, asset_type, location, language
│   └── TTS: Bhashini/ElevenLabs, segment-appropriate voice
│
├── CONVERSATION MANAGER
│   ├── State machine per conversation (from Journey Studio)
│   ├── MCP client: calls sentinel-core, los-engine, monitoring tools
│   ├── Context from graph-memory (returning borrowers: "Welcome back, Rameshbhai")
│   ├── Empathy detection: distress keywords + tonal analysis -> human escalation
│   └── Full conversation log -> audit-trail + graph memory
│
└── FAST BRAIN / SLOW BRAIN (from voiceai-platform)
    ├── Fast: cached responses, FAQ, status lookups (<200ms)
    └── Slow: credit decision, product design, complex queries (parallel exec)
```

**S14: `voice-pipeline`** -- (from voiceai-platform, adapted for Indian languages):

| Component | Specification |
|-----------|--------------|
| ASR | Whisper large-v3 fine-tuned on: Gujarati, Hindi, Tamil, Telugu, Kannada, Bengali, Marathi, Odia, Malayalam, Punjabi, Assamese, Rajasthani. Domain vocab: loan, EMI, interest, Aadhaar, buffalo, cooperative, harvest, monsoon. Target: WER < 15%. |
| TTS | Bhashini API (Indian languages, government-backed) + ElevenLabs fallback. Voice selection: configurable per segment. |
| VAD | Energy-based, 500ms lending-tuned (pause for loan numbers). |
| Telephony | Exotel (India-native) / Twilio (fallback) PSTN integration. Toll-free number. |
| Latency | Target <1.5s (ASR + NLU + response + TTS). Requires GPU inference. |

**S15: `constellation`** -- Multi-model routing (from voiceai-platform):
- Routes to optimal LLM per task (fast model for FAQ, capable model for decisioning assist)
- Failover chain with latency-aware selection
- Parallel fast brain/slow brain execution

### 4.2 Verification Gate

| Criteria | Test |
|----------|------|
| Voice agent completes dairy journey in Gujarati end-to-end | Simulated call: dairy intent -> Aadhaar OTP -> offer -> acceptance. < 10 min. |
| ASR accuracy > 85% on Gujarati lending utterances | 200 labeled audio clips, measure WER |
| WhatsApp agent completes vendor journey in Hindi | Simulated conversation: vendor intent -> photo OCR -> offer -> acceptance |
| Intent classification > 90% across 8 intents | 500 labeled utterances, 3 languages |
| Empathy detection escalates distress call | "Husband passed" + crying tone -> verify human transfer |
| MCP: agent triggers credit decision and receives offer | Voice agent -> MCP -> sentinel -> offer -> present to borrower |
| Conversation log in audit trail | Verify: transcript + audio recording stored |
| Returning borrower recognized via graph memory | Second call from same Aadhaar -> "Welcome back" with context |

### 4.3 Lender Workflow Enabled

**Controlled pilot: 500-1,000 real loans.** 50-100 dairy farmers call toll-free in Gujarati. All applications on REFER (human review mandatory). Lender compares agent-originated vs. branch-originated on data quality and decision quality.

---

## BUILD 5: MONITORING, INTELLIGENCE & RL LOOP (Weeks 21-30)

### 5.1 Services Built

**S18: `monitoring-agent`** -- Continuous post-disbursal watch (from ecaas-v2/tools/monitoring + caas-casestudy patterns):

```
DATA STREAM WATCHERS (configurable per segment):
  CooperativeWatcher: poll dairy coop API daily -> detect pouring drop
  WeatherWatcher: poll IMD API per pincode hourly -> heat wave, flood, drought
  BankAccountWatcher: poll AA weekly -> income pattern shift
  PlatformWatcher: poll gig platform API daily -> earnings drop, rating change
  EPFOWatcher: poll quarterly -> employment change
  GSTWatcher: poll monthly -> business volume trend
  RepaymentWatcher: internal on EMI due -> DPD, early payment, overpayment

TRIGGER EVALUATOR:
  For each (loan, signal):
    1. Look up policy trigger rules
    2. Evaluate condition (threshold breach?)
    3. Determine intervention type
    4. Execute (auto or route to human)
    5. Notify borrower (voice/WhatsApp/SMS)
    6. Log to audit + graph memory
    7. Emit rl.signal for training

INTERVENTIONS:
  AUTO_MORATORIUM      ── pause EMI, recalculate schedule
  SCHEDULE_ADJUST      ── reduce EMI for N months
  EMERGENCY_MICROLOAN  ── offer INR 5-10K emergency loan
  INSURANCE_CLAIM      ── pre-file claim with insurer
  GENTLE_REMINDER      ── nudge via preferred channel
  GRADUATION_OFFER     ── pre-approve larger loan at better terms
  RATE_REDUCTION       ── offer lower rate mid-tenure
  HUMAN_ESCALATION     ── route to resolution team
```

**S17: `context-graph`** enhanced with RL signals:

```
RL SIGNAL COLLECTION (Kafka topic: rl.signal):
  {loan_id, signal_type: "EMI_COLLECTED", amount, on_time: true}
  {loan_id, signal_type: "PRODUCTION_VERIFIED", value: 38, unit: "liters/day"}
  {loan_id, signal_type: "WEATHER_MORATORIUM_TRIGGERED", pincode, temp: 41}
  {loan_id, signal_type: "MORATORIUM_CURE", resumed: true, months_paused: 1}
  {loan_id, signal_type: "DEFAULT", dpd: 90}
  {loan_id, signal_type: "GRADUATION", new_loan_id, amount_increase: 2.0}

ML PIPELINE (weekly batch):
  1. Collect rl.signal events
  2. Join with original decision data
  3. Compute reward: 0.6*verifiable + 0.3*partial + 0.1*proxy
  4. Retrain segment models with updated labels
  5. Evaluate: Gini, KS, AUC on holdout
  6. If improved: register in MLflow -> auto-shadow
  7. CRO approves promotion to production
```

### 5.2 Verification Gate

| Criteria | Test |
|----------|------|
| Weather event triggers moratorium within 1 hour | Inject mock IMD heat wave, verify moratorium activated |
| 40% pouring drop triggers intervention | Mock cooperative data, verify agent calls borrower |
| Graph memory returns full borrower context | 2 loans, 5 repayments, 1 adjustment -> verify complete subgraph |
| Fraud ring detected via graph query | 5 apps sharing reference person -> all flagged |
| RL pipeline produces updated model version | 1,000 test loans -> verify new model in MLflow |
| Graduation trigger fires at 12 months clean | Test loan with 12 on-time -> verify graduation offer |
| Intervention success > 70% in simulation | 100 distress events -> 70+ prevented DPD |
| All monitoring events in audit trail | 5 different interventions -> all in audit with context |

### 5.3 Lender Workflow Enabled

Full post-disbursal lifecycle: monitoring dashboard with early warnings, collections with context ("missed payment due to verified heat wave, 80% cure rate after 1-month moratorium"), CRO reviews model improvements ("DairyScore v3 shows 15% better Gini"), compliance pulls full audit trail.

---

## BUILD 6: SCALE, OPTIMIZATION & ADVANCED (Weeks 27-40+)

### 6.1 Components

| Component | Specification | Source |
|-----------|--------------|--------|
| **Multi-segment expansion** | Add: vendor, SHG, gig, kirana, fisherman, ASHA. Each needs: scoring model, source mapping, product template, monitoring triggers, conversation flow. | ecaas-v2 (38 segment configs ready) |
| **Multi-language expansion** | Add ASR+TTS+NLU for: Tamil, Telugu, Kannada, Bengali, Marathi, Odia, Malayalam, Punjabi. 5,000+ labeled utterances each. | voiceai-platform ASR pipeline |
| **Co-lending engine** | RBI CLA compliant: Bank(80%) + NBFC(20%) + fintech(FLDG). Split disbursement, split repayment, consolidated reporting. | fb-product-l2/prism extension |
| **Climate credit overlay** | Pincode-level climate risk scoring (IMD + ISRO). Moratorium trigger calibration. Climate-risk-adjusted pricing. | New -- from Knowledge-Repo research |
| **Federated fraud detection** | Cross-lender anonymized signals. borrower_hash in default_ring at lender_A -> flag at lender_B. | New -- from caas-casestudy patterns |
| **ONDC embedded lending** | SDK in ONDC seller dashboard. Pre-qualification in background. One-click loan at point of need. | New -- from fb-specifics/ONDC analysis |
| **Performance optimization** | Redis hot policy cache (<1ms), Feast feature pre-compute, ONNX Runtime inference, latent briefing (49% token savings). | gff-case-studies/latent-briefing |
| **Self-service lender onboarding** | Sign up -> configure Prism profile -> connect disbursement API -> shadow test -> go live. Target: <2 weeks. | fb-product-l2/gateway + voiceai/workspace |

### 6.2 Verification Gate

| Criteria | Test |
|----------|------|
| Second segment (vendor) end-to-end | WhatsApp vendor journey in Hindi: apply -> disburse -> 1 week monitoring |
| Co-lending splits correctly | 5L FPO loan: 80% bank, 20% NBFC, FLDG reserved |
| Climate risk integrated into dairy policy | High-risk pincode -> higher moratorium probability, reflected in pricing |
| Cross-lender fraud works (anonymized) | Mark hash as fraud at lender_A, verify flag at lender_B |
| ONDC embedded flow in 3 taps | Seller dashboard -> "Loan available" -> tap -> pre-approved -> tap -> disbursed |
| 1,000 concurrent applications | Load test: p95 decision latency < 500ms |
| 5 lenders simultaneously | Verify: data isolation, correct routing, no cross-contamination |
| Model improving over time | Gini at month 1 vs month 6: statistically significant improvement |

---

# PART IV: CROSS-CUTTING CONCERNS

## 4.1 Compliance Implementation

| Regulation | Implementation | Verification |
|------------|---------------|-------------|
| RBI Digital Lending (Sep 2022) | KYC before disbursement, KFS generation, cooling-off period, grievance redressal | Automated KFS, cooling-off enforced in LOS |
| RBI CLA Directions (Jan 2026) | Co-lending templates, split disbursement/repayment, consolidated reporting | Prism co-lending workflow tests |
| DPDP Act 2023 | Consent ledger per data element, purpose limitation, data minimization, right to erasure | Consent API tests, deletion workflow |
| PSL Classification | Auto-classification at origination (segment + borrower + loan purpose) | PSL report vs manual classification |
| Fair Lending / FREE-AI | Bias detection (gender, caste, religion, geography). SHAP explainability. | Statistical parity, equalized odds, disparate impact |
| RBI DLG 2025 | Digital Lending Guidelines compliance. First Loss Default Guarantee tracking. | DLG reserve computation, reporting |

## 4.2 Testing Strategy Per Build

| Build | Unit Tests | Integration Tests | Load Tests | Security Tests |
|-------|-----------|-------------------|-----------|---------------|
| 0 | Infra health | Service connectivity | Baseline throughput | Penetration test |
| 1 | Parsers, validators, entity resolution | ULI end-to-end sandbox | 100 concurrent fetches | PII encryption |
| 2 | Scoring models, policy eval, fraud | Decision pipeline e2e | 500 concurrent decisions | MCP auth |
| 3 | Matching, journey, LOS workflow | Full loan lifecycle staging | 200 concurrent LOS | RBAC, XSS/CSRF |
| 4 | ASR accuracy, intent, conversation | Agent e2e with mock borrower | 50 concurrent voice calls | Voice recording encryption |
| 5 | Triggers, graph queries, RL signals | Monitoring loop mock streams | 10,000 active loans | Audit immutability |
| 6 | Co-lending, climate, federation | Multi-lender concurrent | 1,000 concurrent apps | Cross-tenant isolation |

## 4.3 Build Timeline

```
WEEK:  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31-40
       ├──BUILD 0──┤
       │Foundation  │
       │+Context    ├──────BUILD 1──────┤
       │            │Data Layer+DPI     │
       │            │ULI+Bank+Device    ├──────BUILD 2──────┤
       │            │                   │Decisioning Core   │
       │            │                   │Sentinel+Product   ├──────BUILD 3──────┤
       │            │                   │+Fraud+MCP         │Distribution       │
       │            │                   │                   │Prism+LOS+Dash     ├────BUILD 4─────┤
       │            │                   │                   │+Journey+Copilot   │Borrower Agents │
       │            │                   │                   │                   │Voice+WA+NLU    ├──BUILD 5──────┤
       │            │                   │                   │                   │                │Monitor+Graph  │
       │            │                   │                   │                   │                │RL Loop        ├──BUILD 6──>
       │            │                   │                   │                   │                │               │Scale
       │            │                   │                   │                   │                │               │
VERIFY:├──GATE 0───├──────GATE 1──────├──────GATE 2──────├──────GATE 3──────├───GATE 4─────├──GATE 5─────├──GATE 6──>
       │            │                   │                   │                   │                │               │
LENDER:│Sec review  │Data validation   │Shadow testing     │Full staging sim  │Pilot 500-1000 │Live portfolio │Multi-seg
       │            │with risk team    │with risk team     │all roles         │real loans      │monitoring     │expansion
       │            │                   │                   │                   │                │               │
AGENTS:│            │Data agents       │Decision agents    │Distribution      │Voice+WA+Web   │Monitor+Graph  │All 6
       │            │deployed          │deployed           │agents deployed   │agents deployed │agents active  │families
```

---

# PART V: CONSOLIDATION EXECUTION PLAN

## 5.1 Base Monorepo

Build from **voiceai-agent-services-platform** architecture:
- Most mature service decomposition (14 services)
- Only real voice pipeline (STT/TTS/VAD/telephony)
- Complete agent lifecycle (eval/shadow/canary)
- SDK (pip-installable)
- Agent Studio web UI

## 5.2 Merge Sequence

| Phase | What Merges | From Repo | Timeline |
|-------|-------------|-----------|----------|
| **Phase 0** (Week 1-2) | Context Layer: Lakehouse, Governance Graph, Decision Traces, Active Ontology | contextgraphs-platform-fintech | Foundation |
| **Phase 0** (Week 1-2) | Shared infra: auth, audit, PII, compliance, KFS generation | fb-product-l2/shared | Foundation |
| **Phase 1** (Week 3-6) | 38 borrower segment configs + bilingual prompts | ecaas-v2/configs | Core |
| **Phase 1** (Week 3-6) | BankConnect: FBCS scoring, cluster intelligence, 30+ lender patterns | fb-product-l2/bankconnect | Core |
| **Phase 1** (Week 3-6) | Sentinel BRE: 14 operators, 3 templates, fraud detection | fb-product-l2/sentinel | Core |
| **Phase 1** (Week 3-6) | India-specific tool interfaces: eKYC, AA, bureau, eSign, NACH | ecaas-v2/tools | Core |
| **Phase 2** (Week 7-10) | SRAL loop + eval/shadow/canary (7-stage) pipeline | caas-casestudy/lifecycle | Lifecycle |
| **Phase 2** (Week 7-10) | Progressive trust engine (4-level + decay) | lia-openclaw/trust | Lifecycle |
| **Phase 2** (Week 7-10) | Lending VLM + latent briefing (49% token savings) | gff-case-studies | Intelligence |
| **Phase 3** (Week 11-14) | WhatsApp channel adapter | ecaas-v2/gateway | Channel |
| **Phase 3** (Week 11-14) | Lender dashboard (Next.js) | ecaas-v2/frontend | UX |
| **Phase 3** (Week 11-14) | Autopilot copilot mode (for enterprise lender adoption) | autopilot-in-enterprise | UX |
| **Phase 4** (Week 15-18) | MCP lending lifecycle servers (origination, UW, servicing) | fb-specifics | Integration |
| **Phase 4** (Week 15-18) | Multi-lender routing (Prism) | fb-product-l2/prism | Routing |

## 5.3 Critical Gap Resolution

| Gap | Resolution | Build Stage |
|-----|-----------|-------------|
| **No real DPI/ULI integration** | uli-orchestrator service with parallel calls to all 136 ULI endpoints, UIDAI sandbox first, production API keys in Build 4 pilot | Build 1 |
| **No durable workflow orchestration** | Temporal deployment in Build 0, all loan lifecycle workflows as Temporal workflows | Build 0 |
| **No real LLM integration** | Constellation service with multi-model routing, fine-tuned Whisper for Indian ASR, Bhashini TTS. GPU inference nodes in K8s. | Build 4 |
| **No multi-tenant database** | PostgreSQL RLS in Build 0, lender_id partitioning, cross-tenant query prevention verified in Gate 0 | Build 0 |
| **No load testing** | Load test at every gate (100 -> 500 -> 200 -> 50 -> 10K -> 1K concurrent) | Every build |
| **No CI/CD** | GitHub Actions pipeline in Build 0: lint -> test -> security scan -> build -> deploy staging -> approve -> deploy prod | Build 0 |

---

# PART VI: FIRST LENDER DEPLOYMENT PLAYBOOK

## 6.1 Pre-Launch Checklist

```
PRE-LAUNCH CHECKLIST (all must be YES)
──────────────────────────────────────

TECHNICAL
[ ] All Build 0-4 verification gates passed
[ ] Penetration test completed, no CRITICAL findings
[ ] Load test: 200 concurrent applications, p95 < 2s
[ ] Disaster recovery tested: failover < 5 min
[ ] Data backup: point-in-time recovery tested
[ ] Monitoring alerts configured (PagerDuty)
[ ] Context graph populated with segment ontology + compliance rules

LENDER VALIDATION
[ ] Risk team approved 3+ Sentinel policies (shadow tested 2+ weeks)
[ ] CRO signed off on risk appetite in Prism
[ ] Compliance validated PSL classification logic
[ ] Compliance validated audit trail completeness
[ ] Underwriters trained on referred case queue + AI assessment
[ ] Operations trained on disbursement monitoring

REGULATORY
[ ] RBI Digital Lending compliance documented
[ ] DPDP Act consent framework approved by legal
[ ] Fair lending bias test passed for all active policies
[ ] KFS template approved by compliance
[ ] Grievance redressal mechanism configured and tested

BORROWER READINESS
[ ] Voice agent tested with 50+ real dairy farmers (controlled)
[ ] ASR accuracy > 85% on real speech (Gujarati lending domain)
[ ] Toll-free number provisioned and tested
[ ] Insurance partner (Pashu Bima) integration tested
[ ] Disbursement API tested with lender bank (real small amount)

OPERATIONAL
[ ] On-call rotation 24/7 for first 30 days
[ ] Escalation matrix: agent -> ops -> PM -> CTO
[ ] Rollback procedure documented and tested
[ ] First 100 loans: ALL on REFER (human review mandatory)
[ ] Weekly review with lender risk team scheduled

AGENT LIFECYCLE
[ ] All borrower-facing agents passed eval suite (tau-bench, pass^k)
[ ] Shadow mode validated: agent decisions align with lender expectations
[ ] Canary deployment tested: auto-rollback on regression working
[ ] Autonomy level set to L1 (human-in-loop) for first lender
[ ] Trust engine configured: decay on accuracy drop below threshold
```

## 6.2 First 90 Days

```
WEEK 1-2:  CONTROLLED PILOT
           50-100 dairy farmers, Gujarat
           All REFER (human reviews every decision)
           Voice agent + WhatsApp
           Lender risk team reviews daily
           Fix: ASR errors, intent misclassification, policy edge cases

WEEK 3-4:  EXPANDED PILOT
           500-1,000 farmers
           Auto-approve for score > 700, REFER for 400-699
           Lender reviews referred only
           Monitor: approval rate, decision time, borrower completion rate
           Fix: Journey Studio A/B tests on drop-off points

WEEK 5-8:  PRODUCTION RAMP
           Full geography rollout (Gujarat dairy)
           Autonomy L2 (supervised autonomy)
           Monitoring agent active on all disbursed loans
           Weekly model performance review with risk team
           Begin collecting RL signals

WEEK 9-12: STEADY STATE
           Full autonomy L3 for auto-approve segment
           First graduation offers generated
           RL pipeline produces DairyScore v2
           Begin shadow testing on second segment (vendor)
           Present results to lender CRO: NPA, cost-per-loan, portfolio growth
```

---

# PART VII: AGGREGATE METRICS

## Platform Statistics (Across All 19 Repos)

| Metric | Value |
|--------|-------|
| Total repos consolidated | 19 (12 with code, 5 docs, 2 empty) |
| Total lines of code | ~75,000+ |
| Total tests | ~1,100+ |
| Core services in unified platform | 18 + 7 cross-cutting = 25 |
| Agent families | 6 (borrower, data, decisioning, distribution, lifecycle, lender ops) |
| Individual agents | 30+ deployed, 100+ designed (EKP-FS blueprint) |
| MCP tools | 49+ unified (data 12, decisioning 8, distribution 7, monitoring 6, agent 8, admin 8) |
| Borrower segments | 38 with YAML configs |
| Lending tools | 60 (ecaas-v2) + 20 (voiceai) + 27 (fb-product-l2 MCP) = 107 total |
| Languages targeted | 12 Indian languages |
| Build stages | 7 (Foundation through Scale) |
| Build timeline | 40 weeks (10 months) to multi-segment, multi-lender platform |
| First lender pilot | Week 17-20 (Build 4 completion) |

## Repo-to-Service Mapping (Final)

| Repo | What Gets Consolidated | What Gets Archived |
|------|----------------------|-------------------|
| voiceai-agent-services-platform | **BASE MONOREPO**: agent-runtime, voice-pipeline, constellation, knowledge-svc, trace-service, eval, explorer, workspace, SDK, Agent Studio | -- |
| contextgraphs-platform-fintech | context-graph service, Governance Graph, Decision Traces, Active Ontology | US mortgage configs |
| fb-product-l2 | sentinel-core, los-engine, prism-matcher, bankconnect-svc, deviceconnect-svc, journey-studio, MCP server, shared infra | In-memory stores (replaced by PostgreSQL RLS) |
| embeddedcredit-as-a-service-v2 | 38 segment configs, 60 tool interfaces, bilingual prompts, WhatsApp adapter, lender dashboard | Monolith agent runtime (replaced by voiceai agent-runtime) |
| caas-casestudy | SRAL loop, eval/shadow/canary pipeline, autonomy levels, agent distribution | Credit card domain logic |
| autopilot-in-enterprise | Copilot agent pattern for enterprise lender adoption | -- |
| fb-specifics | FinBox MCP server, lending lifecycle MCP servers, enterprise compliance | -- |
| gff-case-studies | Latent briefing (token optimization), lending VLM | Agent-fill (low priority) |
| lia-openclaw | Progressive trust engine (4-level + decay), security patterns | Gmail/Calendar integration |
| Knowledge-Repo | ULI integration spec, segment research, startup roadmap | Factory AI research pages (reference only) |
| embeddedcredit-v1 | Thesis narrative (README/about page) | -- |
| EKP-FS | 106-agent blueprint (reference architecture) | -- |
| ecaas-prd | Production roadmap (this document supersedes) | -- |
| eng-docs | Founding engineer onboarding | Separate repo |
| agentic-design-patterns | Pattern reference | Separate repo |
| lendscore | Marketing site | Separate repo |
| finaii-v2 | Agent patterns (orchestrator-workers) | Separate product |
| fb-product-l1 | -- | Empty, delete |
| equal-tsp-fintech | -- | Empty, delete |
