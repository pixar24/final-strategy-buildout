# Voice AI Knowledge Distillation for Indian Lending
## Why Voice Is the Primary Channel, Not a Feature

---

# 1. WHY VOICE CHANGES EVERYTHING FOR INDIAN LENDING

## The 300 Million Borrowers Who Cannot Use Apps

India's underserved lending market -- 300M borrowers across agriculture, micro-enterprise, and gig economy -- is fundamentally different from US/EU mortgage markets in one critical dimension: **the primary interface is voice, not text.**

| Dimension | US Mortgage Market | Indian Underserved Lending |
|-----------|-------------------|---------------------------|
| Primary channel | Web app, PDF upload | Voice call (mobile phone) |
| Literacy level | High (text-first) | Mixed: 30%+ semi-literate |
| Language | English (single) | 12+ languages, code-mixed |
| Device | Laptop/smartphone | Feature phone / low-end Android |
| Connectivity | Broadband | 2G/3G in rural areas |
| Typical borrower | Salaried professional | Dairy farmer, street vendor, gig worker |
| Loan amount | $200K-$500K | ₹10K-₹2L ($120-$2,400) |
| Documentation | W-2, tax returns, bank statements (digital) | Aadhaar, milk pour records, mandi receipts (mostly paper) |

### Voice Is Not Optional

For a dairy farmer in Kutch, Gujarat:
- She cannot navigate an app-based loan application
- She speaks Gujarati with Hindi code-mixing, not English
- She has a feature phone or low-end Android with intermittent 3G
- Her "financial documents" are cooperative milk pour receipts
- She knows her Aadhaar number but needs to speak it, not type it
- She understands EMI amounts when spoken aloud: "pachis hazaar" (₹25,000)

For a street vendor in Chennai:
- He speaks Tamil with English loan terminology mixed in
- He has no formal bank statements (cash business)
- He has a smartphone but low digital literacy
- He can explain his business verbally but cannot fill forms
- His "income proof" is daily sales he can describe but not document

For a gig worker in Bangalore:
- She code-switches between Kannada, Hindi, and English
- Her income is variable (₹800 on good days, ₹200 on bad)
- She can describe her earnings pattern verbally
- She needs to understand complex terms: EMI, FOIR, NACH, CIBIL

**Voice IS the primary channel.** Not a convenience feature. Not an add-on. The entire lending experience -- from application to disbursement to collections -- must be voice-native for these 300M borrowers.

---

# 2. INDIAN LANGUAGE ASR CHALLENGES

## 12 Languages, Domain Vocabulary, and Code-Mixing

### Target Languages

| Phase | Languages | Geography | Borrower Segments |
|-------|-----------|-----------|-------------------|
| Phase 1 | Gujarati, Hindi | Gujarat dairy belt, Hindi-belt MFI | Dairy farmers, SHG women |
| Phase 2 | Tamil, Telugu, Kannada | SFB geographies (South India) | Kirana shops, gig workers |
| Phase 3 | Bengali, Marathi, Odia | MFI expansion (East + West) | Agriculture, micro-enterprise |
| Phase 4 | Malayalam, Punjabi, Assamese, Rajasthani | Full national coverage | All 38 segments |

### Domain-Specific Vocabulary

Standard ASR models (Google, Deepgram, Whisper) are trained on general conversation. Indian lending has specialized vocabulary that these models consistently misrecognize:

