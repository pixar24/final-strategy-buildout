# Production Guide: From Bare LLM to Full Lending Platform
## A 12-Level Progressive Build for AI-First Indian Lending

---

Every level adds one capability. Every capability creates new problems solved by the next level. Skip a level and you accumulate debt that compounds under production load.

```
Level  0: Bare LLM                     ─── "It talks"
Level  1: Conversation Memory          ─── "It remembers"
Level  2: Tool Use                     ─── "It acts"
Level  3: Retrieval / RAG              ─── "It knows"
Level  4: Conversation Flow / Journeys ─── "It guides"
Level  5: Policy Enforcement           ─── "It obeys"
Level  6: Multi-Model Routing          ─── "It optimizes"
Level  7: Observability                ─── "It reports"
Level  8: Evaluation                   ─── "It proves"
Level  9: Version Control & Deployment ─── "It ships"
Level 10: Voice                        ─── "It speaks"
Level 11: DPI/ULI Integration          ─── "It connects"
Level 12: Continuous Improvement       ─── "It learns"
```

---

## LEVEL 0: BARE LLM

### What You're Adding
A single OpenAI API call. The agent is a function: `input → prompt → LLM → output`.

### How the Platform Implements It

**Service:** `constellation` (S15) — but at Level 0 you only need one provider.

```python
# The simplest possible lending agent
import openai

async def lending_agent(user_message: str) -> str:
    response = await openai.ChatCompletion.acreate(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a lending assistant for Indian borrowers."},
            {"role": "user", "content": user_message},
        ],
    )
    return response.choices[0].message.content
```

### Problems It Creates
- **No memory:** Every turn is independent. "What rate did you just quote me?" fails.
- **No tools:** Cannot check a real loan status, verify Aadhaar, or initiate disbursement.
- **Hallucination:** Will fabricate interest rates, policies, and regulatory requirements.
- **No compliance:** No audit trail, no PII protection, no RBI guidelines adherence.
- **Single model:** Paying GPT-4o prices for FAQ answers that need GPT-4o-mini.

---

## LEVEL 1: CONVERSATION MEMORY

### What You're Adding
Session-level memory (within a call) and cross-session memory (returning borrowers). The agent maintains context across turns and recognizes returning users.

### How the Platform Implements It

**Service:** `agent-runtime` (S13) for session state, `context-graph` (S17) for cross-session.

```python
# Session memory: Redis-backed conversation buffer
class SessionManager:
    async def get_history(self, session_id: str) -> list[Message]:
        raw = await self.redis.lrange(f"session:{session_id}:messages", 0, -1)
        return [Message.parse_raw(m) for m in raw]

    async def append(self, session_id: str, message: Message):
        await self.redis.rpush(f"session:{session_id}:messages", message.json())
        await self.redis.expire(f"session:{session_id}:messages", 3600)  # 1h TTL

# Cross-session memory: Neo4j context graph
class CrossSessionMemory:
    async def get_borrower_context(self, phone_hash: str) -> dict:
        """Returns: previous loans, repayment history, preferred language, last interaction."""
        result = await self.neo4j.execute("""
            MATCH (b:Borrower {phone_hash: $ph})
            OPTIONAL MATCH (b)-[:HAS_LOAN]->(l:Loan)
            OPTIONAL MATCH (b)-[:HAD_CONVERSATION]->(c:Interaction)
            RETURN b, collect(l) as loans, collect(c) as interactions
            ORDER BY c.date DESC LIMIT 5
        """, {"ph": phone_hash})
        return self._format(result)
```

**Returning borrower greeting:**
```
"નમસ્તે રમેશભાઈ, પાછા આવ્યા! તમારો છેલ્લો ₹50,000નો ભેંસ ખરીદી લોન
સમયસર ચૂકવાયો. આ વખતે શું મદદ કરી શકું?"
```

### Problems It Creates
- **Memory but no action:** Agent remembers the conversation but still cannot check real data or execute transactions.
- **Stale context:** Cross-session memory can become outdated if loan status changed since last interaction.
- **Token bloat:** Long conversations fill the context window. Need summarization or sliding window.

---

## LEVEL 2: TOOL USE

### What You're Adding
49+ MCP tools that let the agent read real data and execute real actions. The agent moves from conversational to transactional.

