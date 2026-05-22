# OSCE Voice Simulation — Plan v2

> Dograh confirmed as runtime. This plan maps OSCE concepts to Dograh primitives and lays out a concrete build sequence.

---

## Why Dograh Makes Sense

Dograh handles the hard infrastructure problems so this project doesn't have to:

| Problem | Dograh Handles It |
|---|---|
| Streaming STT → LLM → TTS pipeline | Yes — pluggable providers per workflow |
| Barge-in / interrupt detection | Yes — built-in VAD + cancelable TTS |
| Twilio inbound/outbound calls | Yes — native integration |
| Conversation state machine | Yes — workflow nodes + transitions |
| Knowledge base per patient | Yes — built-in KB tool |
| Template variables (persona injection) | Yes |
| Pre-call data fetch | Yes |
| Audio storage | Yes — MinIO |
| Tracing / observability | Yes — Langfuse integration |

Self-hosted Docker stack is already running. Do **not** fork Dograh — build the OSCE-specific layer on top via its API.

---

## Architecture Map

```
OSCE Concept              →  Dograh Primitive
─────────────────────────────────────────────
Patient emotional state   →  Workflow node (own system prompt)
State transition trigger  →  Node condition / LLM tool call
Patient persona + case    →  Template variables + knowledge base
Training vs Exam mode     →  Different LLM/TTS config on same workflow
Twilio phone call         →  Dograh inbound call → workflow
Narrator interruption     →  Webhook event → inject audio or SMS
Evaluator scoring         →  Post-call webhook → external LLM
```

---

## What Dograh Owns vs What We Build

### Dograh owns (no custom code needed):
- STT → LLM → TTS loop
- Streaming + partial decoding
- Barge-in / cancelable TTS
- Twilio call routing
- Audio streaming to browser/phone
- Node graph execution
- Knowledge base retrieval
- Call recording + storage

### We build:
- OSCE Spec JSON schema
- PDF → OSCE Spec extractor (LLM pipeline)
- OSCE Spec → Dograh workflow generator (via Dograh REST API)
- Professor web UX (create / test / publish)
- Evaluator service (post-call transcript → structured score)
- Custom visual flow editor (Phase 3)

---

## OSCE Spec Schema (Canonical Format)

All scenarios are stored as OSCE Specs. This is the source of truth — everything else derives from it.

```json
{
  "id": "osce_chest_pain_v1",
  "title": "Chest Pain — Undifferentiated",
  "patient": {
    "name": "Maria Chen",
    "age": 45,
    "sex": "female",
    "chief_complaint": "chest pain radiating to left arm",
    "backstory": "45F, no cardiac history, presents to ED anxious...",
    "voice_profile": "middle_aged_female"
  },
  "modes": {
    "training": {
      "llm": "llama-3.1-8b",
      "tts": "chatterbox",
      "tts_exaggeration": 0.4
    },
    "exam": {
      "llm": "claude-sonnet-4-6",
      "tts": "elevenlabs-turbo-v2.5",
      "voice_id": "maria_chen_clone"
    }
  },
  "states": [
    {
      "id": "initial",
      "label": "Anxious / Onset",
      "emotional_tone": "anxious",
      "system_prompt_suffix": "You are very scared. Your chest has been hurting for 20 minutes. Answer only what is asked. Do not volunteer information.",
      "is_start": true
    },
    {
      "id": "reassured",
      "label": "Calming Down",
      "emotional_tone": "neutral",
      "system_prompt_suffix": "The student acknowledged your pain. You feel slightly more at ease but still worried."
    },
    {
      "id": "distressed",
      "label": "Worsening Pain",
      "emotional_tone": "distressed",
      "system_prompt_suffix": "Your pain is now 9/10. You are becoming short of breath."
    }
  ],
  "transitions": [
    {
      "from": "initial",
      "to": "reassured",
      "trigger": "student expresses empathy or acknowledges pain"
    },
    {
      "from": "initial",
      "to": "distressed",
      "trigger": "student mentions surgery or invasive procedure"
    }
  ],
  "narrator_events": [
    {
      "id": "ecg_result",
      "trigger": "student orders ECG",
      "action": "speak",
      "text": "The nurse hands you the ECG results.",
      "delay_seconds": 5
    }
  ],
  "evaluation_criteria": [
    { "id": "intro", "label": "Student introduced themselves", "points": 5 },
    { "id": "pain_scale", "label": "Asked pain severity (1-10)", "points": 10 },
    { "id": "radiation", "label": "Explored radiation of pain", "points": 10 },
    { "id": "red_flags", "label": "Screened for diaphoresis, SOB, nausea", "points": 15 },
    { "id": "empathy", "label": "Demonstrated empathy at least once", "points": 10 }
  ]
}
```

