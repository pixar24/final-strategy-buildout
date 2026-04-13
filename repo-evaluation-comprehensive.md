# Comprehensive Repository Evaluation: AI-First Lending Platform
## 19 Repositories at github.com/Annapurnar06

---

## I. EXECUTIVE SUMMARY

19 repositories representing a systematic series of experiments toward building a unified AI-first lending platform for India's 300M underserved borrowers. The corpus contains **~75,000+ lines of working code** across 6 production-grade codebases, plus strategic documentation, research implementations, and UI prototypes. Together, they cover every layer from DPI/ULI infrastructure through voice AI borrower journeys to lender operator dashboards.

**Key Finding:** The repos are NOT scattered experiments -- they form a deliberate layer-by-layer exploration of the complete lending stack. Each repo probes a specific architectural thesis (context graphs, voice AI agent OS, credit infrastructure decomposition, agent lifecycle) that maps to a distinct platform layer. The consolidation path is clear.

---

## II. REPO-BY-REPO EVALUATION

### TIER 1: CORE PLATFORM CODEBASES (Working Production Code)

---

#### 1. `embeddedcredit-as-a-service-v2`
**Role:** The primary AI lending operating system  
**Lang:** Python 53.6% + TypeScript 45.7% | **Commits:** 28 | **Tests:** 388 passing

| Dimension | Assessment |
|-----------|------------|
| **What exists** | FastAPI backend, 60 lending tools (7 categories: identity, data, scoring, product, execution, servicing, monitoring), 38 YAML borrower segment configs, policy engine (7 structural + 10 behavioral rules), agent runtime with bilingual prompts, WhatsApp + Web gateway, Next.js lender dashboard, Docker Compose (app + Postgres 16 + Redis 7) |
| **Architecture** | Borrower-agent-first: single agent handles full loan lifecycle per borrower, tools are lending primitives, policy engine gates all actions. Multi-tenant via lender API keys. Voice: Cartesia TTS + Deepgram STT. Memory: mem0 for borrower context. |
| **Use-case design** | 38 segments spanning agriculture (dairy, sugarcane, cotton), micro-enterprise (kirana, street vendor), asset-backed (gold, warehouse, EV), gig workers, SHGs. Each segment has YAML config defining risk parameters, product eligibility, collection strategy. |
| **Lender outcomes** | Designed to reduce cost-per-loan via AI agent replacing manual field interactions. Segment-specific scoring means product fit, not just access. |
| **Production gaps** | ALL external integrations mocked: eKYC (DigiLocker, Aadhaar), Account Aggregator, credit bureau, eSign, disbursal (NPCI/IMPS), WhatsApp BSP. No durable workflow orchestration (Temporal-style). |
| **Maturity** | MVP with sound architecture. 8-12 weeks from pilot-ready if integration work starts. |
| **Platform position** | **LAYER 2 (Intelligence) + LAYER 3 (UX)** -- the borrower-facing agent and lender dashboard |

---

#### 2. `contextgraphs-platform-fintech`
**Role:** Enterprise Context Layer for lending (Atlan architecture)  
**Lang:** Python 100% | **Commits:** 1 | **Tests:** 31 | **Files:** 71 | **Lines:** 21,636

| Dimension | Assessment |
|-----------|------------|
| **What exists** | Context Layer (Lakehouse with time-travel, Data Graph with column-level lineage, Knowledge Graph with 28 terms, Governance Graph with 7 policies + 14 compliance rules, Active Ontology resolving entities across 8 systems, Decision Traces as first-class objects). 15 specialized agents (origination, underwriting, compliance, servicing, context). 24-state orchestration machine with Temporal-inspired workflow engine. MCP Server with 5 tools + RBAC. Lender workflows (LO pipeline, UW queue, condition management, closing coordination). 3 jurisdiction configs (CA/TX/NY), 3 product configs (Conventional/FHA/VA). Observability with drift detection + feedback loops. |
| **Architecture** | "Context compounds" thesis: every decision writes back as a trace, 10,000th loan benefits from 9,999 prior traces. Deterministic where possible (LTV/DTI = arithmetic), agentic where necessary (doc extraction = LLM). Workflow engine owns state machine, not the LLM. |
| **Use-case design** | Mortgage-focused (US context) but architecture is directly transferable to Indian lending. Jurisdiction-as-config means adding Indian states = config files, not code changes. |
| **Lender outcomes** | Context graphs solve the enterprise AI accuracy problem in regulated industries. Decision traces provide full audit trail. Governance graph ensures compliance. |
| **Production gaps** | Zero external dependencies (stdlib only) means no real LLM/ML integration. US mortgage focus needs adaptation for India. Single commit = generated in one session. |
| **Maturity** | Architectural prototype. The design patterns are production-grade; the implementation needs real integrations. |
| **Platform position** | **LAYER 0 (Platform Foundation)** -- the context graph substrate that all agents read from and write to |