### How the Platform Implements It

**Services:** `mcp-server` (cross-cutting, 49+ tools), every core service exposes tools.

```python
# MCP tool registration via @tool decorator
from platform_mcp.tools import tool

@tool(
    name="fetch_uli_data",
    description="Fetch borrower data from ULI sources in parallel",
    session_state_required="VERIFIED",
    circuit_breaker={"failure_threshold": 5, "recovery_timeout": 30},
)
async def fetch_uli_data(borrower_id: str, segment: str) -> dict:
    """Parallel fetch from credit bureau, bank AA, cooperative, land records."""
    result = await uli_orchestrator.fetch(borrower_id, segment)
    return {
        "sources_fetched": len(result.completed),
        "confidence_score": result.confidence,
        "data_quality_flags": result.flags,
    }
```

**Session-state gating** prevents tools from running before prerequisites:

```
Session State   │ Allowed Tools (49 total)
────────────────┼────────────────────────────────────────
UNVERIFIED      │ 8 tools:  search_knowledge, detect_language, get_faq,
                │           check_eligibility, get_product_info, ...
VERIFIED        │ +25 tools: fetch_uli_data, evaluate_credit, match_lender,
                │            present_offer, get_borrower_context, ...
CONFIRMED       │ +16 tools: initiate_disbursement, execute_esign,
                │            trigger_moratorium, adjust_schedule, ...
```

### Problems It Creates
- **Tool hallucination:** LLM invents tool names or passes wrong parameters.
- **No domain knowledge:** Agent calls tools correctly but cannot explain lending terms, RBI rules, or product features to the borrower.
- **Compliance gaps:** Nothing prevents the agent from giving financial advice or misquoting interest rates.

---

## LEVEL 3: RETRIEVAL / RAG

### What You're Adding
7-partition knowledge base with verbatim retrieval for regulatory text. The agent now has authoritative domain knowledge instead of training-data priors.

### How the Platform Implements It

**Service:** `knowledge-svc` (S16) — ChromaDB/pgvector with partition routing.

```python
# 7 knowledge partitions
PARTITIONS = {
    "rates":            {"update": "daily",    "retrieval": "exact_match"},
    "products":         {"update": "weekly",   "retrieval": "semantic"},
    "compliance_rbi":   {"update": "on_change","retrieval": "verbatim"},   # ← NEVER paraphrase
    "segment_configs":  {"update": "on_change","retrieval": "exact_match"},
    "loss_mitigation":  {"update": "monthly",  "retrieval": "semantic"},
    "uli_docs":         {"update": "on_change","retrieval": "semantic"},
    "faq":              {"update": "weekly",   "retrieval": "semantic"},
}

async def retrieve(query: str, context: TurnContext) -> list[Chunk]:
    # Auto-route to relevant partitions based on query classification
    partitions = await classify_query_partitions(query)

    results = []
    for p in partitions:
        if PARTITIONS[p]["retrieval"] == "verbatim":
            # Compliance text: return exact chunk, no reranking
            chunks = await vector_store.query(p, query, top_k=3)
            for c in chunks:
                c.metadata["verbatim"] = True
            results.extend(chunks)
        else:
            # Semantic search with MMR diversification
            chunks = await vector_store.query(p, query, top_k=5, mmr_lambda=0.7)
            results.extend(chunks)

    return deduplicate_and_rank(results)
```

**Verbatim enforcement for RBI compliance:**
```python
# Agent runtime inserts verbatim chunks directly into response
if any(chunk.metadata.get("verbatim") for chunk in retrieved_chunks):
    # Inject as quoted block — LLM cannot paraphrase
    system_prompt += "\n\nCOMPLIANCE TEXT (quote exactly, do not paraphrase):\n"
    for chunk in retrieved_chunks:
        if chunk.metadata.get("verbatim"):
            system_prompt += f'"""\n{chunk.text}\n"""\n'
```

### Problems It Creates
- **Retrieval without flow:** Agent answers questions but doesn't guide borrowers through a structured journey (KYC → data fetch → decision → offer).
- **No segment awareness:** Same conversation flow for a Gujarat dairy farmer and a Chennai street vendor.
- **Channel blindness:** No adaptation for voice vs. WhatsApp vs. web.