---

## Phase 1 — Core Loop (Target: 2 weeks)

### 1.1 Hardcoded Patient Agent in Dograh
- Create 1 Dograh workflow manually for the chest pain scenario
- Configure: Deepgram STT, Llama 3.1 8B (or Claude), Chatterbox TTS
- Nodes = 3 patient states (initial / reassured / distressed)
- Template variables: inject patient name, backstory, chief complaint
- Knowledge base: patient case summary
- Goal: end-to-end voice call works on localhost

### 1.2 Twilio Integration
- Configure Twilio number → Dograh inbound call handler
- Test: student dials number, reaches AI patient
- Verify barge-in and interruption work correctly

### 1.3 Voice Model Evaluation
Compare TTS options in real calls:

| Provider | Mode | Latency | Emotion | Cost |
|---|---|---|---|---|
| Chatterbox | Training | target <150ms | exaggeration param | free |
| Kokoro | Training | target <100ms | none | free |
| ElevenLabs Turbo v2.5 | Exam | ~200ms | full | ~$0.025/session |

Decision criteria: latency + naturalness + emotional range. Pick Chatterbox for training if latency is acceptable with GPU.

### 1.4 Latency Audit
- Measure end-to-end: mic → first TTS audio byte
- Target: <800ms
- If over budget: colocate TTS/LLM containers, enable streaming decode

---

## Phase 2 — Authoring Pipeline (Target: 2-3 weeks)

### 2.1 PDF → OSCE Spec Extractor
```
Input: PDF or plain text scenario
→ PyMuPDF / pdfplumber (parse)
→ Section detection (regex + heuristics)
→ LLM extraction prompt → OSCE Spec JSON
→ Pydantic validation
→ Output: validated OSCE Spec
```

LLM prompt should extract: patient persona, emotional states, transitions, narrator events, evaluation criteria.

### 2.2 OSCE Spec → Dograh Workflow Generator
- Python/FastAPI service
- Input: OSCE Spec JSON + mode (training/exam)
- Output: Dograh workflow ID (created via Dograh REST API)
- Idempotent: update existing workflow if `osce_id` already exists
- One workflow per (osce_id, mode) pair

This is the critical integration point — confirm Dograh API supports workflow CRUD programmatically.

### 2.3 Professor Web UX (MVP)
Minimal viable UI — no visual editor yet:

```
/create  → text box or PDF upload → generates OSCE Spec → preview → publish
/my-osces → list of created scenarios with status + phone number
/test/:id → simulate call from browser (web call via Dograh)
```

Stack: Next.js frontend + FastAPI backend (wraps Dograh API).

### 2.4 Test Mode
- Professor can initiate a web-based call to their own scenario
- Live transcript panel shows current patient state
- State transitions visible in real time (via Dograh webhook stream)
- One-click "reset" to restart scenario

---

## Phase 3 — Multi-Agent + Evaluation (Target: 3-4 weeks)

### 3.1 Narrator Agent
- Separate Dograh workflow (or webhook-driven service)
- Subscribes to call transcript events
- Detects narrator trigger keywords in real time
- Actions: inject audio cue into call OR send SMS to student via Twilio
- Example trigger: "student says surgery → patient state → distressed + narrator says 'patient becomes visibly upset'"

### 3.2 Evaluator Agent (Post-Call)
```
Call ends
→ Dograh webhook fires with transcript
→ Evaluator service receives transcript + OSCE Spec criteria
→ LLM (Claude) evaluates against rubric
→ Returns structured JSON: { criterion_id, achieved, evidence, score }
→ Aggregate score + narrative feedback
→ Store in DB
→ Display to student + professor dashboard
```

### 3.3 Custom Visual Flow Editor
- React + React Flow library
- Nodes = patient states (color-coded by emotion)
- Edges = transitions (label = trigger condition)
- Click node → edit system prompt suffix, emotional tone
- Click edge → edit trigger text
- "Test" button → spins up a call
- Saves as OSCE Spec JSON → regenerates Dograh workflow on publish
- Replaces Dograh's generic visual editor with OSCE-optimized UX

