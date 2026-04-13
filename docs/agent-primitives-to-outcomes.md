# Agent Primitives to Lending Outcomes
## Architecture Over Instruction: Mapping State, Context, Policy, and Feedback to Every Lending Outcome

---

# 1. INTRODUCTION: ARCHITECTURE OVER INSTRUCTION

## Why Prompts Are the Least Important Lever in Regulated Lending

In building AI agents for lending, teams fixate on prompt engineering -- the wording of system instructions, few-shot examples, chain-of-thought scaffolding. This is a mistake. Prompts are the *seed*; architecture is the *soil, water, sunlight, and seasons*.

Consider the most fundamental operation in Indian lending: **Aadhaar verification**.

### The Aadhaar Verification Example

**Prompt-based approach (fragile):**
```
System prompt: "Before accessing any borrower data, you MUST verify
the borrower's identity using Aadhaar OTP. Never skip this step.
Always confirm the OTP was successful before proceeding."
```

Problems with this approach:
- An LLM can hallucinate that verification succeeded
- A jailbreak can bypass the instruction
- A tool-calling race condition can fetch data before OTP completes
- There is zero audit trail if the prompt is ignored
- RBI compliance officers cannot verify prompt adherence at scale

**Architecture-based approach (structural gate):**
```
Policy Layer (enforced via MCP server, not agent):
  - MCP tool "fetch_uli_data" requires precondition: aadhaar_otp_verified = true
  - Precondition checked by Sentinel (S6) via mcp-server cross-cutting service
  - Tool is literally absent from agent's MCP tool list until gate is satisfied
  - Enforcement at MCP protocol layer (platform-mcp/permissions.py), not by LLM

State Layer:
  - aadhaar_otp_verified: boolean, persisted in session state
  - aadhaar_otp_attempts: counter, max 3
  - verification_timestamp: ISO 8601, immutable once set

Context Layer:
  - On successful verification: "Identity confirmed for Aadhaar XXXX-XXXX-1234"
  - On failure: "Verification failed. {remaining_attempts} attempts remaining."

Feedback Layer:
  - Log: verification_success_rate, average_attempts, time_to_verify
  - Alert: if success_rate < 95%, investigate ASR/telephony issues
```

The structural approach makes non-compliance *impossible*, not merely discouraged. The agent cannot access ULI tools until the policy gate confirms OTP verification. No prompt engineering required.

### The Four Primitives

Every lending outcome maps to four architectural primitives:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    THE FOUR PRIMITIVES                                │
│                                                                      │
│  ┌─────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐          │
│  │  STATE   │   │ CONTEXT  │   │ POLICY  │   │ FEEDBACK │          │
│  │         │   │          │   │         │   │          │          │
│  │ What    │   │ What the │   │ What    │   │ What     │          │
│  │ persists│   │ agent    │   │ governs │   │ improves │          │
│  │ across  │   │ knows at │   │ what    │   │ the      │          │
│  │ turns   │   │ decision │   │ the     │   │ system   │          │
│  │ and     │   │ time     │   │ agent   │   │ over     │          │
│  │ restarts│   │          │   │ can do  │   │ time     │          │
│  └─────────┘   └──────────┘   └─────────┘   └──────────┘          │
│                                                                      │
│  Graph DB      Context         Sentinel       RL signals,           │
│  Session       Lakehouse,      BRE (S6),      analytics (S18),      │
│  Temporal      ULI data,       RBI rules,     A/B results,          │
│  workflow      Active          MCP tool       WER tracking,          │
│  state         Ontology        gates,         trust decay,           │
│  (S17,S13)     (S17,S8-S12)    RBAC (auth)    cure rates (S7)       │
└──────────────────────────────────────────────────────────────────────┘
```

**State** = durable memory (Neo4j graph, Redis session, Temporal workflow state) → *Owned by: S17: context-graph, S13: agent-runtime, Temporal orchestrator*
**Context** = deterministically reconstructed decision surface -- not RAG retrieval, but versioned, reproducible context assembled from the Context Lakehouse (ULI data, bureau, weather, cooperative calendar, Active Ontology entity resolution across 8 systems, Decision Traces as first-class objects). Context is a build artifact with time-travel capability, not a search result. → *Owned by: S17: context-graph, S8-S12: data connectors, ml-platform*
**Policy** = hard constraints on agent behavior (Sentinel BRE, RBI rules, tool preconditions via MCP server, RBAC, autonomy levels) → *Owned by: S6: sentinel-core, mcp-server, auth-svc*
**Feedback** = signals that improve the system over time (RL rewards, analytics, comparison reports, WER, trust decay signals) → *Owned by: S18: analytics-svc, S7: rl-optimizer*

This document maps **every lending outcome** to these four primitives, demonstrating why architecture -- not prompts -- determines platform quality.

---

# 2. IDENTITY VERIFICATION OUTCOMES

## 2.1 Aadhaar OTP Before Data Fetch

The foundational gate: no borrower data is accessible without verified identity. **Owning service: S5: identity-svc.** MCP tools: `verify_aadhaar_otp`, `check_otp_status`. **Build Stage: Build 1 (Wk 5-8).**

| Primitive | Implementation |
|-----------|---------------|
| **State** | `aadhaar_otp_verified: bool`, `otp_attempts: int`, `verification_ts: datetime` in session state (S13: agent-runtime) |
| **Context** | Masked Aadhaar (XXXX-XXXX-1234), verification method (OTP/biometric), device fingerprint. Assembled by S17: context-graph. |
| **Policy** | Structural gate: MCP tools (`fetch_credit_bureau`, `fetch_bank_statement`, `fetch_gstin`) require `aadhaar_otp_verified = true` as Sentinel (S6) precondition. Tool literally absent from agent's MCP tool list until gate passes -- enforced at mcp-server layer, not agent. |
| **Feedback** | `verification_success_rate`, `avg_time_to_verify`, `otp_delivery_failure_rate` → alerts if success < 95% (S18: analytics-svc) |

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐
│ Borrower     │────>│ Aadhaar OTP  │────>│ Sentinel Gate  │
│ calls in     │     │ challenge    │     │ checks:        │
│              │     │              │     │ otp_verified?  │
└─────────────┘     └──────────────┘     └───────┬────────┘
                                                  │
                                          ┌───────┴────────┐
                                          │                │
                                     [YES]│           [NO] │
                                          │                │
                                   ┌──────▼──────┐  ┌─────▼──────┐
                                   │ ULI tools   │  │ Only       │
                                   │ UNLOCKED    │  │ identity   │
                                   │ - Bureau    │  │ tools      │
                                   │ - Bank stmt │  │ available  │
                                   │ - GSTIN     │  │            │
                                   └─────────────┘  └────────────┘
```

## 2.2 Failed Verification Handling

When OTP verification fails, the system must handle gracefully without exposing data.

| Primitive | Implementation |
|-----------|---------------|
| **State** | `otp_attempts` counter incremented on each failure, persisted across reconnections |
| **Context** | Failure reason (wrong OTP, expired OTP, delivery failure), remaining attempts |
| **Policy** | 3-attempt limit enforced by Sentinel. After 3 failures: session locked, human escalation triggered, 30-minute cooldown before retry. No prompt-level "please try again" -- hard architectural lock. |
| **Feedback** | `failure_reason_distribution` (wrong_otp vs expired vs delivery) → informs telephony/ASR optimization |

State machine for verification:
```
  [UNVERIFIED] ──OTP_SENT──> [PENDING]
       │                        │
       │                   [OTP_RECEIVED]
       │                        │
       │              ┌─────────┴──────────┐
       │              │                    │
       │         [CORRECT]            [INCORRECT]
       │              │                    │
       │        [VERIFIED]          attempts < 3?
       │                                   │
       │                          ┌────────┴────────┐
       │                     [YES]│            [NO] │
       │                          │                 │
       │                    [PENDING]          [LOCKED]
       │                    (retry)       (human escalation)
       └──────────────────────────────────────────────┘
```

## 2.3 Returning Borrower Recognition

For borrowers with prior interactions, the platform provides continuity.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Graph memory in Neo4j: `(:Borrower)-[:HAS_LOAN]->(:Loan)`, `(:Borrower)-[:INTERACTED_VIA]->(:Session)`, prior loan history, repayment patterns, preferred language |
| **Context** | Personalized greeting: "Welcome back Rameshbhai, I can see your crop loan from last kharif season was repaid on time." Previous interaction context loaded into agent prompt window. |
| **Policy** | Re-verification required if >30 days since last interaction. Simplified re-verification (voice match + last 4 Aadhaar digits) for <30-day gap. |
| **Feedback** | `returning_borrower_conversion_rate` vs `new_borrower_conversion_rate` → validates personalization value |

---

# 3. CREDIT DECISION OUTCOMES

## 3.1 Sentinel Policy Evaluation