---

## LEVEL 4: CONVERSATION FLOW / JOURNEYS

### What You're Adding
Segment-specific, channel-aware conversation journeys. The agent stops being reactive (answer questions) and becomes proactive (guide borrowers through origination).

### How the Platform Implements It

**Service:** `journey-studio` (S11) — flow definitions as JSON/YAML.

```yaml
# configs/journeys/dairy_voice_gujarati.json
{
  "id": "dairy_voice_gu",
  "segment": "DAIRY_FARMER",
  "channel": "VOICE",
  "language": "gu",
  "steps": [
    {"id": "greeting", "type": "template",
     "text": "નમસ્તે {name}, હું તમારો લોન સહાયક છું. ભેંસ ખરીદી લોન માટે મદદ કરું?"},
    {"id": "kyc", "type": "tool", "tool": "verify_aadhaar", "required": true,
     "on_failure": "kyc_failed_empathetic"},
    {"id": "data_fetch", "type": "tool", "tool": "fetch_uli_data", "parallel": true},
    {"id": "decision", "type": "tool", "tool": "evaluate_credit"},
    {"id": "offer", "type": "tool", "tool": "present_offer",
     "wait_for_acceptance": true, "timeout_hours": 24},
    {"id": "esign", "type": "tool", "tool": "execute_esign"},
    {"id": "disbursement", "type": "tool", "tool": "initiate_disbursement",
     "session_state": "CONFIRMED"}
  ]
}
```

```yaml
# configs/journeys/vendor_whatsapp_hindi.json — different flow, same platform
{
  "id": "vendor_wa_hi",
  "segment": "STREET_VENDOR",
  "channel": "WHATSAPP",
  "language": "hi",
  "steps": [
    {"id": "greeting", "type": "template",
     "text": "नमस्ते {name}! मैं आपका लोन सहायक हूं। ₹15,000 तक का कार्यशील पूंजी ऋण चाहिए?"},
    {"id": "photo_kyc", "type": "tool", "tool": "verify_pm_svanidhi_id"},
    {"id": "upi_analysis", "type": "tool", "tool": "analyze_upi_history"},
    {"id": "decision", "type": "tool", "tool": "evaluate_credit"},
    {"id": "offer", "type": "interactive_button",
     "options": ["स्वीकार करें", "अभी नहीं"]},
    {"id": "disbursement", "type": "tool", "tool": "initiate_disbursement"}
  ]
}
```

### Problems It Creates
- **No guardrails:** Journey defines what the agent *should* do, but nothing prevents it from going off-script (giving financial advice, skipping KYC).
- **Compliance is implicit:** RBI KFS verbatim requirement exists in knowledge base but isn't enforced.
- **No authority limits:** Agent could theoretically approve a ₹10Cr loan for a dairy farmer.

---

## LEVEL 5: POLICY ENFORCEMENT

### What You're Adding
Three-layer policy enforcement: structural (deterministic gates), behavioral (LLM supervisor), and authority (hard limits). The agent is now governed.

### How the Platform Implements It

**Service:** `sentinel-core` (S6) + `agent-runtime` (S13) policy middleware.

```
┌─────────────────────────────────────────────────────────┐
│         THREE-LAYER POLICY ENFORCEMENT                  │
│                                                         │
│  LAYER 1 — STRUCTURAL (if/else, fail-closed)            │
│  ┌────────────────────────────────────────────────────┐ │
│  │ Aadhaar gate: KYC before data fetch                │ │
│  │ Consent gate: DPDP token per data element          │ │
│  │ Velocity gate: max 2 apps per Aadhaar per 48h      │ │
│  │ Blacklist gate: RBI defaulter list check           │ │
│  │ Session-state gate: tools gated by auth level      │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  LAYER 2 — BEHAVIORAL (LLM supervisor, fail-open)       │
│  ┌────────────────────────────────────────────────────┐ │
│  │ RBI KFS: all 18 mandatory fields present           │ │
│  │ No financial advice: don't recommend specific loans │ │
│  │ Empathetic hardship: detect distress → escalate    │ │
│  │ Language appropriate: no jargon in vernacular       │ │
│  │ Consent boundary: only access consented data       │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  LAYER 3 — AUTHORITY (numeric, fail-closed)             │
│  ┌────────────────────────────────────────────────────┐ │
│  │ Disbursement max: segment.product.amount_range.max │ │
│  │ Fee waiver max: lender.config.max_fee_waiver_pct   │ │
│  │ Moratorium max: policy.monitoring.max_moratorium    │ │
│  │ Rate floor: rbi.config.min_rate                    │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

```python
# Policy check in agent-runtime turn processor (step 7 of 16)
async def check_policy(context: TurnContext) -> TurnContext:
    # Layer 1: Structural
    for gate in STRUCTURAL_GATES:
        result = gate.check(context)
        if not result.passed:
            context.block(reason=result.reason)  # Hard stop
            return context

    # Layer 3: Authority (check before tool execution)
    if context.pending_tool:
        for limit in AUTHORITY_LIMITS:
            result = limit.check(context.pending_tool, context.params)
            if not result.passed:
                context.block(reason=f"Authority limit: {result.reason}")
                return context

    # Layer 2: Behavioral (check after response generation)
    # Applied in step 13 (apply_compliance) — fail-open
    return context
