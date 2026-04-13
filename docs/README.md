# Platform Architecture Documentation

> 9 design documents covering the complete AI-first lending platform architecture.
> 6,300+ lines of architectural rationale, decision frameworks, and implementation specifications.

## Document Index

| # | Document | Lines | Focus | Cross-References |
|---|----------|-------|-------|-----------------|
| 1 | [platform-evolution.md](platform-evolution.md) | 503 | Platform genesis: from 19 repos to unified architecture. Founding thesis, exploration phase, consolidation, go-to-market evolution. | [unified-platform-specification.md](../unified-platform-specification.md) Part I |
| 2 | [ai-engineering-primitives.md](ai-engineering-primitives.md) | 529 | Pandhi's 4 primitives (State, Context, Policy, Feedback) mapped to 25 services. 5 lending-specific extensions: DPI orchestration, segment intelligence, regulatory compliance, post-disbursal lifecycle, context compounding. | [mission-implementation-plan.md](../mission-implementation-plan.md) Missions 1-8 |
| 3 | [platform-spec.md](platform-spec.md) | 911 | Full technical specification: all 25 services with APIs, data models, infrastructure, security. The authoritative service reference. | [unified-platform-specification.md](../unified-platform-specification.md) Part II |
| 4 | [adlc-production-pipeline.md](adlc-production-pipeline.md) | 879 | Agent Development Lifecycle: 8-phase pipeline from discovery through continuous improvement. Lending-specific eval scenarios, regulatory compliance gating. | [mission-implementation-plan.md](../mission-implementation-plan.md) Missions 8-9 |
| 5 | [fde-lender-engagement.md](fde-lender-engagement.md) | 683 | Lender Engagement Engineering model: embedded engineers who understand RBI regulations + platform. Engagement lifecycle, decision framework, knowledge transfer, metrics. | [unified-platform-specification.md](../unified-platform-specification.md) Part VI |
| 6 | [agent-tooling-stack.md](agent-tooling-stack.md) | 613 | Architectural decisions: service mesh topology, shared types, Constellation routing, 3-layer policy, knowledge pipeline, observability, eval, voice, config-as-code, SDK. | [mission-implementation-plan.md](../mission-implementation-plan.md) Missions 0-1 |
| 7 | [production-guide-agents.md](production-guide-agents.md) | 773 | 12-level progressive guide: bare LLM to production lending agent. Each level adds one capability, shows implementation, explains what breaks without it. | [mission-implementation-plan.md](../mission-implementation-plan.md) Missions 2-8 |
| 8 | [agent-primitives-to-outcomes.md](agent-primitives-to-outcomes.md) | 745 | Every lending outcome mapped to State/Context/Policy/Feedback primitives. Identity verification, credit decisions, disbursement, monitoring, collections, voice. | [ai-engineering-primitives.md](ai-engineering-primitives.md) |
| 9 | [voice-ai-lending-distillation.md](voice-ai-lending-distillation.md) | 697 | Voice AI for Indian lending: 12-language ASR, Bhashini TTS, lending pronunciation, latency engineering, emotional intelligence, Exotel telephony. | [mission-implementation-plan.md](../mission-implementation-plan.md) Mission 5 |

## Reading Order

**For architects:** platform-evolution → ai-engineering-primitives → platform-spec → agent-tooling-stack

**For engineers:** production-guide-agents → platform-spec → agent-tooling-stack → adlc-production-pipeline

**For lender-facing teams:** fde-lender-engagement → adlc-production-pipeline → agent-primitives-to-outcomes

**For voice/NLU engineers:** voice-ai-lending-distillation → production-guide-agents (Level 10-12)

## Companion Documents

| Document | Location | Relationship |
|----------|----------|-------------|
| [Unified Platform Specification](../unified-platform-specification.md) | Root | System architecture, build stages, deployment playbook |
| [Mission Implementation Plan](../mission-implementation-plan.md) | Root | Granular engineering specs, SQL schemas, code patterns |
| [Repo Evaluation](../repo-evaluation-comprehensive.md) | Root | 19-repo analysis that informed the architecture |

## Inspired By

These documents are modeled after the [voiceai-agent-services-platform docs](https://github.com/Annapurnar06/voiceai-agent-services-platform/tree/master/docs) — 10 design documents covering Sierra AI platform evolution, Pandhi's primitives, LendingOS technical spec, ADLC pipeline, FDE engagement model, agent tooling stack, production guide, primitives-to-outcomes mapping, voice AI distillation, and internal development analysis.