The 3-layer Sentinel BRE evaluates every credit decision through Policy, with Context providing the enriched profile. **Owning service: S6: sentinel-core.** MCP tools: `evaluate_credit`, `get_decision_trace`. **Build Stage: Build 2 (Wk 9-12).**

| Primitive | Implementation |
|-----------|---------------|
| **State** | Decision trace: full audit trail of every rule evaluation, stored in Temporal workflow (S3: los-engine, S14: audit-trail) |
| **Context** | Enriched borrower profile reconstructed from Context Lakehouse (S17): ULI (bureau score, bank statements, GSTIN via S8: uli-connector), FinBox (device, BankConnect via S9), cooperative data (milk pour records, crop yield via S10), weather (IMD district-level via S12). Active Ontology resolves entity across all 8 sources. Context is versioned, reproducible, and 10-50x richer than traditional bureau-only. |
| **Policy** | 3-layer Sentinel (S6): (1) Hard gates (age, geography, blacklist), (2) Scoring model (segment-specific, ml-platform), (3) Business rules (ticket size limits, FOIR caps, PSL classification). Each layer can REJECT, APPROVE, or REFER. All layers must APPROVE for auto-decision. Data confidence score (see §3.1.1 above) modifies thresholds. |
| **Feedback** | `approval_rate_by_segment`, `default_rate_30dpd`, `model_discrimination` (KS, Gini, AUC) → triggers model retraining if AUC < 0.70 (S7: rl-optimizer, S18: analytics-svc) |

```
┌───────────────────────────────────────────────────────────┐
│              SENTINEL 3-LAYER EVALUATION                   │
│                                                           │
│  Layer 1: HARD GATES (Policy)                             │
│  ┌─ Age >= 18 && Age <= 65                                │
│  ├─ Not in blacklist (fraud ring, legal proceedings)      │
│  ├─ Geography: serviceable pincode                        │
│  └─ KYC: Aadhaar + PAN verified                          │
│         │                                                 │
│    [ALL PASS] ──────────────> [ANY FAIL] ──> REJECT       │
│         │                                                 │
│  Layer 2: SCORING MODEL (Policy + Context)                │
│  ┌─ Segment-specific model (dairy, kirana, gig, etc.)    │
│  ├─ Input: enriched context (bureau + alt data + cash)   │
│  ├─ Output: PD score, risk band (A/B/C/D/E)             │
│  └─ Threshold: segment-specific cutoff                    │
│         │                                                 │
│    [ABOVE CUT] ─────────────> [BELOW CUT] ──> REJECT     │
│         │                                                 │
│  Layer 3: BUSINESS RULES (Policy)                         │
│  ┌─ Ticket size within segment limits                     │
│  ├─ FOIR (EMI/income) < segment threshold                │
│  ├─ PSL classification check                              │
│  └─ Co-lending split rules (if applicable)                │
│         │                                                 │
│    [ALL PASS] ──> APPROVE     [ANY REFER] ──> HUMAN UW   │
└───────────────────────────────────────────────────────────┘
```

### Data Confidence Scoring: The Bridge Between Context and Policy

The quality of Context directly governs what Policy permits. The confidence scoring formula computes a composite score from Context completeness, freshness, cross-validation, and anomaly checks. This score then modifies Sentinel Policy behavior:

```
conf = 0.3 × (sources_available / sources_required)     # Completeness
     + 0.3 × avg_freshness_score                        # Freshness (1.0 = <24h, 0.5 = <7d, 0.0 = >30d)
     + 0.2 × cross_validation_rate                      # Cross-source agreement
     + 0.2 × (1 - anomaly_flags / total_checks)         # Anomaly absence

Owning service: S6: sentinel-core (evaluation), S17: context-graph (computation)
```

**How confidence modifies Policy (S6: sentinel-core):**

| Confidence Band | Sentinel Behavior | Auto-Decision | Amount Cap |
|-----------------|-------------------|---------------|------------|
| **High** (≥ 0.85) | Full auto-decision path | Enabled | Segment maximum |
| **Medium** (0.60-0.84) | Auto with reduced limits | Enabled | 60% of segment max |
| **Low** (0.40-0.59) | Refer to human underwriter | Disabled | N/A (human decides) |
| **Very Low** (< 0.40) | Auto-reject, request more data | Disabled | N/A |

This formula ensures that Context gaps don't silently degrade decisions. A borrower with only bureau data (2 of 6 sources) gets `conf = 0.3×(2/6) + 0.3×1.0 + 0.2×0 + 0.2×0.8 = 0.56` → referred to human underwriter, not auto-decided with incomplete data.

> **Cross-reference:** Formula defined in [unified-platform-specification.md §Build 2: Intelligence Layer](../unified-platform-specification.md). Implementation in mission-implementation-plan.md Mission 3 (`sentinel-core/_evaluate_confidence()`).

## 3.2 Shadow / Champion-Challenger Decisions

Every model update goes through shadow testing before live deployment.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Parallel decision traces: champion model decision AND challenger model decision stored for every application during shadow period. Both decisions persisted in Temporal workflow state. |
| **Context** | Identical context fed to both models simultaneously. Feature store ensures reproducibility. |
| **Policy** | Champion decision used for actual lending. Challenger is shadow-only (no real impact). Shadow period: minimum 1,000 decisions or 30 days. Promotion requires: challenger AUC > champion AUC + 0.02, no adverse segment impact. |
| **Feedback** | Comparison analytics: approval rate delta, score distribution shift, segment-level impact analysis, back-tested default prediction accuracy. Automated report generated at shadow period end. |

## 3.3 Explainability

Every credit decision must be explainable to the borrower, the lender, and the regulator.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Decision trace includes: raw feature values, model score, SHAP values per feature, triggered rules, final outcome. Immutable, stored for 7 years (RBI retention). |
| **Context** | SHAP values computed for top-5 contributing features. Translated to borrower-friendly language: "Your strong repayment history with the cooperative contributed positively." |
| **Policy** | Reason codes REQUIRED for every rejection (minimum 3 reasons). Format: machine-readable code + borrower-friendly text + lender-technical explanation. Adverse action notice template per RBI Digital Lending guidelines. |
| **Feedback** | `reason_code_distribution` → identifies if a single reason dominates (potential model bias). `borrower_comprehension_rate` (post-call survey) → refines explanation language. |

---

# 4. PRODUCT DESIGN OUTCOMES

## 4.1 Seasonal Dynamic EMI Schedule

Agricultural borrowers need EMI schedules aligned to harvest cycles, not uniform monthly payments.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Loan schedule object: `[{month, emi_amount, principal, interest, status}]` stored in LOS. Schedule is mutable based on cooperative calendar events. |
| **Context** | Cooperative calendar (milk flush season: Oct-Feb high pour, Mar-Jun lean), crop harvest dates (kharif: Oct, rabi: Mar), market price data (APMC mandi prices). |
| **Policy** | Structural constraint: `sum(all_EMIs) = total_repayment_obligation`. Higher EMIs during harvest/flush (up to 1.5x base), lower during lean (down to 0.5x base). Schedule generated algorithmically, not by LLM. |
| **Feedback** | `on_time_payment_rate_seasonal` vs `on_time_payment_rate_uniform` → validates seasonal scheduling improves repayment. `schedule_modification_requests` → indicates misalignment with actual cash flow. |

```
Monthly EMI Pattern for Dairy Farmer:
                                                        
  EMI ₹ │                                               
  3000  │         ██                                    
  2500  │      ██ ██ ██                                 
  2000  │   ██ ██ ██ ██ ██         ██                   
  1500  │██ ██ ██ ██ ██ ██ ██   ██ ██                   
  1000  │██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██           
   500  │██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██           
        └──────────────────────────────────────          
         O  N  D  J  F  M  A  M  J  J  A  S            
         ▲──── Flush Season ───▲  ▲─ Lean ─▲            
         (Higher EMIs)           (Lower EMIs)            
                                                         
  Constraint: sum(EMIs) = ₹24,000 (same as uniform)     
```

## 4.2 Moratorium with Weather Trigger

Automatic payment moratorium when verified weather events impact borrower's livelihood.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Weather event node in Neo4j graph: `(:WeatherEvent {type: "drought", severity: "severe", district: "Kutch", start_date, end_date})-[:AFFECTS]->(:Borrower)`. Moratorium status persisted in loan state machine. |
| **Context** | IMD (India Meteorological Department) district-level rainfall data, heat wave advisories, flood warnings. Pincode-to-district mapping. Historical yield impact data by event type. |
| **Policy** | Auto-moratorium rules: (1) District rainfall deficit > 40% of LPA triggers drought moratorium, (2) Heat wave advisory for >5 consecutive days triggers heat moratorium, (3) Flood alert (IMD red warning) triggers flood moratorium. Moratorium duration: event duration + 30-day recovery. Interest continues to accrue but no penalty. |
| **Feedback** | `cure_rate_post_moratorium` (% who resume payment after moratorium ends), `false_positive_moratorium_rate` (moratorium triggered but borrower unaffected) → refines trigger thresholds. |