```

### Problems It Creates
- **Single model does everything:** FAQ answers use the same expensive model as credit decisions. Classification uses the same model as reasoning.
- **No Indian language optimization:** Generic ASR/TTS performs poorly on Gujarati lending terms.
- **No failover:** If OpenAI is down, everything stops.

---

## LEVEL 6: MULTI-MODEL ROUTING

### What You're Adding
Constellation — the model abstraction layer. Routes tasks to optimal models by type. Adds Indian language specialist models and failover chains.

### How the Platform Implements It

**Service:** `constellation` (S15).

```python
# Task-type routing table
ROUTING_TABLE = {
    "classification":  {"primary": "gpt-4o-mini",    "fallback": "claude-3-haiku",    "target_ms": 200},
    "reasoning":       {"primary": "gpt-4o",          "fallback": "claude-3.5-sonnet", "target_ms": 2000},
    "extraction":      {"primary": "gpt-4o-mini",    "fallback": "gemini-1.5-flash",  "target_ms": 500},
    "generation":      {"primary": "gpt-4o",          "fallback": "claude-3.5-sonnet", "target_ms": 1000},
    "embedding":       {"primary": "text-embed-3-sm", "fallback": "voyage-3",          "target_ms": 100},
    "vision":          {"primary": "gpt-4o-vision",   "fallback": "gemini-1.5-pro",    "target_ms": 3000},
}

# Indian language specialist routing
LANGUAGE_ROUTING = {
    "asr": {"primary": "deepgram-nova-2", "fallback": "whisper-large-v3-finetuned"},
    "tts": {"primary": "bhashini",         "fallback": "elevenlabs-multilingual"},
    "translate": {"primary": "indictrans2", "fallback": "google-translate"},
}
```

**Cost impact:** Classification calls (intent detection, segment routing) use gpt-4o-mini at ~$0.15/1M tokens instead of gpt-4o at ~$2.50/1M tokens. For a platform handling 10K calls/day with 5 classification calls per turn, this saves ~$350/day.

### Problems It Creates
- **Flying blind:** Multiple models, multiple tools, multiple services — but no visibility into what happened during a turn. When a borrower complains, you cannot reconstruct the conversation flow.
- **No per-lender metrics:** Lender A cannot see their own agent performance vs. Lender B.

---

## LEVEL 7: OBSERVABILITY

### What You're Adding
End-to-end trace service with hierarchical spans: Session → Turn → Span. Per-lender dashboards. ULI source latency tracking.

### How the Platform Implements It

**Services:** `trace-service` (observability), Prometheus + Grafana + Loki + Jaeger.

```python
# Trace model — every turn produces a full execution trace
@dataclass
class TurnTrace:
    session_id: str
    turn_id: str
    spans: list[Span]  # Ordered execution spans

    # Example spans for a dairy farmer voice turn:
    # [0] language_detection:     12ms   provider=fasttext    result=gu
    # [1] asr_transcription:     340ms   provider=deepgram    confidence=0.87
    # [2] intent_classification:  89ms   provider=gpt-4o-mini result=NEW_LOAN
    # [3] knowledge_retrieval:   156ms   partition=products   chunks=3
    # [4] policy_check:           23ms   layer=structural     result=PASS
    # [5] tool_execution:       2100ms   tool=fetch_uli_data  sources=5
    #     ├─ sub: cibil_fetch    980ms   circuit=CLOSED       status=SUCCESS
    #     ├─ sub: aa_fetch      1800ms   circuit=CLOSED       status=SUCCESS
    #     ├─ sub: coop_fetch     650ms   circuit=CLOSED       status=SUCCESS
    #     ├─ sub: land_fetch    1200ms   circuit=CLOSED       status=TIMEOUT
    #     └─ sub: weather_fetch  400ms   circuit=CLOSED       status=SUCCESS
    # [6] model_call:            890ms   model=gpt-4o         tokens_in=1240 tokens_out=380
    # [7] policy_check:           18ms   layer=behavioral     result=PASS
    # [8] tts_synthesis:         210ms   provider=bhashini    language=gu
