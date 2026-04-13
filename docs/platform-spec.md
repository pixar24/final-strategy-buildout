# Unified AI-First Lending Platform: Technical Specification
## 18 Core Services + 7 Cross-Cutting Services — Multi-Agent Orchestration for India's 300M Underserved

---

## 1. Platform Overview

### 1.1 Dimensions Summary

| Dimension | Specification |
|-----------|--------------|
| **Core Services** | 18 microservices across 4 layers (Data, Decision, Distribution, Intelligence) |
| **Cross-Cutting Services** | 7 shared infrastructure services (gateway, auth, audit, compliance, ML, observability, MCP) |
| **Multi-Tenancy** | `lender_id` on every table + PostgreSQL Row-Level Security (RLS); zero cross-tenant data leakage |
| **Multi-Channel** | Voice (Exotel PSTN/VoIP), WhatsApp (BSP), Web (React), Partner SDK (ONDC/gig platforms) |
| **Multi-Model** | Constellation router: 6 task types → model selection with health monitoring and automatic failover |
| **Borrower Segments** | 38 YAML-configured segments (agriculture, micro-enterprise, gig, salaried NTC, asset-backed, SHG) |
| **Languages** | 12 Indian languages: Hindi, Gujarati, Marathi, Tamil, Telugu, Kannada, Bengali, Malayalam, Punjabi, Odia, Assamese, English |
| **DPI Integration** | ULI (136 services), Account Aggregator, Aadhaar eKYC, DigiLocker, eSign, eNACH, GSTN, EPFO |
| **MCP Tools** | 49+ unified tools via JSON-RPC 2.0 across data, decisioning, distribution, monitoring, and agent categories |
| **Compliance** | RBI Digital Lending Guidelines, RBI CLA (Jan 2026), DPDP Act 2023, PSL classification, FREE-AI framework |

### 1.2 Layered Architecture

```
BORROWER CHANNELS                PLATFORM CORE                    EXTERNAL SYSTEMS
┌─────────────────┐    ┌──────────────────────────────┐    ┌──────────────────────┐
│ Voice (PSTN)    │───▶│  INTELLIGENCE LAYER (S13-S18) │◀──▶│ DPI / ULI (136 svc)  │
│ WhatsApp (BSP)  │───▶│  agent-runtime, voice-pipeline│    │ Account Aggregator   │
│ Web (React)     │───▶│  constellation, knowledge-svc │    │ Credit Bureaus       │
│ Partner SDK     │───▶│  context-graph, monitoring     │    │ UIDAI / eKYC         │
│ Lender Dashboard│───▶│                                │    │ NPCI (UPI/NEFT)      │
└─────────────────┘    │  DISTRIBUTION LAYER (S9-S12)   │    │ GSTN / EPFO          │
                       │  prism, los, journey, notify    │    │ Cooperatives / Mandi │
                       │                                │    │ Gig Platforms        │
                       │  DECISION LAYER (S6-S8)         │    │ Satellite / Weather  │
                       │  sentinel, product, fraud       │    └──────────────────────┘
                       │                                │
                       │  DATA LAYER (S1-S5)             │    INFRASTRUCTURE
                       │  uli, dq, bankconnect,          │    ┌──────────────────────┐
                       │  deviceconnect, identity         │    │ K8s HA, PostgreSQL   │
                       └──────────────────────────────────┘    │ Redis, Kafka, Neo4j  │
                                                               │ Temporal, Vault      │
                       CROSS-CUTTING (7 services)              │ S3/MinIO, MLflow     │
                       api-gateway, auth-svc, audit-trail      │ Prometheus+Grafana   │
                       compliance-reporter, ml-platform        └──────────────────────┘
                       observability, mcp-server
```

---

## 2. Service Specifications

---

### S1 — `uli-orchestrator` (port 9001)

**Role:** Single entry-point for parallel ULI/DPI data fetch. Maps borrower segment to required data sources, manages consent validation, executes parallel HTTP/2 calls with per-source circuit breakers, and assembles enriched profiles with graceful degradation when sources are unavailable.

