# Agent Tooling Stack: Architectural Reconstruction
## AI-First Lending Platform — Complete Tooling Decision Map

---

# 1. ARCHITECTURE PATTERN: SERVICE MESH

## 25-Service Topology

```
                            ┌─────────────────────────────────────┐
                            │          API GATEWAY (Kong)          │
                            │   JWT · Rate-limit · Request-ID      │
                            └──────────┬──────────────────────────┘
                                       │
               ┌───────────────────────┼───────────────────────────┐
               │                       │                           │
   ┌───────────▼──────────┐ ┌─────────▼──────────┐ ┌──────────────▼──────────┐
   │   BORROWER CHANNELS  │ │  LENDER DASHBOARD   │ │   PARTNER SDK / MCP     │
   │  Voice · WhatsApp    │ │  React + TypeScript  │ │  pip install lending-sdk│
   │  Web · ONDC Partner  │ │  Portfolio · Policy  │ │  @tool · JourneyBuilder │
   └───────────┬──────────┘ │  Copilot · Monitor   │ └──────────────┬─────────┘
               │            └─────────┬──────────┘                  │
               └──────────────────────┼─────────────────────────────┘
                                      │
   ┌──────────────────────────────────▼──────────────────────────────────────┐
   │                    CROSS-CUTTING SERVICES (7)                           │
   │  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌────────────┐ ┌───────────┐ │
   │  │ auth-svc │ │audit-trail│ │compliance-│ │ ml-platform│ │observabil-│ │
   │  │ JWT+RBAC │ │ tamper-   │ │reporter   │ │ MLflow +   │ │ity stack  │ │
   │  │ RLS      │ │ evident   │ │ PSL+DPDP  │ │ Feast      │ │ Prom+Graf │ │
   │  └──────────┘ └──────────┘ └───────────┘ └────────────┘ └───────────┘ │
   │  ┌──────────────────┐  ┌──────────────────────┐                        │
   │  │ mcp-server (49+  │  │ api-gateway (Kong)    │                        │
   │  │ tools, JSON-RPC) │  │ routing + rate limit  │                        │
   │  └──────────────────┘  └──────────────────────┘                        │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
   ┌────────────────────────────────▼───────────────────────────────────────┐
   │                       CORE SERVICES (18)                               │
   │                                                                        │
   │  DATA LAYER (5)        DECISIONING (3)      DISTRIBUTION (4)           │
   │  ┌────────────────┐   ┌──────────────┐     ┌──────────────────┐       │
   │  │S1 uli-orchestr │   │S6 sentinel   │     │S9  prism-matcher │       │
   │  │S2 data-quality │   │S7 product-eng│     │S10 los-engine    │       │
   │  │S3 bankconnect  │   │S8 fraud-eng  │     │S11 journey-studio│       │
   │  │S4 deviceconnect│   └──────────────┘     │S12 notification  │       │
   │  │S5 identity-svc │                        └──────────────────┘       │
   │  └────────────────┘                                                    │
   │  INTELLIGENCE (6)                                                      │
   │  ┌────────────────────────────────────────────────────────────────┐    │
   │  │S13 agent-runtime    S14 voice-pipeline   S15 constellation     │    │
   │  │S16 knowledge-svc    S17 context-graph    S18 monitoring-agent  │    │
   │  └────────────────────────────────────────────────────────────────┘    │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
   ┌────────────────────────────────▼───────────────────────────────────────┐
   │                        INFRASTRUCTURE                                  │
   │  PostgreSQL 16 (RLS) │ Redis 7 Cluster │ Kafka (MSK, Avro)            │
   │  Neo4j Causal Cluster│ Temporal         │ ChromaDB / pgvector          │
   │  S3 (encrypted)      │ HashiCorp Vault  │ EKS (CPU + GPU node pools)  │
   │  Prometheus + Grafana + Loki + Jaeger   │ MLflow + Feast               │
   └────────────────────────────────────────────────────────────────────────┘
```