```

**Per-lender Grafana dashboard panels:**
- Applications funnel: created → KYC → decision → offer → disburse (conversion %)
- Decision latency distribution (p50/p95/p99) by segment
- ULI source availability heatmap (CIBIL, AA, cooperative, land, weather)
- Agent metrics: calls/hr, completion rate, escalation %, avg turn count
- Portfolio: AUM, DPD buckets, NPA %, monitoring triggers/day

### Problems It Creates
- **Observable but unproven:** You can see what the agent does, but you cannot prove it works correctly across all scenarios before deploying a new version.
- **No regression detection:** A policy change might break the dairy happy path while improving the vendor path.

---

## LEVEL 8: EVALUATION

### What You're Adding
tau-bench style lending scenarios with LLM-as-borrower simulator. State-based scoring. pass^k metric. Automated regression gating in CI/CD.

### How the Platform Implements It

**Service:** `evaluation-svc` (S11 port 8011).

```python
# Evaluation scenario definition
SCENARIOS = [
    Scenario(
        id="dairy_happy_path",
        persona="Ramesh, Gujarat dairy farmer, 45, cooperative member for 8 years",
        language="gu",
        goal="Apply for ₹50,000 animal purchase loan",
        expected_state_path=["CREATED", "KYC_CHECK", "DATA_FETCH", "APPROVED", "ACTIVE"],
        expected_tools=["verify_aadhaar", "fetch_uli_data", "evaluate_credit",
                        "present_offer", "initiate_disbursement"],
        max_turns=20,
    ),
    Scenario(
        id="weather_moratorium",
        persona="Active dairy borrower, 3 months into loan, heat wave in district",
        language="gu",
        goal="Receive moratorium notification and understand schedule change",
        expected_state_path=["ACTIVE → MORATORIUM"],
        expected_tools=["trigger_moratorium", "adjust_schedule"],
        max_turns=10,
    ),
    Scenario(
        id="fraud_ring_detection",
        persona="Synthetic applicant sharing references with 3 defaulters",
        language="hi",
        goal="Application should be flagged for fraud",
        expected_flags=["SHARED_REFERENCE", "FRAUD_RING"],
        max_turns=15,
    ),
    # ... 5 more scenarios (vendor, gig emergency, KYC failure, returning, hardship)
]

# pass^k scoring
def compute_pass_at_k(results: list[bool], k: int = 3) -> float:
    """Run scenario k times. pass^k = P(pass at least once in k runs)."""
    n = len(results)
    c = sum(results)
    if n - c < k:
        return 1.0
    return 1.0 - math.comb(n - c, k) / math.comb(n, k)
```

**CI/CD gate:** New agent versions must achieve pass^k ≥ 0.85 across all 8 scenarios before promotion to staging. Shadow mode runs both old and new in parallel — only the old agent's decisions are live.

### Problems It Creates
- **Eval passes but deployment is risky:** No versioned config management, no canary pipeline, no rollback mechanism.
- **Config changes bypass eval:** A segment YAML change (new scoring weights) doesn't trigger the eval suite.

---

## LEVEL 9: VERSION CONTROL & DEPLOYMENT

### What You're Adding
Workspace service for configuration-as-code. Temporal workflow durability. Segment config versioning with diff engine. Canary deployment pipeline.

### How the Platform Implements It

**Service:** `workspace-service` (S9 / journey-studio S11).

```
Config Promotion Pipeline:
  DRAFT ──► QA (auto-eval, pass^k ≥ 0.85) ──► STAGING (shadow test) ──► PRODUCTION
    │                                                                        │
    └── Every change gets: version bump, SHA-256 hash, diff vs previous ─────┘
                                                                         Rollback: instant
                                                                         (revert to prev hash)