---

# 5. DISBURSEMENT OUTCOMES

## 5.1 Temporal Workflow Durability

Disbursement is the most critical workflow: money leaves the system. It MUST be durable across failures.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Temporal workflow: `DisbursementWorkflow` with activities: `validate_approval`, `check_authority_limit`, `initiate_transfer`, `confirm_receipt`, `update_loan_status`. Workflow state persists across service restarts, network partitions, deployment updates. Each activity is independently retriable. |
| **Context** | Approved loan details, borrower bank account (verified via penny drop), lender's bank API credentials (from Vault), current authority limits for approver. |
| **Policy** | Authority limit matrix: Junior UW (up to ₹50K), Senior UW (up to ₹2L), Chief Credit Officer (up to ₹10L), Board committee (above ₹10L). Dual-approval required above ₹5L. Weekend/holiday disbursement blocked unless override by CCO. |
| **Feedback** | `disbursement_success_rate`, `avg_time_to_disburse`, `retry_rate_by_bank` → identifies problematic bank integrations. `weekend_override_frequency` → auditable metric. |

```
Temporal Workflow: Disbursement
                                                                    
  ┌──────────┐   ┌──────────────┐   ┌──────────────┐              
  │ Validate  │──>│ Check Auth   │──>│ Initiate     │              
  │ Approval  │   │ Limits       │   │ Transfer     │              
  │           │   │              │   │ (NEFT/IMPS)  │              
  └──────────┘   └──────────────┘   └──────┬───────┘              
                                           │                       
                                    ┌──────▼───────┐              
                                    │ Confirm      │              
                      ┌─────────────│ Receipt      │              
                      │             │ (penny drop) │              
                      │             └──────┬───────┘              
                      │                    │                       
                 [FAIL/TIMEOUT]      [CONFIRMED]                   
                      │                    │                       
               ┌──────▼───────┐    ┌──────▼───────┐              
               │ Retry with   │    │ Update Loan  │              
               │ exponential  │    │ Status:      │              
               │ backoff      │    │ DISBURSED    │              
               │ (max 3)      │    │              │              
               └──────────────┘    └──────────────┘              
                                                                    
  KEY: Temporal persists state at every arrow.                     
  Service crash at any point → resume from last checkpoint.        
```

## 5.2 Idempotent Disbursement

Preventing double-disbursement is a hard architectural requirement, not a prompt concern.

| Primitive | Implementation |
|-----------|---------------|
| **State** | `disbursement_ref` (UUID) generated at approval time, used as idempotency key for bank API calls. Stored in PostgreSQL with UNIQUE constraint. Any retry with same `disbursement_ref` is deduplicated at DB level. |
| **Context** | Prior disbursement attempts for this loan (if any), bank API response codes, transfer confirmation reference numbers. |
| **Policy** | Exactly-once semantics: (1) Generate `disbursement_ref` at approval, (2) Use as idempotency key in bank API, (3) DB UNIQUE constraint prevents duplicate records, (4) Temporal workflow ensures single execution path. Authority limits enforced pre-transfer, not post. |
| **Feedback** | `duplicate_attempt_count` → should be near-zero if Temporal working correctly. Any non-zero triggers investigation. `disbursement_reconciliation_rate` → daily reconciliation with bank statements. |

## 5.3 Co-Lending Structure

Co-lending (CLA) requires precise split management per RBI guidelines.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Co-lending agreement: `{lender_a_share: 80%, lender_b_share: 20%, lender_a_rate: 12%, lender_b_rate: 9%, blended_rate: 11.4%}`. Split tracked per disbursement tranche and per EMI collection. |
| **Context** | Lender profiles (NBFC vs bank), RBI CLA circular requirements (Jan 2026), lender-specific risk appetite, segment coverage preferences. |
| **Policy** | RBI CLA split rules: (1) Bank must take minimum 80% on-book, (2) Both lenders must independently underwrite, (3) Borrower sees single blended rate, (4) Collection split proportional to outstanding. Sentinel enforces compliance at origination AND at every collection event. |
| **Feedback** | `co_lending_portfolio_quality` (by split partner), `split_reconciliation_accuracy`, `regulatory_audit_findings` → zero tolerance for split calculation errors. |

---

# 6. POST-DISBURSAL / LIFECYCLE OUTCOMES

## 6.1 Weather Moratorium Trigger (Automated)

The complete lifecycle of weather-triggered moratorium, from data ingestion to resolution.

| Primitive | Implementation |
|-----------|---------------|
| **State** | IMD data ingested into Neo4j graph: `(:District)-[:HAS_WEATHER]->(:WeatherData {rainfall_mm, lpa_deficit_pct, date})`. Moratorium status in loan state machine: `ACTIVE -> MORATORIUM -> ACTIVE`. |
| **Context** | Pincode-to-district mapping, borrower's primary livelihood (dairy, crop, fisheries), weather impact matrix (drought affects crop > dairy, flood affects fisheries > crop). |
| **Policy** | Auto-moratorium rules by livelihood: Crop farmer + rainfall deficit > 40% → 90-day moratorium. Dairy farmer + heat wave > 5 days → 30-day moratorium. Fisheries + cyclone warning → 60-day moratorium. Interest accrues, no penalty, no negative bureau reporting during moratorium. |
| **Feedback** | RL signal: `cure_rate_post_moratorium` feeds back into moratorium duration optimization. If cure rate < 70%, extend default moratorium period. If > 90%, consider shorter period to reduce interest burden. |

## 6.2 Graduation Offer

Rewarding good borrowers with better terms incentivizes repayment and builds loyalty.

| Primitive | Implementation |
|-----------|---------------|
| **State** | 12-month repayment history in graph: `(:Borrower)-[:PAID_EMI {on_time: true, date}]->(:EMI)`. Clean history = 12 consecutive on-time payments. Graduation eligibility flag computed nightly by batch job. |
| **Context** | Current loan terms, borrower's repayment capacity growth (income trend from bank statements), market rates, competitive offers in borrower's segment. |
| **Policy** | Graduation criteria: (1) 12 months clean repayment, (2) No restructuring in current loan, (3) Income stable or growing, (4) No adverse bureau events. Graduation offer: rate reduction (50-100 bps), higher ticket size (up to 1.5x), longer tenure option. |
| **Feedback** | `graduation_acceptance_rate`, `post_graduation_default_rate` (must be < pre-graduation), `borrower_lifetime_value_improvement` → validates graduation program ROI. |

## 6.3 Fraud Ring Detection

Graph-native fraud detection identifies connected entities that traditional systems miss.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Neo4j graph relationships: `(:Borrower)-[:SHARES_DEVICE]->(:Device)<-[:SHARES_DEVICE]-(:Borrower)`, `(:Borrower)-[:SHARES_ADDRESS]->(:Address)`, `(:Borrower)-[:SHARES_EMPLOYER]->(:Employer)`, `(:Borrower)-[:REFERRED_BY]->(:Borrower)`. |
| **Context** | Multi-hop query: "Find all borrowers within 3 hops of flagged borrower X who share 2+ attributes (device, address, employer, phone, reference)." Enriched with: application timing (clustered applications = suspicious), document similarity scores. |
| **Policy** | Fraud ring threshold: 3+ borrowers sharing 2+ attributes with clustered application timing → auto-flag for investigation. Flagged applications: held (not rejected) pending human review. No automated rejection for fraud suspicion (false positive protection). |
| **Feedback** | `fraud_ring_detection_precision` (of flagged rings, % confirmed fraud), `fraud_ring_detection_recall` (of confirmed fraud, % caught by graph), `false_positive_rate` → tunes thresholds. |

---

# 7. COLLECTIONS OUTCOMES

## 7.1 Context-Aware Collections

Collections conversations must account for *why* a borrower missed payment.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Payment history: `missed_payments: [{date, amount, days_overdue}]`. Graph relationships linking borrower to weather events, hospital visits, family events. |
| **Context** | "Borrower missed EMI due to verified heat wave in Kutch district (IMD confirmed, 47°C for 8 consecutive days). Dairy output dropped 40% district-wide. 23 other borrowers in same district also impacted." This context transforms the collections conversation from adversarial to empathetic. |
| **Policy** | Empathetic tone mandate when weather/health event verified. No threatening language. Offer restructuring options proactively. Collections script adapts: (1) Acknowledge hardship, (2) Confirm current situation, (3) Present recovery options, (4) Document agreed plan. |
| **Feedback** | `cure_rate_by_approach` (empathetic vs standard), `borrower_satisfaction_post_collections`, `escalation_to_human_rate` → empathetic approach consistently shows higher cure rates in pilot data. |

## 7.2 RBI Fair Practices Code Compliance