**Why microservices:** Each lending concern (data fetch, scoring, origination, monitoring) has different scaling profiles, failure modes, and deployment cadences. A circuit-breaker failure in the weather API must not cascade to loan disbursement. Sentinel policy can ship hourly while voice pipeline ships weekly.

**Contract layer:** `packages/platform-common` is the shared library installed as an editable dependency in every service. It enforces consistent auth, logging, encryption, and event schemas — the monorepo's immune system.

**Service discovery:** Environment variables per-pod via Helm ConfigMaps. Each service knows its neighbors by `SERVICE_NAME.platform.svc.cluster.local:PORT`. Istio service mesh handles mTLS, traffic splitting (canary), and retry policies.

---

# 2. LANGUAGE AND FRAMEWORK STACK

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend services** | Python 3.10+, FastAPI | Async-first, Pydantic for validation, OpenAPI auto-generation, ML ecosystem access |
| **Dashboard** | TypeScript, React, Vite | Type-safe lender UI. recharts for portfolio viz, TanStack Query for data fetching |
| **HTTP client** | httpx (async) | Replaces requests — async-native, HTTP/2, connection pooling for ULI parallel fetch |
| **ORM** | SQLAlchemy 2.0 async | Async session factory, RLS middleware, EncryptedString custom type for PII |
| **Migrations** | Alembic | Versioned schema migrations per mission. 6 migration files covering foundation → monitoring |
| **Cache / Pub-sub** | Redis 7 Cluster (6-node) | Session state, tool result cache, rate limiting, pub-sub for real-time dashboard updates |
| **Event streaming** | Kafka (MSK, 3-broker) | 22 topics, Avro schemas, exactly-once for audit events. 12-partition ULI topics for parallelism |
| **Graph database** | Neo4j Causal Cluster (3 core) | Context Lakehouse, fraud ring detection (multi-hop), decision traces, borrower relationship graph |
| **Workflow engine** | Temporal | Durable loan origination (survives pod restarts). Signal-based offer acceptance. 48h human referral timeout |
| **Vector store** | ChromaDB / pgvector | 7-partition knowledge base. Verbatim RBI text in dedicated compliance partition |
| **ML platform** | MLflow + Feast | Model versioning, champion/challenger promotion. Feast for real-time feature serving to Sentinel |
| **Containerization** | Docker + Kubernetes (EKS) | Base images: `Dockerfile.python`, `Dockerfile.node`. Helm charts with HPA, PDB, network policies |
| **IaC** | Terraform (modules) | 8 Terraform modules: EKS, RDS, ElastiCache, MSK, Neo4j, Temporal, S3, networking |
| **Security** | HashiCorp Vault, Trivy, bandit | Secrets injection, container scanning, static analysis. PII encryption keys in Vault |
| **Observability** | Prometheus + Grafana + Loki + Jaeger | Metrics, dashboards (5 standard), log aggregation, distributed tracing |
| **CI/CD** | GitHub Actions | 4 workflows: CI (lint+test+security), CD-staging (auto), CD-prod (manual gate), model-deploy |

---

# 3. SHARED TYPES ARCHITECTURE — `packages/platform-common`