```

**Canary deployment (7-stage):**
```
Stage 1:  1% traffic,   4h,   100 requests  → check error rate < 5%
Stage 2:  5% traffic,   8h,   500 requests  → check latency p95 < 2s
Stage 3: 10% traffic,  12h, 1,000 requests  → check task completion > 85%
Stage 4: 25% traffic,  24h, 2,500 requests  → check policy violations < 1%
Stage 5: 50% traffic,  24h, 5,000 requests  → all metrics green
Stage 6: 75% traffic,  24h, 7,500 requests  → all metrics green
Stage 7: 100% traffic, 48h, 10,000 requests → promote to production
Auto-rollback: any metric breach → revert to previous version instantly
```

**Temporal durability:** Loan origination workflow survives pod restarts. If a pod dies mid-disbursement, Temporal replays the workflow from the last completed activity — no double-disbursement, no lost state.

### Problems It Creates
- **Text-only agent:** Platform works for WhatsApp and web, but 40% of target borrowers (rural, low-literacy) need voice in their language.
- **No phone integration:** Cannot receive a call from a toll-free number.

---

## LEVEL 10: VOICE

### What You're Adding
Indian language ASR (12 languages), Bhashini TTS, lending pronunciation engine, Exotel PSTN integration.

### How the Platform Implements It

**Service:** `voice-pipeline` (S14).

```
Borrower Phone Call (toll-free)
       │
       ▼
  Exotel PSTN ──WebSocket──► Voice Pipeline (S14)
                                 │
                                 ├── VAD (energy-based, 300ms silence = end-of-utterance)
                                 │
                                 ├── ASR
                                 │   ├─ Deepgram Nova-2 + lending vocab boost
                                 │   │   gu: ["લોન", "EMI", "ભેંસ", "સહકારી"]
                                 │   │   hi: ["लोन", "किस्त", "ब्याज", "आधार"]
                                 │   └─ Fallback: Whisper large-v3 (if confidence < 0.7)
                                 │
                                 ├── Agent Runtime (S13) — 16-step turn processor
                                 │   ├─ Fast Brain: cached FAQ → <200ms response
                                 │   └─ Slow Brain: MCP tool call → <2s response
                                 │
                                 ├── TTS
                                 │   ├─ Bhashini (Indian govt TTS, natural prosody)
                                 │   └─ Fallback: ElevenLabs multilingual
                                 │
                                 └── Pronunciation Engine
                                     EMI → "ee-em-ai"
                                     FOIR → "foy-ar"
                                     CIBIL → "sih-bil"
                                     NACH → "naach"
                                     KFS → "key fact statement"
                                     FLDG → "first loss default guarantee"