```
┌────────────────────────────────────────────────────────────────┐
│  LENDING DOMAIN VOCABULARY (HIGH RECOGNITION ERROR TERMS)      │
│                                                                │
│  Financial Terms:                                              │
│    EMI         → Often recognized as "Emmy" or "Amy"           │
│    CIBIL       → Misheard as "civil" or "sybil"               │
│    NACH        → Misheard as "match" or "much"                 │
│    FOIR        → Almost never recognized correctly              │
│    KYC         → Misheard as "Casey"                           │
│    NPA         → Misheard as "N-P-A" (correct) or "inpa"      │
│    PSL         → Misheard as "P-S-L" (correct) or garbled     │
│    ULI         → Misheard as "Julie" or "Yuli"                │
│                                                                │
│  Agricultural Terms:                                           │
│    buffalo     → "bhens" (Gujarati) / "bhains" (Hindi)        │
│    cooperative → "sahkari mandali" (Gujarati)                  │
│    milk pour   → "dudh bharvu" (Gujarati)                      │
│    harvest     → "laṇṇī" (Gujarati) / "fasal" (Hindi)        │
│    monsoon     → "chomasu" (Gujarati) / "baarish" (Hindi)     │
│    mandi       → "mandi" (universal, market yard)              │
│                                                                │
│  Aadhaar Numbers:                                              │
│    12-digit sequences spoken as groups: "4823-5167-9034"       │
│    Often spoken with pauses, repetitions, corrections          │
│    Critical: single-digit error = verification failure          │
│                                                                │
│  Rupee Amounts:                                                │
│    "pachis hazaar" (₹25,000 in Hindi)                          │
│    "das hazaar" (₹10,000 in Hindi)                             │
│    "ek lakh" (₹1,00,000 -- Indian numbering system)           │
│    Regional variants for the same number                       │
└────────────────────────────────────────────────────────────────┘
```

### Code-Mixing Challenge

Indian borrowers routinely code-mix -- switching languages mid-sentence:

**Gujarati-Hindi-English mixing:**
> "Maro CIBIL score ketlo chhe? Mane EMI kitni hogi monthly?"
> (My CIBIL score how-much is? To-me EMI how-much will-be monthly?)

**Tamil-English mixing:**
> "Ennoda loan status enna? EMI amount reduce panna mudiyuma?"
> (My loan status what? EMI amount reduce do can-you?)

**Hindi-English mixing:**
> "Mera Aadhaar number hai twelve-four-eight... wait... four-eight-two-three..."
> (My Aadhaar number is twelve-four-eight... wait... four-eight-two-three...)

ASR models must handle:
- Language switching within a single utterance
- English loan terminology embedded in vernacular speech
- Number representation mixing ("twenty-five thousand" / "pachis hazaar" / "25,000")
- Borrower self-corrections (especially for Aadhaar numbers)

### Target Performance: WER < 15%

| Metric | Target | Current Baseline (off-the-shelf) |
|--------|--------|----------------------------------|
| Overall WER | < 15% | 25-35% (Gujarati lending domain) |
| Number recognition accuracy | > 98% | 85-90% |
| Aadhaar digit accuracy | > 99% | 90-92% |
| Domain term recognition | > 90% | 40-60% |
| Code-mixed utterance WER | < 20% | 35-45% |

Achieving these targets requires: domain vocabulary boosting, fine-tuned models, and lending-specific language models.

---

# 3. VOICE PIPELINE ARCHITECTURE