```
packages/platform-common/src/platform_common/
├── auth/
│   ├── jwt.py               # Validate + generate HS256 JWTs
│   ├── rbac.py              # 6 roles: CRO, RISK_ANALYST, UNDERWRITER, OPS, COMPLIANCE, ADMIN
│   ├── api_key.py           # bcrypt-hashed partner API keys
│   └── middleware.py         # FastAPI middleware → sets RLS session variable
├── crypto/
│   ├── pii.py               # AES-256-GCM encrypt/decrypt, EncryptedString SQLAlchemy type
│   ├── hashing.py           # SHA-256 for Aadhaar/PAN dedup (never store raw)
│   └── consent.py           # DPDP Act consent token management
├── events/
│   ├── kafka.py             # Avro producer/consumer with schema registry
│   ├── schemas.py           # 22 event schemas (ULIFetchRequest, DecisionResult, etc.)
│   └── handlers.py          # Base event handler with dead-letter routing
├── logging/
│   ├── structured.py        # structlog JSON output, ISO timestamps
│   ├── tracing.py           # OpenTelemetry trace-context propagation
│   └── pii_filter.py        # Regex redaction: Aadhaar, PAN, phone, email → [REDACTED_*]
├── audit/
│   ├── trail.py             # Immutable append-only writer, SHA-256 hash chain
│   └── decorators.py        # @audited(entity_type, action) → Kafka audit.event
├── health/
│   ├── checks.py            # /healthz (liveness) + /readyz (readiness) probes
│   └── metrics.py           # Prometheus: request_count, latency_seconds, active_connections
├── db/
│   ├── postgres.py          # Async engine + session factory
│   ├── redis.py             # Async client + @cached decorator
│   ├── models.py            # Base model: id, lender_id, created_at, updated_at
│   └── rls.py               # SET app.current_lender_id on every session
├── config/
│   ├── loader.py            # YAML config with env variable override
│   ├── vault.py             # Vault client for secret rotation
│   └── segment.py           # 38-segment registry, typed access
├── errors/
│   ├── exceptions.py        # Hierarchy: PlatformError → AuthError, DataError, PolicyError...
│   └── handlers.py          # FastAPI exception → structured JSON response
└── testing/
    ├── fixtures.py          # Shared: test DB, Redis, Kafka mock
    ├── factories.py         # Factory-boy: borrower, loan, lender, decision
    └── assertions.py        # assert_pii_encrypted, assert_rls_enforced, assert_audit_logged
```

**Key design principle:** Every service imports `platform-common` as an editable dependency. Auth middleware, PII encryption, structured logging, and audit decorators are not optional — they are enforced at the library level. A new service gets compliance for free by inheriting the base patterns.

---

# 4. MODEL ABSTRACTION LAYER — CONSTELLATION (S15)

## Provider Interface

```python
class ModelProvider(Protocol):
    async def generate(self, messages: list[Message], **kwargs) -> ModelResponse: ...
    async def embed(self, texts: list[str]) -> list[list[float]]: ...
    def health_check(self) -> bool: ...
```

## 6-Task Routing Table

| Task Type | Primary Model | Fallback | Latency Target | Use Case |
|-----------|--------------|----------|----------------|----------|
| `classification` | gpt-4o-mini | claude-3-haiku | <200ms | Intent detection, segment classification |
| `reasoning` | gpt-4o | claude-3.5-sonnet | <2s | Credit decision explanation, policy evaluation |
| `extraction` | gpt-4o-mini | gemini-1.5-flash | <500ms | Entity extraction from voice transcript |
| `generation` | gpt-4o | claude-3.5-sonnet | <1s | Natural language response, KFS generation |
| `embedding` | text-embedding-3-small | voyage-3 | <100ms | Knowledge retrieval, semantic search |
| `vision` | gpt-4o-vision | gemini-1.5-pro | <3s | Bank statement OCR, document verification |

## Indian Language Model Routing

| Capability | Primary | Fallback | Languages |
|-----------|---------|----------|-----------|
| **ASR (Speech-to-Text)** | Deepgram Nova-2 (vocab boost) | Whisper large-v3 fine-tuned | gu, hi, mr, ta, te, kn, ml, bn, pa, or, as, ur |
| **TTS (Text-to-Speech)** | Bhashini (govt-backed) | ElevenLabs multilingual | Same 12 languages |
| **Translation** | IndicTrans2 | Google Translate API | Same 12 + en |

## Health Monitoring + Failover

```
Primary Model (healthy)  ──────────► Use primary
       │ (latency > 2x target OR error rate > 5%)
       ▼
Fallback Model (healthy) ──────────► Use fallback
       │ (fallback also degraded)
       ▼
Circuit Breaker OPEN     ──────────► Return cached response OR escalate to human
       │ (30s cooldown)
       ▼
Half-Open (1 test request)──────────► If success → close breaker → resume primary
```