```

**12 supported languages:** Gujarati (gu), Hindi (hi), Marathi (mr), Tamil (ta), Telugu (te), Kannada (kn), Malayalam (ml), Bengali (bn), Punjabi (pa), Odia (or), Assamese (as), Urdu (ur).

**Latency budget per voice turn:**
```
VAD end-of-utterance:  300ms
ASR transcription:     350ms (target)
Agent processing:      800ms (fast) / 2000ms (slow)
TTS synthesis:         250ms
Network + buffering:   200ms
─────────────────────────────────────
Total:                1.9s (fast) / 3.1s (slow)
Target:               <1.5s (fast) / <3.0s (slow)
```

### Problems It Creates
- **Mocked integrations:** All 136 ULI services, credit bureaus, cooperative APIs, and disbursement channels are still mocked.
- **No real data flow:** Cannot actually verify an Aadhaar, fetch a bank statement, or disburse funds.

---

## LEVEL 11: DPI/ULI INTEGRATION

### What You're Adding
Real connections to India's Digital Public Infrastructure. Parallel fetch engine with circuit breakers. Consent management. Graceful degradation when sources are unavailable.

### How the Platform Implements It

**Service:** `uli-orchestrator` (S1) + `identity-svc` (S5) + `bankconnect-svc` (S3).

```python
# Parallel ULI fetch with circuit breakers per source
class ULIFetchEngine:
    SOURCE_CIRCUIT_BREAKERS = {
        "CREDIT_BUREAU_CIBIL":  CircuitBreaker(failure_threshold=5, recovery_timeout=30),
        "BANK_ACCOUNT_AA":      CircuitBreaker(failure_threshold=5, recovery_timeout=30),
        "DAIRY_COOPERATIVE":    CircuitBreaker(failure_threshold=3, recovery_timeout=60),
        "LAND_RECORDS":         CircuitBreaker(failure_threshold=3, recovery_timeout=120),
        "IMD_WEATHER":          CircuitBreaker(failure_threshold=3, recovery_timeout=60),
        "SATELLITE_NDVI":       CircuitBreaker(failure_threshold=3, recovery_timeout=120),
        # ... 136 total services in ULI catalog
    }

    async def fetch_parallel(self, borrower_id: str, segment: str) -> FetchResult:
        config = segment_registry.get(segment)

        # Launch all fetches in parallel with per-source timeouts
        tasks = {
            source.name: asyncio.create_task(
                self._fetch_with_breaker(source, borrower_id)
            )
            for source in config.required_sources + config.optional_sources
        }

        # Gather with global timeout
        results = await asyncio.gather(
            *[self._wrap_timeout(name, task, config.max_total_timeout_s)
              for name, task in tasks.items()],
            return_exceptions=True,
        )

        return self._build_result(results, config.required_sources)
```

**Consent management (DPDP Act):**
```python
# Every data element fetch requires a valid consent token
async def validate_consent(borrower_id: str, data_element: str) -> bool:
    consent = await consent_ledger.get_active(borrower_id, data_element)
    if not consent or consent.expired:
        raise ConsentRequiredError(f"No active consent for {data_element}")
    return True
```

**Graceful degradation:** If optional sources (weather, satellite) fail, the decision proceeds with reduced confidence. If required sources (CIBIL, AA) fail, the application is held in `PARTIAL` state and retried.

### Problems It Creates
- **Static system:** Agent doesn't learn from outcomes. A dairy farmer who repaid perfectly gets the same treatment as a first-time applicant.
- **No feedback loop:** Model drift goes undetected. Policy effectiveness is unknown.
- **No graduation:** Successful borrowers are not offered better terms or larger loans.

---

## LEVEL 12: CONTINUOUS IMPROVEMENT

### What You're Adding
SRAL (Sense-Reason-Act-Learn) feedback loop. Monitoring-agent for real-time triggers. RL signals for model retraining. Graduation detection.

### How the Platform Implements It

**Services:** `monitoring-agent` (S18), `ml-platform` (cross-cutting), `context-graph` (S17).

```python
# SRAL Loop — every loan outcome feeds back into the system
class SRALLoop:
    async def sense(self, loan_id: str):
        """Collect outcome signals."""
        signals = []
        signals.append(await self.repayment_tracker.get_status(loan_id))
        signals.append(await self.cooperative_watcher.get_production(loan_id))
        signals.append(await self.weather_watcher.get_conditions(loan_id))
        return signals

    async def reason(self, signals: list[Signal]):
        """Classify situation and determine intervention."""
        if signals.repayment.dpd > 0 and signals.weather.severe:
            return Intervention(type="AUTO_MORATORIUM", reason="weather_distress")
        if signals.repayment.months_clean >= 12:
            return Intervention(type="GRADUATION_OFFER", reason="excellent_repayment")
        if signals.cooperative.production_drop_pct > 40:
            return Intervention(type="SCHEDULE_ADJUST", reason="production_decline")
        return None

    async def act(self, intervention: Intervention):
        """Execute the intervention."""
        if intervention.type == "AUTO_MORATORIUM":
            await moratorium_tool.trigger(intervention.loan_id, intervention.months)
        elif intervention.type == "GRADUATION_OFFER":
            await graduation_tool.offer(intervention.loan_id)
        elif intervention.type == "SCHEDULE_ADJUST":
            await schedule_tool.adjust(intervention.loan_id)

    async def learn(self, intervention: Intervention, outcome: Outcome):
        """Emit RL signal for model retraining."""
        await kafka.send("rl.signal", {
            "loan_id": intervention.loan_id,
            "intervention_type": intervention.type,
            "outcome": outcome.status,  # IMPROVED, UNCHANGED, WORSENED
            "reward": self._compute_reward(outcome),
        })