## End-to-End: Phone to Agent to Phone

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VOICE PIPELINE ARCHITECTURE                       │
│                                                                     │
│  BORROWER                                                           │
│  (Feature Phone / Smartphone)                                       │
│       │                                                             │
│       │ PSTN / Mobile Call                                          │
│       ▼                                                             │
│  ┌──────────┐                                                       │
│  │ EXOTEL   │  India-native telephony (toll-free, PSTN, IVR)       │
│  │ (SIP)    │  TRAI-compliant, call recording, consent capture     │
│  └────┬─────┘                                                       │
│       │ WebSocket (audio stream)                                    │
│       ▼                                                             │
│  ┌──────────────┐                                                   │
│  │ VAD          │  Voice Activity Detection                         │
│  │ (500ms       │  Lending-tuned: 500ms silence threshold           │
│  │  threshold)  │  (vs typical 300ms) because borrowers pause       │
│  │              │  to recall Aadhaar numbers, loan amounts           │
│  └────┬─────────┘                                                   │
│       │ Audio segments                                              │
│       ▼                                                             │
│  ┌──────────────┐    ┌───────────────────┐                         │
│  │ STT PRIMARY  │    │ STT FALLBACK      │                         │
│  │ Deepgram     │    │ Whisper Fine-tuned │                         │
│  │ (streaming,  │    │ (higher accuracy,  │                         │
│  │  low latency │    │  higher latency)   │                         │
│  │  ~150ms)     │    │  ~400ms)           │                         │
│  └────┬─────────┘    └───────────────────┘                         │
│       │ Text (transcription)                                        │
│       ▼                                                             │
│  ┌──────────────────────────────────────┐                           │
│  │ AGENT RUNTIME                        │                           │
│  │                                      │                           │
│  │  ┌─────────┐    ┌────────────────┐   │                           │
│  │  │ Fast    │    │ Slow Brain     │   │                           │
│  │  │ Brain   │    │ (Full agent    │   │                           │
│  │  │ (<50ms) │    │  runtime,      │   │                           │
│  │  │ Keyword │    │  MCP tools,    │   │                           │
│  │  │ match + │    │  Sentinel      │   │                           │
│  │  │ ack     │    │  policy)       │   │                           │
│  │  └────┬────┘    └───────┬────────┘   │                           │
│  │       │                 │             │                           │
│  │       ▼                 ▼             │                           │
│  │  [Immediate ack]  [Full response]    │                           │
│  └──────────────────────────────────────┘                           │
│       │ Response text                                               │
│       ▼                                                             │
│  ┌──────────────┐    ┌───────────────────┐                         │
│  │ TTS PRIMARY  │    │ TTS FALLBACK      │                         │
│  │ Bhashini     │    │ ElevenLabs        │                         │
│  │ (Indian lang │    │ (English, high    │                         │
│  │  native,     │    │  quality,         │                         │
│  │  streaming)  │    │  streaming)       │                         │
│  │  ~150ms      │    │  ~200ms           │                         │
│  └────┬─────────┘    └───────────────────┘                         │
│       │ Audio stream                                                │
│       ▼                                                             │
│  ┌──────────┐                                                       │
│  │ EXOTEL   │  Audio playback to caller                            │
│  │ (SIP)    │                                                       │
│  └────┬─────┘                                                       │
│       │                                                             │
│       ▼                                                             │
│  BORROWER HEARS RESPONSE                                           │
│                                                                     │
│  Total target latency: < 1.5 seconds end-to-end                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Selection Rationale

| Component | Primary | Fallback | Why |
|-----------|---------|----------|-----|
| Telephony | Exotel | — | India-native, toll-free support, TRAI compliant, PSTN connectivity (critical for feature phones) |
| VAD | Silero VAD (500ms) | — | Open-source, low-latency, customizable threshold |
| STT | Deepgram | Whisper fine-tuned | Deepgram: streaming + low latency. Whisper: higher accuracy for Indic languages after fine-tuning |
| Agent | Custom runtime | — | Lending-specific: Sentinel policy, MCP tools, graph context |
| TTS | Bhashini | ElevenLabs | Bhashini: government-backed Indian language TTS. ElevenLabs: superior English quality |

---

# 4. FAST BRAIN / SLOW BRAIN FOR LENDING

## Why Sub-50ms Acknowledgment Matters

Conversational AI has a perception threshold: if the system takes > 1.5 seconds to respond, borrowers perceive it as broken or unresponsive. But meaningful lending responses (checking loan status, computing EMI, evaluating policy) require 500-1000ms of processing.

**Solution: Fast Brain for acknowledgment, Slow Brain for substance.**

### Fast Brain (<50ms)

Pattern-matching engine with zero API calls:

```
┌──────────────────────────────────────────────────────────┐
│  FAST BRAIN: KEYWORD → ACKNOWLEDGMENT MAP                 │
│                                                          │
│  Detected Intent         Fast Brain Response             │
│  ─────────────────────────────────────────────────────  │
│  "loan status"        → "Let me check your loan status"  │
│  "EMI amount/kitni"   → "Looking up your EMI details"    │
│  "next payment/agla"  → "Let me find your payment date"  │
│  "Aadhaar/aadhar"     → "Ready for your Aadhaar number"  │
│  "balance/bakaya"     → "Checking your balance now"       │
│  "apply/loan chahiye" → "Let me help you with that"       │
│  "problem/dikkat"     → "I understand, let me look into"  │
│  "speak to someone"   → "Connecting you now"              │
│                                                          │
│  Implementation: Redis hash lookup, <50ms                │
│  NO LLM call. NO API call. Pure pattern match.           │
└──────────────────────────────────────────────────────────┘
```