---

## Infrastructure

```
┌─────────────────────────────────────────────┐
│  Self-hosted Docker stack                    │
│                                             │
│  dograh-api    :8000  (orchestration)        │
│  dograh-ui     :3010  (Dograh admin)         │
│  postgres      :5432  (shared DB)            │
│  redis         :6379  (state/pubsub)         │
│  minio         :9000  (audio storage)        │
│  cloudflared          (tunnel for webhooks)  │
│                                             │
│  osce-backend  :8001  (our FastAPI service)  │
│  osce-frontend :3000  (professor UX)         │
│  chatterbox    :8888  (TTS, GPU)             │
│  llm-service   :11434 (Ollama/Llama, GPU)    │
└─────────────────────────────────────────────┘
         ↕ Twilio (calls + SMS)
         ↕ ElevenLabs (exam TTS)
         ↕ Deepgram (STT)
         ↕ Claude API (exam LLM + evaluator)
```

GPU required for: Chatterbox TTS, self-hosted LLM (Llama). For Phase 1, API fallbacks are fine.

---

## Open Questions (Resolve in Phase 1)

1. **Dograh workflow CRUD API** — does the REST API support programmatic workflow creation/update? This is required for Phase 2.2. Check `http://localhost:8000/api/v1/` or Dograh GitHub.

2. **Multi-agent on one call** — can a Narrator agent observe + interject into an active Patient agent call? Or does this require a custom webhook → Twilio `<Play>` injection?

3. **Latency with GPU Chatterbox** — is colocated Chatterbox (Docker, same host) fast enough for <800ms end-to-end? Run a latency benchmark in Phase 1.4.

4. **Dograh state transfer** — can nodes pass context between each other (e.g., "patient mentioned chest pain was 8/10" carries forward to next state)? Or does LLM context window handle this automatically?

---

## Success Criteria per Phase

| Phase | Done When |
|---|---|
| Phase 1 | Student calls Twilio number, speaks to AI patient, barge-in works, <800ms latency |
| Phase 2 | Professor uploads PDF, reviews spec, publishes, student calls that scenario |
| Phase 3 | Post-call score shows in dashboard; narrator fires on keyword; professor edits scenario in visual editor |

---

## Architectural Decision Records

> Captured decisions that shape the system. Status: Accepted unless noted.

---

### ADR-001: Use Dograh as Agent Orchestration Runtime

**Status:** Accepted

**Context:**
The platform needs real-time voice orchestration with <800ms latency, streaming STT/LLM/TTS, barge-in detection, Twilio integration, and a conversation state machine. Building this infrastructure from scratch is 3-4 months of work orthogonal to the core product value (OSCE simulation).

Alternatives considered:
- **Vapi / Retell** — managed, per-minute pricing, no self-hosting, limited control over TTS/LLM routing
- **Build from scratch** — full control, but months of infra work before any OSCE feature ships
- **Dograh (self-hosted, open-source)** — handles the full voice pipeline, pluggable models, Twilio native, Docker deployment in minutes

**Decision:**
Use Dograh as the runtime layer. It owns the voice pipeline, telephony, streaming, state execution, and tool calls. OSCE-specific logic lives in a thin application layer on top, interacting with Dograh via its REST API.

**Consequences:**
- Pro: STT/LLM/TTS pipeline, barge-in, Twilio, streaming, and state machine handled without custom code
- Pro: Self-hosted = no per-minute Vapi/Retell costs; GPU-colocated services for low latency
- Pro: Any OpenAI-compatible LLM or TTS endpoint is pluggable per workflow
- Con: Dependent on Dograh's API surface being stable enough for programmatic workflow CRUD — must verify in Phase 1
- Con: Some features (multi-agent per call, Narrator injection) may need workarounds if not natively supported

---

### ADR-002: Never Fork Dograh

**Status:** Accepted

**Context:**
Dograh is MIT-licensed. Forking gives full control but creates a permanent maintenance burden — every upstream improvement requires a manual merge, and divergence compounds over time.

**Decision:**
Run Dograh exactly as published via Docker images (`ghcr.io/dograh-hq/dograh`). All customization lives in our own services via Dograh's API and extension points (custom TTS endpoints, webhooks, pre-call data fetch). Never modify Dograh source.