Indian collections must comply with the **RBI Fair Practices Code (FPC)** (Master Direction DoR.FIN.REC.23/26.03.001/2024-25), **RBI Digital Lending Guidelines** (September 2022, RBI/2022-23/111), and evolving guidelines on outsourcing collections to AI agents. These are the governing Indian regulations -- US frameworks such as FDCPA do not apply.

| Primitive | Implementation | Owning Service |
|-----------|---------------|----------------|
| **State** | Call log: `{timestamp, duration, agent_type (AI/human), consent_given, recording_id, outcome}`. Contact attempt counter per day/week. All records immutable, retained 8 years per RBI retention norms. | S3: los-engine, S14: audit-trail |
| **Context** | Borrower's preferred contact time, language preference, prior interaction sentiment, current DPD (days past due) bucket. RBI FPC clause applicability based on loan type and DPD bracket. | S17: context-graph |
| **Policy** | Calling hours: 8 AM - 7 PM borrower's local time (enforced by Sentinel scheduler rule `collections.calling_hours`, not prompt). Max 3 contact attempts per week (RBI FPC §6). Consent verification at call start (recorded). No third-party contact without borrower consent (RBI FPC §7). No workplace calls. Language must match borrower preference. AI agent must disclose non-human nature at conversation start (Digital Lending Guidelines §4.2). | S6: sentinel-core, S13: agent-runtime |
| **Feedback** | `compliance_violation_count` (must be zero), `consent_recording_coverage` (must be 100%), `calling_hours_violation_count` → automated audit trail for RBI inspectors. Monthly compliance report generated per `compliance-reporter` cross-cutting service. | S18: analytics-svc, compliance-reporter |

---

# 8. LENDER OPERATIONS OUTCOMES

## 8.1 Copilot Natural Language Query

Lender operations teams interact with the platform through natural language, routed to appropriate MCP tools. **Owning service: S13: agent-runtime + mcp-server.** MCP tools: `query_portfolio`, `get_borrower_details`, `run_analytics_query` + 46 more (49+ total). **Build Stage: Build 3 (Wk 13-16).**

| Primitive | Implementation |
|-----------|---------------|
| **State** | Query history: prior questions and results for context continuity (S13: agent-runtime). User session state for multi-turn exploration ("Now filter that by Gujarat" refers to previous query results). |
| **Context** | MCP tool routing via mcp-server: "Show me all loans in Kutch district with DPD > 30" → routes to MCP tool `query_portfolio` with parameters `{district: "Kutch", dpd_min: 30}`. Agent selects from 49+ MCP tools (6 categories: identity, data, decisioning, portfolio, lifecycle, reporting) based on query intent. Tool descriptions in MCP registry guide selection. |
| **Policy** | RBAC per role (auth-svc), enforced at mcp-server layer: Branch Manager's MCP tool list physically excludes other branches' query capabilities. Regional Head sees their region. Risk Officer sees risk metrics but not PII. Collections Manager sees DPD buckets but not credit scores. Tool availability filtered by mcp-server BEFORE agent sees tool list -- agent cannot even attempt unauthorized queries. |
| **Feedback** | `query_success_rate` (did the tool return useful results?), `query_latency`, `tool_routing_accuracy` (was the right MCP tool selected?), `user_satisfaction_rating` → improves MCP tool descriptions and routing logic (S18: analytics-svc). |

## 8.2 Policy Shadow Testing

New lending policies (interest rate changes, eligibility criteria updates) can be tested in shadow before deployment.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Shadow decision log: for every real application, compute both current-policy and proposed-policy decisions. Store both outcomes with full audit trail. |
| **Context** | Policy change description, expected impact analysis, affected segments, historical application data for back-testing. |
| **Policy** | Shadow mode enforced: proposed policy NEVER affects real decisions during testing. Minimum shadow period: 500 applications or 14 days. Promotion requires: no adverse impact on target segments, impact within expected range, approval from Risk Head. |
| **Feedback** | Automated comparison report: approval rate impact, portfolio composition shift, expected default rate change, revenue impact estimate, segment-level breakdown. |

## 8.3 PSL Classification

Priority Sector Lending (PSL) classification is automated and auditable.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Classification audit trail: `{application_id, segment, sub_segment, psl_category, psl_sub_category, classification_rules_applied, classification_timestamp}`. Immutable once set. |
| **Context** | Borrower profile (occupation, income, loan purpose, geography), RBI PSL master circular definitions, segment-specific thresholds (agriculture up to ₹2Cr, micro enterprise up to ₹1Cr). |
| **Policy** | Auto-classification rules based on RBI PSL norms. Categories: Agriculture (crop, allied activities), MSME (micro, small, medium), Weaker Sections (SHG, SC/ST, women), Housing (up to ₹35L in metro, ₹25L non-metro). Classification determines: reporting bucket, sub-target compliance, inter-bank PSL certificate eligibility. |
| **Feedback** | `psl_classification_accuracy` (auditor-verified sample vs auto-classification), `psl_target_achievement_by_category`, `psl_shortfall_penalty_avoided` → validates classification engine accuracy. |

---

# 9. VOICE OUTCOMES

## 9.1 Gujarati ASR Accuracy

Voice agents for dairy farmers in Gujarat require domain-specific ASR optimization.

| Primitive | Implementation |
|-----------|---------------|
| **State** | ASR model version, custom vocabulary list, per-borrower pronunciation profile (if returning borrower). |
| **Context** | Lending vocabulary boost: domain-specific terms weighted higher in ASR decoding. Terms: EMI, CIBIL, Aadhaar, NACH, FOIR, buffalo (bhens), cooperative (sahkari mandali), milk pour (dudh bharvu), harvest (laṇṇī). Code-mixing handling: Gujarati-Hindi-English mixed utterances common. |
| **Policy** | WER target: < 15% for lending domain vocabulary. Fallback: if ASR confidence < 0.6, ask borrower to repeat. If 3 low-confidence utterances, offer Hindi fallback. If still failing, transfer to human agent with ASR transcript for context. |
| **Feedback** | `wer_by_language`, `wer_by_term_category` (numbers, names, domain terms), `asr_fallback_rate`, `human_transfer_rate_due_to_asr` → prioritizes vocabulary fine-tuning. |

## 9.2 Fast Brain Acknowledgment

Sub-50ms responses keep the conversation feeling natural while the slow brain processes.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Keyword-to-acknowledgment mapping cached in Redis. No API call needed. Keywords: "loan status", "EMI amount", "next payment", "balance", "Aadhaar". |
| **Context** | Not needed for fast brain -- this is pure pattern matching, not contextual reasoning. |
| **Policy** | Fast brain ONLY for acknowledgment ("Let me check your loan status", "Looking up your EMI details"). Never for decisions. Never for amounts. Never for dates. All substantive responses come from slow brain (full agent runtime). |
| **Feedback** | `fast_brain_trigger_rate`, `fast_brain_accuracy` (was the acknowledgment relevant?), `perceived_latency_with_fast_brain` vs `without` → validates latency improvement. |

```
Fast Brain / Slow Brain Pipeline:
                                                                    
  Borrower: "What's my EMI?"                                       
       │                                                            
       ▼                                                            
  ┌──────────────┐                                                  
  │ VAD + ASR    │                                                  
  │ (200ms)      │                                                  
  └──────┬───────┘                                                  
         │                                                          
    ┌────▼─────┐      ┌──────────────────┐                         
    │ FAST     │      │ SLOW BRAIN       │                         
    │ BRAIN    │      │ (parallel)       │                         
    │ <50ms    │      │                  │                         
    │          │      │ Agent Runtime    │                         
    │ Match:   │      │ → MCP Tool Call  │                         
    │ "EMI"    │      │ → Fetch actual   │                         
    │          │      │   EMI amount     │                         
    │ Respond: │      │ → Format: "Your  │                         
    │ "Let me  │      │   EMI is ₹2,450  │                         
    │  check   │      │   due on 15th"   │                         
    │  your    │      │                  │                         
    │  EMI"    │      │ (~800ms)         │                         
    └────┬─────┘      └────────┬─────────┘                         
         │                     │                                    
    [IMMEDIATE]          [AFTER 800ms]                              
         │                     │                                    
         ▼                     ▼                                    
    Borrower hears:  Borrower hears:                               
    "Let me check    "Your EMI is                                  
     your EMI"       ₹2,450, due on                                
                     the 15th of this                               
                     month"                                         
```

## 9.3 Empathy Detection and Escalation

