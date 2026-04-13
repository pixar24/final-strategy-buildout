# Field Deployment Engineering — Lender Engagement Model

> Adapted from the FDE model for AI-first Indian lending platform.
> Engineers embedded with lenders who ship autonomous lending agents to production.

---

## Table of Contents

1. [What is Lender Engagement Engineering](#what-is-lender-engagement-engineering)
2. [Lender Engagement Engineer Role](#lender-engagement-engineer-role)
3. [Lender Engagement Lifecycle](#lender-engagement-lifecycle)
4. [Organizational Structure](#organizational-structure)
5. [Decision Framework](#decision-framework)
6. [Tooling](#tooling)
7. [Knowledge Transfer](#knowledge-transfer)
8. [Metrics](#metrics)

---

## What is Lender Engagement Engineering

### Not Software Licenses

The lending platform is not a SaaS product you configure via a settings page.
Every lender has unique risk appetites, regulatory interpretations, borrower
segments, language requirements, and operational cultures. A dashboard with
toggles does not capture this complexity.

The traditional model — sell a license, hand over documentation, wait for
support tickets — fails catastrophically for autonomous lending agents because:

1. **Regulatory stakes are non-negotiable.** A misconfigured policy can violate
   RBI norms. There is no "oops, we'll fix it in the next sprint."

2. **Borrower segments are deeply contextual.** A dairy farmer in Gujarat and a
   gig worker in Bangalore require fundamentally different journeys, not just
   different templates.

3. **Trust is earned through competence, not contracts.** A lender's CRO will
   not approve autonomous disbursement based on a slide deck. They need to see
   an engineer who understands their risk framework building the policies.

4. **The platform improves through lender engagement.** Every lender deployment
   surfaces edge cases, regulatory nuances, and borrower behaviors that feed
   back into the platform. The engineer is the conduit.

### Embedded Engineers Who Understand RBI + Platform + Lending

A Lender Engagement Engineer (LEE) is not a solutions architect, not a
customer success manager, not a support engineer. They are a **hybrid**:

```
┌───────────────────────────────────────────────────┐
│           Lender Engagement Engineer               │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Platform  │  │ Lending   │  │ ML/AI        │    │
│  │ Engineer  │  │ Domain    │  │ Intuition    │    │
│  │           │  │ Expert    │  │              │    │
│  │ Python    │  │ RBI regs  │  │ Prompt eng   │    │
│  │ YAML      │  │ PSL norms │  │ Eval design  │    │
│  │ MCP tools │  │ KYC/AML   │  │ Bias detect  │    │
│  │ DAG flows │  │ Fair Prac │  │ Quality gates│    │
│  │ CI/CD     │  │ Co-lending│  │ RL signals   │    │
│  └──────────┘  └──────────┘  └──────────────┘    │
│                                                    │
│  + Lender Relationship Management                  │
│  + Stakeholder Communication                       │
│  + Training & Knowledge Transfer                   │
└───────────────────────────────────────────────────┘
```

### Services-Led Product Strategy

This is not services bolted onto a product. The services ARE the product
strategy:

- **Every lender engagement improves the platform.** The LEE files platform
  feedback tickets that become features. Segment-specific patterns become
  reusable templates. Regulatory edge cases become policy primitives.

- **The platform enables the engineer.** Without Agent Studio, Workspace
  Service, and Evaluation Service, the LEE would be writing custom code for
  every lender. The platform provides leverage; the engineer provides judgment.

- **Scale comes from self-sufficiency, not headcount.** The goal is not to
  permanently embed engineers. It is to bring lenders to self-sufficiency
  (L1 -> L4) while the platform absorbs complexity.

```
Revenue Model:
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  Engagement Fee ─────> Covers LEE time + platform    │
  │  (months 1-6)          setup + first segments        │
  │                                                      │
  │  Platform Fee ────────> Per-conversation pricing      │
  │  (month 4+)            scales with lender volume      │
  │                                                      │
  │  Expansion Fee ───────> New segments, new channels,   │
  │  (ongoing)             new lender integrations        │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

---

## Lender Engagement Engineer Role

### Day-to-Day Allocation

The LEE's time is structured but not rigid. Allocation shifts across the
engagement lifecycle, but the steady-state breakdown is:

| Activity                | % Time | Description                                                |
|-------------------------|--------|------------------------------------------------------------|
| Segment Configuration   | 20%    | Knowledge partitions, product specs, rate configs           |
| Journey Design          | 20%    | DAG flows, prompt templates, channel-specific UX            |
| Policy Configuration    | 15%    | 3-layer policy YAML, authority limits, compliance rules     |
| Eval Authoring          | 15%    | tau-bench scenarios, pass^k targets, adversarial tests      |
| Production Monitoring   | 15%    | Trace review, Explorer alerts, incident response            |
| Lender Training         | 10%    | Agent Studio training, conversation review, eval walkthrough|
| Platform Feedback       | 5%     | Feature requests, bug reports, pattern proposals            |

### Time Allocation Across Lifecycle Phases

```
                Build Phase         Pilot Phase         Steady State
                (weeks 5-10)       (weeks 13-16)       (week 17+)

Segment Config  ████████ 30%       ████ 15%            ██ 10%
Journey Design  ████████ 30%       ███ 12%             ██ 8%
Policy Config   ██████ 20%         ███ 10%             ██ 8%
Eval Authoring  ████ 10%           ██████ 25%          ███ 12%
Prod Monitoring ░░ 0%              ██████ 22%          ██████ 25%
Lender Training ██ 5%              ████ 12%            ██████ 25%
Platform Fdbk   ██ 5%              ██ 4%               ███ 12%
```

### Skill Stack

The LEE is a T-shaped engineer. Deep in platform + lending, with breadth across
ML, data, and relationship management.

**Must-have:**

| Skill                    | Depth    | Why                                              |
|--------------------------|----------|--------------------------------------------------|
| Python                   | Deep     | Platform tools, eval scripts, data pipelines     |
| YAML / Config-as-Code   | Deep     | Policies, journeys, knowledge partitions         |
| RBI Regulations          | Working  | DLG, Fair Practices, KYC, CLA 2026, PSL norms   |
| Lending Domain           | Working  | Credit decisioning, collections, loss mitigation |
| Prompt Engineering       | Deep     | System prompts, few-shot examples, tone control  |
| Evaluation Design        | Deep     | tau-bench authoring, pass^k methodology          |
| Stakeholder Management   | Working  | CRO communication, compliance officer alignment  |

**Nice-to-have:**

| Skill                    | Depth    | Why                                              |
|--------------------------|----------|--------------------------------------------------|
| ML/AI Fundamentals       | Aware    | Model behavior intuition, RL signal design       |
| Indian Languages         | Aware    | Hindi/Gujarati/regional language nuances         |
| Data Analysis            | Working  | Explorer insights, trace analysis                |
| Agricultural Lending     | Working  | Crop cycles, weather triggers, cooperative norms |

---

## Lender Engagement Lifecycle

### Timeline Overview

```
Pre-sales    Kickoff      Build         Shadow    Pilot       Production   Steady
(2 weeks)    (2 weeks)    (6 weeks)     (2 weeks) (4 weeks)   (ongoing)    State
│            │            │             │         │           │            │
├── Week 1-2 ├── Week 3-4 ├── Week 5-10 ├── Wk 11 ├── Wk 13  ├── Week 17+ │
│            │            │             │   -12   │   -16     │            │
│ Assess     │ Align      │ Build       │ Shadow  │ Graduated │ Monitor    │ Self-
│ Scope      │ Staff      │ Ship        │ Verify  │ Rollout   │ Improve    │ Sufficient
│ Qualify     │ Plan       │ Test        │ Tune    │ Expand    │ Transfer   │
│            │            │             │         │           │            │
▼            ▼            ▼             ▼         ▼           ▼            ▼
Scoping      Kickoff      Journey +     Shadow    First       Full         Lender
Doc          Charter      Policy +      Report    Pilot       Production   L3-L4
             + Team       Eval Suite              Live
```

**First pilot conversation: Week 12-13.**

### Phase Details

#### Pre-sales (Weeks 1-2)

| Activity                         | Output                              | Owner          |
|----------------------------------|-------------------------------------|----------------|
| Lender discovery call            | Needs assessment                    | Sales + LEE    |
| Regulatory landscape review      | Compliance requirements list        | LEE            |
| Segment prioritization           | Priority matrix (impact x feasibility)| LEE + Lender |
| Technical feasibility assessment | Integration checklist               | LEE + Platform |
| Effort estimation                | SOW draft                           | LEE            |
| Go/no-go recommendation          | Internal review document            | LEE Lead       |

#### Kickoff (Weeks 3-4)

| Activity                         | Output                              | Owner          |
|----------------------------------|-------------------------------------|----------------|
| Stakeholder mapping workshop     | RACI matrix                         | LEE            |
| Segment deep-dive                | Borrower personas per segment       | LEE + Lender   |
| Compliance alignment             | Regulatory constraint document      | LEE + Compliance|
| Integration planning             | API inventory + timeline            | LEE + IT       |
| Workspace provisioning           | Draft + QA workspaces ready         | Platform       |
| Project charter sign-off         | Approved charter + milestones       | CRO            |

#### Build (Weeks 5-10)

| Week | Focus                                  | Deliverable                         |
|------|----------------------------------------|-------------------------------------|
| 5-6  | Knowledge engineering + segment config | Populated knowledge partitions      |
| 6-7  | Journey design (priority segment)      | DAG flows + prompt templates        |
| 7-8  | Tool integration + policy config       | MCP manifest + 3-layer policies     |
| 8-9  | Eval suite authoring                   | tau-bench scenarios (15+ per segment)|
| 9-10 | Iteration: fix failures, tune prompts  | pass^k >= 0.80 (QA)                |

#### Shadow Test (Weeks 11-12)

| Activity                         | Output                              | Threshold      |
|----------------------------------|-------------------------------------|----------------|
| Mirror live traffic to agent     | Shadow transcripts                  | 200+ sessions  |
| Risk team review                 | Agreement scores                    | >= 92%         |
| Policy compliance audit          | Violation report                    | 0 violations   |
| Tone/empathy assessment          | Human eval scores                   | >= 4.0 / 5.0   |
| Go/no-go for pilot               | Shadow summary report               | CRO sign-off   |

#### Pilot (Weeks 13-16)

| Week | Traffic % | Segments Active        | Key Metrics Monitored              |
|------|-----------|------------------------|-------------------------------------|
| 13   | 5%        | FAQ + status only       | Error rate, escalation rate         |
| 14   | 25%       | + pre-approved loans    | + conversion rate, disbursement acc |
| 15   | 50%       | + moratorium, graduation| + compliance score, CSAT            |
| 16   | 100%      | All conversation types  | All metrics at full scale           |

#### Production (Week 17+)

Ongoing operations: monitoring, incident response, continuous improvement,
segment expansion. The LEE shifts from builder to advisor.

#### Steady State

The lender operates with increasing self-sufficiency. The LEE transitions to
advisory role, handling escalations and new segment expansion only.

### Lender Self-Sufficiency Levels

| Level | Name            | Capabilities                                           | LEE Role          |
|-------|-----------------|--------------------------------------------------------|-------------------|
| L1    | Dependent       | Cannot operate without LEE. All changes go through LEE | Builder           |
| L2    | Assisted        | Can review conversations, minor config edits in Studio | Builder + Trainer |
| L3    | Capable         | Can author evals, modify journeys, adjust policies     | Advisor           |
| L4    | Self-sufficient | Full operational independence. LEE for expansion only  | On-call           |

```
Self-Sufficiency Progression:
                                                         
L1 ─────────────> L2 ─────────────> L3 ─────────────> L4
  Kickoff to        Build to           Pilot to          Production
  Build end         Shadow end         Prod stable       month 3+
  (Week 3-10)       (Week 10-12)       (Week 13-20)      (Week 20+)
                                                         
  LEE: 100%         LEE: 80%           LEE: 40%          LEE: 10%
  Lender: 0%        Lender: 20%        Lender: 60%       Lender: 90%
```

---

## Organizational Structure

### Vertical Leads by Lending Segment

The LEE team is organized by lending vertical, not by lender. This ensures
deep segment expertise and cross-lender pattern sharing.

```
                    ┌──────────────────────┐
                    │   Head of Lender     │
                    │   Engagement         │
                    └──────────┬───────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
┌─────────v─────────┐ ┌──────v────────┐ ┌─────────v─────────┐
│  Agriculture       │ │ Micro-        │ │  Gig Economy       │
│  Vertical Lead     │ │ Enterprise    │ │  Vertical Lead     │
│                    │ │ Vertical Lead │ │                    │
│  LEEs: 3-4         │ │ LEEs: 3-4    │ │  LEEs: 2-3         │
│                    │ │              │ │                    │
│  Segments:         │ │ Segments:    │ │  Segments:         │
│  - Dairy           │ │ - Street     │ │  - Ride-hailing    │
│  - Crop finance    │ │   vendors    │ │  - Delivery        │
│  - Fisheries       │ │ - Artisans   │ │  - Freelancers     │
│  - Poultry         │ │ - Retailers  │ │  - Content creators│
│  - Allied agri     │ │ - Workshops  │ │  - Home services   │
└───────────────────┘ └──────────────┘ └───────────────────┘
```

### Feedback Loop to Platform Team

The LEE team is the platform team's highest-signal input channel. Every
engagement generates platform improvements.

```
LEE Feedback Pipeline:

Lender Engagement ──> LEE observes gap/friction
                          │
                     ┌────v─────┐
                     │ Classify  │
                     │           │
                     ├── Bug     │──> P1/P2 ticket -> Platform fix
                     ├── Feature │──> RFC -> Platform roadmap
                     ├── Pattern │──> Template -> Shared library
                     └── Insight │──> Knowledge base -> Documentation
                     
Monthly LEE <-> Platform sync:
  - Top 5 friction points from field
  - Pattern proposals for platform primitives
  - Eval methodology improvements
  - Tool gap analysis
```

| Feedback Category | Example                                           | Platform Action               |
|-------------------|---------------------------------------------------|-------------------------------|
| Bug               | "Circuit breaker not resetting after ULI recovery"| Hotfix                        |
| Feature           | "Need weather-triggered auto-moratorium policy"   | Roadmap item (Q+1)           |
| Pattern           | "Agriculture KCC journey reusable across lenders" | Template in shared library   |
| Insight           | "Gujarati ASR needs crop-specific vocabulary"     | ASR model enhancement        |
| Tooling Gap       | "No bulk eval re-run capability in CLI"           | CLI feature addition         |

---

## Decision Framework

Clear ownership prevents decision paralysis. In lending, ambiguity about who
decides what costs weeks and creates compliance risk.

### Engineer Decides

These are technical-operational decisions within the platform's capability
envelope. The LEE has full authority.

| Decision                     | Rationale                                          |
|------------------------------|---------------------------------------------------|
| Segment configuration        | Knowledge partition structure, data mapping        |
| Journey structure            | DAG flow design, node sequencing, branching logic  |
| Model routing                | Which LLM for which task, fallback chains          |
| Eval scenario design         | tau-bench scenarios, adversarial test cases        |
| Prompt engineering           | System prompts, few-shot examples, tone templates  |
| Tool wiring                  | MCP tool selection, circuit breaker thresholds     |
| Workspace promotion timing   | When to promote draft -> QA -> staging             |
| Shadow mode duration         | How long to shadow before pilot recommendation     |

### Lender Decides

These are business-risk decisions that reflect the lender's strategy and
regulatory interpretation. The LEE advises but does not decide.

| Decision                     | Rationale                                          |
|------------------------------|---------------------------------------------------|
| Risk appetite                | How aggressive on credit scoring, DPD tolerance    |
| Authority limits             | Fee waiver caps, disbursement limits, auto-approve |
| Brand voice                  | Tone guidelines, prohibited phrases, formality     |
| PSL targets                  | Priority sector allocation, segment prioritization |
| Escalation thresholds        | When agent must hand off to human                  |
| Collections strategy         | Settlement ranges, legal action triggers           |
| Rollout pace                 | How fast to move through 5% -> 100%               |
| Go-live approval             | Final CRO sign-off for production                  |

### Platform Decides

These are architectural decisions that ensure consistency, security, and
scalability across all lender deployments.

| Decision                     | Rationale                                          |
|------------------------------|---------------------------------------------------|
| Policy enforcement arch.     | How policies are evaluated (not which policies)    |
| Trace format                 | Standard trace schema for audit and analysis       |
| Eval methodology             | pass^k metric, tau-bench framework, gating rules   |
| Tool protocol (MCP)          | Standard tool interface, session state machine     |
| Knowledge partition schema   | Standard partition structure, versioning approach   |
| Security model               | Authentication, encryption, data residency         |
| Workspace architecture       | Environment isolation, promotion pipeline          |
| Circuit breaker defaults     | Default thresholds (overridable per engagement)    |

### Decision Matrix Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    DECISION AUTHORITY                         │
├──────────────────┬───────────────┬──────────────────────────┤
│   ENGINEER       │    LENDER     │     PLATFORM             │
│   (LEE)          │    (CRO+Ops)  │     (Platform Team)      │
├──────────────────┼───────────────┼──────────────────────────┤
│ HOW to build     │ WHAT to build │ WHERE it runs            │
│ Journey DAGs     │ Risk appetite │ Policy engine            │
│ Prompt design    │ Authority $   │ Trace format             │
│ Eval scenarios   │ Brand voice   │ Eval methodology         │
│ Tool wiring      │ PSL targets   │ Security model           │
│ Shadow duration  │ Go-live call  │ Tool protocol            │
└──────────────────┴───────────────┴──────────────────────────┘
```

---

## Tooling

The LEE operates through a standardized toolchain. Every tool has a specific
role in the ADLC pipeline. No ad-hoc scripts, no shadow spreadsheets.

### Tool Inventory

| Tool               | Purpose                                         | Primary User    | ADLC Phase    |
|--------------------|-------------------------------------------------|-----------------|---------------|
| Agent Studio       | Visual journey design, prompt editing, testing   | LEE + Lender    | 3, 5, 8       |
| Workspace Service  | Environment management (draft/QA/staging/prod)   | LEE             | 7             |
| Evaluation Service | tau-bench execution, pass^k scoring              | LEE             | 6, 8          |
| Experience Manager | Stratified sampling, human review queue           | LEE + Lender    | 8             |
| Explorer           | Failure clustering, anomaly detection, trends     | LEE             | 8             |
| Trace Service      | Conversation traces, audit trail, compliance log  | LEE + Compliance| All           |
| CLI / Makefile     | Automation: config deploy, eval runs, promotions  | LEE             | All           |

### Tool-Phase Mapping

```
Phase 1: Discovery      -> Trace Service (analyze existing interactions)
Phase 2: Knowledge Eng  -> CLI (partition management, knowledge import)
Phase 3: Journey Design  -> Agent Studio (visual DAG builder)
Phase 4: Tool Integration-> CLI (MCP manifest, circuit breaker config)
Phase 5: Policy Config   -> Agent Studio + CLI (policy YAML authoring)
Phase 6: Eval & Gating   -> Evaluation Service + CLI (tau-bench runs)
Phase 7: Staged Rollout  -> Workspace Service (promote, rollback)
Phase 8: Continuous Impr -> Experience Manager + Explorer (monitor, learn)
```

### Agent Studio: The LEE's Primary Interface

Agent Studio is not a demo tool. It is the production interface where journeys
are designed, tested, and iterated. Key capabilities:

- **Journey Builder**: Visual DAG editor with segment/channel/language config
- **Prompt Workshop**: Test prompts against real conversation fragments
- **Policy Editor**: Author and test policy YAML with instant validation
- **Eval Dashboard**: View pass^k scores, drill into failures
- **Conversation Viewer**: Review production traces with policy annotations
- **Knowledge Manager**: Browse, edit, and version knowledge partitions

### CLI / Makefile Workflows

```bash
# Knowledge partition management
make knowledge-import LENDER=hdfc PARTITION=rates SOURCE=./rates.yaml
make knowledge-validate LENDER=hdfc
make knowledge-diff LENDER=hdfc FROM=v1.2 TO=v1.3

# Evaluation
make eval-run LENDER=hdfc ENV=qa SUITE=full
make eval-report LENDER=hdfc ENV=qa RUN=latest
make eval-compare LENDER=hdfc RUN_A=20260401 RUN_B=20260408

# Workspace management
make workspace-promote LENDER=hdfc FROM=qa TO=staging
make workspace-rollback LENDER=hdfc ENV=staging TO=v2.1
make workspace-status LENDER=hdfc

# Production monitoring
make traces-export LENDER=hdfc ENV=production SINCE=24h
make explorer-report LENDER=hdfc PERIOD=weekly
make sampling-review LENDER=hdfc QUEUE=escalated
```

---

## Knowledge Transfer

### Objective

Bring lender teams from L1 (fully dependent) to L4 (self-sufficient) through
structured training on the platform, evaluation methodology, and operational
practices.

### Training Program

| Module                     | Duration | Audience               | Prerequisites     | Outcome                       |
|----------------------------|----------|------------------------|--------------------|-------------------------------|
| 1. Platform Overview       | 2 hours  | All stakeholders       | None               | Understand agent capabilities |
| 2. Agent Studio Basics     | 4 hours  | Ops + Compliance       | Module 1           | Navigate Studio, view traces  |
| 3. Conversation Review     | 4 hours  | Ops + Compliance       | Module 2           | Review and score conversations|
| 4. Eval Interpretation     | 3 hours  | Ops + Risk             | Module 2           | Read pass^k reports, identify gaps|
| 5. Journey Basics          | 6 hours  | Ops (power users)      | Module 2           | Understand DAGs, minor edits  |
| 6. Policy Fundamentals     | 4 hours  | Compliance + Risk      | Module 2           | Understand 3-layer model      |
| 7. Knowledge Management    | 4 hours  | Ops (power users)      | Module 5           | Update FAQ, rate partitions   |
| 8. Eval Authoring          | 6 hours  | Ops (advanced)         | Module 4 + 5       | Write tau-bench scenarios     |
| 9. Production Operations   | 4 hours  | Ops (advanced)         | Modules 3, 4, 7    | Handle alerts, minor incidents|
| 10. Advanced Journey Design| 8 hours  | Ops (L4 candidates)    | Module 5 + 8       | Design new segment journeys   |

### Training Timeline Aligned to Engagement

```
Kickoff        Build          Shadow         Pilot          Steady State
(Week 3-4)     (Week 5-10)    (Week 11-12)   (Week 13-16)   (Week 17+)
    │              │              │              │              │
    │  Module 1    │  Module 2-4  │  Module 5-6  │  Module 7-8  │  Module 9-10
    │  (overview)  │  (basics)    │  (journey +  │  (knowledge  │  (operations
    │              │              │   policy)    │   + eval)    │   + advanced)
    │              │              │              │              │
    ▼              ▼              ▼              ▼              ▼
    L1             L1 -> L2       L2             L2 -> L3       L3 -> L4
```

### Self-Sufficiency Spectrum

```
Capability                          L1    L2    L3    L4
─────────────────────────────────────────────────────────
View conversation traces            ──    ✓     ✓     ✓
Review and score conversations      ──    ✓     ✓     ✓
Interpret eval reports              ──    ✓     ✓     ✓
Update FAQ knowledge partitions     ──    ──    ✓     ✓
Update rate/fee configurations      ──    ──    ✓     ✓
Modify journey prompts              ──    ──    ✓     ✓
Write tau-bench scenarios           ──    ──    ✓     ✓
Adjust authority limits             ──    ──    ✓     ✓
Design new segment journeys         ──    ──    ──    ✓
Author new policy rules             ──    ──    ──    ✓
Manage workspace promotions         ──    ──    ──    ✓
Handle production incidents         ──    ──    ──    ✓
Onboard new borrower segments       ──    ──    ──    ✓
```

### Knowledge Transfer Artifacts

| Artifact                         | Format      | Maintained By | Updated       |
|----------------------------------|-------------|---------------|---------------|
| Lender Runbook                   | Markdown    | LEE           | Per milestone |
| Segment Configuration Guide      | YAML + docs | LEE           | Per segment   |
| Eval Scenario Library            | JSON + docs | LEE + Lender  | Weekly        |
| Incident Response Playbook       | Markdown    | LEE           | Per incident  |
| Journey Design Patterns          | Studio + MD | LEE           | Per journey   |
| Regulatory Update Procedure      | Markdown    | LEE + Compliance| Per circular |
| Training Recordings              | Video       | LEE           | Per module    |

---

## Metrics

### Why These Metrics

Metrics must answer four questions:
1. **Speed**: How fast are we getting lenders to production?
2. **Quality**: How good are the agents we deploy?
3. **Transfer**: Are lenders becoming self-sufficient?
4. **Platform**: Is the platform improving from engagements?

### Speed Metrics

| Metric                     | Target          | Measurement                              |
|----------------------------|-----------------|------------------------------------------|
| Time to first pilot        | 12 weeks        | Kickoff to first live borrower interaction|
| Time to full production    | 16 weeks        | Kickoff to 100% traffic                  |
| Time to new segment        | 4 weeks         | Decision to segment live in production   |
| Shadow-to-pilot conversion | < 2 weeks       | Shadow complete to pilot start           |
| Eval iteration cycle       | < 3 days        | Failure identified to fix deployed in QA |
| Incident resolution time   | < 4 hours       | Alert to resolution (P1)                |
| Knowledge update cycle     | < 24 hours      | Regulatory change to knowledge deployed  |

### Quality Metrics

| Metric                     | Target          | Measurement                              |
|----------------------------|-----------------|------------------------------------------|
| Resolution rate            | >= 78%          | Conversations resolved without escalation|
| pass^k (k=5, production)  | >= 0.95         | tau-bench pass^k on production eval suite|
| Compliance score           | 100%            | Policy violations per 1000 conversations |
| ASR accuracy               | >= 92% WER      | Word Error Rate on voice channels        |
| CSAT                       | >= 4.2 / 5.0   | Borrower satisfaction post-interaction   |
| Escalation rate            | < 15%           | Conversations requiring human handoff    |
| Tool call success rate     | >= 98%          | MCP tool calls succeeding first attempt  |
| Journey completion rate    | >= 85%          | Conversations reaching wrap node         |
| Tone compliance            | >= 95%          | Automated tone eval on production sample |

### Transfer Metrics

| Metric                     | Target          | Measurement                              |
|----------------------------|-----------------|------------------------------------------|
| Self-sufficiency level     | L3 by week 20   | Assessed against capability matrix       |
| Training completion        | 100% by week 16 | Modules 1-8 completed by all Ops staff   |
| Lender-authored evals      | >= 20%          | % of eval scenarios authored by lender   |
| Lender knowledge updates   | >= 50%          | % of knowledge updates done by lender    |
| LEE time allocation        | < 20% by week 24| LEE hours spent on this lender           |
| Incident self-resolution   | >= 60%          | P2/P3 incidents resolved without LEE     |

### Platform Contribution Metrics

| Metric                     | Target          | Measurement                              |
|----------------------------|-----------------|------------------------------------------|
| Platform feedback tickets  | >= 5 / month    | Actionable tickets filed per engagement  |
| Pattern contributions      | >= 2 / quarter  | Reusable templates added to shared library|
| Eval methodology impr.     | >= 1 / quarter  | New eval techniques from field experience|
| Regulatory edge cases doc. | >= 3 / quarter  | New compliance scenarios documented      |
| Cross-lender pattern reuse | >= 40%          | % of new lender config from templates    |

### Metrics Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│              LENDER ENGAGEMENT DASHBOARD                      │
├──────────────────────────┬──────────────────────────────────┤
│  SPEED                   │  QUALITY                          │
│                          │                                   │
│  Time to Pilot: 11w ✓   │  Resolution Rate: 81% ✓          │
│  Time to Prod:  15w ✓   │  pass^k (5):     0.96 ✓          │
│  Segment Add:   3w  ✓   │  Compliance:     100% ✓          │
│  Eval Cycle:    2d  ✓   │  ASR WER:        91%  ⚠          │
│                          │  CSAT:           4.3  ✓          │
├──────────────────────────┼──────────────────────────────────┤
│  TRANSFER                │  PLATFORM                         │
│                          │                                   │
│  Self-Sufficiency: L2 ⚠ │  Feedback Tickets: 7/mo ✓        │
│  Training:        75% ⚠ │  Patterns Added:   1/Q  ⚠        │
│  Lender Evals:    15% ⚠ │  Reg Edge Cases:   4/Q  ✓        │
│  LEE Allocation:  65% ── │  Template Reuse:   45%  ✓        │
│                          │                                   │
│  ✓ On track  ⚠ Watch  ── In progress                       │
└──────────────────────────┴──────────────────────────────────┘
```

---

## Appendix: Engagement Checklist

### Pre-sales Gate

- [ ] Lender has RBI-regulated entity status confirmed
- [ ] Minimum conversation volume (> 1000/month) validated
- [ ] Integration feasibility (core banking API, ULI readiness) assessed
- [ ] Segment priority aligned with platform capability
- [ ] LEE assigned with relevant vertical experience

### Kickoff Gate

- [ ] Project charter signed by CRO
- [ ] RACI matrix approved by all stakeholders
- [ ] Workspace provisioned (draft + QA)
- [ ] Integration timeline agreed with IT
- [ ] Compliance constraints documented and acknowledged

### Build Gate

- [ ] Knowledge partitions populated and version-controlled
- [ ] Priority segment journeys designed and prompt-tested
- [ ] 3-layer policies configured and unit-tested
- [ ] tau-bench suite passing pass^k >= 0.80 in QA
- [ ] Circuit breakers tested with injected failures

### Shadow Gate

- [ ] 200+ shadow sessions completed
- [ ] Response agreement rate >= 92%
- [ ] Policy compliance rate = 100%
- [ ] Risk team confidence score >= 4.0
- [ ] CRO sign-off on pilot readiness

### Production Gate

- [ ] 100% traffic stable for 1 week
- [ ] All quality metrics within target ranges
- [ ] Rollback procedure tested and documented
- [ ] Lender team at L2+ self-sufficiency
- [ ] Steady-state monitoring processes established

---

**The goal is not permanent dependency. It is structured independence.**
**Every engagement should make the next one faster, and the lender more capable.**