```

**Graduation detection:**
```python
# Borrowers with 12+ months clean repayment get pre-approved for larger loans
async def detect_graduation_candidates(lender_id: str) -> list[GraduationOffer]:
    candidates = await context_graph.query("""
        MATCH (b:Borrower)-[:HAS_LOAN]->(l:Loan)
        WHERE l.status = 'ACTIVE'
          AND l.lender_id = $lender_id
          AND l.months_clean >= 12
          AND NOT (l)-[:GRADUATED_TO]->()
        RETURN b, l
    """, {"lender_id": lender_id})

    offers = []
    for b, l in candidates:
        new_amount = l.disbursed_amount * 1.5  # 50% increase
        new_rate = l.rate - 0.5                # 50bps reduction
        offers.append(GraduationOffer(borrower=b, current_loan=l,
                                       new_amount=new_amount, new_rate=new_rate))
    return offers
```

**Model retraining pipeline:**
```
RL Signals (Kafka)  ──►  Feature Store (Feast)  ──►  Model Retrain (MLflow)
                                                          │
                                                          ▼
                                                    Champion/Challenger
                                                    (shadow for 2 weeks)
                                                          │
                                                          ▼
                                                    If Gini improvement > 2%:
                                                      Promote to Production
```

---

# COST OF SKIPPING LEVELS

| Skipped Level | Immediate Consequence | Production Consequence |
|---------------|----------------------|----------------------|
| **L0 → L2** (skip memory) | Agent asks borrower to repeat Aadhaar every turn | 80% call abandonment rate; borrowers stop calling |
| **L0 → L3** (skip tools) | Agent can talk about loans but cannot process them | Zero originations; expensive chatbot with no ROI |
| **L2 → L5** (skip policy) | Agent originates loans without compliance checks | RBI enforcement action; license risk for lender |
| **L3 → L5** (skip journeys) | Same flow for dairy farmer and gig worker | Poor conversion; wrong product fit; high NPA |
| **L5 → L7** (skip routing) | GPT-4o for every FAQ answer | 10x higher API costs; no Indian language optimization |
| **L5 → L8** (skip observability) | Cannot debug failed conversations | Blind to agent errors; borrower complaints unresolvable |
| **L7 → L9** (skip eval) | Ship agent changes without regression testing | "Works on my machine" → broken dairy flow in production |
| **L8 → L10** (skip versioning) | Config changes go live without review or rollback | Bad policy deployed at 2am → auto-approving bad loans |
| **L9 → L11** (skip voice) | 40% of target borrowers cannot use the platform | Excludes rural, low-literacy segments (core ICP) |
| **L10 → L12** (skip DPI) | All data mocked; no real Aadhaar/AA/bureau integration | Platform is a demo, not a product |
| **L11 → L12** (skip learning) | Agent never improves; models drift; no graduation | NPA rises over time; borrowers never get better terms |
| **Any 2 levels** | Compounding debt | Each skipped level makes the next harder to add; platform becomes unmaintainable |

---

# LEVEL DEPENDENCY GRAPH

```
L0 (Bare LLM)
 └── L1 (Memory)
      └── L2 (Tools)
           ├── L3 (RAG)
           │    └── L4 (Journeys)
           │         └── L5 (Policy)
           │              ├── L6 (Multi-Model)
           │              │    └── L7 (Observability)
           │              │         └── L8 (Evaluation)
           │              │              └── L9 (Versioning)
           │              │                   ├── L10 (Voice)
           │              │                   └── L11 (DPI/ULI)
           │              │                        └── L12 (Continuous Improvement)
           │              └── (L6-L12 all depend on L5 being solid)
           └── (L3-L12 all depend on L2 tool infrastructure)
```

**Critical path to first pilot:** L0 → L1 → L2 → L3 → L4 → L5 → L9 → L11 (8 levels, ~20 weeks)
**Full production:** All 13 levels, ~40 weeks with 8-10 engineers.