---

#### 3. `voiceai-agent-services-platform` (LendingOS)
**Role:** Sierra-grade voice AI agent platform for lending  
**Lang:** Python 88.2% + JS/CSS 12% | **Commits:** 8 | **Files:** 156 | **Lines:** 14,420

| Dimension | Assessment |
|-----------|------------|
| **What exists** | 14 microservices: Gateway (8000), Constellation/multi-model router (8001), Tool Executor with 20 lending tools (8002), Policy Engine with structural+behavioral+supervisor (8003), Knowledge Service with 7-partition RAG (8004), Agent Runtime with 16-step turn processor (8005), Trace Service (8006), Voice Pipeline with STT/TTS/VAD/telephony (8007), Agent Data Platform with cross-session memory (8008), Workspace Service with config version control (8009), Experience Manager for QA+annotation (8010), Evaluation with tau-bench pass^k (8011), Explorer with AI-powered analysis (8012). Plus Agent Studio web UI (3000) and pip-installable Agent SDK. |
| **Architecture** | Implements Pandhi's 4 structural primitives: State (append-only logs), Context (entity resolution, policy-scoped retrieval, cross-session memory), Policy (Journeys as structural enforcement, supervisor models), Feedback (tau-bench, pass^k, regression gating). Voice pipeline: Deepgram STT + ElevenLabs TTS + energy-based VAD + Twilio telephony. Fast brain/slow brain parallel execution. |
| **Use-case design** | 11 lending journeys (identity verification, loan status, payment, rate lock, hardship, dispute, general, etc.). Demo lender config with rate sheets, policies, TILA/ECOA disclosures. |
| **Lender outcomes** | Complete agent development lifecycle: dev -> evals -> shadow mode -> canary -> production. Experience Manager for human annotation. Explorer for failure clustering and anomaly detection. This is the operational backbone lenders need. |
| **Production gaps** | Mock mode for all external AI providers (works without API keys). US lending context. No DPI/ULI integration. |
| **Maturity** | Most architecturally complete platform in the portfolio. 14 services with clear separation of concerns. |
| **Platform position** | **LAYER 1 (Governed Execution) + LAYER 2 (Intelligence)** -- the agent runtime, policy enforcement, voice pipeline, observability, and agent lifecycle |

---

#### 4. `fb-product-l2` (FinBox Credit Infrastructure)
**Role:** Credit infrastructure operating system (FinBox product decomposition)  
**Lang:** TypeScript 69.9% + Python 28.3% | **Commits:** 1 | **Tests:** 217 | **Files:** 182

| Dimension | Assessment |
|-----------|------------|
| **What exists** | 11-package monorepo: @finbox/shared (auth, audit, PII, compliance, KFS), @finbox/sentinel (BRE with 14 operators, fraud detection), @finbox/los (14-state loan origination), @finbox/prism (multi-lender waterfall routing), @finbox/journey_studio (orchestration + A/B testing), @finbox/gateway (REST API + Prometheus), @finbox/mcp-server (27 MCP tools, dual transport), bankconnect (statement analysis, FBCS 0-900, Cluster TX), riskengine (ML scoring, ensemble, feedback loop), deviceconnect (FIS scoring, device fraud). Plus 5 end-to-end integration test scenarios. |
| **Architecture** | Identity -> Risk Intelligence -> Fit Decision -> Journey Orchestration -> Execution -> Monitoring. Polyglot monorepo with clean package boundaries. MCP server enables AI agent access to all platform capabilities. |
| **Use-case design** | 8 borrower segments (salaried, self-employed, gig, micro-entrepreneur, NTC, MSME, farmer, SHG). Each segment has specific scoring path and product fit. |
| **Lender outcomes** | Credit FIT not just credit ACCESS. Dynamic pricing, champion/challenger analytics. Multi-lender waterfall means borrower gets best available offer. RBI DLG compliant KFS generation. |
| **Production gaps** | FinBox product simulation -- not connected to real FinBox APIs. Single commit. |
| **Maturity** | Comprehensive product decomposition. Shows deep understanding of credit infrastructure stack. |
| **Platform position** | **LAYER 0 (Platform Foundation) + LAYER 1 (Governed Execution)** -- credit data services (BankConnect, DeviceConnect, RiskEngine) + decisioning (Sentinel) + origination (LOS) + routing (Prism) |