### Slow Brain (~800ms)

Full agent runtime with MCP tool calls, Sentinel policy evaluation, graph context:

```
Slow Brain Pipeline:
  1. Load borrower context from graph (50ms)
  2. Evaluate Sentinel policy gates (30ms)
  3. Select and execute MCP tools (500ms)
     - fetch_loan_status, compute_emi, check_eligibility, etc.
  4. Generate natural language response (200ms)
  5. Apply pronunciation rules (20ms)
```

### Parallel Execution

Fast and slow brain run in parallel. Borrower hears fast brain immediately, then slow brain response replaces/augments:

```
  Timeline (ms):
  0        50       200      500      800      1200     1500
  |--------|--------|--------|--------|--------|--------|
  
  [VAD+ASR: 200ms        ]
           [Fast Brain: 50ms]
           [──── Slow Brain: 800ms ────────────]
                                                [TTS: 200ms]
  
  Borrower hears:
  @ 450ms:  "Let me check your EMI details"  (fast brain via TTS)
  @ 1400ms: "Your EMI is ₹2,450, due on the 15th" (slow brain via TTS)
```

### Why 500ms VAD (Not 300ms)

Standard voice assistants use 300ms silence threshold (if user is silent for 300ms, assume they're done talking). This fails for lending:

- **Aadhaar numbers:** Borrowers pause between digit groups: "4823... [400ms pause] ...5167... [350ms pause] ...9034"
- **Loan amounts:** Borrowers think about exact numbers: "I want... [450ms] ...fifty thousand"
- **Complex questions:** "My EMI... [500ms] ...when is the next one due?"

A 300ms VAD would cut off borrowers mid-sentence. **500ms is calibrated for lending conversations** where borrowers need time to recall numbers and formulate questions about unfamiliar financial concepts.

---

# 5. PRONUNCIATION ENGINE FOR INDIAN LENDING

## Getting Financial Terms Right

Mispronouncing financial terms destroys trust. If the TTS says "Emmy" instead of "E-M-I", or "civil score" instead of "CIBIL score", the borrower doubts the system's competence.

### Pronunciation Rules

```
┌──────────────────────────────────────────────────────────────┐
│  PRONUNCIATION ENGINE: LENDING-SPECIFIC RULES                 │
│                                                              │
│  ACRONYMS (spell out):                                       │
│    EMI    → "E-M-I"       (not "Emmy")                       │
│    FOIR   → "F-O-I-R"     (not "foy-er")                    │
│    KYC    → "K-Y-C"       (not "kick")                       │
│    NPA    → "N-P-A"       (not "npa")                        │
│    PSL    → "P-S-L"       (not "psl")                        │
│    SFB    → "S-F-B"       (not "sfb")                        │
│    NBFC   → "N-B-F-C"     (not "nbfc")                      │
│    RBI    → "R-B-I"       (not "rbi")                        │
│                                                              │
│  DOMAIN TERMS (specific pronunciation):                       │
│    CIBIL  → "SIB-il"      (not "see-bil" or "civil")        │
│    NACH   → "NATCH"       (not "naash" or "match")          │
│    Aadhaar→ "AAH-dhaar"   (not "a-dar" or "adder")          │
│    NEFT   → "NEFT"        (rhymes with "left")              │
│    IMPS   → "IMPS"        (as written)                       │
│    UPI    → "U-P-I"       (not "upee")                       │
│                                                              │
│  RUPEE AMOUNTS (language-specific):                           │
│    ₹25,000 →                                                 │
│      Hindi:    "pachis hazaar rupaye"                         │
│      Gujarati: "pachis hazar rupiya"                          │
│      Tamil:    "irupattaiynthu aayiram rupaai"               │
│                                                              │
│    ₹1,00,000 (1 lakh) →                                     │
│      Hindi:    "ek laakh rupaye"                              │
│      Gujarati: "ek laakh rupiya"                              │
│      Tamil:    "oru latcham rupaai"                           │
│                                                              │
│  AADHAAR NUMBERS (digit-by-digit with grouping):             │
│    4823-5167-9034 →                                          │
│      "four-eight-two-three [pause]                            │
│       five-one-six-seven [pause]                              │
│       nine-zero-three-four"                                   │
│                                                              │
│    In Hindi: "chaar-aath-do-teen [pause]                      │
│              paanch-ek-chhah-saat [pause]                     │
│              nau-shunya-teen-chaar"                            │
└──────────────────────────────────────────────────────────────┘
```

### Implementation

Pronunciation rules applied as a post-processing step between agent response generation and TTS:

```
Agent Output:  "Your EMI is ₹25,000, due on the 15th"
                     │
                     ▼
Pronunciation Engine (regex + lookup table):
  - "EMI" → SSML: <say-as interpret-as="characters">EMI</say-as>
  - "₹25,000" → language-specific: "pachis hazaar rupaye" (Hindi)
  - "15th" → "pandrahvi" (Hindi) / "pandharmi" (Gujarati)
                     │
                     ▼
TTS Input:     Processed text with SSML tags
```

---

# 6. LATENCY ENGINEERING

## Target: <1.5s End-to-End

Every millisecond matters. Rural borrowers on 2G/3G have limited patience and high background noise. Long pauses create confusion.

### Latency Budget

```
┌───────────────────────────────────────────────────────────┐
│  END-TO-END LATENCY BUDGET                                 │
│                                                           │
│  Component              Target    Optimization             │
│  ──────────────────────────────────────────────────────── │
│  Network (up)           100ms     Exotel India PoPs        │
│  VAD processing          20ms     Silero VAD on CPU        │
│  ASR (Deepgram)         200ms     Streaming mode, GPU      │
│  Agent Runtime          800ms     Fast brain parallel       │
│    - Context load:  50ms          Redis cache for hot data  │
│    - Policy eval:   30ms          Pre-compiled rules        │
│    - Tool calls:   500ms          Async, parallel where ok  │
│    - NLG:          200ms          Streaming generation      │
│    - Pronunciation: 20ms          Regex + lookup table      │
│  TTS (Bhashini)         200ms     Streaming, chunk-based    │
│  Network (down)         100ms     Exotel India PoPs        │
│  ──────────────────────────────────────────────────────── │
│  TOTAL                 1420ms     Within 1.5s budget       │
│                                                           │
│  With fast brain:       450ms     First response to user   │
│  (Network + VAD + ASR + Fast + TTS)                        │
└───────────────────────────────────────────────────────────┘
```

### Optimization Strategies

**1. GPU Inference for ASR**
- Whisper fine-tuned model runs on g5.xlarge (NVIDIA A10G)
- Batch size 1 (real-time), FP16 inference
- Reduces Whisper latency from 800ms (CPU) to 200ms (GPU)

**2. Streaming TTS**
- Chunk-based synthesis: start speaking first sentence while generating rest
- Reduces perceived latency by 200-400ms
- Bhashini streaming API returns audio chunks as generated

**3. Edge Deployment for Rural Connectivity**
- ASR model replicated to edge nodes closer to rural borrowers
- Reduces network latency from 200ms to 50ms for edge-served regions
- Fallback to cloud if edge unavailable

**4. Context Pre-Loading**
- On call connect (before borrower speaks), pre-load:
  - Borrower profile from graph (if returning borrower, via caller ID)
  - Active loan details from Redis cache
  - District-level weather data
- Reduces agent runtime context load from 50ms to ~5ms

**5. Connection Pooling**
- Persistent WebSocket connections to ASR/TTS providers
- Database connection pools (PostgreSQL, Redis, Neo4j)
- Eliminates connection setup overhead (~50ms per cold connection)

---

# 7. EMOTIONAL INTELLIGENCE IN LENDING VOICE

## Distress Detection Is Structural, Not Optional

Lending conversations carry emotional weight. A borrower calling about a missed payment due to a family member's death or a flood destroying their crops is in genuine distress. The voice agent must detect this and respond appropriately.

### Distress Detection System

```
┌───────────────────────────────────────────────────────────┐
│  DISTRESS DETECTION: MULTI-SIGNAL APPROACH                 │
│                                                           │
│  Signal 1: KEYWORD DETECTION                               │
│  ┌─ "husband/wife passed" / "pati/patni ka dehaant"       │
│  ├─ "flood" / "baadh" / "paani aa gaya"                   │
│  ├─ "drought" / "sukhaa" / "paani nahi"                   │
│  ├─ "hospital" / "aspatal" / "beemar"                     │
│  ├─ "accident" / "hadsa"                                   │
│  ├─ "died" / "mar gaya" / "guzar gaye"                    │
│  ├─ "lost everything" / "sab khatam"                       │
│  └─ "can't pay" / "nahi de sakta" / "paisa nahi hai"      │
│                                                           │
│  Signal 2: TONAL ANALYSIS                                  │
│  ┌─ Pitch variance: high variance = emotional distress     │
│  ├─ Speech rate: sudden decrease = hesitation/sadness       │
│  ├─ Volume: sudden drop = withdrawal/hopelessness          │
│  └─ Extended silence: > 3 seconds = processing grief       │
│                                                           │
│  Signal 3: CONVERSATION PATTERN                            │
│  ┌─ Repeated "I don't know" responses                      │
│  ├─ Inability to provide requested information             │
│  ├─ Asking to "please understand" / "samjhiye"             │
│  └─ Requesting to speak to someone else                    │
│                                                           │
│  THRESHOLD: Any 2 signals → DISTRESS FLAG                  │
└───────────────────────────────────────────────────────────┘
```

### Escalation Protocol

When distress is detected, the policy engine (not the prompt) controls the response:

1. **Immediate:** Soft acknowledgment: "I understand this is a very difficult time."
2. **Block:** Collections script is disabled. Agent CANNOT mention payment amounts, due dates, or penalties.
3. **Offer:** "Let me connect you with a specialist who can discuss options to help."
4. **Transfer:** Human agent receives full transcript + distress signals + borrower context.
5. **Follow-up:** Human agent has authority to offer: moratorium, restructuring, interest waiver.

**This is Policy, not Prompt.** The agent's tool set changes when distress flag is raised. Collections tools are removed. Only empathy and transfer tools remain.

---

# 8. TELEPHONY FOR INDIA

## Exotel: India-Native Telephony Infrastructure

### Why Exotel (Not Twilio)

| Dimension | Twilio | Exotel |
|-----------|--------|--------|
| India PSTN | Via international routing (high latency) | Native Indian PSTN (low latency) |
| Toll-free | Limited India support | Full toll-free support (1800 numbers) |
| TRAI compliance | Requires manual compliance | Built-in TRAI compliance |
| Feature phone support | WebRTC focus (requires data) | PSTN native (works on any phone) |
| Rural connectivity | Poor (depends on data network) | Good (PSTN backbone) |
| Pricing | USD-based (expensive for high volume) | INR-based (optimized for India) |
| Call recording | Available | Available + consent management |
| DND registry | Manual integration | Built-in DND check |

### Telephony Architecture

```
┌──────────────────────────────────────────────────────┐
│  TELEPHONY FLOW                                       │
│                                                      │
│  INBOUND (Borrower-initiated):                       │
│  Borrower ──PSTN──> Exotel ──WebSocket──> Platform   │
│  - Toll-free number (1800-XXX-XXXX)                  │
│  - IVR bypass: direct to AI agent (no "press 1")    │
│  - Language detection from caller ID region           │
│  - Caller ID lookup for returning borrower context   │
│                                                      │
│  OUTBOUND (Platform-initiated):                      │
│  Platform ──API──> Exotel ──PSTN──> Borrower         │
│  - DND registry check (mandatory)                    │
│  - Calling hours enforcement (8 AM - 7 PM local)     │
│  - Consent verification at call start                │
│  - Use cases:                                        │
│    * Moratorium notification                         │
│    * Graduation offer                                │
│    * EMI reminder (3 days before due)                │
│    * Disbursement confirmation                       │
│    * Weather alert (moratorium triggered)            │
│                                                      │
│  TRAI COMPLIANCE:                                    │
│  - DND (Do Not Disturb) registry check               │
│  - Calling hour restrictions                         │
│  - Consent recording and storage                     │
│  - Call recording with borrower notification         │
│  - Caller ID display (no anonymous calls)            │
└──────────────────────────────────────────────────────┘
```

### IVR Bypass

Traditional IVR ("Press 1 for loan status, press 2 for EMI...") fails for semi-literate borrowers who cannot navigate numeric menus. The platform bypasses IVR entirely:

1. Borrower calls toll-free number
2. Brief greeting in detected language: "Namaste, main aapki madad kaise kar sakti hoon?" (Hello, how can I help you?)
3. Direct to AI voice agent -- no menu, no tree, no "press 1"
4. Agent determines intent from natural speech

---

# 9. MULTI-LANGUAGE EXPANSION STRATEGY

## Phased Rollout: Depth Before Breadth

### Phase 1: Gujarati + Hindi (Dairy Pilot)

**Target:** Gujarat dairy belt -- cooperative dairy farmers (Amul, Banas)

| Component | Requirement |
|-----------|-------------|
| ASR training data | 5,000+ labeled utterances in Gujarati lending domain |
| Domain vocabulary | Milk pour, cooperative, buffalo breed, flush/lean season, AMCS rate |
| TTS voice | Female Gujarati voice (culturally appropriate for dairy cooperative context) |
| Code-mixing model | Gujarati-Hindi-English tri-lingual |
| Pronunciation rules | Gujarati rupee amounts, Aadhaar in Gujarati, EMI/CIBIL/NACH |
| Validation | 100 real dairy farmer conversations, WER < 15% |

### Phase 2: Tamil, Telugu, Kannada (SFB Geography)

**Target:** South India -- Small Finance Bank customer base

| Component | Requirement |
|-----------|-------------|
| ASR training data | 5,000+ labeled utterances per language |
| Domain vocabulary | Kirana store terms, gig economy terms, micro-enterprise vocabulary |
| TTS voices | Regional voices with appropriate dialect |
| Code-mixing model | Tamil-English, Telugu-Hindi, Kannada-English |
| Pronunciation rules | South Indian rupee amount conventions, regional number words |
| Validation | 100 conversations per language, WER < 15% |

### Phase 3: Bengali, Marathi, Odia (MFI Expansion)

**Target:** East and West India -- MFI customer base (SHG women, agricultural workers)

| Component | Requirement |
|-----------|-------------|
| ASR training data | 5,000+ labeled utterances per language |
| Domain vocabulary | SHG terminology, agricultural implements, seasonal crops by region |
| TTS voices | Regional voices |
| Code-mixing model | Bengali-Hindi, Marathi-Hindi, Odia-Hindi |
| Validation | 100 conversations per language, WER < 15% |

### Data Collection Strategy

```
┌────────────────────────────────────────────────────────────┐
│  LABELED UTTERANCE COLLECTION PER LANGUAGE                  │
│                                                            │
│  Source 1: SYNTHETIC (2,000 utterances)                     │
│  - LLM-generated lending conversations                     │
│  - Covers: application, status check, EMI query,           │
│    collections, complaints, Aadhaar verification           │
│  - Human-reviewed for naturalness                          │
│                                                            │
│  Source 2: FIELD RECORDINGS (2,000 utterances)             │
│  - Recorded at cooperative offices, SFB branches           │
│  - Real borrower speech patterns                           │
│  - Consented, anonymized, transcribed                      │
│                                                            │
│  Source 3: AGENT INTERACTION LOGS (1,000+ utterances)      │
│  - From live pilot conversations                           │
│  - Highest value: real errors surface real gaps             │
│  - Continuous accumulation (grows over time)               │
│                                                            │
│  TOTAL PER LANGUAGE: 5,000+ labeled utterances             │
│                                                            │
│  ANNOTATION:                                               │
│  - Transcription (verbatim)                                │
│  - Intent label (loan_status, emi_query, complaint, etc.)  │
│  - Entity extraction (Aadhaar, amount, date, name)         │
│  - Language tags (for code-mixed segments)                  │
│  - Emotional tone (neutral, distressed, frustrated, happy) │
└────────────────────────────────────────────────────────────┘
```

---

# 10. KEY PRINCIPLES

## Voice AI for Indian Lending: Non-Negotiable Truths

### Principle 1: Voice Is Primary, Not Secondary

Voice is not a feature added to a text-based platform. For 300M underserved Indian borrowers, voice IS the platform. Every architectural decision -- latency budget, ASR accuracy, pronunciation rules, VAD threshold -- flows from this principle. If the voice experience fails, the platform fails, regardless of how sophisticated the credit model is.

### Principle 2: Empathy Is Structural, Not Prompt-Based

"Be empathetic" in a system prompt is worthless. Empathy in voice lending means:
- Distress detection that **removes collections tools** from the agent (Policy)
- Weather moratorium that **automatically pauses EMIs** for affected districts (State + Policy)
- Context-aware collections that **knows why** a borrower missed payment (Context)
- Human escalation that **transfers full transcript** so the borrower doesn't repeat their story (State)

Empathy is architecture. Tools available to the agent change based on detected emotional state. This is not prompt engineering.

### Principle 3: Latency Is Non-Negotiable

<1.5 seconds end-to-end. No exceptions. Strategies:
- Fast brain acknowledgment (<50ms) to fill perceived gaps
- Streaming TTS to start speaking before full response is ready
- GPU inference for ASR to cut recognition time
- Context pre-loading on call connect
- Edge deployment for rural low-connectivity areas

### Principle 4: Pronunciation Is Domain-Critical

A voice agent that says "Emmy" instead of "E-M-I" or "civil" instead of "CIBIL" loses borrower trust instantly. Lending has dozens of terms that standard TTS mispronounces. The pronunciation engine is not optional -- it is as critical as the credit model.

### Principle 5: Consent Is Audible and Recorded

In voice lending, consent cannot be a checkbox. It must be:
- **Spoken:** "This call will be recorded for quality and compliance. Do you consent?"
- **Recorded:** The consent utterance itself is stored with the call recording
- **Verified:** ASR must confirm affirmative response ("haan", "yes", "theek hai")
- **Timestamped:** Exact timestamp of consent for regulatory audit
- **Revocable:** Borrower can say "stop recording" at any time

```
┌──────────────────────────────────────────────────────────┐
│  THE FIVE PRINCIPLES: VOICE AI FOR INDIAN LENDING         │
│                                                          │
│  1. VOICE IS PRIMARY                                      │
│     Not a feature. The platform itself.                   │
│     300M borrowers access finance through voice.          │
│                                                          │
│  2. EMPATHY IS STRUCTURAL                                 │
│     Architecture, not prompts. Tool availability           │
│     changes based on detected emotional state.            │
│                                                          │
│  3. LATENCY IS NON-NEGOTIABLE                             │
│     < 1.5s end-to-end. Fast brain for immediate ack.     │
│     Streaming everything. GPU inference. Edge deploy.     │
│                                                          │
│  4. PRONUNCIATION IS DOMAIN-CRITICAL                      │
│     EMI ≠ Emmy. CIBIL ≠ Civil. NACH ≠ Match.            │
│     Every financial term has an exact pronunciation.      │
│                                                          │
│  5. CONSENT IS AUDIBLE                                    │
│     Spoken, recorded, verified, timestamped, revocable.  │
│     No silent checkboxes in voice-first lending.          │
└──────────────────────────────────────────────────────────┘
```

---

*This document distills voice AI knowledge specific to Indian lending: 12 languages, domain pronunciation, latency engineering, emotional intelligence, and telephony infrastructure -- all designed for 300M borrowers who access finance through voice, not screens.*