---

# 5. TOOL EXECUTION FRAMEWORK

## 49+ MCP Tools Across 5 Lending Categories

| Category | Count | Key Tools |
|----------|-------|-----------|
| **Data** | 12 | `fetch_uli_data`, `verify_aadhaar`, `verify_pan`, `fetch_bank_statement`, `analyze_bank_statement`, `fetch_credit_bureau`, `fetch_device_score`, `fetch_cooperative_data`, `fetch_land_records`, `fetch_weather`, `fetch_satellite_ndvi`, `check_insurance` |
| **Decisioning** | 8 | `evaluate_credit`, `explain_decision`, `run_what_if`, `check_fraud`, `generate_kfs`, `compare_policies`, `fetch_model_performance`, `compute_shap` |
| **Distribution** | 7 | `match_lender`, `create_application`, `present_offer`, `execute_esign`, `initiate_disbursement`, `enroll_insurance`, `check_psl_classification` |
| **Monitoring** | 6 | `check_portfolio_health`, `get_early_warnings`, `trigger_moratorium`, `adjust_schedule`, `offer_graduation`, `generate_compliance_report` |
| **Agent** | 8 | `get_borrower_context`, `search_knowledge`, `log_interaction`, `escalate_to_human`, `detect_language`, `translate_text`, `fetch_journey_config`, `emit_rl_signal` |
| **Copilot** | 8 | `query_portfolio`, `run_what_if_scenario`, `explain_rejection`, `compare_segments`, `forecast_npa`, `audit_trail_search`, `bias_analysis`, `generate_report` |

## Session-State Gating

```
  UNVERIFIED ──────────► VERIFIED ──────────► CONFIRMED
  (browsing)             (KYC passed)         (consent granted)
      │                      │                      │
  Can use:               Can use:              Can use:
  - search_knowledge     - all UNVERIFIED +    - all VERIFIED +
  - detect_language      - fetch_uli_data      - initiate_disbursement
  - get_faq              - evaluate_credit     - execute_esign
  - check_eligibility    - match_lender        - trigger_moratorium
                         - present_offer       - adjust_schedule
                         - get_borrower_ctx    - offer_graduation
```

**Why gating matters:** A borrower who has not completed KYC must never trigger a data fetch (consent violation). A borrower who has not accepted the offer must never trigger disbursement. The session-state machine enforces this at the tool layer, not at the LLM prompt layer.

## Circuit Breaker Pattern

```python
@circuit(failure_threshold=5, recovery_timeout=30, expected_exception=ExternalServiceError)
async def call_external_service(client, params):
    """5 consecutive failures → OPEN (reject immediately) → 30s → HALF-OPEN (try 1) → CLOSED"""
    return await client.fetch(params)
```

Applied to: all ULI source fetches, credit bureau calls, disbursement APIs, weather/satellite services, cooperative APIs. Each external dependency has its own circuit breaker — a cooperative API outage does not prevent Aadhaar verification.

---

# 6. POLICY ENFORCEMENT ARCHITECTURE

## Three-Layer Model