---

#### 5. `autopilot-in-enterprise`
**Role:** Indian Lending Autopilot (Blend-style for India)  
**Lang:** Python 99.7% | **Commits:** 4

| Dimension | Assessment |
|-----------|------------|
| **What exists** | FastAPI application with Alembic migrations, Docker Compose, evals directory, full src/autopilot package. Architecture docs: Blend Autopilot adapted for Indian lending, LOS Cloud architecture. Phase 0 + Phase 1 implemented. |
| **Architecture** | Enterprise autopilot pattern: AI copilot sits alongside existing LOS/LMS, augments lender operators with recommendations. Not replacing, augmenting. |
| **Lender outcomes** | Designed for lenders who already have LOS. Doesn't require rip-and-replace. Autopilot recommends, human decides. |
| **Production gaps** | Early implementation. Needs more phases. |
| **Maturity** | Early MVP. The architectural thesis (autopilot-in-enterprise, not replacement) is strategically sound for initial lender adoption. |
| **Platform position** | **LAYER 3 (Lender UX)** -- lender operator copilot that works alongside existing systems |

---

#### 6. `caas-casestudy` (Hyperface Agentic AI Platform)
**Role:** Credit Cards-as-a-Service multi-agent system  
**Lang:** Python 100% | **Commits:** 10 | **Tests:** 182

| Dimension | Assessment |
|-----------|------------|
| **What exists** | 18 domain-specific agents across 5 products (Onboard IQ, Credit Pro, Embed Next, Smart Engage, AI Labs). SRAL (Sense-Reason-Act-Learn) loop architecture. L0-L4 autonomy levels per agent per bank. Full agent development lifecycle: evals (6 scorers), shadow mode, canary (7-stage 1%->100%), observability (PSI drift), chaos engineering, HITL queue, model management, policy management with diff engine. Agentic payments (Visa TAP + Mastercard VI). Agent distribution: 13 MCP tools, CLI, OpenAPI spec, AGENTS.md. |
| **Architecture** | Events-only communication via MSK. PCI scrubbing at event boundary. Write-before-act audit. Fail-closed policy. Per-bank autonomy overrides. Thompson Sampling for engagement optimization. |
| **Lender outcomes** | Full credit card lifecycle from onboarding through collections. Per-bank autonomy means each bank controls agent freedom. |
| **Production gaps** | Credit card focused (not lending). But SRAL pattern, lifecycle pipeline, and agent distribution layer are directly reusable. |
| **Maturity** | Most mature agent lifecycle implementation. The eval/shadow/canary pipeline is production-grade pattern work. |
| **Platform position** | **LAYER 1 (Agent Lifecycle)** -- the SRAL loop, eval pipeline, shadow/canary deployment, and autonomy framework transfer directly |

---

### TIER 2: RESEARCH & CASE STUDIES (Working Code + Research)

---

#### 7. `gff-case-studies`
**Role:** Global Fintech Frontiers research implementations  
**Lang:** Python 95.5% | **Commits:** 3 | **Tests:** 49+