Voice agents must detect borrower distress and escalate to humans.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Sentiment trajectory across conversation turns: `[{turn: 1, sentiment: "neutral"}, {turn: 3, sentiment: "distressed"}, ...]`. Distress keywords detected and logged. |
| **Context** | Distress keyword dictionary: "husband passed", "flood", "drought", "hospital", "accident", "died", "lost everything". Tonal analysis: pitch variance, speech rate changes, silence duration. |
| **Policy** | Distress detected → immediate soft escalation: "I understand this is a difficult time. Let me connect you with a specialist who can help with your situation." No further collections messaging. No loan amount discussion. Human agent receives full context transcript. Escalation is POLICY, not prompt -- the agent literally cannot continue collections after distress flag. |
| **Feedback** | `distress_detection_accuracy` (human-verified sample), `escalation_appropriateness`, `post_escalation_borrower_outcome` (restructuring offered? moratorium granted?) → validates escalation improves outcomes. |

---

# 10. COMPLETE MAPPING MATRIX

> Enhanced with: owning service IDs (S#), MCP tool references, verification gates, and build-stage alignment.
> Cross-references: Service IDs per [unified-platform-specification.md](../unified-platform-specification.md), build stages and gates per [mission-implementation-plan.md](../mission-implementation-plan.md).

## 10.1 Identity Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Aadhaar OTP verification | S5: identity-svc | `verify_aadhaar_otp`, `check_otp_status` | otp_verified flag, attempt counter | Masked Aadhaar, device fingerprint | Structural gate: ULI tools require `otp_verified=true` (mcp-server precondition) | Success rate ≥ 95%, delivery failure < 2% | Gate 1: 100 OTP flows, 0 data leaks pre-verify | Build 1 (Wk 5-8) |
| Failed verification handling | S5: identity-svc | `lock_session`, `schedule_retry` | Attempt counter, lock status | Failure reason, remaining attempts | 3-attempt limit (Sentinel rule `identity.max_attempts`), 30-min cooldown | Failure reason distribution | Gate 1: Lock verified in load test (1K concurrent) | Build 1 (Wk 5-8) |
| Returning borrower recognition | S17: context-graph | `fetch_borrower_history`, `load_session_context` | Graph memory (loan history, interactions) | Personalized greeting, prior context from Context Lakehouse | Re-verification rules by recency (Sentinel `identity.reverify_days`) | Returning vs new conversion rate | Gate 2: Personalization accuracy > 90% in tau-bench | Build 2 (Wk 9-12) |
| PAN-Aadhaar linkage check | S5: identity-svc | `verify_pan_aadhaar_link` | Linkage status, verification timestamp | PAN details, name match score via Active Ontology | Mandatory pre-disbursement check (Sentinel hard gate) | Linkage failure rate by segment | Gate 1: 0 disbursements without linkage | Build 1 (Wk 5-8) |

## 10.2 Decisioning Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Sentinel 3-layer evaluation | S6: sentinel-core | `evaluate_credit`, `get_decision_trace` | Decision trace in Temporal | Enriched profile from Context Lakehouse (confidence-scored) | Hard gates → Scoring → Biz rules. conf < 0.60 → refer human. | Approval rate, default 30dpd, AUC > 0.70 | Gate 2: 1K decisions, Sentinel p95 < 200ms | Build 2 (Wk 9-12) |
| Shadow/champion-challenger | S6: sentinel-core, ml-platform | `run_shadow_decision`, `compare_models` | Parallel decision traces | Identical versioned context to both models | Champion live, challenger shadow (Sentinel `model.shadow_mode`) | AUC delta, approval rate delta, segment impact | Gate 4: Shadow infra handles 2x decision load | Build 4 (Wk 17-20) |
| Explainability | S6: sentinel-core | `get_shap_values`, `get_reason_codes` | SHAP values, reason codes, full trace (7yr retention) | Top-5 feature contributions via Decision Traces | Min 3 reason codes per rejection (RBI Digital Lending §5) | Reason code distribution, comprehension rate | Gate 2: Reason codes on 100% of rejections | Build 2 (Wk 9-12) |
| Segment-specific scoring | S6: sentinel-core, ml-platform | `select_segment_model`, `evaluate_segment` | Segment model selection trace | Segment features (milk pour, crop yield, gig earnings) | Segment threshold config (YAML per 38 segments) | Per-segment default rates, KS > 0.30 | Gate 3: Each segment model validated on holdout | Build 3 (Wk 13-16) |

## 10.3 Product Design Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Seasonal EMI schedule | S3: los-engine | `generate_emi_schedule`, `modify_schedule` | Schedule object in LOS | Cooperative calendar, harvest dates from Context Lakehouse | `sum(EMIs) = total_obligation` (DB CHECK constraint) | On-time rate: seasonal vs uniform (A/B) | Gate 3: Invariant holds across 10K loans | Build 3 (Wk 13-16) |
| Weather moratorium | S4: monitor-svc, S12 | `check_weather_moratorium`, `apply_moratorium` | Moratorium status in loan FSM | IMD district weather data (7-day cache) | Auto-trigger thresholds (Sentinel `weather.moratorium.*`) | Cure rate post-moratorium ≥ 70% | Gate 5: Weather→moratorium in < 1hr | Build 5 (Wk 21-30) |
| Flexible tenure | S3: los-engine | `calculate_tenure_options` | Tenure selection trace | Borrower cash flow from Context Lakehouse | Min/max tenure by segment (Sentinel biz rules) | Prepayment rate by tenure | Gate 3: Tenure within regulatory limits | Build 3 (Wk 13-16) |
| Top-up on existing loan | S3: los-engine, S6 | `check_topup_eligibility`, `create_topup` | Existing loan state, eligibility flag | Repayment history, capacity growth | Top-up: min 6 months clean (Sentinel rule) | Top-up default rate vs fresh loans | Gate 4: Eligibility 100% consistent | Build 4 (Wk 17-20) |

## 10.4 Disbursement Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Temporal workflow durability | S3: los-engine, Temporal | `initiate_disbursement`, `check_disbursement_status` | Workflow checkpoints, activity state | Loan details, bank account, vault creds | Authority limit matrix (Sentinel `disburse.authority.*`) | Success rate ≥ 99.5%, retry rate | Gate 2: Survive worker kill mid-disbursement | Build 2 (Wk 9-12) |
| Idempotent disbursement | S3: los-engine | `disburse_loan` (idempotency key required) | disbursement_ref UUID, DB UNIQUE | Prior attempts, bank responses | Exactly-once: UUID + UNIQUE + Temporal single path | Duplicate attempt count = 0 | Gate 2: 0 double-disbursements in chaos test | Build 2 (Wk 9-12) |
| Co-lending split | S3: los-engine | `calculate_colending_split`, `reconcile_split` | Split agreement, per-tranche tracking | Lender profiles, RBI CLA circular (Jan 2026) | 80/20 min bank share, dual UW (Sentinel) | Portfolio quality by partner | Gate 4: Split reconciliation 100% accurate | Build 4 (Wk 17-20) |
| NACH mandate registration | S3: los-engine | `register_nach_mandate`, `check_mandate_status` | Mandate status, UMRN reference | Borrower bank details, mandate amount | Mandate before disbursement (Sentinel hard gate) | Mandate success rate by bank | Gate 2: 0 disbursements without mandate | Build 2 (Wk 9-12) |

## 10.5 Lifecycle Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Weather moratorium (auto) | S4: monitor-svc | `trigger_moratorium`, `resolve_moratorium` | IMD data in graph, moratorium FSM | Pincode weather, livelihood type, impact matrix | Auto-trigger by livelihood (Sentinel `monitor.weather.*`) | Cure rate (RL-optimized duration), FP rate | Gate 5: Triggers within 1hr of IMD alert | Build 5 (Wk 21-30) |
| Graduation offer | S4: monitor-svc | `check_graduation_eligibility`, `create_graduation_offer` | 12-month history, eligibility flag | Income trend, market rates from Context Lakehouse | Clean repayment + no restructuring (Sentinel) | Acceptance rate, post-grad default < pre-grad | Gate 4: Criteria 100% auditable | Build 4 (Wk 17-20) |
| Fraud ring detection | S17: context-graph | `detect_fraud_ring`, `flag_application` | Graph relationships (device, address, employer) | Multi-hop query, timing via Active Ontology | 3+ borrowers + 2+ shared attrs → flag (Sentinel) | Precision ≥ 80%, recall ≥ 60%, FP < 15% | Gate 3: Catches injected synthetic rings | Build 3 (Wk 13-16) |
| Early warning signals | S4: monitor-svc | `classify_ews`, `trigger_intervention` | Payment pattern features, bureau alerts | Cash flow deterioration, market stress | EWS → SMA-0/1/2 (Sentinel + ML) | EWS AUC for 90dpd prediction | Gate 4: Fires ≥ 30 days before default | Build 4 (Wk 17-20) |

## 10.6 Collections Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Context-aware collections | S13: agent-runtime | `get_collection_context`, `log_collection_attempt` | Payment history, weather/health links | Verified hardship reason, district data from Lakehouse | Empathetic tone on hardship (Sentinel `collections.hardship_mode`) | Cure rate by approach (empathetic > standard) | Gate 5: Hardship detection accuracy ≥ 85% | Build 5 (Wk 21-30) |
| RBI FPC compliance | S13, S14: audit-trail | `check_contact_eligibility`, `log_consent` | Call log, consent records, contact counter | Preferred time, language, DPD, RBI FPC clauses | Hours 8-7, max 3/week, consent, AI disclosure (Sentinel `collections.rbi_fpc.*`) | Violations = 0, consent coverage = 100% | Gate 5: 0 violations in 1K simulated calls | Build 5 (Wk 21-30) |
| Restructuring offer | S3: los-engine | `assess_restructure_eligibility`, `create_restructure` | Current loan terms, hardship verification | Borrower capacity reassessment from Lakehouse | Eligibility by DPD (Sentinel biz rules) | Restructuring success rate (cure in 6mo) | Gate 5: Restructured loans tracked separately | Build 5 (Wk 21-30) |
| Legal escalation | S3: los-engine | `initiate_legal_process`, `generate_notice` | DPD history, communication trail (immutable) | Total outstanding, collateral value | DPD > 180 + comm exhausted (Sentinel hard gate) | Recovery rate post-legal | Gate 6: Full audit trail for proceedings | Build 6 (Wk 31-40) |

## 10.7 Lender Operations Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Copilot NL query | S13, mcp-server | `query_portfolio`, `get_borrower_details` + 46 more | Query history, session context | MCP routing (49+ tools), role-filtered via mcp-server | RBAC per role (auth-svc), tools filtered BEFORE agent | Query success ≥ 90%, routing accuracy | Gate 3: RBAC isolation (role X ≠ role Y data) | Build 3 (Wk 13-16) |
| Policy shadow testing | S6: sentinel-core | `create_shadow_policy`, `run_shadow_eval`, `get_shadow_report` | Shadow decision log, both outcomes | Policy change, historical backtest data | Shadow-only (Sentinel `policy.shadow_mode`) | Comparison: approval Δ, default Δ, revenue Δ | Gate 4: Shadow provably isolated from prod | Build 4 (Wk 17-20) |
| PSL classification | S3: los-engine | `classify_psl`, `get_psl_report` | Classification audit trail (immutable) | Borrower profile, RBI PSL master circular | Auto-classification per RBI norms (Sentinel) | Accuracy ≥ 99% (auditor-verified) | Gate 3: Matches manual audit sample | Build 3 (Wk 13-16) |
| MIS/regulatory reporting | S18, compliance-reporter | `generate_regulatory_report`, `submit_report` | Report state, delivery log | Aggregated portfolio metrics | Templates per regulator (RBI, NHB, NABARD) | Accuracy, on-time delivery = 100% | Gate 4: Reports match manual reconciliation | Build 4 (Wk 17-20) |

## 10.8 Voice Outcomes

| Outcome | Owning Service | MCP Tools | State | Context | Policy | Feedback | Verification Gate | Build Stage |
|---------|---------------|-----------|-------|---------|--------|----------|-------------------|-------------|
| Gujarati ASR accuracy | S15: voice-gateway | `transcribe_audio`, `boost_vocabulary` | ASR model version, custom vocab | Lending vocab boost (Bhashini + custom), code-mixing | WER < 15% (Sentinel `voice.wer_threshold`), fallback | WER by language, by term category | Gate 5: WER < 15% on 500 lending utterances | Build 5 (Wk 21-30) |
| Fast brain acknowledgment | S15: voice-gateway | N/A (Redis pattern match, no MCP) | Keyword-to-ack mapping (Redis) | None (pattern match only) | Ack only, never decisions (Sentinel hard gate) | Trigger rate, accuracy, latency < 200ms | Gate 5: Never returns substantive data | Build 5 (Wk 21-30) |
| Empathy detection | S15, S13: agent-runtime | `detect_distress`, `escalate_to_human` | Sentiment trajectory per turn | Distress keywords, tonal analysis | Distress → hard escalation (Sentinel `voice.distress_escalation`) | Detection accuracy ≥ 85%, escalation outcomes | Gate 5: 0 missed distress in test corpus | Build 5 (Wk 21-30) |
| Language switching | S15: voice-gateway | `switch_language`, `detect_language` | Current language state, switch history | Detected vs preferred language from Lakehouse | Supported set per region (config) | Switch frequency, post-switch WER | Gate 5: Seamless switch < 500ms | Build 5 (Wk 21-30) |

---

# 11. THE PRIMITIVES DEPENDENCY CHAIN

Not all primitives contribute equally to every outcome. Each outcome category has a PRIMARY primitive -- the one without which the outcome is impossible.

```
┌─────────────────────────────────────────────────────────────────┐
│          PRIMARY PRIMITIVE BY OUTCOME CATEGORY                   │
│                                                                  │
│  CATEGORY              PRIMARY       WHY                         │
│  ─────────────────────────────────────────────────────────────  │
│  Identity              POLICY        Gates are structural,       │
│                                      not advisory                │
│                                                                  │
│  Decisioning           CONTEXT       Richer context = better     │
│                                      decisions (10-50x more      │
│                                      features than bureau-only)  │
│                                                                  │
│  Product Design        POLICY        Constraints (sum of EMIs)   │
│                                      are mathematical, not       │
│                                      suggestive                  │
│                                                                  │
│  Disbursement          STATE         Durability and idempotency  │
│                                      are state problems          │
│                                                                  │
│  Lifecycle             FEEDBACK      System improves moratorium  │
│                                      duration, graduation        │
│                                      criteria via RL signals     │
│                                                                  │
│  Collections           CONTEXT       Knowing WHY someone missed  │
│                                      payment changes everything  │
│                                                                  │
│  Lender Ops            POLICY        RBAC and tool filtering     │
│                                      define what each role can   │
│                                      see and do                  │
│                                                                  │
│  Voice                 FEEDBACK      WER tracking, latency       │
│                                      monitoring, sentiment       │
│                                      accuracy drive improvement  │
└─────────────────────────────────────────────────────────────────┘
```

### Dependency Hierarchy

```
                    ┌──────────┐
                    │ FEEDBACK │  ← Improves all other primitives
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ POLICY   │  ← Governs what State/Context can do
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼─────┐        ┌─────▼────┐
         │ CONTEXT  │        │  STATE   │
         └──────────┘        └──────────┘
         What agent           What persists
         knows                across time
```

**Feedback** sits at the top because it refines Policy thresholds, enriches Context sources, and optimizes State management over time. A lending platform without feedback is static; one with feedback compounds intelligence.

**Policy** governs what State and Context can be accessed and how. Without Policy, Context is ungoverned and State is unprotected.

**Context** and **State** are peers: Context is what the agent knows at decision time; State is what persists across time. Context without State has no memory; State without Context has no intelligence.

---

# 12. SRAL: THE EXECUTION PATTERN THAT BINDS ALL FOUR PRIMITIVES

> **Cross-reference:** SRAL (Sense → Reason → Act → Learn) is the foundational agent lifecycle pattern defined in [unified-platform-specification.md §Agent Runtime Architecture](../unified-platform-specification.md) and derived from caas-casestudy. It is the actual execution loop that binds all four primitives into a single agent turn.

The four primitives are not independent -- they execute together in every agent cycle through the SRAL loop. One complete SRAL cycle for a credit decision:

```
┌──────────────────────────────────────────────────────────────────────┐
│  SRAL CYCLE: CREDIT DECISION FOR DAIRY FARMER                       │
│  Owning services: S13: agent-runtime (orchestrator),                 │
│  S17: context-graph, S6: sentinel-core, S7: rl-optimizer            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ SENSE (= Context primitive)                    [~200ms]     │    │
│  │ S17: context-graph + S8-S12: data connectors               │    │
│  │                                                             │    │
│  │ • Reconstruct borrower context from Context Lakehouse       │    │
│  │ • Active Ontology resolves entity across ULI, FinBox,       │    │
│  │   cooperative, weather systems (8 sources)                  │    │
│  │ • Context is versioned (time-travel capable), not fetched   │    │
│  │ • Output: enriched borrower profile with confidence score   │    │
│  │ • Kafka: context.assembled.v1 → (topic: context.enrichment)│    │
│  └─────────────────────────┬───────────────────────────────────┘    │
│                            ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ REASON (= Policy + Context primitives)         [~150ms]     │    │
│  │ S6: sentinel-core                                           │    │
│  │                                                             │    │
│  │ • Sentinel 3-layer evaluation against enriched context      │    │
│  │ • Hard gates → Scoring model → Business rules               │    │
│  │ • Confidence score modifies Policy thresholds:              │    │
│  │   conf ≥ 0.85 → auto-decide | conf < 0.60 → refer human   │    │
│  │ • Autonomy level (L0-L4) gates which decisions are allowed  │    │
│  │ • MCP tool availability filtered by Policy before agent sees│    │
│  │ • Kafka: sentinel.decision.completed → (topic: decisions)   │    │
│  └─────────────────────────┬───────────────────────────────────┘    │
│                            ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ ACT (= State mutation, gated by Policy)        [~300ms]     │    │
│  │ S3: los-engine + S13: agent-runtime                         │    │
│  │                                                             │    │
│  │ • State mutation ONLY if Policy approved                    │    │
│  │ • Loan state machine: APPLIED → APPROVED | REJECTED         │    │
│  │ • Decision trace written to immutable audit log             │    │
│  │ • Temporal activity checkpoint (survives restart)            │    │
│  │ • Idempotency key prevents duplicate state mutations        │    │
│  │ • Kafka: loan.state.changed → (topic: loan.events)         │    │
│  └─────────────────────────┬───────────────────────────────────┘    │
│                            ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ LEARN (= Feedback primitive)                   [async]      │    │
│  │ S7: rl-optimizer + S18: analytics-svc                       │    │
│  │                                                             │    │
│  │ • Emit RL signal: {decision, features, outcome (deferred)}  │    │
│  │ • Update segment-level approval/default rate trackers       │    │
│  │ • Feed champion-challenger comparison if shadow active      │    │
│  │ • Trust decay: if accuracy drops, autonomy level decreases  │    │
│  │ • Kafka: rl.signal → (topic: feedback.signals)              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Total cycle: ~650ms synchronous + async feedback                    │
│  Temporal persists state at every phase boundary.                    │
└──────────────────────────────────────────────────────────────────────┘
```

**Why SRAL matters:** Without SRAL, the four primitives are described abstractly. SRAL shows their *execution order*, *latency budget*, and *data flow* in a concrete agent turn. Every outcome in Sections 2-9 executes through this loop.

---

# 13. AUTONOMY LEVELS AND TRUST ENGINE

> **Cross-reference:** Autonomy levels L0-L4 defined in [unified-platform-specification.md §Agent Runtime Architecture](../unified-platform-specification.md), trust decay model from lia-openclaw repository analysis.

Autonomy levels are the most consequential Policy mechanism -- more impactful than RBAC, because they govern what the *agent itself* can decide, not just what a human user can access.

### 13.1 Five Autonomy Levels

| Level | Name | Agent Capability | Human Role | Example |
|-------|------|-----------------|------------|---------|
| **L0** | Fully Supervised | Draft responses, no execution | Approves every action | New lender onboarding (first 2 weeks) |
| **L1** | Suggest + Confirm | Proposes action + rationale | Confirms or overrides | Credit decisions during shadow period |
| **L2** | Act + Notify | Executes, notifies human post-facto | Reviews async, can reverse | Standard collections outreach |
| **L3** | Autonomous + Audit | Full autonomy, audit trail only | Periodic review | Returning borrower EMI inquiries |
| **L4** | Self-Improving | Modifies own parameters within bounds | Reviews parameter changes | A/B testing schedule variations |

### 13.2 Trust Decay: Feedback Modifies Policy

The Trust Engine (S6: sentinel-core + S7: rl-optimizer) continuously computes agent trust scores. When accuracy degrades, autonomy level automatically decreases:

```
trust_score = weighted_average(
    accuracy_30d    × 0.4,    # Decision accuracy over 30 days
    compliance_rate × 0.3,    # Zero-violation compliance
    escalation_apt  × 0.2,    # Appropriate escalation rate
    user_satisfaction× 0.1     # Lender/borrower CSAT
)

┌─────────────────────────────────────────────────────────────┐
│  TRUST DECAY → AUTONOMY ADJUSTMENT                           │
│                                                              │
│  trust_score ≥ 0.95  →  L3/L4 (autonomous)                  │
│  trust_score 0.85-0.94 →  L2 (act + notify)                 │
│  trust_score 0.70-0.84 →  L1 (suggest + confirm)            │
│  trust_score < 0.70    →  L0 (fully supervised) + ALERT     │
│                                                              │
│  Decay trigger: any 7-day rolling window where               │
│  accuracy drops > 5% from baseline → immediate               │
│  one-level demotion + Kafka alert on topic:                  │
│  agent.trust.decay                                           │
│                                                              │
│  Recovery: 14-day sustained performance at threshold          │
│  required to regain previous level. Recovery is slower        │
│  than decay (asymmetric by design).                          │
│                                                              │
│  Owning services: S6: sentinel-core (enforcement),           │
│  S7: rl-optimizer (computation), S18: analytics-svc (signals)│
└─────────────────────────────────────────────────────────────┘
```

### 13.3 Autonomy × Outcome Category

| Outcome Category | Starting Level | Target Level (Build 6) | Trust Threshold |
|-----------------|---------------|----------------------|----------------|
| Identity verification | L2 | L3 | 0.95 |
| Credit decisioning | L0 | L2 | 0.90 |
| Disbursement | L1 | L2 | 0.95 (financial risk) |
| Collections | L1 | L3 | 0.85 |
| Voice interactions | L2 | L3 | 0.90 |
| Lender copilot | L1 | L3 | 0.85 |
| Fraud detection | L0 | L1 | 0.98 (never fully autonomous) |

---

# 14. KAFKA EVENT TOPOLOGY: HOW PRIMITIVES CONNECT

> **Cross-reference:** Full Kafka topic registry (24 topics) in [mission-implementation-plan.md §Kafka Topic Registry](../mission-implementation-plan.md). Event-driven architecture in [unified-platform-specification.md §Infrastructure](../unified-platform-specification.md).

The four primitives are not isolated subsystems. They are connected by **Kafka events** -- the nervous system of the platform. Every State mutation emits an event. Every Feedback signal travels via Kafka. Every Context enrichment is triggered by a Kafka event.

```
┌──────────────────────────────────────────────────────────────────────┐
│  KAFKA EVENT TOPOLOGY: PRIMITIVE INTERCONNECTIONS                    │
│                                                                      │
│  STATE MUTATIONS → Events                                            │
│  ├─ loan.events           : loan state changes (applied→approved→    │
│  │                          disbursed→active→closed)                 │
│  ├─ identity.verified     : Aadhaar OTP success/failure              │
│  ├─ disbursement.initiated: transfer started (idempotency key)       │
│  └─ session.events        : agent session lifecycle                  │
│                                                                      │
│  CONTEXT ENRICHMENT ← Triggered by events, produces events           │
│  ├─ uli.fetch.requested   : trigger ULI data pull                    │
│  ├─ uli.fetch.completed   : enriched data available                  │
│  ├─ context.enrichment    : assembled context snapshot               │
│  ├─ graph.ingest          : new relationships for Neo4j              │
│  └─ weather.alert         : IMD weather event ingested               │
│                                                                      │
│  POLICY DECISIONS → Events                                           │
│  ├─ sentinel.decision.completed : credit decision with full trace    │
│  ├─ sentinel.rule.triggered     : specific rule fired (audit)        │
│  ├─ agent.trust.decay           : autonomy level changed             │
│  └─ tool.gate.blocked           : MCP tool access denied by policy   │
│                                                                      │
│  FEEDBACK SIGNALS ← Consumes all above, produces                     │
│  ├─ rl.signal             : reward/penalty for RL optimizer          │
│  ├─ monitor.trigger.intervened : lifecycle monitoring detected issue  │
│  ├─ analytics.metric      : aggregated metrics for dashboards        │
│  └─ audit.event           : compliance audit trail entry             │
│                                                                      │
│  PRIMITIVE FLOW VIA KAFKA:                                           │
│                                                                      │
│  identity.verified ──────> uli.fetch.requested ──> uli.fetch.completed│
│        (State)                  (Context trigger)     (Context ready) │
│                                                            │          │
│                                               sentinel.decision.completed│
│                                                    (Policy decision)  │
│                                                            │          │
│                                                    loan.events        │
│                                                    (State mutation)   │
│                                                            │          │
│                                                    rl.signal          │
│                                                    (Feedback emitted) │
│                                                            │          │
│                                             [feeds back to Policy     │
│                                              thresholds via           │
│                                              rl-optimizer (S7)]       │
└──────────────────────────────────────────────────────────────────────┘
```

**Key insight:** Kafka makes the primitive dependencies *observable*. Every connection between State, Context, Policy, and Feedback is a named, typed, monitored event stream. Broken connections surface immediately in observability dashboards (Grafana + Prometheus, per observability cross-cutting service).

---

# 15. FAILURE MODES AND RESILIENCE PATTERNS

> **Cross-reference:** Circuit breaker, retry, and degradation patterns in [mission-implementation-plan.md §Mission 2: Data Intelligence](../mission-implementation-plan.md).

Each primitive has characteristic failure modes. A production system must handle all of them structurally.

### 15.1 State Failures

| Failure Mode | Impact | Resilience Pattern | Owning Service |
|-------------|--------|-------------------|----------------|
| Neo4j unavailable | No graph queries, no fraud detection | Read from PostgreSQL fallback (reduced features), circuit breaker opens after 5 failures, half-open after 30s | S17: context-graph |
| Redis session loss | Agent loses conversation context | Session state also persisted in Temporal; reload from last Temporal checkpoint | S13: agent-runtime |
| Temporal worker crash | Workflow paused mid-execution | Temporal server replays from last checkpoint on new worker. Workflows are durable by design -- this is WHY we use Temporal. | Temporal orchestrator |
| PostgreSQL write failure | Loan state mutation lost | Retry with exponential backoff (max 3), idempotency key prevents duplicates on retry | S3: los-engine |

### 15.2 Context Failures

| Failure Mode | Impact | Resilience Pattern | Owning Service |
|-------------|--------|-------------------|----------------|
| ULI API timeout (>5s) | Credit decision delayed | Per-source configurable timeout (default 5s). Partial context triggers confidence recalculation. Sentinel evaluates with available data at reduced confidence → may refer to human. | S8: uli-connector |
| Bureau data stale (>30 days) | Outdated credit info | Freshness score in confidence formula drops to 0. Sentinel auto-refers to human UW if bureau freshness < 0.5. | S17: context-graph |
| Weather API down | Moratorium triggers miss | Cache last 7 days of IMD data locally. Moratorium decisions delayed (not skipped) until data available. Alert: `weather.data.stale`. | S12: weather-connector |
| Context Lakehouse corruption | Non-reproducible decisions | Versioned snapshots with time-travel rollback. Every decision references a context version ID; can reconstruct exact context for audit. | S17: context-graph, ml-platform |

### 15.3 Policy Failures

| Failure Mode | Impact | Resilience Pattern | Owning Service |
|-------------|--------|-------------------|----------------|
| Sentinel evaluation timeout | Decision blocked | Fail-closed: if Sentinel doesn't respond within 2s, default to REJECT (not APPROVE). Conservative by design -- never auto-approve on failure. | S6: sentinel-core |
| RBAC service unavailable | Tool filtering broken | Fail-closed: agent gets EMPTY tool list (no tools). Cannot operate without authorization. Alert: `auth.svc.unavailable`. | auth-svc |
| MCP tool registration stale | Agent sees outdated tools | MCP tool registry has TTL-based refresh (60s). On refresh failure, retain last known-good registry. Alert on drift > 5 minutes. | mcp-server |
| Policy rule conflict | Contradictory decisions | Rule priority ordering (hard gates > scoring > business rules). First rejecting rule wins. Conflicts logged to audit trail for review. | S6: sentinel-core |

### 15.4 Feedback Failures

| Failure Mode | Impact | Resilience Pattern | Owning Service |
|-------------|--------|-------------------|----------------|
| Kafka producer failure | Feedback signals lost | Local buffer with retry (100 events, 30s flush). If buffer full, write to local disk WAL. Replayed on recovery. | S7: rl-optimizer |
| RL optimizer divergence | Bad model updates | Shadow deployment for all RL parameter changes. 500-decision minimum before promotion. Automatic rollback if KPI degrades > 2% from baseline. | S7: rl-optimizer |
| Analytics pipeline lag | Stale dashboards | Separate real-time (Kafka Streams, <5s) and batch (Spark, daily) pipelines. Real-time for alerts, batch for reporting. Lag > 1 hour triggers alert. | S18: analytics-svc |
| Trust score computation failure | Autonomy levels frozen | Retain last known trust score. If stale > 24h, auto-demote one level (conservative). Alert: `trust.score.stale`. | S6: sentinel-core |

**Design principle:** Every failure mode follows **fail-closed** for Policy (never auto-approve) and **fail-degraded** for Context (reduce confidence, refer to human). The system becomes more conservative under stress, never more permissive.

---

# 16. CONCLUSION: ARCHITECTURE OVER INSTRUCTION

## Prompts Are Seeds. Primitives Are Soil, Water, Sunlight, and Seasons.

A seed (prompt) planted in barren soil (no State persistence) with no water (no Context enrichment), no sunlight (no Policy governance), and no seasons (no Feedback loops) will not grow.

The same seed planted in rich soil with irrigation, sunlight, and seasonal management becomes a productive system.

### The Ratio That Matters

Lines of code are a crude proxy. What matters is the **enforcement surface area** -- the count of decision points that are structurally governed vs. prompt-governed.

```
┌─────────────────────────────────────────────────────────────┐
│  ENFORCEMENT SURFACE AREA                                    │
│                                                              │
│  Prompt-governed decision points:          ~12               │
│  (system instructions, few-shot examples, tone directives)   │
│  ──────────────────────────────────────────────               │
│  Architecture-governed enforcement points:                    │
│                                                              │
│  Sentinel policy rules (S6):               ~200 rules        │
│  RBAC role-permission pairs:               ~85 pairs         │
│  MCP tool preconditions (mcp-server):      ~49 tool gates    │
│  Temporal workflow checkpoints:            ~35 activities     │
│  DB constraints (UNIQUE, CHECK, FK):       ~120 constraints  │
│  Kafka event contracts:                    ~24 topic schemas  │
│  Circuit breaker + retry policies:         ~18 policies      │
│  ──────────────────────────────────────────────               │
│  TOTAL ENFORCEMENT POINTS:                 ~531              │
│                                                              │
│  Ratio: 12 prompt directives : 531 structural gates          │
│       = 1 : 44                                               │
│                                                              │
│  For every 1 prompt-governed behavior, there are 44          │
│  architecturally-enforced decision points.                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  CODEBASE DISTRIBUTION (across 19 consolidated repos)        │
│                                                              │
│  Prompts/System Instructions:       ~200 lines    (0.3%)    │
│  State Management (Neo4j, Redis,                             │
│    Temporal, PostgreSQL):           ~18,000 lines  (24%)     │
│  Context Enrichment (ULI, bureau,                            │
│    weather, graph, lakehouse):      ~20,000 lines  (27%)     │
│  Policy Engine (Sentinel, RBAC,                              │
│    tool gates, authority, compliance): ~22,000 lines (29%)   │
│  Feedback Systems (analytics, RL,                            │
│    A/B, monitoring, alerts):        ~15,000 lines  (20%)     │
│  ──────────────────────────────────────────────               │
│  TOTAL ARCHITECTURE:                ~75,000+ lines           │
│                                                              │
│  Code ratio: 200 : 75,000 = 1 : 375                        │
│                                                              │
│  The enforcement-point ratio (1:44) is the number that       │
│  matters for regulators. The code ratio (1:375) is the       │
│  number that matters for engineering investment decisions.    │
└─────────────────────────────────────────────────────────────┘
```

### What This Means for Platform Builders

1. **Stop optimizing prompts first.** If your Aadhaar verification can be bypassed by a clever prompt, you have an architecture problem, not a prompt problem.

2. **Invest in State durability.** Temporal workflows, Neo4j graph persistence, and Redis session management are the foundation. An agent that forgets is an agent that fails.

3. **Enrich Context relentlessly.** The difference between bureau-only scoring and ULI-enriched scoring is the difference between excluding 200M borrowers and including them.

4. **Make Policy structural, not advisory.** Sentinel gates that prevent tool access are infinitely more reliable than prompt instructions that suggest tool usage patterns.

5. **Close the Feedback loop.** Every decision should generate a signal. Every signal should feed back into the system. The 10,000th loan should be measurably smarter than the 1st.

### The Architecture-Over-Instruction Principle

```
  UNRELIABLE: "Dear agent, please verify Aadhaar before accessing data."
  
  RELIABLE:   MCP tool gate (mcp-server + S6: sentinel-core). ULI tools
              absent from agent's MCP tool list until S5: identity-svc
              confirms verification_state = true. No prompt needed.
              
  UNRELIABLE: "Please don't disburse twice for the same loan."
  
  RELIABLE:   Idempotency key (disbursement_ref) with DB UNIQUE
              constraint (S3: los-engine). Temporal workflow ensures
              single execution path. Double-disburse is impossible.
              
  UNRELIABLE: "Be empathetic when the borrower mentions hardship."
  
  RELIABLE:   Distress detection (S15: voice-gateway) triggers hard
              policy switch (S6: sentinel-core rule collections.distress).
              Agent cannot continue collections -- MCP collections tools
              are removed from tool list after distress flag.
              
  UNRELIABLE: "Only show portfolio data for the user's branch."
  
  RELIABLE:   RBAC (auth-svc) filters MCP tools at mcp-server layer
              BEFORE agent sees them. Branch manager's MCP tool list
              physically excludes other branches' query capabilities.
```

**The principle:** Every behavior that matters for regulatory compliance, borrower safety, or financial integrity must be enforced by architecture (State + Context + Policy + Feedback), not by instruction (prompts).

Prompts set the tone. Architecture sets the truth.

---

*This document maps every lending outcome to the four primitives (State, Context, Policy, Feedback), demonstrating that a production lending platform is 99% architecture and 1% prompts. The primitives are the platform.*