**Consequences:**
- Pro: `docker compose pull` picks up all upstream improvements with zero effort
- Pro: Zero maintenance burden on the orchestration layer itself
- Con: Feature requests must go upstream or use workarounds — we cannot patch Dograh internals

---

### ADR-003: Professors Get a Custom UX — No Direct Dograh Access

**Status:** Accepted

**Context:**
Dograh's admin UI (`localhost:3010`) is a generic voice agent builder. Professors need a purpose-built experience: PDF upload, scenario preview, test mode, publish. Exposing the Dograh admin to professors would leak internal infrastructure detail and produce a product that requires training to use.

**Decision:**
Build a dedicated professor-facing web app (Next.js, `osce-frontend`). Professors exclusively interact with this app. Our `osce-backend` (FastAPI) translates professor actions (create/edit/publish/test) into Dograh API calls. Dograh's admin UI stays internal — ops access only.

**Consequences:**
- Pro: Full control over professor mental model and UX; can enforce OSCE-specific validation
- Pro: Dograh upgrade never breaks professor-facing UI (abstraction layer absorbs changes)
- Con: Additional service to build and maintain (frontend + backend abstraction)

---

### ADR-004: OSCE Spec JSON as the Canonical Source of Truth

**Status:** Accepted

**Context:**
OSCE scenarios need to drive four distinct systems: agent behavior (Dograh workflow), voice simulation (TTS config), evaluation (rubric), and visual editing (flow editor). Dograh's native workflow format is generic and carries none of the OSCE-specific semantics — emotional states, narrator triggers, or evaluation criteria.

**Decision:**
Define an OSCE Spec JSON schema (see schema section above) as the single source of truth. Every other system is a derived artifact:
- Dograh workflow = generated from OSCE Spec
- Visual editor = renders OSCE Spec
- Evaluator = reads criteria from OSCE Spec

OSCE Specs are stored in our DB. Dograh workflows are regenerated on every publish — they are not edited directly.

**Consequences:**
- Pro: Portable — scenarios are not locked to Dograh's internal format
- Pro: OSCE Spec can be versioned, diffed, exported, and imported independently
- Pro: If Dograh is ever replaced, only the workflow generator changes; scenarios survive
- Con: OSCE Spec and Dograh workflow must stay in sync — stale workflows are a risk if the generator fails silently

---

### ADR-005: TTS Shim for Self-Hosted Models (Chatterbox / Kokoro)

**Status:** Accepted

**Context:**
Training mode uses Chatterbox or Kokoro (self-hosted, free, emotional). Dograh natively integrates with ElevenLabs. Chatterbox and Kokoro do not expose an ElevenLabs-compatible API.

**Decision:**
Build a thin FastAPI shim (~50 lines) that accepts ElevenLabs-shaped POST requests and proxies them to Chatterbox or Kokoro, translating parameters as needed (e.g., `stability` → `exaggeration`). Dograh is configured with the shim's URL as if it were ElevenLabs. Switching training↔exam mode = changing a URL and API key in workflow config only. No code change required.

**Consequences:**
- Pro: Dograh's model config layer stays fully generic (just URLs and keys)
- Pro: Training/exam mode swap is configuration-only — no deployment required
- Pro: Shim can inject Chatterbox-specific params (exaggeration, voice clone reference) transparently
- Con: Additional service to run; must restart if Chatterbox/Kokoro API shape changes
- Con: If Dograh changes how it calls TTS, the shim's request shape must be updated

---

### ADR-006: Training vs Exam as Per-Workflow Config, Not Separate Code Paths

**Status:** Accepted

**Context:**
Two modes differ only in LLM and TTS provider — the patient state machine, transitions, and evaluation criteria are identical. A naive approach would duplicate the workflow or create if/else branches in the agent logic.

**Decision:**
Each OSCE Spec carries a `modes` block (training, exam). The workflow generator creates two Dograh workflows per scenario — one per mode — using the same state graph but different model configs. Students select their mode at call time (or are assigned by a professor). Professors author once; two workflows are generated automatically on publish.

**Consequences:**
- Pro: Single source of truth per scenario — state machine authored once, modes diverge only in model config
- Pro: Adding a new mode (e.g., "exam-strict") requires only a new `modes` block entry, no workflow logic changes
- Con: Two Dograh workflows per scenario doubles workflow count in Dograh; namespace them clearly (e.g., `osce_chest_pain_v1__training`)