| Component | What It Does | Relevance |
|-----------|-------------|-----------|
| **agent-fill/** | AI PDF form filler (Gemini + Claude + PyMuPDF). 6 tests. | Document automation for lending (W-9, KYC forms) |
| **latent-briefing/** | KV cache compaction for multi-agent systems. 17 tests. Up to 49% token savings. | Reduces cost of multi-agent lending platform at scale |
| **ecaas-briefing-integration/** | Latent briefing integrated with ecaas-v2. 38-segment threshold map + A/B testing. 29 tests, 285 total with zero regressions. | Production optimization for the core platform |
| **snowflake-cortex-lending/** | Snowflake Cortex AI lending case study. IAM/RBAC, workflows, security, compliance, DR. | Data platform architecture reference |
| **src/lending_vlm/** | Fine-tune LFM2.5-VL-450M for lending lifecycle analysis. 4 tasks, 3 training configs, 5-stage lifecycle orchestrator. 49 tests. | Visual document understanding for lending |

**Platform position:** **LAYER 2 (Intelligence R&D)** -- optimization research that feeds into production systems

---

#### 8. `lia-openclaw-CoS`
**Role:** AI Chief of Staff agent (OpenClaw plugin)  
**Lang:** TypeScript 100% | **Commits:** 1

| Dimension | Assessment |
|-----------|------------|
| **What exists** | Security-first AI agent: AES-256-GCM encrypted OAuth, full audit logging with auto-redaction, per-user rate limiting, confirmation gates on destructive actions, progressive trust engine (4-level autonomy with decay), Gmail/Calendar integration, email triage, 22 tools across 7 surfaces, SOUL.md persona, persistent memory with daily transcripts and compaction. |
| **Relevance** | Progressive trust engine and security patterns are directly applicable to lender-facing agents. The 4-level autonomy with decay is a more sophisticated version of what caas-casestudy implements as L0-L4. |
| **Platform position** | **PATTERN LIBRARY** -- trust engine, security patterns, audit design for agent systems |

---

### TIER 3: MCP & INTEGRATION LAYER

---

#### 9. `fb-specifics`
**Role:** FinBox MCP implementation + MCP Summit 2026 lending demos  
**Lang:** Python 68.5% + TypeScript 29.3% | **Commits:** 7

| Dimension | Assessment |
|-----------|------------|
| **What exists** | `finbox-mcp-server/`: Production MCP server wrapping FinBox Embedded Finance APIs (17 tools, TypeScript). `servers/`: 3 MCP servers for loan origination, underwriting, servicing. `apps/`: MCP Apps (loan comparison, status tracker, amortization). `agents/`: Multi-agent A2A + MCP lending orchestrator. `config/`: Enterprise compliance (OAuth 2.1, PII redaction, tamper-evident audit). Strategic docs: BankConnect, Sentinel AI, Journey Studio, RiskEngine, Prism analysis. |
| **Platform position** | **LAYER 1 (Integration)** -- MCP as the interoperability layer between platform components and external AI agents |

---

### TIER 4: STRATEGIC DOCUMENTATION & KNOWLEDGE

---

#### 10. `embeddedcredit-as-a-service` (v1)
**Role:** Original thesis document | **Commits:** 4 | **Code:** Zero  
Strategic narrative covering platform overview, borrower lifecycle, lender transformation, system architecture.  
**Position:** The "WHY" document. Everything else is implementation of this thesis.

#### 11. `EKP-FS`
**Role:** AI-first lending architecture specs | **Commits:** 1  
5 markdown files: 106-agent architecture mapping, enterprise architecture diagrams, platform mechanics analysis, production build roadmap evaluation, orchestration patterns analysis.  
**Position:** The architectural blueprint. Maps the complete agent inventory and enterprise deployment.

#### 12. `Knowledge-Repo`
**Role:** Research knowledge base | **Commits:** 3  
AI-First Lending Startup Roadmap, ULI integration spec, borrower journeys, FBR market data, agentic architecture, + 24 Factory AI research pages.  
**Position:** The research foundation. ULI integration spec is critical for DPI layer.

#### 13. `ecaas-prd`
**Role:** Production implementation roadmap | **Commits:** 1  
Roadmap for lender deployment. `implementation/` directory.  
**Position:** The deployment playbook.

#### 14. `eng-docs`
**Role:** Founding engineer onboarding | **Commits:** 1  
Single file: `founding-engineer-onboarding.md`. Comprehensive onboarding for risk analytics data engineer.  
**Position:** Team-building artifact.

#### 15. `agentic-design-patterns-2025`
**Role:** Pattern reference | **Commits:** 1  
Single PDF: 20 agentic design patterns.  
**Position:** Pattern library reference.

---

### TIER 5: UI/FRONTEND EXPERIMENTS

---

#### 16. `lendscore`
**Role:** Product marketing site / LendScore frontend  
**Lang:** TypeScript 99.5% | **Commits:** 121 | **Stack:** Vite + React + shadcn-ui + Tailwind  
Built with Lovable. Includes Agents SDK panel. Marketing/product positioning for the lending platform.  
**Position:** **LAYER 3 (UX)** -- borrower/lender-facing web presence

#### 17. `finaii-v2`
**Role:** Personal finance companion (consumer app)  
**Lang:** TypeScript 98.9% | **Commits:** 109 | **Stack:** Vite + React + Supabase  
5 agent workers (Budget, Goals, Risk, Portfolio, Insights), financial health engine (8 sub-scores), 360-degree profiling, Spotify Wrapped-style recaps. Forked from Annapurnar07/finaii.  
**Position:** **Adjacent** -- consumer finance companion, not core lending platform. But agent patterns (orchestrator-workers, policy engine, progressive disclosure) are directly reusable.

---

### TIER 6: EMPTY/PLACEHOLDER

#### 18. `fb-product-l1` - Empty repository (described as "agents use-cases over finbox platform")
#### 19. `equal-tsp-fintech` - Empty repository

---

## III. UNIFIED ARCHITECTURE: HOW REPOS MAP TO PLATFORM LAYERS

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: LENDER + BORROWER UX                                                 │
│                                                                                 │
│  embeddedcredit-v2 (borrower WhatsApp/Web agent + lender dashboard)             │
│  autopilot-in-enterprise (lender operator copilot alongside existing LOS)       │
│  lendscore (product web presence)                                               │
│  voiceai-platform/agent-studio (lender agent configuration UI)                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 2: AI INTELLIGENCE + VOICE                                               │
│                                                                                 │
│  embeddedcredit-v2 (60 lending tools, 38 segment configs, agent runtime)        │
│  voiceai-platform (voice pipeline: STT/TTS/VAD, fast/slow brain, telephony)     │
│  voiceai-platform (constellation: multi-model routing + failover)               │
│  gff-case-studies (lending VLM, latent briefing for token optimization)         │
│  fb-product-l2/riskengine (ML scoring, ensemble, feedback loop)                 │
│  fb-product-l2/bankconnect (FBCS 0-900, Cluster TX)                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 1: GOVERNED EXECUTION + AGENT LIFECYCLE                                  │
│                                                                                 │
│  voiceai-platform (policy engine, agent runtime, 16-step turn processor)        │
│  voiceai-platform (trace service, evaluation, experience manager, explorer)     │
│  caas-casestudy (SRAL loop, eval/shadow/canary pipeline, autonomy levels)       │
│  fb-product-l2/sentinel (BRE, 14 operators, fraud detection)                    │
│  fb-product-l2/los (14-state machine, KFS, disbursement)                        │
│  fb-product-l2/prism (multi-lender waterfall routing)                           │
│  fb-product-l2/journey_studio (orchestration, A/B testing)                      │
│  fb-specifics (MCP servers for lending lifecycle, enterprise compliance)         │
│  lia-openclaw (progressive trust engine, security patterns)                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 0: PLATFORM FOUNDATION + DATA INFRASTRUCTURE                             │
│                                                                                 │
│  contextgraphs-platform (Context Lakehouse, Data Graph, Knowledge Graph,        │
│    Governance Graph, Active Ontology, Decision Traces)                           │
│  voiceai-platform (knowledge service: 7-partition RAG, agent data platform)     │
│  fb-product-l2/shared (auth, audit, PII, compliance, rate limiting)             │
│  Knowledge-Repo (ULI integration spec, DPI data layer design)                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  STRATEGIC FOUNDATION (Documentation + Research)                                │
│                                                                                 │
│  embeddedcredit-v1 (thesis), EKP-FS (106-agent blueprint),                     │
│  ecaas-prd (deployment roadmap), agentic-design-patterns (20 patterns)          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## IV. CROSS-REPO COMPONENT INVENTORY

### A. Agent Runtime & Orchestration
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| Agent Runtime (16-step turn processor) | voiceai-platform | Production-grade, multi-service |
| Agent Runtime (bilingual, 60 tools) | ecaas-v2 | MVP, FastAPI monolith |
| SRAL Loop (Sense-Reason-Act-Learn) | caas-casestudy | Production pattern, 18 agents |
| 24-state Lending State Machine | contextgraphs | Temporal-style, human checkpoints |
| 14-state LOS Machine | fb-product-l2 | TypeScript, transition validation |
| Journey Engine | voiceai-platform | YAML-driven, 11 journeys |
| Journey Studio | fb-product-l2 | Builder + runtime + A/B testing |

### B. Policy & Compliance
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| Policy Engine (structural + behavioral + supervisor) | voiceai-platform | 3-layer enforcement |
| Sentinel BRE (14 operators, 3 templates) | fb-product-l2 | Champion/challenger analytics |
| Policy Engine (7 structural + 10 behavioral) | ecaas-v2 | FastAPI |
| Governance Graph (7 policies, 14 rules) | contextgraphs | Atlan-style |
| RBI FREE-AI Compliance | caas-casestudy | Pre-act checking |
| Progressive Trust (4-level + decay) | lia-openclaw | Security-first |
| KFS Generation (RBI DLG) | fb-product-l2/shared | Newton-Raphson APR |
| Enterprise Compliance (OAuth 2.1, PII, audit) | fb-specifics | MCP integration |

### C. Voice & Communication
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| STT (Deepgram + Whisper fallback) | voiceai-platform | Lending vocab boost |
| TTS (ElevenLabs + Google fallback) | voiceai-platform | SSML prosody |
| VAD (energy-based, 500ms lending-tuned) | voiceai-platform | Pause for loan numbers |
| Fast Brain / Slow Brain | voiceai-platform | Parallel execution |
| Telephony (Twilio PSTN/SIP) | voiceai-platform | WebSocket audio |
| Voice (Cartesia TTS + Deepgram STT) | ecaas-v2 | In agent runtime |
| WhatsApp BSP + Web Gateway | ecaas-v2 | Channel adapters |

### D. Data & Scoring
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| BankConnect (FBCS 0-900, Cluster TX) | fb-product-l2 | 20+ categories, 30+ lender patterns |
| DeviceConnect (FIS 0-900) | fb-product-l2 | 20+ features, fraud detection |
| RiskEngine (ensemble, drift detection) | fb-product-l2 | GB + sequence model |
| 60 Lending Tools (identity -> monitoring) | ecaas-v2 | 7 tool categories |
| 20 Lending Tools (session-state gated) | voiceai-platform | Circuit breaker |
| Knowledge Service (7-partition RAG) | voiceai-platform | Verbatim compliance store |
| Knowledge Graph (28 terms) | contextgraphs | Semantic search |
| Context Lakehouse (versioned metadata) | contextgraphs | Time travel |

### E. Agent Lifecycle (Evals, Observability, Guardrails)
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| Eval System (tau-bench, pass^k) | voiceai-platform | 8 scenarios |
| Eval Harness (6 scorers, datasets) | caas-casestudy | Multi-dataset suites |
| Shadow Mode + Canary (7-stage) | caas-casestudy | Auto-graduate, auto-rollback |
| Observability (metrics, SLA, drift) | voiceai-platform | PSI-based drift |
| Explorer (failure clustering, anomaly) | voiceai-platform | AI-powered analysis |
| Experience Manager (QA + annotation) | voiceai-platform | HITL feedback loop |
| Trace Service (execution graphs) | voiceai-platform | Span timeline |
| Workspace Service (config versioning) | voiceai-platform | Draft->QA->staging->prod |
| Circuit Breaker | caas-casestudy | Per-agent |
| Decision Traces (first-class objects) | contextgraphs | Full provenance |

### F. MCP & Interoperability
| Component | Source Repo | Implementation |
|-----------|------------|----------------|
| MCP Server (27 tools, dual transport) | fb-product-l2 | stdio + HTTP/SSE |
| MCP Server (5 tools, RBAC) | contextgraphs | Lending-specific |
| FinBox MCP Server (17 tools) | fb-specifics | Production wrapper |
| MCP Lending Lifecycle (origination/UW/servicing) | fb-specifics | A2A + MCP orchestration |
| Agent SDK (pip installable) | voiceai-platform | 15 platform API methods |
| Agent Distribution (13 MCP + CLI + OpenAPI) | caas-casestudy | Full distribution layer |

---

## V. POSITIONING FRAMEWORK: REPOS AS EXPERIMENTS

### The Thesis Arc

```
embeddedcredit-v1 (WHY)
    │
    ├── "What does an AI-first lending stack look like?"
    │       ├── EKP-FS (106-agent blueprint)
    │       ├── Knowledge-Repo (ULI/DPI integration spec)
    │       └── agentic-design-patterns (pattern library)
    │
    ├── "How do you build the data + context substrate?"
    │       ├── contextgraphs-platform (Atlan Context Graph)
    │       └── fb-product-l2/bankconnect+riskengine+deviceconnect
    │
    ├── "How do you build the governed execution layer?"
    │       ├── fb-product-l2 (FinBox product decomposition)
    │       ├── fb-specifics (MCP integration layer)
    │       └── caas-casestudy (agent lifecycle: eval/shadow/canary)
    │
    ├── "How do you build the intelligence + voice layer?"
    │       ├── voiceai-platform (Sierra-grade voice AI OS)
    │       ├── gff-case-studies (VLM, latent briefing, optimization)
    │       └── ecaas-v2 (60-tool lending agent, 38 segments)
    │
    ├── "How do you build the lender/borrower UX?"
    │       ├── ecaas-v2 (WhatsApp agent + lender dashboard)
    │       ├── autopilot-in-enterprise (operator copilot)
    │       ├── lendscore (web presence)
    │       └── finaii-v2 (consumer finance companion)
    │
    └── "What patterns from adjacent domains apply?"
            ├── caas-casestudy (Hyperface credit card lifecycle)
            ├── lia-openclaw (trust engine, security patterns)
            └── gff-case-studies (Ramp context graphs, form filling)
```

### Experiment Classification

| Repo | Experiment Type | Key Discovery | Carry Forward |
|------|----------------|---------------|---------------|
| ecaas-v2 | Core platform prototype | Single-agent-per-borrower works for 38 segments | Segment configs, tool design, bilingual prompts |
| contextgraphs | Architecture thesis | Context graphs > knowledge graphs for regulated AI | Lakehouse, Governance Graph, Decision Traces, Active Ontology |
| voiceai-platform | Full platform architecture | 14-service Sierra-grade platform design is viable | Agent runtime, voice pipeline, agent lifecycle services, SDK |
| fb-product-l2 | Product decomposition | FinBox's credit infra can be rebuilt as composable packages | Sentinel, LOS, Prism, BankConnect, RiskEngine, MCP |
| caas-casestudy | Agent lifecycle | SRAL + eval/shadow/canary pipeline is non-negotiable for production | Lifecycle pipeline, autonomy levels, agent distribution |
| autopilot-in-enterprise | Go-to-market strategy | Enterprise adoption = autopilot alongside LOS, not replacement | Copilot pattern for initial lender sales |
| gff-case-studies | Optimization research | VLM for doc understanding, latent briefing for token cost | Lending VLM, KV cache compaction at scale |
| fb-specifics | Integration layer | MCP as universal tool protocol for lending services | MCP server patterns, compliance hooks |
| lia-openclaw | Trust patterns | Progressive trust with decay for financial agents | Trust engine, audit design |

---

## VI. DPI/ULI DATA INTEGRATION ASSESSMENT

### Current State
- **Knowledge-Repo** has a ULI integration spec document
- **ecaas-v2** has 60 tools that map to ULI service categories but ALL are mocked
- **EKP-FS** has architecture diagrams showing DPI data flow
- **No repo has actual DPI/ULI API integration code**

### What Needs Building (136 ULI data services)
| ULI Service Category | Platform Component | Source of Design |
|---------------------|-------------------|-----------------|
| Identity (eKYC, Aadhaar, DigiLocker) | ecaas-v2/tools/identity | ecaas-v2 has tool interfaces |
| Account Aggregator (financial data) | fb-product-l2/bankconnect | BankConnect has analysis logic |
| Credit Bureau (CIBIL, Experian) | fb-product-l2/riskengine | RiskEngine has scoring pipeline |
| eSign (Aadhaar eSign, DSC) | ecaas-v2/tools/execution | Mocked interfaces exist |
| eNACH / NPCI | fb-product-l2/los | LOS has disbursement processor |
| GSTN / EPFO | ecaas-v2/tools/data | Tool interfaces defined |

---

## VII. LENDER WORKFLOW DESIGN ACROSS REPOS

### How Different Lender Types Would Interact

| Lender Type | Primary Interface | Key Components | Autonomy Level |
|-------------|------------------|----------------|----------------|
| **MFI** | WhatsApp agent (ecaas-v2) + voice (voiceai) | Low-doc journey, field agent support, vernacular voice | L3 (high autonomy, low ticket) |
| **Small NBFC** | Autopilot copilot (autopilot-in-enterprise) | Alongside existing LOS, recommendation engine | L1-L2 (copilot mode) |
| **SFB** | Agent Studio (voiceai-platform) + Dashboard (ecaas-v2) | Full journey configuration, A/B testing, policy management | L2 (supervised autonomy) |
| **Large NBFC** | fb-product-l2 infrastructure + MCP integration | API-first, multi-product routing via Prism, BRE via Sentinel | L1 (API integration) |
| **Bank** | Full platform (all layers) | Context graphs, full audit trail, RBI compliance, multi-channel | L0-L1 (approval required) |
| **LSP** | MCP tools (fb-specifics, fb-product-l2) | Embed platform capabilities via MCP, partner waterfall | L2-L3 (operational) |

---

## VIII. CONSOLIDATION PATH

### Base Monorepo: Build from `voiceai-agent-services-platform` architecture

**Rationale:** It has the most mature service decomposition (14 services), the only real voice pipeline, the complete agent lifecycle (eval/shadow/canary), and the SDK. Everything else can be merged as packages/modules.

### Merge Sequence

| Phase | What Merges In | From Repo | Priority |
|-------|---------------|-----------|----------|
| **Phase 0** | Context Layer (Lakehouse, Governance Graph, Decision Traces) | contextgraphs | Foundation |
| **Phase 0** | Shared infrastructure (auth, audit, PII, compliance, KFS) | fb-product-l2/shared | Foundation |
| **Phase 1** | 38 borrower segment configs + bilingual prompts | ecaas-v2 | Core |
| **Phase 1** | BankConnect analysis + FBCS scoring | fb-product-l2/bankconnect | Core |
| **Phase 1** | Sentinel BRE + fraud detection | fb-product-l2/sentinel | Core |
| **Phase 1** | India-specific tool interfaces (eKYC, AA, bureau) | ecaas-v2/tools | Core |
| **Phase 2** | SRAL loop + eval/shadow/canary pipeline | caas-casestudy/lifecycle | Lifecycle |
| **Phase 2** | Progressive trust engine | lia-openclaw | Lifecycle |
| **Phase 2** | Lending VLM + latent briefing | gff-case-studies | Intelligence |
| **Phase 3** | WhatsApp channel adapter | ecaas-v2/gateway | Channel |
| **Phase 3** | Lender dashboard (Next.js) | ecaas-v2/frontend | UX |
| **Phase 3** | Autopilot copilot mode | autopilot-in-enterprise | UX |
| **Phase 4** | MCP lending lifecycle servers | fb-specifics | Integration |
| **Phase 4** | Multi-lender routing (Prism) | fb-product-l2/prism | Routing |

### What to Discard
- `embeddedcredit-v1`: Already captured in thesis. No code to merge.
- `fb-product-l1`: Empty.
- `equal-tsp-fintech`: Empty.
- `EKP-FS`: Reference documentation only. Keep as knowledge base.
- `ecaas-prd`: Planning document. Keep for roadmap reference.
- `eng-docs`: HR document. Separate concern.
- `agentic-design-patterns-2025`: PDF reference. Not code.
- `lendscore`: Marketing site. Separate repo.
- `finaii-v2`: Consumer finance app. Separate product.

---

## IX. CRITICAL GAPS ACROSS ALL REPOS

### 1. No Real DPI/ULI Integration
Zero repos have actual API calls to Indian DPI services. All 136 ULI services are mocked.

### 2. No Durable Workflow Orchestration
contextgraphs has a "Temporal-inspired" engine but it's in-memory. No repo uses actual Temporal/Conductor for durable workflows -- which is non-negotiable for lending journeys that span days/weeks.

### 3. No Real LLM Integration in Core
voiceai-platform falls back to mock mode without API keys. ecaas-v2 uses in-process agents. No repo demonstrates production LLM deployment (inference, fine-tuning, latency optimization) in a lending context.

### 4. No Multi-Tenant Database Layer
ecaas-v2 has Postgres but no tenant isolation. fb-product-l2 uses in-memory stores. No repo has row-level security, tenant-scoped queries, or data partitioning for multi-lender deployment.

### 5. No Load Testing / Performance Benchmarks
No repo has load testing results. For a platform targeting MFIs processing thousands of daily loans, performance at scale is untested.

### 6. No CI/CD Pipeline
Single-commit repos dominate. No GitHub Actions, no deployment automation, no environment promotion.

---

## X. AGGREGATE STATISTICS

| Metric | Value |
|--------|-------|
| Total repos | 19 |
| Repos with working code | 12 |
| Empty repos | 2 |
| Documentation-only repos | 5 |
| Total estimated lines of code | ~75,000+ |
| Total tests across all repos | ~1,100+ |
| Languages | Python (primary), TypeScript, JavaScript, CSS |
| Key frameworks | FastAPI, Next.js, React, Vite, Docker Compose |
| AI/ML integrations | OpenAI, Anthropic, Deepgram, ElevenLabs, Cartesia, Gemini, mem0 |
| MCP tools implemented | ~80+ across all repos |
| Unique agents designed | 100+ (across EKP-FS blueprint + implementations) |
| Borrower segments defined | 38 (ecaas-v2) |
| Lending tools implemented | 60 (ecaas-v2) + 20 (voiceai) + 27 MCP (fb-product-l2) |