```
┌──────────────────────────────────────────────────────────────────┐
│ LAYER 3: AUTHORITY (Hard Limits — Deterministic, No Override)    │
│                                                                  │
│  • Disbursement max: segment.product.amount_range.max            │
│  • Fee waiver max: lender.config.max_fee_waiver_pct              │
│  • Moratorium max months: policy.monitoring.max_moratorium        │
│  • Interest rate floor: rbi.config.min_rate                      │
│  • Single-borrower exposure limit                                │
│  Enforcement: numeric comparison, fail-closed, no LLM involved   │
├──────────────────────────────────────────────────────────────────┤
│ LAYER 2: BEHAVIORAL (LLM Supervisor — Soft Guardrails)           │
│                                                                  │
│  • RBI KFS verbatim: must include all 18 mandatory fields        │
│  • No financial advice: agent must not recommend specific loans   │
│  • Empathetic hardship: distress keywords → tone shift + escalate│
│  • Language appropriateness: no jargon for vernacular channels    │
│  • No data beyond consent: only access consented data elements   │
│  Enforcement: LLM classifier on agent response, fail-open        │
├──────────────────────────────────────────────────────────────────┤
│ LAYER 1: STRUCTURAL (Deterministic Gates — Binary Pass/Fail)     │
│                                                                  │
│  • Aadhaar gate: KYC must precede data fetch                     │
│  • Consent gate: DPDP consent token required per data element    │
│  • KYC gate: identity verified before credit evaluation          │
│  • Velocity gate: max 2 applications per Aadhaar per 48h         │
│  • Blacklist gate: check against RBI defaulter list              │
│  • Session-state gate: tool access restricted by auth level      │
│  Enforcement: if/else in code, fail-closed, no LLM involved     │
└──────────────────────────────────────────────────────────────────┘
```

**Design philosophy:** Compliance = fail-closed (Layers 1 & 3). A structural check failure immediately blocks the action. Tone/behavioral = fail-open (Layer 2). A borderline empathy score logs a warning but does not block the response. This prevents both regulatory violations and user-experience degradation.

---

# 7. KNOWLEDGE PIPELINE

## 7+ Knowledge Partitions

| Partition | Content | Update Cadence | Retrieval Mode |
|-----------|---------|---------------|----------------|
| `rates` | Interest rates, fee schedules per lender per segment | Daily | Exact match by lender_id + segment |
| `products` | Product specs, eligibility criteria, schedule types | Weekly | Semantic search |
| `compliance_rbi` | RBI Digital Lending guidelines, CLA 2026, PSL rules | On regulatory change | **Verbatim** (no paraphrase) |
| `segment_configs` | 38 segment YAML configs (dairy, vendor, gig, etc.) | On config change | Exact match by segment_id |
| `loss_mitigation` | Hardship options, moratorium triggers, restructuring rules | Monthly | Semantic search |
| `uli_docs` | ULI API specs, 136 service catalog, error codes | On API change | Semantic search |
| `faq` | Borrower FAQs per language per segment | Weekly | Semantic search (multilingual) |
| `dpdp_compliance` | DPDP Act requirements, consent templates, data retention | On regulatory change | **Verbatim** |

## Ingestion Pipeline

```
Source Document (Markdown/PDF/YAML)
       │
       ▼
  Chunk (512-token window, 50-token overlap)
       │
       ▼
  Embed (text-embedding-3-small, 1536 dimensions)
       │
       ▼
  Store in Partition (ChromaDB collection per partition)
       │
       ▼
  Metadata: {partition, source_file, version, language, last_updated}
```

## Retrieval: Auto-Partition Routing

```python
async def retrieve(query: str, context: TurnContext) -> list[KnowledgeChunk]:
    # 1. Classify query to determine partition(s)
    partitions = await classify_partition(query)  # e.g., ["compliance_rbi", "products"]

    # 2. If compliance partition → use verbatim retrieval (no reranking/rewriting)
    # 3. For all other partitions → semantic search with MMR diversification
    # 4. Merge results, deduplicate, rank by relevance * recency
```

**Verbatim rule:** For RBI regulatory text and DPDP Act requirements, the agent must quote the exact source text. The knowledge service marks these chunks as `verbatim=true`, and the agent runtime inserts them into the response without paraphrase. This prevents hallucinated compliance language.

---

# 8. OBSERVABILITY STACK

## Trace Model

