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
Policy Layer:
  - Tool "fetch_uli_data" requires precondition: aadhaar_otp_verified = true
  - Precondition checked by Sentinel policy engine, not by LLM
  - Tool is literally unavailable to the agent until gate is satisfied

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
│  Graph DB      Enriched        Sentinel       RL signals,           │
│  Session       profiles,       BRE,           analytics,            │
│  Temporal      ULI data,       RBI rules,     A/B results,          │
│  workflow      weather,        RBAC,          WER tracking,          │
│  state         market data     tool gates     cure rates             │
└──────────────────────────────────────────────────────────────────────┘
```

**State** = durable memory (Neo4j graph, Redis session, Temporal workflow state)
**Context** = enriched information available at decision time (ULI data, bureau, weather, cooperative calendar)
**Policy** = hard constraints on agent behavior (Sentinel BRE, RBI rules, tool preconditions, RBAC)
**Feedback** = signals that improve the system over time (RL rewards, analytics, comparison reports, WER)

This document maps **every lending outcome** to these four primitives, demonstrating why architecture -- not prompts -- determines platform quality.

---

# 2. IDENTITY VERIFICATION OUTCOMES

## 2.1 Aadhaar OTP Before Data Fetch

The foundational gate: no borrower data is accessible without verified identity.

| Primitive | Implementation |
|-----------|---------------|
| **State** | `aadhaar_otp_verified: bool`, `otp_attempts: int`, `verification_ts: datetime` in session state |
| **Context** | Masked Aadhaar (XXXX-XXXX-1234), verification method (OTP/biometric), device fingerprint |
| **Policy** | Structural gate: ULI tools (`fetch_credit_bureau`, `fetch_bank_statement`, `fetch_gstin`) require `aadhaar_otp_verified = true` as Sentinel precondition. Tool literally absent from agent's available tools until gate passes. |
| **Feedback** | `verification_success_rate`, `avg_time_to_verify`, `otp_delivery_failure_rate` → alerts if success < 95% |

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

The 3-layer Sentinel BRE evaluates every credit decision through Policy, with Context providing the enriched profile.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Decision trace: full audit trail of every rule evaluation, stored in Temporal workflow |
| **Context** | Enriched borrower profile assembled from ULI (bureau score, bank statements, GSTIN), FinBox (device, BankConnect), cooperative data (milk pour records, crop yield), weather (IMD district-level). Context is 10-50x richer than traditional bureau-only. |
| **Policy** | 3-layer Sentinel: (1) Hard gates (age, geography, blacklist), (2) Scoring model (segment-specific), (3) Business rules (ticket size limits, FOIR caps, PSL classification). Each layer can REJECT, APPROVE, or REFER. All layers must APPROVE for auto-decision. |
| **Feedback** | `approval_rate_by_segment`, `default_rate_30dpd`, `model_discrimination` (KS, Gini, AUC) → triggers model retraining if AUC < 0.70 |

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

## 7.2 FDCPA-Equivalent Compliance

Indian collections must comply with RBI Fair Practices Code and evolving digital lending rules.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Call log: `{timestamp, duration, agent_type (AI/human), consent_given, recording_id, outcome}`. Contact attempt counter per day/week. |
| **Context** | Borrower's preferred contact time, language preference, prior interaction sentiment, current DPD (days past due) bucket. |
| **Policy** | Calling hours: 8 AM - 7 PM borrower's local time (enforced by scheduler, not prompt). Max 3 contact attempts per week. Consent verification at call start (recorded). No third-party contact without borrower consent. No workplace calls. Language must match borrower preference. |
| **Feedback** | `compliance_violation_count` (must be zero), `consent_recording_coverage` (must be 100%), `calling_hours_violation_count` → automated audit trail for regulator. |

---

# 8. LENDER OPERATIONS OUTCOMES

## 8.1 Copilot Natural Language Query

Lender operations teams interact with the platform through natural language, routed to appropriate tools.

| Primitive | Implementation |
|-----------|---------------|
| **State** | Query history: prior questions and results for context continuity. User session state for multi-turn exploration ("Now filter that by Gujarat" refers to previous query results). |
| **Context** | MCP tool routing: "Show me all loans in Kutch district with DPD > 30" → routes to `query_portfolio` tool with parameters `{district: "Kutch", dpd_min: 30}`. Agent selects appropriate tool from 80+ available MCP tools based on query intent. |
| **Policy** | RBAC per role: Branch Manager sees only their branch portfolio. Regional Head sees their region. Risk Officer sees risk metrics but not PII. Collections Manager sees DPD buckets but not credit scores. Tool availability filtered by role before agent sees tool list. |
| **Feedback** | `query_success_rate` (did the tool return useful results?), `query_latency`, `tool_routing_accuracy` (was the right tool selected?), `user_satisfaction_rating` → improves tool descriptions and routing logic. |

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

## 10.1 Identity Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Aadhaar OTP verification | otp_verified flag, attempt counter | Masked Aadhaar, device fingerprint | Structural gate on ULI tools | Success rate, delivery failure rate |
| Failed verification handling | Attempt counter, lock status | Failure reason, remaining attempts | 3-attempt limit, 30-min cooldown | Failure reason distribution |
| Returning borrower recognition | Graph memory (loan history, interactions) | Personalized greeting, prior context | Re-verification rules by recency | Returning vs new conversion rate |
| PAN-Aadhaar linkage check | Linkage status, verification timestamp | PAN details, name match score | Mandatory pre-disbursement check | Linkage failure rate by segment |

## 10.2 Decisioning Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Sentinel 3-layer evaluation | Decision trace, rule evaluation log | Enriched profile (ULI + alt data) | Hard gates, scoring model, biz rules | Approval rate, default rate, AUC |
| Shadow/champion-challenger | Parallel decision traces | Identical context to both models | Champion live, challenger shadow-only | Comparison analytics, AUC delta |
| Explainability | SHAP values, reason codes, full trace | Top-5 feature contributions | Reason codes required for rejection | Reason code distribution, comprehension |
| Segment-specific scoring | Segment model selection trace | Segment-specific features (milk, crop) | Segment threshold configuration | Per-segment default rates, KS stat |

## 10.3 Product Design Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Seasonal EMI schedule | Schedule object in LOS | Cooperative calendar, harvest dates | sum(EMIs) = total obligation | On-time rate: seasonal vs uniform |
| Weather moratorium | Moratorium status in loan FSM | IMD district weather data | Auto-moratorium trigger thresholds | Cure rate post-moratorium |
| Flexible tenure | Tenure selection trace | Borrower cash flow analysis | Min/max tenure by segment | Prepayment rate by tenure |
| Top-up on existing loan | Existing loan state, eligibility flag | Repayment history, capacity growth | Top-up criteria (min 6 months clean) | Top-up default rate vs fresh |

## 10.4 Disbursement Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Temporal workflow durability | Workflow checkpoints, activity state | Loan details, bank account, vault creds | Authority limit matrix | Success rate, retry rate |
| Idempotent disbursement | disbursement_ref UUID, DB unique | Prior attempts, bank responses | Exactly-once semantics | Duplicate attempt count |
| Co-lending split | Split agreement, per-tranche tracking | Lender profiles, RBI CLA rules | 80/20 minimum split, dual UW | Portfolio quality by partner |
| NACH mandate registration | Mandate status, UMRN reference | Borrower bank details, mandate amount | Mandate before disbursement (gate) | Mandate success rate by bank |

## 10.5 Lifecycle Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Weather moratorium (auto) | IMD data in graph, moratorium status | Pincode weather, livelihood type | Auto-trigger thresholds by crop/dairy | Cure rate, RL-optimized duration |
| Graduation offer | 12-month history, eligibility flag | Income trend, market rates, competition | Graduation criteria (clean repayment) | Acceptance rate, post-grad default |
| Fraud ring detection | Graph relationships (device, address) | Multi-hop query results, timing analysis | 3+ borrowers, 2+ shared attributes | Precision, recall, false positive rate |
| Early warning signals | Payment pattern features, bureau alerts | Cash flow deterioration, market stress | EWS classification (SMA-0/1/2) | EWS prediction accuracy |

## 10.6 Collections Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Context-aware collections | Payment history, weather/health links | Verified hardship reason, district data | Empathetic tone when hardship verified | Cure rate by approach |
| FDCPA-equivalent compliance | Call log, consent records, contact counter | Preferred time, language, DPD bucket | Calling hours, max contacts, consent | Compliance violations (target: 0) |
| Restructuring offer | Current loan terms, hardship verification | Borrower capacity reassessment | Restructuring eligibility by DPD bucket | Restructuring success rate |
| Legal escalation | DPD history, communication trail | Total outstanding, collateral value | DPD > 180 + communication exhausted | Recovery rate post-legal |

## 10.7 Lender Operations Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Copilot NL query | Query history, session context | MCP tool routing, 80+ tools | RBAC per role, tool filtering | Query success rate, routing accuracy |
| Policy shadow testing | Shadow decision log, both outcomes | Policy change description, backtest | Shadow-only (no real impact) | Comparison report, impact analysis |
| PSL classification | Classification audit trail | Borrower profile, RBI PSL norms | Auto-classification rules | Classification accuracy (auditor-verified) |
| MIS/regulatory reporting | Report generation state, delivery log | Aggregated portfolio metrics | Report templates per regulator | Report accuracy, on-time delivery |

## 10.8 Voice Outcomes

| Outcome | State | Context | Policy | Feedback |
|---------|-------|---------|--------|----------|
| Gujarati ASR accuracy | ASR model version, custom vocab | Lending vocab boost, code-mixing | WER < 15%, fallback rules | WER by language, by term category |
| Fast brain acknowledgment | Keyword-to-ack mapping (Redis) | None (pattern match only) | Ack only, never decisions/amounts | Trigger rate, accuracy, latency |
| Empathy detection | Sentiment trajectory per turn | Distress keywords, tonal analysis | Distress → human escalation (hard) | Detection accuracy, escalation outcomes |
| Language switching | Current language state, switch history | Detected language vs preferred | Supported language set per region | Switch frequency, post-switch WER |

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

# 12. CONCLUSION: ARCHITECTURE OVER INSTRUCTION

## Prompts Are Seeds. Primitives Are Soil, Water, Sunlight, and Seasons.

A seed (prompt) planted in barren soil (no State persistence) with no water (no Context enrichment), no sunlight (no Policy governance), and no seasons (no Feedback loops) will not grow.

The same seed planted in rich soil with irrigation, sunlight, and seasonal management becomes a productive system.

### The Ratio That Matters

```
┌────────────────────────────────────────────────────────┐
│  LENDING PLATFORM: WHERE THE LINES OF CODE LIVE        │
│                                                        │
│  Prompts/System Instructions:     ~200 lines           │
│  ──────────────────────────────────────────────         │
│  State Management (Neo4j schemas,                      │
│    Redis session, Temporal workflows,                   │
│    PostgreSQL models):             ~3,000 lines         │
│                                                        │
│  Context Enrichment (ULI integration,                  │
│    bureau parsing, weather ingestion,                   │
│    cooperative calendar, graph queries): ~4,000 lines   │
│                                                        │
│  Policy Engine (Sentinel BRE rules,                    │
│    RBAC configuration, tool gates,                      │
│    authority matrices, compliance):  ~5,000 lines       │
│                                                        │
│  Feedback Systems (analytics pipelines,                │
│    RL reward functions, A/B framework,                  │
│    monitoring dashboards, alerts):   ~3,000 lines       │
│  ──────────────────────────────────────────────         │
│  TOTAL ARCHITECTURE:              ~15,000+ lines        │
│                                                        │
│  Ratio: 200 : 15,000+ = 1 : 75                        │
│                                                        │
│  For every 1 line of prompt, there are 75 lines        │
│  of architecture making that prompt effective.          │
└────────────────────────────────────────────────────────┘
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
  
  RELIABLE:   Policy gate on tool availability. Tool does not exist
              in agent's tool list until verification state = true.
              
  UNRELIABLE: "Please don't disburse twice for the same loan."
  
  RELIABLE:   Idempotency key (disbursement_ref) with DB UNIQUE
              constraint. Temporal workflow ensures single execution.
              
  UNRELIABLE: "Be empathetic when the borrower mentions hardship."
  
  RELIABLE:   Distress detection triggers hard policy switch.
              Agent literally cannot continue collections script
              after distress flag. Escalation is automatic.
              
  UNRELIABLE: "Only show portfolio data for the user's branch."
  
  RELIABLE:   RBAC filters available tools BEFORE agent sees them.
              Branch manager's tool list physically excludes other
              branches' query capabilities.
```

**The principle:** Every behavior that matters for regulatory compliance, borrower safety, or financial integrity must be enforced by architecture (State + Context + Policy + Feedback), not by instruction (prompts).

Prompts set the tone. Architecture sets the truth.

---

*This document maps every lending outcome to the four primitives (State, Context, Policy, Feedback), demonstrating that a production lending platform is 99% architecture and 1% prompts. The primitives are the platform.*