**Dependencies:** identity-svc (consent tokens), data-quality-engine (downstream enrichment), Kafka (`uli.fetch.*`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/data/fetch` | Initiate parallel data fetch for a borrower; accepts segment hint, consent tokens, idempotency key |
| GET | `/v1/data/fetch/{request_id}` | Poll status of an in-flight or completed fetch request |
| GET | `/v1/data/sources/{segment}` | Return required + optional sources for a given segment |
| POST | `/v1/data/fetch/{request_id}/retry` | Retry failed sources for a partial fetch |
| GET | `/v1/health` | Liveness and readiness probe with per-source circuit breaker status |

**Data Models:**
- `DataFetchRequest`: borrower_id (UUID), aadhaar_token (encrypted), aa_consent_handle, segment_hint (enum of 38), lender_id (UUID), request_id (UUID idempotency key)
- `SourceResult`: source (string), status (SUCCESS | TIMEOUT | FAILED | SKIPPED), latency_ms (int), data_ref (string nullable)
- `DataFetchResponse`: request_id, status (COMPLETE | PARTIAL | FAILED), sources_fetched (list[SourceResult]), confidence_score (float 0–1), data_quality_flags (list[str]), enriched_profile (dict)

**Decision Rationale:** A dedicated orchestrator isolates the complexity of parallel DPI calls — each with different timeout profiles, auth mechanisms, and failure modes — from downstream services. Segment-aware source mapping (e.g., DAIRY_FARMER requires cooperative + land records + satellite; GIG_WORKER requires platform API + UPI history) prevents unnecessary calls and keeps p95 fetch latency under 5 seconds. Circuit breakers (open after 5 consecutive failures, half-open after 30s) prevent cascading failures when a ULI source degrades. Graceful degradation — reducing max approved amount and flagging for human review rather than hard-rejecting — ensures thin-file borrowers are never excluded solely due to data infrastructure issues.

---

### S2 — `data-quality-engine` (port 9002)

**Role:** Entity resolution across heterogeneous ULI sources, cross-source validation, freshness scoring, anomaly detection, and composite confidence scoring. Transforms raw multi-source data into a unified enriched borrower profile.

**Dependencies:** uli-orchestrator (upstream), Kafka (`dq.enriched`), context-graph (Active Ontology)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/quality/enrich` | Consume raw source results, perform entity resolution + cross-validation, return enriched profile |
| GET | `/v1/quality/profile/{borrower_id}` | Retrieve latest enriched profile with confidence breakdown |
| POST | `/v1/quality/validate` | Run cross-source validation on specific field pairs (e.g., income: bank vs. cooperative) |
| GET | `/v1/quality/freshness/{borrower_id}` | Return per-source data freshness scores |

**Data Models:**
- `EnrichedProfile`: borrower_id, confidence_score (float), freshness_scores (dict[source, float]), cross_validation_results (list), anomaly_flags (list[str]), resolved_entity (dict)
- Freshness enum: FRESH (<30d, 1.0), RECENT (30–90d, 0.8), STALE (90–180d, 0.5), VERY_STALE (>180d, 0.2)
- Confidence formula: `0.3 * (sources_fetched / sources_required) + 0.3 * avg_freshness + 0.2 * cross_validation_pass_rate + 0.2 * (1 - anomaly_count / total_checks)`

**Decision Rationale:** Entity resolution is the first defense against duplicate applications and identity fraud. Jaro-Winkler (threshold >0.85) handles the name-matching problem across Indian transliterations (e.g., cooperative ledger in Gujarati vs. bank records in English). Cross-source validation (e.g., declared income vs. bank deposits, cooperative production records vs. claimed livestock count) catches both honest errors and deliberate inflation. The confidence score drives graceful degradation in Sentinel: low confidence → reduced loan amount + higher rate + human review flag, rather than rejection. This is philosophically critical — the platform serves thin-file borrowers who may legitimately have fewer verifiable data points.

---

### S3 — `bankconnect-svc` (port 9003)

**Role:** Bank statement parsing (PDF/OCR), transaction categorization into 15 categories, income stability analysis, FinBox Credit Score (FBCS 0–900) computation, and cluster intelligence fraud detection (circular transfers, round-trip, income inflation).

**Dependencies:** uli-orchestrator (source input), fraud-engine (fraud flag forwarding), Kafka (`bc.analyzed`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/bankconnect/analyze` | Submit bank statement (PDF/image ref) for full analysis pipeline |
| GET | `/v1/bankconnect/result/{analysis_id}` | Retrieve completed analysis with FBCS, income, fraud flags |
| POST | `/v1/bankconnect/categorize` | Categorize raw transactions without full scoring (utility endpoint) |
| GET | `/v1/bankconnect/cluster/{borrower_id}` | Retrieve cluster intelligence results for a borrower |

**Data Models:**
- `BankConnectResult`: fbcs_score (int 0–900), monthly_income (list[{month, amount, source_breakdown}]), income_stability (float stddev/mean), fraud_flags (list[str]), foir (float), transaction_summary (dict), cluster_analysis (dict)
- 15 transaction categories: SALARY, UPI_CREDIT, UPI_DEBIT, CASH_DEPOSIT, CASH_WITHDRAWAL, LOAN_EMI, COOPERATIVE_CREDIT, PLATFORM_PAYOUT, RENT, UTILITY, TRANSFER_IN, TRANSFER_OUT, BOUNCE, INTEREST, OTHER
- Fraud flag enum: CIRCULAR_TRANSFER, ROUND_TRIP, INCOME_INFLATION, CASH_DEPOSIT_PATTERN, BOUNCE_FREQUENCY

**Decision Rationale:** BankConnect is the single richest data source for underserved borrowers — a 6-month bank statement reveals income patterns, expenditure discipline, existing obligations, and fraud signals that no bureau score can capture. The 15-category taxonomy was chosen because lending-specific categories (COOPERATIVE_CREDIT, PLATFORM_PAYOUT) are essential for segment scoring but missing from generic transaction classifiers. Cluster intelligence (graph-based detection of circular transfers within 48h windows) catches sophisticated income inflation schemes that per-transaction analysis misses. FBCS composite scoring (0–900) provides a bureau-equivalent signal for thin-file borrowers.

---

### S4 — `deviceconnect-svc` (port 9004)

**Role:** Android/iOS SDK data collection (consent-gated), behavioral feature extraction (5,000+ raw features → scoring features), FinBox Inclusion Score (FIS 0–1,000) computation, and device-level fraud detection.

**Dependencies:** identity-svc (consent validation), Kafka (`dc.scored`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/device/score` | Submit device telemetry payload, return FIS score + fraud flags |
| GET | `/v1/device/score/{borrower_id}` | Retrieve latest device score for a borrower |
| POST | `/v1/device/consent/revoke` | Revoke consent; triggers raw data deletion within 24h |
| GET | `/v1/device/features/{borrower_id}` | Return extracted feature vector (not raw data) |

**Data Models:**
- `DeviceScoreResult`: fis_score (int 0–1000), feature_vector (dict), fraud_flags (list[str]), consent_token (str), scored_at (datetime)
- Feature categories: app_usage (financial app presence, frequency), location_stability (home/work pattern), behavioral (typing speed, screen time patterns), network (WiFi vs mobile, SIM changes)
- Privacy invariant: raw device data deleted after feature extraction; only feature vectors and scores persist

**Decision Rationale:** Device behavioral data provides an inclusion signal for borrowers with no bureau history — stable location patterns, financial app usage, and consistent device behavior correlate with repayment discipline. The 5,000+ raw features are reduced to scoring features via XGBoost, preventing overfitting while capturing signal. Consent-gating is non-negotiable: DPDP Act requires explicit per-purpose consent, and raw data deletion post-extraction respects data minimization principles. The FIS 0–1,000 range (wider than FBCS's 0–900) provides finer granularity for the inclusion scoring use case.

---

### S5 — `identity-svc` (port 9005)

**Role:** Aadhaar eKYC (OTP + biometric modes), PAN verification + Aadhaar-PAN seeding, DigiLocker document fetch, Account Aggregator consent lifecycle management, and unified KYC pipeline orchestration.

**Dependencies:** api-gateway (inbound), auth-svc (JWT), consent ledger (compliance-reporter), Kafka (`id.verified`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/identity/aadhaar/otp/initiate` | Send OTP to Aadhaar-linked mobile |
| POST | `/v1/identity/aadhaar/otp/verify` | Verify OTP and fetch eKYC data |
| POST | `/v1/identity/pan/verify` | Verify PAN and check Aadhaar-PAN link |
| POST | `/v1/identity/digilocker/fetch` | Fetch documents from DigiLocker (income proof, PMFBY) |
| POST | `/v1/identity/aa/consent` | Initiate Account Aggregator consent flow |
| GET | `/v1/identity/aa/consent/{handle}/status` | Poll AA consent status |
| POST | `/v1/identity/aa/fetch` | Fetch financial data post-consent |

**Data Models:**
- `IdentityVerification`: borrower_id, verification_type (AADHAAR_OTP | AADHAAR_BIOMETRIC | PAN | DIGILOCKER | AA_CONSENT | VIDEO_KYC), status (PENDING | SUCCESS | FAILED | EXPIRED), result_json (encrypted), consent_id (FK), verified_at, expires_at
- `AAConsentRequest`: borrower_id, fi_types (list: DEPOSIT, MUTUAL_FUNDS, INSURANCE, GST), purpose (CREDIT_SCORING | KYC | MONITORING), data_range_months (int)

**Decision Rationale:** Identity is the mandatory first step in every lending journey — RBI Digital Lending Guidelines require KYC completion before any data fetch or credit decision. Aadhaar eKYC via OTP mode enables fully digital verification for borrowers without smartphones (via USSD/IVR fallback), while biometric mode handles assisted channel scenarios. Account Aggregator consent management is separated from data fetch because consent can expire or be revoked independently. The consent lifecycle (grant → active → expired/revoked) maps directly to the DPDP Act's purpose-limitation requirements, making compliance structural rather than procedural.

---

### S6 — `sentinel-core` (port 9006)

**Role:** Unified 3-layer policy engine (structural rules → ML scoring → supervisor override). Evaluates credit decisions against versioned YAML policy definitions per segment per lender. Supports shadow mode (parallel evaluation without state mutation), canary deployment (graduated traffic split), and champion/challenger analytics.

**Dependencies:** data-quality-engine (enriched profiles), ml-platform (model registry), fraud-engine (fraud signals), Kafka (`sentinel.decision`), context-graph (Decision Traces)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/decision/evaluate` | Run full 3-layer policy evaluation for a borrower application |
| POST | `/v1/decision/what-if` | Simulate policy changes against historical portfolio data |
| GET | `/v1/decision/{decision_id}` | Retrieve decision with full explainability (SHAP, reason codes) |
| POST | `/v1/policy/create` | Create new policy version (DRAFT status) |
| PUT | `/v1/policy/{id}/promote` | Promote policy: DRAFT → SHADOW → CANARY → PRODUCTION (CRO approval required) |
| GET | `/v1/policy/{id}/shadow-results` | Compare shadow policy results against production |
| GET | `/v1/decision/kfs/{application_id}` | Generate RBI-compliant Key Fact Statement |

**Data Models:**
- `PolicyDefinition` (YAML): policy_id, version, status (DRAFT | SHADOW | CANARY | PRODUCTION | RETIRED), segment, scoring (model_id, features with sources/weights), decision_rules (auto_approve thresholds, refer criteria, auto_reject triggers), pricing (base_rate, grade adjustments), product (type, amount_range, insurance, moratorium_triggers, graduation criteria)
- `CreditDecision`: borrower_id, policy_id, policy_version, is_shadow (bool), raw_score (0–900), grade (A1/A2/B1/B2), feature_values, feature_weights, decision (APPROVE | REFER | REJECT), reason_codes, shap_values, proposed_amount/rate/tenure, fraud_flags, fraud_score, data_confidence
- MCP tools: `trigger_credit_decision`, `run_what_if`, `fetch_model_performance`, `detect_drift`, `explain_decision`, `generate_kfs`

**Decision Rationale:** The 3-layer architecture — structural checks first (hard rules: velocity limits, blacklists, regulatory gates), ML scoring second (segment-specific models), supervisor override third (anomaly detection on the decision itself) — ensures that compliance-critical rules can never be overridden by ML outputs. YAML policy definitions make policies auditable by non-engineers (regulators, CROs) and enable version-controlled policy changes with full diff history. Shadow/canary deployment is non-negotiable for production lending: a new policy must prove itself on real traffic before any borrower is affected. Champion/challenger analytics (approval rate, score distribution, projected NPA) give CROs data-driven confidence to promote or retire policies.

---

### S7 — `product-engine` (port 9007)

**Role:** Dynamic repayment schedule generation from 8 specialized generators, each designed for a specific cash-flow pattern. Produces schedules where sum(EMIs) = principal + interest, respects moratorium triggers, and supports mid-term schedule adjustments.

**Dependencies:** sentinel-core (approved decisions with grade/amount), monitoring-agent (moratorium triggers), Kafka (`product.designed`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/product/design` | Generate repayment schedule for approved application |
| POST | `/v1/product/redesign` | Recalculate schedule after moratorium or adjustment |
| GET | `/v1/product/schedule/{loan_id}` | Retrieve current repayment schedule with status per installment |
| GET | `/v1/product/types/{segment}` | List eligible product types for a segment |

**Data Models:**
- 8 schedule generators:

| Generator | Logic | Target Segment |
|-----------|-------|----------------|
| SEASONAL_DYNAMIC | EMI scaled by flush/lean/transition months from cooperative calendar | Dairy farmer, crop farmer |
| DAILY_MICRO | Daily micro-deductions adjusted for earning pattern; skip if below threshold | Street vendor |
| CATCH_SALE_LINKED | Repayment triggered by mandi/warehouse sale event | FPO, warehouse receipt |
| INCOME_CONTINGENT | Monthly = verified_income × share_rate (EPFO/platform verified) | Gig worker, salaried NTC |
| MORATORIUM_WITH_TRIGGER | Standard EMI + auto-pause on weather/health event triggers | All agriculture segments |
| BRIDGE_ZERO_INTEREST | Principal-only repayment, auto-deduct on DBT/insurance arrival | Emergency, bridge loans |
| STAGED_DISBURSEMENT | Tranche at milestones (e.g., medical: 30/40/30 at admission/procedure/discharge) | Medical, asset purchase |
| PAYOUT_LINKED | Auto-deduct percentage from platform settlement (e.g., 15% of Swiggy payout) | Gig worker delivery/ridehail |

- `Installment`: installment_no, due_date, principal, interest, total_emi, season_tag (FLUSH | LEAN | TRANSITION), status (SCHEDULED | PAID | PARTIAL | OVERDUE | MORATORIUM | SKIPPED), paid_amount, paid_at, moratorium_reason

**Decision Rationale:** One-size-fits-all EMIs fail underserved borrowers whose income is seasonal, irregular, or event-driven. A dairy farmer earns 3× more during flush season (Oct–Feb) than lean season (Apr–Jun); forcing equal EMIs causes lean-season defaults that have nothing to do with creditworthiness. The 8 generators address 8 distinct cash-flow archetypes observed across the 38 segments. Moratorium triggers (weather, health, production drop) are built into schedules rather than being post-hoc restructuring — this reduces NPA by preventing defaults before they happen. The invariant sum(EMIs) = total_repayment is validated at generation time to prevent rounding drift.

---

### S8 — `fraud-engine` (port 9008)

**Role:** Multi-signal fraud detection combining velocity checks, BankConnect cluster analysis, and Neo4j graph-based ring detection. Outputs fraud flags and composite fraud scores consumed by Sentinel for credit decisions.

**Dependencies:** bankconnect-svc (cluster analysis), context-graph (Neo4j for ring detection), Kafka (`fraud.alert`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/fraud/check` | Run full fraud check pipeline for an application |
| GET | `/v1/fraud/result/{check_id}` | Retrieve fraud check result with flags and scores |
| POST | `/v1/fraud/ring-detect` | Run graph-based fraud ring detection for a borrower |
| GET | `/v1/fraud/velocity/{aadhaar_hash}` | Check application velocity for an identity |

**Data Models:**
- `FraudCheckResult`: borrower_id, fraud_score (float 0–1), flags (list[str]), velocity_result (dict), cluster_result (dict), ring_result (dict), is_hard_reject (bool)
- Velocity rules: max 2 applications per Aadhaar in 48h; max 5 per phone in 7d; max 1 active loan per lender per borrower
- Ring detection: Neo4j multi-hop query (shared reference persons → shared devices → shared bank accounts → shared addresses within 2-hop distance)

**Decision Rationale:** Single-signal fraud detection is trivially evadable. Velocity checks catch volume attacks (same identity applying across lenders). BankConnect cluster analysis catches financial fraud (income inflation, circular transfers). Graph-based ring detection catches organized fraud (syndicates sharing reference persons, devices, or addresses). By combining three independent detection mechanisms, the system requires attackers to evade all three simultaneously — which is orders of magnitude harder. Neo4j's native graph traversal makes multi-hop ring detection feasible at the latency required for real-time decisioning (<200ms for 3-hop queries).

---

### S9 — `prism-matcher` (port 9009)

**Role:** Multi-lender matching engine that optimizes across PSL (Priority Sector Lending) gap, risk appetite, cost to borrower, and capital availability. Supports co-lending structures (RBI CLA compliant 80/20 split) when ticket size warrants it.

**Dependencies:** sentinel-core (approved decisions), los-engine (downstream origination), compliance-reporter (PSL classification), Kafka (`prism.matched`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/prism/match` | Match approved application to optimal lender(s) |
| GET | `/v1/prism/match/{match_id}` | Retrieve match result with scoring breakdown |
| POST | `/v1/prism/co-lend/structure` | Build RBI CLA compliant co-lending structure |
| GET | `/v1/prism/lenders/{segment}` | List active lenders for a segment with PSL gap and capacity |
| PUT | `/v1/prism/lender/{id}/config` | Update lender Prism profile (segments, geography, risk appetite, PSL targets) |

**Data Models:**
- `LenderMatch`: application_id, matched_lender_id, match_score (composite), psl_gap_score (0–40), approval_probability (0–30), cost_to_borrower (0–20), fldg_availability (0–10), co_lending (bool), co_lend_structure (nullable)
- `CoLendStructure`: primary_lender_id, secondary_lender_id, split_ratio (default 80/20), rbi_cla_reference, risk_share_terms

**Decision Rationale:** Prism transforms the platform from a single-lender origination tool into a multi-lender marketplace. PSL gap scoring (weighted at 40% of match score) is the key insight: banks and SFBs must meet PSL targets (40% of net credit to priority sectors), and loans to underserved segments directly fill these gaps. By routing dairy farmer loans to the bank with the largest agriculture PSL gap, Prism aligns borrower need with lender incentive — creating a win-win that drives lender adoption. Co-lending structures allow small NBFCs to originate loans beyond their capital base by partnering with banks that need PSL assets, expanding the effective supply of credit.

---

### S10 — `los-engine` (port 9010)

**Role:** Temporal-based durable loan origination workflow engine managing the full 16-state lifecycle from application creation through disbursement and activation. Handles human-in-loop referral queues, e-sign orchestration, insurance enrollment, and disbursement via lender bank APIs.

**Dependencies:** All data/decision/distribution services (orchestrates the full pipeline), Temporal (durable execution), Kafka (`los.event.*`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/los/application` | Create new loan application (initiates Temporal workflow) |
| GET | `/v1/los/application/{id}` | Retrieve application with current state and history |
| POST | `/v1/los/application/{id}/human-decision` | Submit human decision for referred application |
| POST | `/v1/los/application/{id}/esign` | Trigger Aadhaar e-sign flow |
| POST | `/v1/los/application/{id}/disburse` | Initiate disbursement via lender bank API |
| GET | `/v1/los/queue/{lender_id}` | Retrieve human review queue for a lender |

**Data Models:**
- 16 states: CREATED → KYC_CHECK → DATA_FETCH → DECISIONING → APPROVED/REFERRED/REJECTED → PRODUCT_DESIGN → LENDER_MATCHING → OFFER_PRESENTED → ACCEPTED/DECLINED → ESIGN → DISBURSING → ACTIVE → CLOSED/GRADUATED/WRITTEN_OFF
- `LoanApplication`: id, lender_id, borrower_id, segment, channel (VOICE | WHATSAPP | WEB | PARTNER), current_state, state_history (list[{state, timestamp, actor}]), decision_id, product_id, match_id, disbursement_ref, activated_at
- Every state transition emits: Kafka event (`los.event.*`), audit log entry (tamper-evident), graph node (context-graph)

**Decision Rationale:** Temporal provides durable execution guarantees that are non-negotiable for financial workflows — a disbursement that starts but crashes mid-way must resume from the exact point of failure, not restart. The 16-state machine is intentionally linear with explicit branch points (APPROVED/REFERRED/REJECTED, ACCEPTED/DECLINED) rather than a free-form graph, because lending workflows are fundamentally sequential with regulatory gates. Human-in-loop for referred cases preserves lender control while AI handles the 70–85% of applications that are clearly approvable or rejectable. The GRADUATED state (borrower earns better terms after successful repayment) is a structural feature enabling credit ladder progression.

---

### S11 — `journey-studio` (port 9011)

**Role:** No-code journey builder for borrower application flows. Supports GenAI-generated journey definitions from natural language prompts, A/B testing with statistical significance (chi-squared), and per-segment per-channel flow customization.

**Dependencies:** agent-runtime (journey execution), notification-svc (channel delivery), Kafka (`journey.analytics`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/journey/create` | Create journey definition (JSON schema: screens, fields, validations, branching) |
| POST | `/v1/journey/generate` | Generate journey from natural language prompt (e.g., "Dairy farmer, Gujarati, voice, animal purchase") |
| PUT | `/v1/journey/{id}/ab-test` | Configure A/B test: control vs variant, borrower hash split |
| GET | `/v1/journey/{id}/analytics` | Conversion funnel: start → KYC → data → offer → accept → disburse |
| GET | `/v1/journey/list/{segment}/{channel}` | List active journeys for segment + channel combo |

**Data Models:**
- `JourneyDefinition`: id, segment, channel, language, screens (list[Screen]), branching_rules, data_bindings, validation_rules, ab_variant (nullable)
- `Screen`: id, type (FORM | INFO | CONSENT | DOCUMENT_UPLOAD | VOICE_PROMPT), fields (list), next_screen_rules

**Decision Rationale:** Every borrower segment + channel combination requires a different flow. A dairy farmer on voice in Gujarati needs a 5-minute conversational flow focused on animal count and cooperative membership; the same loan type via WhatsApp needs interactive buttons and document upload prompts. Building these flows in code creates a bottleneck; a no-code builder with GenAI generation lets product teams ship new segment journeys in hours rather than sprint cycles. A/B testing is structural (not bolted on) because conversion optimization is the primary lever for reducing cost-per-loan.

---

### S12 — `notification-svc` (port 9012)

**Role:** Multi-channel notification dispatch: SMS (OTP, status updates), WhatsApp templates (application status, repayment reminders), voice callbacks (missed borrower re-engagement), and email (lender reports, compliance notifications).

**Dependencies:** los-engine (lifecycle events), monitoring-agent (intervention notifications), Kafka (`notify.*`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/notify/send` | Dispatch notification via specified channel |
| POST | `/v1/notify/template/create` | Create or update notification template (WhatsApp pre-approved) |
| GET | `/v1/notify/status/{notification_id}` | Delivery status (SENT, DELIVERED, READ, FAILED) |
| POST | `/v1/notify/bulk` | Batch notification dispatch (repayment reminders, portfolio alerts) |

**Data Models:**
- `NotificationRequest`: borrower_id, channel (SMS | WHATSAPP | VOICE_CALLBACK | EMAIL), template_id, params (dict), scheduled_at (nullable), priority (HIGH | NORMAL | LOW)
- `DeliveryStatus`: notification_id, channel, status, delivered_at, read_at, failure_reason

**Decision Rationale:** Borrower re-engagement is the highest-ROI notification use case — a voice callback to a borrower who dropped off during data fetch recovers 20–30% of abandoned applications. WhatsApp template messages (pre-approved by BSP) enable rich interactions (buttons, quick replies) that SMS cannot provide. Channel selection is borrower-preference-aware: some segments prefer voice (agricultural), others WhatsApp (gig workers), others SMS (feature phone users). The template system enforces compliance (all borrower-facing messages must be pre-approved, include regulatory disclosures).

---

### S13 — `agent-runtime` (port 9013)

**Role:** Central multi-agent orchestrator implementing a 16-step turn processor. Routes borrower interactions to specialized domain agents, manages conversation state, executes MCP tool calls, and implements the SRAL (Sense-Reason-Act-Learn) pattern for continuous agent improvement.

**Dependencies:** constellation (LLM routing), knowledge-svc (RAG), voice-pipeline (voice channel), context-graph (memory), all service-layer APIs via MCP

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/agent/turn` | Process one agent turn: input → NLU → reasoning → tool calls → response |
| POST | `/v1/agent/session/create` | Create new agent session with channel, language, and segment context |
| GET | `/v1/agent/session/{id}` | Retrieve session state and conversation history |
| POST | `/v1/agent/escalate` | Escalate to human agent with full context transfer |
| POST | `/v1/agent/evaluate` | Run agent evaluation (tau-bench, pass^k scoring) |
| POST | `/v1/agent/promote` | Promote agent version: SHADOW → CANARY → PRODUCTION |

**Data Models:**
- 16-step turn processor: (1) receive input → (2) language detect → (3) ASR if voice → (4) intent classify → (5) context load (graph memory) → (6) policy check (journey gate) → (7) knowledge retrieve (RAG) → (8) LLM reason (constellation) → (9) tool plan → (10) tool execute (MCP) → (11) result validate → (12) response generate → (13) TTS if voice → (14) response deliver → (15) state persist → (16) RL signal emit
- `AgentSession`: session_id, borrower_id, channel, language, segment, journey_id, current_step, conversation_history, tool_call_log, escalation_reason (nullable)
- SRAL loop: Sense (channel input + ULI data + graph memory) → Reason (LLM via constellation + knowledge RAG) → Act (MCP tool execution, policy-gated) → Learn (outcome feedback → RL signal → model update)

**Decision Rationale:** The 16-step turn processor makes every agent interaction deterministic and auditable — critical for financial services where regulators may inspect any borrower interaction. The SRAL pattern (from production credit card agent systems) ensures agents improve from every interaction: repayment outcomes feed back as RL signals that retrain intent classifiers and response generators. Multi-agent orchestration (voice agent, data agent, decision agent, monitoring agent as specialized sub-agents) enables each agent to be narrow and accurate rather than general and error-prone. Policy gating on tool execution (step 6) prevents agents from taking actions the lender hasn't authorized at their current autonomy level.

---

### S14 — `voice-pipeline` (port 9014)

**Role:** End-to-end voice processing: Indian language ASR (Whisper fine-tuned on lending vocabulary), TTS (Bhashini for government-supported languages, ElevenLabs for premium quality), Voice Activity Detection (500ms lending-tuned for pauses when stating loan amounts), fast brain/slow brain parallel execution, and Exotel telephony integration.

**Dependencies:** agent-runtime (orchestration), constellation (LLM routing), Kafka (`voice.stream`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| WebSocket | `/v1/voice/stream` | Bidirectional audio stream (borrower ↔ agent) |
| POST | `/v1/voice/call/initiate` | Initiate outbound call via Exotel |
| POST | `/v1/voice/transcribe` | Offline transcription of voice recording |
| GET | `/v1/voice/call/{id}/status` | Call status and metadata |
| POST | `/v1/voice/tts` | Text-to-speech synthesis for a given language and text |

**Data Models:**
- `VoiceSession`: call_id, borrower_id, language, asr_provider (WHISPER_FINETUNED | DEEPGRAM), tts_provider (BHASHINI | ELEVENLABS), telephony_provider (EXOTEL | TWILIO), call_duration_ms, transcript (list[{speaker, text, timestamp}])
- VAD config: silence_threshold_ms=500 (lending-tuned: borrowers pause when recalling loan amounts), energy_threshold (adaptive), endpointing_mode (AGGRESSIVE for confirmations, RELAXED for open questions)
- Fast/slow brain: fast brain generates immediate acknowledgment ("Hmm, I see...") while slow brain processes the full reasoning pipeline, reducing perceived latency from 2s to <500ms

**Decision Rationale:** Voice is the primary channel for agricultural and micro-enterprise borrowers — 65% of the target 300M speak primarily in vernacular languages and may not be comfortable with text-based interfaces. Whisper fine-tuned on lending vocabulary (EMI, moratorium, cooperative, lakh, guarantee) dramatically improves transcription accuracy for domain-specific terms that generic ASR models consistently misrecognize. The 500ms VAD threshold is lending-tuned: generic VAD (200–300ms) cuts off borrowers mid-pause when they're calculating amounts, while 500ms accommodates the natural pause-to-recall pattern. Bhashini (government TTS) provides acceptable quality for 12 languages at zero cost; ElevenLabs provides premium quality for high-value interactions. Fast brain/slow brain parallel execution is borrowed from production voice AI systems where perceived latency directly impacts borrower completion rates.

---

### S15 — `constellation` (port 9015)

**Role:** Multi-model routing across 6 task types, with per-model health monitoring, automatic failover, and cost optimization. Routes each LLM request to the optimal model based on task complexity, latency requirements, and provider availability.

**Dependencies:** agent-runtime (consumer), voice-pipeline (fast/slow brain), knowledge-svc (embedding models)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/constellation/route` | Route LLM request to optimal model based on task type and constraints |
| GET | `/v1/constellation/health` | Model health dashboard: latency, error rate, cost per 1K tokens |
| PUT | `/v1/constellation/config` | Update routing rules, model priorities, and failover chains |
| GET | `/v1/constellation/models` | List available models with current status and capabilities |

**Data Models:**
- 6 task types: INTENT_CLASSIFICATION (fast, cheap → GPT-4o-mini / Gemini Flash), RESPONSE_GENERATION (quality → Claude Sonnet / GPT-4o), DOCUMENT_ANALYSIS (multimodal → Gemini Pro / GPT-4V), KNOWLEDGE_SYNTHESIS (RAG → Claude Haiku / GPT-4o-mini), TRANSLATION (specialized → IndicTrans / Google), SUMMARIZATION (balanced → Claude Haiku)
- `RouteDecision`: task_type, selected_model, fallback_chain (list), estimated_latency_ms, estimated_cost
- Health tracking: per-model p50/p95/p99 latency, error rate (5-min window), cost per 1K tokens, availability (up/degraded/down)

**Decision Rationale:** No single LLM is optimal for all lending tasks. Intent classification needs to be fast and cheap (100ms budget, millions of calls/day); credit document analysis needs multimodal capability and accuracy (latency less critical). Constellation abstracts model selection from the application layer, enabling transparent model upgrades and cost optimization without code changes. Automatic failover (e.g., Claude → GPT-4o → Gemini if primary is degraded) ensures the voice pipeline never blocks on a provider outage. Health monitoring with 5-minute sliding windows detects degradation before it impacts borrowers.

---

### S16 — `knowledge-svc` (port 9016)

**Role:** 7-partition RAG (Retrieval-Augmented Generation) system with a verbatim compliance store for RBI regulatory text, auto-partition routing based on query classification, and lending domain knowledge management.

**Dependencies:** constellation (embedding + generation), context-graph (knowledge graph), Kafka (`knowledge.query`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/knowledge/query` | Query knowledge base with auto-partition routing |
| POST | `/v1/knowledge/ingest` | Ingest new knowledge document into appropriate partition |
| GET | `/v1/knowledge/partitions` | List partitions with document counts and freshness |
| POST | `/v1/knowledge/compliance/check` | Verbatim lookup of regulatory text (no paraphrasing) |

**Data Models:**
- 7 partitions: (1) REGULATORY — RBI circulars, DPDP Act text, PSL guidelines (verbatim store, never paraphrased); (2) PRODUCT — loan product specifications, eligibility, pricing; (3) SEGMENT — 38 segment knowledge bases (dairy farming practices, gig economy patterns); (4) PROCESS — operational procedures, escalation paths, SLAs; (5) FAQ — borrower frequently asked questions per language; (6) LENDER — per-lender policies, brand voice, custom disclosures; (7) TRAINING — agent training materials, example conversations
- `KnowledgeQuery`: query_text, detected_partition (auto-routed), top_k results, confidence_scores, verbatim_flag (bool, for regulatory)

**Decision Rationale:** The 7-partition architecture prevents regulatory text from being paraphrased by embedding similarity — when an agent quotes an RBI rule, it must be verbatim, not a semantic approximation. Auto-partition routing (query → classifier → partition) ensures product questions don't pollute compliance results and vice versa. Segment-specific partitions (dairy farming practices, monsoon patterns, cooperative structures) give agents domain expertise that generic knowledge bases lack. The verbatim compliance store is a regulatory necessity: if an auditor asks "what did the agent tell the borrower about interest rates?", the answer must trace to the exact regulatory text, not an LLM interpretation.

---

### S17 — `context-graph` (port 9017)

**Role:** Neo4j-backed context graph implementing 6 sub-graphs: Context Lakehouse (versioned metadata with time travel), Data Graph (column-level lineage across ULI sources), Knowledge Graph (lending domain ontology), Governance Graph (regulatory rules and policy mappings), Active Ontology (entity resolution across systems), and Decision Traces (first-class provenance objects).

**Dependencies:** All services emit to Kafka `graph.ingest`; agent-runtime, sentinel-core, and monitoring-agent query the graph

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/graph/ingest` | Ingest event as graph node (every state change becomes a node) |
| POST | `/v1/graph/query` | Execute Cypher query against context graph |
| GET | `/v1/graph/borrower/{id}/context` | Retrieve full borrower context subgraph (relationships, history, decisions) |
| GET | `/v1/graph/decision-trace/{id}` | Retrieve decision trace: input → reasoning → output with full provenance |
| POST | `/v1/graph/similar-borrowers` | Find similar borrowers by graph embedding distance (for risk assessment) |
| GET | `/v1/graph/lineage/{data_element}` | Column-level data lineage from ULI source to decision |

**Data Models:**
- Graph entities: `(:Borrower)-[:HAS_LOAN]->(:Loan)`, `(:Borrower)-[:MEMBER_OF]->(:Cooperative)`, `(:Loan)-[:DECIDED_BY]->(:Policy)`, `(:Loan)-[:HAS_REPAYMENT]->(:RepaymentEvent)`, `(:Pincode)-[:HAD_WEATHER]->(:WeatherEvent)`, `(:Loan)-[:GRADUATED_TO]->(:Loan)`
- `DecisionTrace`: trace_id, input_snapshot (all features at decision time), reasoning (model + policy + rules applied), output (decision + amount + rate), timestamp, policy_version, model_version
- Context compounding: the 10,000th loan benefits from 9,999 prior decision traces — similar-borrower default patterns, segment-level risk trends, cooperative-level performance data

**Decision Rationale:** Context graphs solve the "amnesia problem" in ML-based lending — traditional scoring treats each application as independent, losing the compounding value of prior decisions. Decision Traces as first-class objects (not just log entries) enable regulatory audit ("show me exactly why this borrower was rejected, including the model version, policy version, and every feature value at decision time"). The Data Graph with column-level lineage answers "where did this income number come from?" — tracing from the final credit decision through data quality enrichment back to the specific bank statement line item or cooperative ledger entry. The Active Ontology resolves the same borrower across 8+ systems (Aadhaar, PAN, cooperative membership, bank account, device ID) into a unified identity without requiring a master data management system.

---

### S18 — `monitoring-agent` (port 9018)

**Role:** Continuous post-disbursal monitoring across 7 data stream watchers. Evaluates triggers against borrower-specific thresholds, executes 8 intervention types, and emits RL signals for model retraining.

**Dependencies:** uli-orchestrator (data re-fetch), product-engine (schedule adjustment), notification-svc (borrower alerts), context-graph (pattern detection), Kafka (`monitor.trigger.*`)

**Key APIs:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/monitor/loan/{id}/status` | Current monitoring status with active watchers and recent triggers |
| POST | `/v1/monitor/loan/{id}/intervene` | Manually trigger intervention (lender override) |
| GET | `/v1/monitor/portfolio/{lender_id}` | Portfolio-level monitoring dashboard (DPD distribution, trigger frequency) |
| PUT | `/v1/monitor/config/{segment}` | Update monitoring thresholds for a segment |
| GET | `/v1/monitor/interventions/{loan_id}` | History of all interventions for a loan |

**Data Models:**
- 7 data stream watchers: (1) COOPERATIVE_PRODUCTION — daily milk pouring, crop yield (polling); (2) WEATHER — IMD temperature, rainfall, cyclone alerts (push); (3) BANK_ACCOUNT — balance, transaction pattern changes via AA (scheduled); (4) PLATFORM_EARNINGS — gig platform payout changes (API); (5) EPFO — employment status changes (periodic); (6) GST — business volume changes for MSMEs (quarterly); (7) REPAYMENT — payment tracking on due dates (event-driven)
- 8 intervention types: (1) AUTO_MORATORIUM — auto-pause EMI on trigger (e.g., heatwave >38°C); (2) SCHEDULE_ADJUST — recalculate EMI based on changed income; (3) EMERGENCY_MICROLOAN — issue bridge loan for acute need; (4) INSURANCE_CLAIM — initiate Pashu Bima/crop claim; (5) GRADUATION_OFFER — pre-approve better terms after consistent repayment; (6) HUMAN_ESCALATION — flag for lender attention; (7) COLLECTIONS_SOFT — empathetic reminder via preferred channel; (8) COLLECTIONS_GRADUATED — progressive collections with context
- `TriggerEvent`: loan_id, watcher_type, trigger_condition, observed_value, threshold_value, severity (INFO | WARNING | CRITICAL), intervention_recommended
- RL signal: every intervention outcome (did moratorium prevent default? did graduation lead to re-borrowing?) feeds back as reward signal for model retraining

**Decision Rationale:** Post-disbursal monitoring is the platform's unfair advantage over traditional lending. A dairy farmer's cooperative production drop (detected via daily polling) predicts a repayment problem 60–90 days before the first missed EMI — enabling proactive moratorium that prevents the default entirely. The 7 watchers cover the data streams that matter for each segment: weather for agriculture, platform earnings for gig, EPFO for salaried. The 8 intervention types form a graduated response spectrum from soft (schedule adjust) to hard (collections) — and the RL feedback loop ensures the system learns which intervention type works best for which trigger pattern in which segment. This is where context compounds most powerfully: the 10,000th intervention benefits from 9,999 prior outcome observations.

---

### Cross-Cutting Services

#### `api-gateway`

**Role:** Kong-based API gateway handling TLS termination, JWT validation, per-lender rate limiting, request routing, and request ID injection for distributed tracing.

| Method | Path | Description |
|--------|------|-------------|
| ALL | `/v1/data/*` | Route to data layer services (S1–S5) |
| ALL | `/v1/decision/*` | Route to decision layer services (S6–S8) |
| ALL | `/v1/los/*` | Route to distribution layer (S9–S12) |
| ALL | `/v1/agent/*` | Route to intelligence layer (S13–S18) |
| POST | `/mcp` | Route to unified MCP server |

**Decision Rationale:** A single entry point simplifies client integration, enables consistent auth and rate-limiting, and provides a natural point for API versioning. Kong was chosen over custom implementation for its production-proven plugin ecosystem (JWT, rate-limiting, logging, CORS).

---

#### `auth-svc`

**Role:** JWT token issuance and validation for lender users, API key validation for partner integrations, Aadhaar OTP session management for borrowers. Sets PostgreSQL session variable (`app.current_lender_id`) for RLS enforcement.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/auth/login` | Lender user login → JWT + refresh token |
| POST | `/v1/auth/refresh` | Refresh JWT token |
| POST | `/v1/auth/api-key/validate` | Validate partner API key |
| GET | `/v1/auth/roles` | List RBAC roles: CRO, RISK_ANALYST, UNDERWRITER, OPS, COMPLIANCE, ADMIN |

**Decision Rationale:** JWT with RLS session variables makes multi-tenancy invisible to application code — every SQL query automatically filters by the authenticated lender's ID. RBAC roles map to lending organization structure: CRO approves policy promotions, risk analysts create policies, underwriters handle referrals, compliance officers generate reports.

---

#### `audit-trail`

**Role:** Immutable append-only audit log with tamper-evident hash chain. Every API call, agent action, credit decision, and state transition is logged with actor identity, timestamp, payload hash, and chain hash linking to the previous entry.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/audit/entity/{type}/{id}` | Retrieve full audit trail for an entity |
| GET | `/v1/audit/verify/{entity_id}` | Verify hash chain integrity for an entity's audit trail |
| GET | `/v1/audit/search` | Search audit log by actor, entity type, date range, action |

**Decision Rationale:** Tamper-evident hash chaining (each entry's `prev_hash` = SHA-256 of previous entry) makes it cryptographically provable that no audit entries have been inserted, modified, or deleted. This is a regulatory requirement under RBI Digital Lending Guidelines and provides the foundation for dispute resolution. Monthly partitioning keeps query performance manageable as the log grows.

---

#### `compliance-reporter`

**Role:** Automated PSL classification (agriculture, micro-enterprise, weaker sections), RBI regulatory reporting, fair lending bias detection (approval rates by gender, geography, segment), and DPDP Act consent ledger management.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/compliance/psl/classify` | Classify loan for PSL category |
| GET | `/v1/compliance/psl/report/{lender_id}` | Generate PSL achievement report |
| GET | `/v1/compliance/bias/report/{lender_id}` | Generate fair lending bias analysis |
| GET | `/v1/compliance/dpdp/consent/{borrower_id}` | Retrieve consent ledger for a borrower |
| POST | `/v1/compliance/dpdp/consent/revoke` | Revoke consent for specific data elements |

**Decision Rationale:** PSL auto-classification eliminates manual reporting effort and ensures accuracy — a dairy farmer loan is automatically tagged as agriculture priority sector, contributing to the lender's 18% agriculture sub-target. Bias detection (comparing approval rates across protected categories at the same score levels) is proactive regulatory risk management. The DPDP consent ledger makes consent a queryable, revocable data structure rather than a one-time checkbox.

---

#### `ml-platform`

**Role:** MLflow model registry, Feast feature store, training pipeline orchestration, and A/B model evaluation. Manages the full ML lifecycle from feature engineering through model training to production deployment with drift monitoring.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/ml/model/register` | Register new model version in MLflow |
| POST | `/v1/ml/model/{id}/promote` | Promote model to staging/production |
| GET | `/v1/ml/model/{id}/performance` | Model performance metrics: Gini, KS, AUC, approval rate |
| POST | `/v1/ml/features/compute` | Compute feature vector from Feast store |
| GET | `/v1/ml/drift/{model_id}` | PSI-based feature distribution drift report |

**Decision Rationale:** MLflow + Feast provides a production-standard ML platform without custom infrastructure. Feast pre-computes segment-level features (average cooperative production, regional default rates) enabling sub-50ms feature retrieval at scoring time. PSI-based drift detection catches feature distribution shifts (e.g., cooperative production patterns changing due to drought) before they degrade model accuracy.

---

#### `observability`

**Role:** Prometheus metrics collection, Grafana dashboards (platform overview, loan lifecycle, agent performance, data quality, per-lender), Loki log aggregation, and Jaeger distributed tracing across all 25 services.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/metrics` | Prometheus scrape endpoint (per service) |
| GET | `/v1/observability/dashboards` | List available Grafana dashboards |
| GET | `/v1/observability/traces/{request_id}` | Retrieve distributed trace across services |
| GET | `/v1/observability/alerts` | Active alert list with severity and affected services |

**Decision Rationale:** Prometheus + Grafana + Loki + Jaeger is the industry-standard open-source observability stack. Every service exposes request_count, latency_seconds (p50/p95/p99), and error_rate metrics. Distributed tracing with a single request_id propagated across all services enables root-cause analysis of slow loan applications ("which source timed out in the ULI fetch?").

---

#### `mcp-server`

**Role:** Unified MCP (Model Context Protocol) endpoint exposing all 49+ platform tools via JSON-RPC 2.0. Enables any AI agent (internal or external) to interact with lending services through a standardized tool-calling interface with per-tool RBAC.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/mcp` | JSON-RPC 2.0 endpoint for MCP tool calls |
| GET | `/mcp/tools` | List available tools with schemas and required permissions |
| GET | `/mcp/health` | MCP server health with per-tool availability |

**Tool categories:**
- DATA (12 tools): fetch_borrower_data, analyze_bank_statement, score_device_behavior, verify_identity, fetch_aa_data, resolve_entity, compute_data_quality, fetch_cooperative_data, fetch_weather_data, fetch_gst_data, fetch_epfo_data, fetch_platform_data
- DECISIONING (8 tools): trigger_credit_decision, run_what_if, fetch_model_performance, detect_drift, check_fraud, design_product, explain_decision, generate_kfs
- DISTRIBUTION (7 tools): match_lender, build_co_lending_structure, create_application, execute_esign, initiate_disbursement, enroll_insurance, check_application_status
- MONITORING (6 tools): check_portfolio_health, trigger_intervention, check_graduation_eligibility, fetch_borrower_context, detect_fraud_ring, generate_compliance_report
- AGENT (8 tools): classify_intent, generate_response, detect_empathy_escalation, fetch_conversation_history, log_interaction, evaluate_agent, promote_agent, configure_journey

**Permission roles:** AGENT_VOICE, AGENT_WHATSAPP, AGENT_WEB, AGENT_PARTNER, RISK_ANALYST, CRO, UNDERWRITER, COMPLIANCE, INTERNAL_COPILOT, PLATFORM_ADMIN

**Decision Rationale:** MCP as the universal tool interface decouples AI agents from service implementations. An external partner's agent can call `trigger_credit_decision` without knowing whether Sentinel uses XGBoost or a neural network. Per-tool RBAC ensures a voice agent can call `classify_intent` but not `promote_agent`, while a CRO copilot can call `run_what_if` but not `initiate_disbursement`. The 49+ tool count covers the complete lending lifecycle, making the platform fully operable via AI agents without any human-operated UI.

---

## 3. Data Models Reference

### 3.1 Key Enums

```
BorrowerSegment: DAIRY_FARMER | CROP_FARMER_RICE | CROP_FARMER_COTTON | CROP_FARMER_SUGARCANE |
  FISHERMAN | FPO_MEMBER | STREET_VENDOR | KIRANA_ONDC | SHG_MEMBER | ASHA_WORKER |
  GIG_WORKER_DELIVERY | GIG_WORKER_RIDEHAIL | EV_DRIVER | SALARIED_NTC | GOLD_LOAN |
  WAREHOUSE_RECEIPT | MEDICAL_LOAN | ... (38 total)

LoanState: CREATED | KYC_CHECK | DATA_FETCH | DECISIONING | APPROVED | REFERRED | REJECTED |
  PRODUCT_DESIGN | LENDER_MATCHING | OFFER_PRESENTED | ACCEPTED | DECLINED | ESIGN |
  DISBURSING | ACTIVE | CLOSED | GRADUATED | WRITTEN_OFF

ActiveSubState: CURRENT | MORATORIUM | ADJUSTED | DELINQUENT | RESTRUCTURED | EMERGENCY_LOAN

DecisionOutcome: APPROVE | REFER | REJECT

CreditGrade: A1 | A2 | B1 | B2

PolicyStatus: DRAFT | SHADOW | CANARY | PRODUCTION | RETIRED

Channel: VOICE | WHATSAPP | WEB | PARTNER

LenderType: PSB | SFB | NBFC | MFI | LSP

ScheduleType: SEASONAL_DYNAMIC | DAILY_MICRO | CATCH_SALE_LINKED | INCOME_CONTINGENT |
  MORATORIUM_WITH_TRIGGER | BRIDGE_ZERO_INTEREST | STAGED_DISBURSEMENT | PAYOUT_LINKED

FraudFlag: CIRCULAR_TRANSFER | ROUND_TRIP | INCOME_INFLATION | CASH_DEPOSIT_PATTERN |
  BOUNCE_FREQUENCY | VELOCITY_BREACH | RING_DETECTED | DEVICE_ANOMALY

InterventionType: AUTO_MORATORIUM | SCHEDULE_ADJUST | EMERGENCY_MICROLOAN | INSURANCE_CLAIM |
  GRADUATION_OFFER | HUMAN_ESCALATION | COLLECTIONS_SOFT | COLLECTIONS_GRADUATED

WatcherType: COOPERATIVE_PRODUCTION | WEATHER | BANK_ACCOUNT | PLATFORM_EARNINGS |
  EPFO | GST | REPAYMENT
```

### 3.2 Core Dataclasses

```
Borrower:
  id: UUID
  lender_id: UUID
  aadhaar_hash: str (SHA-256, never raw)
  aadhaar_encrypted: str (AES-256-GCM)
  pan_encrypted: str
  name_encrypted: str
  phone_encrypted: str
  pincode: str
  segment: BorrowerSegment
  data_quality_score: float (0.0–1.0)
  profile_json: dict (enriched non-PII features)

Loan:
  id: UUID
  lender_id: UUID
  borrower_id: UUID
  segment: BorrowerSegment
  channel: Channel
  current_state: LoanState
  active_sub_state: ActiveSubState (nullable)
  decision_id: UUID (FK → CreditDecision)
  product_type: ScheduleType
  amount: Decimal
  rate: Decimal
  tenure_months: int
  matched_lender_id: UUID
  disbursement_ref: str
  activated_at: datetime
  closed_at: datetime (nullable)

Decision:
  id: UUID
  lender_id: UUID
  borrower_id: UUID
  policy_id: UUID
  policy_version: int
  is_shadow: bool
  raw_score: int (0–900)
  grade: CreditGrade
  decision: DecisionOutcome
  reason_codes: list[str]
  shap_values: dict (nullable)
  proposed_amount: Decimal
  proposed_rate: Decimal
  fraud_flags: list[FraudFlag]
  data_confidence: float

Policy:
  id: UUID
  lender_id: UUID
  policy_id: str
  version: int
  status: PolicyStatus
  segment: BorrowerSegment
  definition_yaml: str
  definition_hash: str (SHA-256)
  scoring: ScoringConfig (model_id, features, grade_bands)
  decision_rules: DecisionRules (auto_approve, refer, auto_reject thresholds)
  pricing: PricingConfig (base_rate, grade_adjustments)

Repayment:
  id: UUID
  loan_id: UUID
  installment_no: int
  due_date: date
  principal: Decimal
  interest: Decimal
  total_emi: Decimal
  season_tag: str (FLUSH | LEAN | TRANSITION, nullable)
  status: str (SCHEDULED | PAID | PARTIAL | OVERDUE | MORATORIUM | SKIPPED)
  paid_amount: Decimal
  paid_at: datetime (nullable)
  moratorium_reason: str (nullable)

Consent:
  id: UUID
  lender_id: UUID
  borrower_id: UUID
  data_element: str (AADHAAR | PAN | BANK_STATEMENT | DEVICE | COOPERATIVE | ...)
  purpose: str (KYC | CREDIT_SCORING | MONITORING)
  consent_token: str
  granted_at: datetime
  expires_at: datetime (nullable)
  revoked_at: datetime (nullable)
  status: str (ACTIVE | EXPIRED | REVOKED)

AuditEntry:
  id: bigint
  lender_id: UUID
  entity_type: str
  entity_id: UUID
  action: str
  actor_type: str (USER | AGENT | SYSTEM)
  actor_id: str
  payload_hash: str (SHA-256)
  payload: dict
  prev_hash: str (chain hash)
  timestamp: datetime
```

---

## 4. Infrastructure Specification

### 4.1 Kubernetes

- **Cluster:** 3-node HA control plane (EKS), autoscaling worker nodes
- **Node pools:** CPU pool (general services), GPU pool (g5.xlarge for voice/ML inference)
- **Namespaces:** `platform` (core services), `data` (data layer), `ml` (ML workloads), `monitoring` (observability), `staging` (pre-production)
- **Service mesh:** Istio with mTLS between all services
- **Pod Disruption Budget:** min 2 replicas available for all services
- **HPA:** CPU-based autoscaling (target 70%) with custom metrics (queue depth) for Kafka consumers

### 4.2 PostgreSQL

- **Version:** 16 with extensions: uuid-ossp, pgcrypto, pg_partman
- **Topology:** Primary + 2 read replicas (multi-AZ)
- **Row-Level Security:** Enabled on all data tables; `lender_id` column on every row; session variable `app.current_lender_id` set by auth middleware
- **PII Encryption:** AES-256-GCM via pgcrypto for all PII columns (aadhaar, PAN, name, phone, DOB, address)
- **Partitioning:** audit_log partitioned by month (pg_partman automated); loan tables partitioned by lender_id for large deployments
- **Backups:** Continuous WAL archiving to S3, daily snapshots, 30-day retention

### 4.3 Redis

- **Topology:** 6-node cluster (3 primary + 3 replica), TLS enforced
- **Use cases:** Agent conversation state, OTP sessions (5-min TTL), policy cache (<5ms lookup), feature cache (pre-computed from Feast), rate limiting (per-lender, per-API, per-borrower), distributed locks (disbursement idempotency)
- **Eviction policy:** allkeys-lru

### 4.4 Kafka

- **Topology:** 3-broker MSK cluster with Avro schema registry
- **Retention:** 7 days default, 30 days for audit.event
- **24 topic groups:**

| Topic Pattern | Partitions | Purpose |
|--------------|------------|---------|
| `uli.fetch.{requested\|completed\|failed\|partial}` | 12 | Data fetch lifecycle |
| `dq.enriched` | 12 | Enriched borrower profiles |
| `bc.analyzed` | 6 | BankConnect results |
| `dc.scored` | 6 | DeviceConnect scores |
| `id.verified` | 6 | Identity verification results |
| `sentinel.decision.{requested\|completed\|referred\|rejected}` | 12 | Credit decisions |
| `product.designed` | 6 | Product/schedule design |
| `fraud.alert` | 6 | Fraud detection alerts |
| `prism.matched` | 6 | Lender matching results |
| `los.event.{created\|signed\|disbursed\|activated}` | 12 | Loan origination events |
| `journey.analytics` | 6 | Journey funnel events |
| `notify.dispatch` | 6 | Notification requests |
| `agent.interaction` | 12 | Agent conversation events |
| `voice.stream` | 24 | Voice audio chunks (high throughput) |
| `monitor.trigger.{detected\|intervened\|resolved}` | 6 | Monitoring triggers |
| `graph.ingest` | 12 | Context graph events |
| `audit.event` | 12 | Immutable audit entries |
| `rl.signal` | 6 | Reinforcement learning rewards |
| `knowledge.query` | 6 | Knowledge service queries |

### 4.5 Neo4j

- **Topology:** Causal cluster (3 core nodes)
- **Use cases:** Context graph (6 sub-graphs), fraud ring detection (multi-hop traversal), similar-borrower analysis (graph embeddings), data lineage
- **Backup:** Daily to S3
- **Query budget:** <200ms for 3-hop traversals (fraud ring), <50ms for single-borrower context fetch

### 4.6 Temporal

- **Topology:** HA deployment on Kubernetes (3 history nodes, 3 matching nodes)
- **Use cases:** Durable loan origination workflows (16-state machine), data fetch orchestration (parallel with timeout), scheduled monitoring jobs
- **Visibility:** Temporal Web UI for workflow inspection and debugging
- **Retention:** 30-day workflow history

### 4.7 Object Storage (S3/MinIO)

- **Buckets:** model-artifacts (MLflow), voice-recordings (encrypted, 90-day retention), documents (bank statements, KYC docs, encrypted), backups (DB snapshots, Neo4j dumps)
- **Encryption:** Server-side AES-256, bucket policies enforcing TLS-only access

### 4.8 HashiCorp Vault

- **Use cases:** Database credentials, API keys (ULI sources, LLM providers, payment gateways), PII encryption keys, TLS certificates
- **Key rotation:** Automated 90-day rotation for encryption keys, 30-day for API keys
- **Injection:** Kubernetes sidecar injector for service pods

---

## 5. Security Model

### 5.1 PII Encryption

- **Algorithm:** AES-256-GCM (authenticated encryption with associated data)
- **Key management:** Encryption keys stored in HashiCorp Vault, never in code/config/environment variables
- **Scope:** All personally identifiable information — Aadhaar number, PAN, name, phone, DOB, address, bank account numbers
- **Associated data:** lender_id used as associated data in AEAD, binding ciphertext to tenant
- **Key rotation:** 90-day automated rotation with re-encryption of active records

### 5.2 RBAC (Role-Based Access Control)

| Role | Capabilities |
|------|-------------|
| CRO | Promote policies (SHADOW → PRODUCTION), approve model deployments, view all reports |
| RISK_ANALYST | Create/edit policies (DRAFT), run what-if simulations, view shadow test results |
| UNDERWRITER | Process referred applications, override agent recommendations (logged) |
| OPS | Monitor disbursements, handle exceptions, view operational dashboards |
| COMPLIANCE | Generate regulatory reports, search audit trails, manage consent |
| ADMIN | User management, lender configuration, system settings |

### 5.3 Consent Ledger

- **Granularity:** Per data element per borrower per purpose (e.g., "Aadhaar for KYC" is a separate consent from "Aadhaar for credit scoring")
- **Lifecycle:** ACTIVE → EXPIRED (time-based) or ACTIVE → REVOKED (borrower-initiated)
- **Enforcement:** Every data fetch checks consent_ledger for active consent matching (borrower_id, data_element, purpose)
- **Revocation:** Borrower can revoke consent via any channel; revocation triggers data deletion where required by DPDP Act
- **Audit:** Every consent grant, expiry, and revocation logged to audit trail

### 5.4 Audit Immutability

- **Hash chain:** Each audit entry includes `prev_hash = SHA-256(previous_entry)`, creating a blockchain-like tamper-evident chain per entity
- **Verification:** Any entry can be verified by recomputing the chain from the first entry; broken chain = tamper detected
- **Partitioning:** Monthly partitions prevent archive queries from impacting current operations
- **Retention:** 7-year retention (RBI requirement for lending records)
- **Access control:** Audit entries are append-only; no UPDATE or DELETE permissions granted to any application role

### 5.5 Network Security

- **Service mesh:** Istio with mutual TLS (mTLS) between all services
- **External TLS:** TLS 1.3 for all external-facing endpoints
- **Network policies:** Kubernetes NetworkPolicy restricting inter-service communication to declared dependencies
- **PCI scrubbing:** At every event boundary (Kafka producer), PII is stripped or encrypted before emission
- **Container scanning:** Trivy on every build; CRITICAL/HIGH CVEs block deployment
- **Penetration testing:** Pre-launch mandatory; quarterly ongoing

### 5.6 Data Isolation

- **Database level:** PostgreSQL RLS policies on every data table; session variable `app.current_lender_id` set by auth middleware
- **Application level:** Lender ID extracted from JWT and propagated through all service calls
- **Event level:** Every Kafka message includes lender_id; consumers filter by authorized lender
- **Graph level:** Neo4j queries scoped by lender_id parameter
- **Cache level:** Redis keys prefixed with lender_id

---

*Document version: 1.0 | Generated: 2026-04-13 | Covers: 18 core + 7 cross-cutting services*