```
Session (borrower phone call or chat)
  └── Turn (single user message + agent response)
        ├── Span: language_detection      [12ms]
        ├── Span: asr_transcription       [340ms]
        ├── Span: intent_classification   [89ms]
        ├── Span: knowledge_retrieval     [156ms]  ← partition: compliance_rbi
        ├── Span: model_call              [890ms]  ← model: gpt-4o, tokens: 1240
        ├── Span: tool_execution          [2100ms] ← tool: fetch_uli_data
        │     ├── Sub-span: cibil_fetch   [980ms]
        │     ├── Sub-span: aa_fetch      [1800ms]
        │     └── Sub-span: coop_fetch    [650ms]  ← circuit: CLOSED
        ├── Span: policy_check            [23ms]   ← layer: structural, result: PASS
        ├── Span: uli_fetch               [2100ms] ← sources: 5, success: 4, timeout: 1
        └── Span: tts_synthesis           [210ms]  ← language: gu, provider: bhashini
```

## Prometheus Metrics (Key Subset)

```
# Per-service standard
request_total{service, method, status}
request_duration_seconds{service, method, quantile}
active_connections{service}

# Lending-specific
loan_applications_total{segment, channel, status}
decision_latency_seconds{segment, policy_version, quantile}
uli_fetch_duration_seconds{source, status, quantile}
uli_source_availability{source}
disbursement_total{lender, status}
agent_turn_duration_seconds{channel, language, quantile}
asr_word_error_rate{language, provider}
policy_violations_total{layer, rule}
knowledge_retrieval_latency_seconds{partition, quantile}
monitoring_triggers_total{trigger_type, severity}
```

## Grafana Dashboards (5 Standard)

| Dashboard | Key Panels |
|-----------|------------|
| **Platform Health** | Service uptime, API latency heatmap, Kafka consumer lag, Redis hit rate, pod resource usage |
| **Loan Lifecycle** | Applications funnel (created → KYC → decision → offer → disburse), conversion by segment, avg time-to-disburse |
| **Agent Performance** | Calls/hr, avg call duration, completion rate, human escalation %, ASR confidence distribution |
| **Per-Lender** | Lender-scoped: AUM, DPD buckets, NPA %, approval rate, PSL classification, disbursement volume |
| **Data Quality** | ULI source latency per-provider, circuit breaker state map, data confidence distribution, entity resolution match rate |

---

# 9. EVALUATION ARCHITECTURE

## tau-bench Style Scenarios for Lending

The evaluation framework runs LLM-as-borrower against the actual agent, scoring task completion, policy compliance, accuracy, empathy, latency, and language quality.

## User Simulator

```python
class BorrowerSimulator:
    """LLM plays a borrower with a specific persona and goal."""

    async def generate_utterance(self, scenario: Scenario, history: list[Turn]) -> str:
        return await self.llm.generate(
            system=f"You are {scenario.persona}. Goal: {scenario.goal}. "
                   f"Language: {scenario.language}. Behavior: {scenario.behavior}.",
            messages=history,
        )
```

## State-Based Scoring

Each scenario defines expected state transitions. The scorer verifies that the loan application reached the expected terminal state (APPROVED, REJECTED, MORATORIUM, etc.) and that all intermediate states were visited.

## pass^k Metric

Run each scenario `k` times (typically k=3). `pass^k` = probability of passing at least once. This accounts for LLM non-determinism — an agent that passes 2/3 runs is still production-viable.

## 8+ Lending Scenarios

| Scenario | Persona | Language | Goal | Key Verification |
|----------|---------|----------|------|-----------------|
| Dairy happy path | Ramesh, Gujarat dairy farmer | gu | Apply for ₹50K animal purchase loan | Full lifecycle → ACTIVE |
| Weather moratorium | Active dairy borrower, heat wave | gu | Receive auto-moratorium notification | sub_status = MORATORIUM, schedule recalculated |
| Vendor working capital | Lakshmi, Chennai street vendor | ta | ₹15K working capital via WhatsApp | WhatsApp channel, correct segment routing |
| Gig emergency | Rajesh, Mumbai delivery driver | hi | Emergency ₹5K loan, low balance | Fast-track decisioning, <5min disbursement |
| Fraud ring detection | Synthetic borrower, shared references | hi | Application should be flagged | fraud_flags includes SHARED_REFERENCE |
| KYC failure | Invalid Aadhaar | hi | Agent handles gracefully | status = KYC_FAILED, empathetic response |
| Returning borrower | Previous loan fully repaid | gu | Apply for graduation loan | Graph memory loads context, pre-approved offer |
| Hardship escalation | Distressed borrower, crop failure | hi | Agent detects distress, escalates | Human escalation triggered, empathy score > 0.8 |

---

# 10. VOICE ARCHITECTURE

## Pipeline

```
Phone (PSTN/VoIP)
  │
  ▼
Exotel/Twilio Telephony  ─── WebSocket ───►  Voice Pipeline (S14)
  │                                              │
  ▼                                              ▼
Inbound Audio Stream                        VAD (energy-based)
                                                 │
                                                 ▼
                                            STT (ASR)
                                            ├─ Primary: Deepgram Nova-2 + vocab boost
                                            └─ Fallback: Whisper large-v3 fine-tuned
                                                 │
                                                 ▼
                                          Agent Runtime (S13)
                                          ├─ Fast Brain (cached FAQ: <200ms)
                                          └─ Slow Brain (MCP tool call: <2s)
                                                 │
                                                 ▼
                                            TTS (Synthesis)
                                            ├─ Primary: Bhashini (govt Indian TTS)
                                            └─ Fallback: ElevenLabs multilingual
                                                 │
                                                 ▼
                                          Outbound Audio Stream ──► Phone
```

## Fast Brain / Slow Brain

**Fast brain** runs in parallel with slow brain. If the query matches a cached FAQ or simple intent (loan status, next EMI date), the fast brain responds in <200ms — the slow brain result is discarded. For complex queries (new application, credit decision), the slow brain result is used.

## Indian Language ASR/TTS

- **12 languages:** Gujarati, Hindi, Marathi, Tamil, Telugu, Kannada, Malayalam, Bengali, Punjabi, Odia, Assamese, Urdu
- **Vocab boost:** Per-language lending keyword list injected into ASR model. Examples for Gujarati: `["લોન", "EMI", "ભેંસ", "સહકારી", "લણણી", "ચોમાસું"]`
- **Pronunciation engine for lending terms:** EMI → "ee-em-ai", FOIR → "foy-ar", CIBIL → "sih-bil", NACH → "naach", KFS → "key fact statement"

## Telephony Integration

| Provider | Use Case | Integration |
|----------|---------|-------------|
| Exotel | Primary PSTN (India toll-free, local DIDs) | WebSocket streaming, call recording |
| Twilio | Fallback / international | SIP trunking, TwiML for IVR fallback |

---

# 11. CONFIGURATION-AS-CODE — WORKSPACE SERVICE (S11)

## Everything in YAML

```
configs/
├── segments/              # 38 borrower segment definitions
│   ├── dairy_farmer.yaml  #   data_sources, scoring, product, monitoring, voice
│   ├── street_vendor.yaml
│   └── ...
├── policies/              # Sentinel policy templates per segment per lender
│   ├── dairy_animal_purchase_v1.yaml
│   └── vendor_working_capital_v1.yaml
├── journeys/              # Agent conversation flow definitions
│   ├── dairy_voice_gujarati.json
│   └── vendor_whatsapp_hindi.json
├── lenders/               # Lender Prism profiles
│   └── example_sfb.yaml   #   segments, geography, risk_appetite, PSL targets
└── compliance/            # Regulatory config
    ├── rbi_digital_lending.yaml
    ├── dpdp_act.yaml
    └── psl_classification.yaml
```

## Promotion Pipeline

```
  DRAFT ──► QA ──► STAGING ──► PRODUCTION
    │         │        │            │
    │         │        │            └── Live traffic. Rollback within 60s.
    │         │        └── Shadow test against historical data.
    │         └── Automated eval suite (pass^k > 0.85 required).
    └── Author edits. Version bumped. Diff generated.
```

## Diff Engine

Every config change produces a human-readable diff:

```diff
# policies/dairy_animal_purchase_v2.yaml vs v1
- scoring.grade_bands.A1.min_score: 750
+ scoring.grade_bands.A1.min_score: 720
- decision_rules.auto_approve.min_confidence: 0.85
+ decision_rules.auto_approve.min_confidence: 0.80
```

CRO reviews the diff before promotion to production. Rollback reverts to the previous version hash — instant, no redeploy.

---

# 12. SDK DESIGN

## pip-Installable Python SDK

```bash
pip install lending-platform-sdk
```

## Key Decorators and Builders

```python
from lending_sdk import tool, JourneyBuilder, ScenarioBuilder

# --- @tool decorator: register a new lending tool ---
@tool(
    name="check_cattle_health",
    description="Query veterinary database for livestock health records",
    category="data",
    session_state_required="VERIFIED",
)
async def check_cattle_health(borrower_id: str, animal_tag: str) -> dict:
    """Custom tool added by lender's engineering team."""
    records = await vet_api.query(borrower_id, animal_tag)
    return {"health_status": records.status, "last_checkup": records.date}


# --- JourneyBuilder: define conversation flows ---
journey = (
    JourneyBuilder("fpo_member_onboarding")
    .set_segment("FPO_MEMBER")
    .set_channel("WHATSAPP")
    .set_language("hi")
    .add_step("greeting", template="नमस्ते {name}, FPO सदस्य ऋण सेवा में आपका स्वागत है")
    .add_step("kyc", tool="verify_aadhaar", required=True)
    .add_step("data_fetch", tool="fetch_uli_data", parallel=True)
    .add_step("decision", tool="evaluate_credit")
    .add_step("offer", tool="present_offer", wait_for_acceptance=True)
    .add_step("disbursement", tool="initiate_disbursement", session_state="CONFIRMED")
    .build()
)


# --- ScenarioBuilder: define eval scenarios ---
scenario = (
    ScenarioBuilder("fpo_member_happy_path")
    .set_persona("Suresh, FPO member, 3 years, Hindi-speaking")
    .set_goal("Apply for ₹2L group loan for seed purchase")
    .set_language("hi")
    .expect_state_transition("CREATED → KYC_CHECK → DATA_FETCH → APPROVED → ACTIVE")
    .expect_tool_calls(["verify_aadhaar", "fetch_uli_data", "evaluate_credit", "initiate_disbursement"])
    .set_max_turns(15)
    .build()
)
```

## SDK Capabilities

| Feature | Description |
|---------|-------------|
| `@tool` decorator | Register custom MCP tools with type hints, session-state gating, and circuit-breaker config |
| `JourneyBuilder` | Fluent API for defining segment-specific conversation flows without YAML |
| `ScenarioBuilder` | Fluent API for evaluation scenarios — used in CI/CD to gate deployments |
| `PlatformClient` | Authenticated client for all platform APIs (decision, data, LOS, monitoring) |
| `WebhookHandler` | Receive Kafka events via HTTP webhooks for external system integration |

---

# APPENDIX: SERVICE PORT MAP

| Port | Service | Protocol |
|------|---------|----------|
| 8000 | api-gateway | HTTP |
| 8001 | constellation | HTTP |
| 8002 | mcp-server | JSON-RPC 2.0 |
| 8003 | sentinel-core | HTTP |
| 8004 | knowledge-svc | HTTP |
| 8005 | agent-runtime | HTTP + WebSocket |
| 8006 | trace-service | HTTP |
| 8007 | voice-pipeline | HTTP + WebSocket |
| 8008 | context-graph | HTTP |
| 8009 | workspace-service | HTTP |
| 8010 | los-engine | HTTP |
| 8011 | evaluation-svc | HTTP |
| 8012 | monitoring-agent | HTTP |
| 3000 | dashboard | HTTP |
| 5432 | PostgreSQL | TCP |
| 6379 | Redis Cluster | TCP |
| 9092 | Kafka (MSK) | TCP |
| 7687 | Neo4j Bolt | TCP |
| 7233 | Temporal | gRPC |
