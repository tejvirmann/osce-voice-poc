# OSCE Voice Simulation Platform

> An OSCE Simulation Operating System — not just a voice app.

A real-time, ultra-low latency platform where medical students interact with AI patients over phone or web, and professors can create OSCE scenarios in under 1 minute.

**Docs:** [planv2.md](planv2.md) — detailed implementation plan, OSCE Spec schema, infrastructure map, and ADRs.

---

## Vision

- Students call (or receive a call from) an AI patient and practice clinical skills
- Professors upload a PDF or write a scenario — the system generates a structured OSCE spec automatically
- Conversations feel natural: interruptible, emotionally realistic, sub-800ms latency
- Two modes: **Training** (free, fast) and **Exam** (premium, high-realism)

---

## System Layers

```
1. Authoring Layer   — Custom Professor UX (PDF/text → OSCE Spec → Visual Editor)
                       Professors never interact with Dograh directly.
2. Runtime Layer     — Dograh (agent orchestration, state, interrupts, tools)
                       Self-hosted Docker. Our services talk to it via REST API.
3. Model Layer       — Pluggable STT / LLM / TTS per mode (training vs exam)
                       Self-hosted (Chatterbox, Llama) + API (ElevenLabs, Claude, Deepgram)
```

---

## Agents

| Agent | Role |
|---|---|
| **Patient Agent** | Role-plays the patient, manages emotion + state transitions |
| **Narrator Agent** | Sends contextual cues via voice or SMS, can interrupt the call |
| **Evaluator Agent** | Scores student performance, generates structured feedback |

---

## Voice Pipeline (Real-Time)

```
Mic Input
→ VAD (interrupt detection)
→ Streaming STT (Deepgram / Whisper)
→ Dograh (orchestration)
→ LLM (patient reasoning)
→ TTS (see model layer below)
→ Streaming Audio Output (cancelable)
```

**Requirements:** streaming at every step, cancelable TTS, partial decoding, <800ms end-to-end.

---

## Model Layer

### STT
| Provider | Notes |
|---|---|
| Deepgram | API, low latency, real-time partial transcripts |
| Whisper | Self-hosted fallback |

### LLM
| Mode | Model |
|---|---|
| Training | Small open-source (Llama, Mistral) |
| Exam | High-quality API model (GPT-class) |

### TTS
| Mode | Provider | Emotion | Cost |
|---|---|---|---|
| **Training (free)** | Chatterbox (Resemble AI) | Yes — exaggeration param + voice clone | Free / self-hosted |
| **Training (free)** | Kokoro | No — clean neutral voice only | Free / self-hosted |
| **Exam (premium)** | ElevenLabs Turbo v2.5 | Full — stability, style, SSML cues | ~$0.025/session |

> **Note on Chatterbox:** Open source (MIT), supports voice cloning from a short reference clip and an `exaggeration` parameter for emotional intensity. Strong candidate for the free tier.
>
> **Note on Voxtral:** Mistral's Voxtral is primarily a speech-understanding/STT model — verify intended use before integrating as TTS.

---

## Two Modes

### Training Mode
- Smaller LLM, Chatterbox or Kokoro TTS
- Patient voice is neutral/fast
- Lower cost — students can practice freely
- Unlimited sessions

### Exam Mode (Premium)
- High-quality LLM, ElevenLabs TTS
- Emotionally realistic patient (anxious, in pain, evasive)
- Voice cloning for specific patient personas
- Topic-triggered interruptions from patient or narrator

---

## Authoring Pipeline (Professor UX)

**Goal: create a full OSCE in under 1 minute**

```
Input (PDF or Text)
→ Parsing + Section Detection
→ LLM Extraction
→ OSCE Spec (structured JSON)
→ Visual Flow Editor (nodes = states, edges = transitions)
→ Test Mode (simulate full call, watch state transitions live)
→ Publish
```

### OSCE Spec Output Includes
- Patient persona + backstory
- Emotional states + transitions
- Narrator events + triggers
- Evaluation criteria + scoring rubric

---

## Visual Editor

Custom UI (not Dograh's default UI):
- Nodes = patient states
- Edges = transitions (triggered by topic, keyword, or timing)
- Drag, edit, preview in real-time
- Topic-triggered interruption config (e.g. "if student mentions surgery → patient becomes distressed")

---

## Telephony

- Twilio for inbound/outbound calls and narrator SMS
- Supports concurrent users

---

## Infrastructure Notes

- **Do not fork Dograh** — run published Docker images, extend via its REST API only (ADR-002)
- Two service layers: Dograh stack (`:8000`, `:3010`, Postgres, Redis, MinIO) + our services (`osce-backend :8001`, `osce-frontend :3000`, `chatterbox-shim :8889`, Ollama `:11434`)
- Colocate TTS and LLM containers on the same host as Dograh to minimize network hops
- GPU recommended for self-hosted TTS (Chatterbox) and LLM (Llama); API fallbacks work for Phase 1
- Stream at every step — no full-sentence waits; target <800ms mic → first audio byte

---

## Clients

| Institution | Description |
|---|---|
| [SimLab — UW Madison](https://simlab.wisc.edu/mission-approach/) | Simulation-Based Learning Lab at the Wisconsin Center for Education Research. Uses AI and modern assessment design to provide realistic, structured, evaluative simulation experiences for practitioners. |

---

## MVP Phases

> Full detail, OSCE Spec schema, and ADRs in [planv2.md](planv2.md).

### Phase 1 — Core Loop (2 weeks)
Goal: student calls a Twilio number, speaks to an AI patient, barge-in works, latency <800ms.
- [ ] Hardcoded chest pain scenario as a Dograh workflow (3 patient states, Deepgram STT, Llama + Claude, Chatterbox TTS)
- [ ] Twilio inbound call → Dograh → patient agent
- [ ] Barge-in and interrupt verified in a real call
- [ ] TTS evaluation: Chatterbox vs Kokoro vs ElevenLabs (latency + naturalness + cost)
- [ ] End-to-end latency benchmark; tune until <800ms
- [ ] Verify Dograh REST API supports programmatic workflow CRUD (required for Phase 2)

### Phase 2 — Authoring Pipeline (2–3 weeks)
Goal: professor uploads a PDF, reviews the extracted scenario, publishes, and a student calls it.
- [ ] OSCE Spec JSON schema (Pydantic, versioned)
- [ ] PDF → OSCE Spec extractor (PyMuPDF + LLM extraction prompt + validation)
- [ ] OSCE Spec → Dograh workflow generator (creates training + exam workflows via API)
- [ ] TTS shim for Chatterbox/Kokoro (ElevenLabs-compatible endpoint, ~50 lines)
- [ ] Professor web app MVP: `/create`, `/my-osces`, `/test/:id`
- [ ] Test mode: browser call with live state transition panel

### Phase 3 — Multi-Agent + Evaluation (3–4 weeks)
Goal: post-call scores visible in dashboard; narrator fires on keywords; visual flow editor live.
- [ ] Evaluator service: post-call webhook → Claude grades transcript against rubric → structured score
- [ ] Student + professor score dashboard
- [ ] Narrator agent: keyword detection → inject audio cue or send SMS via Twilio
- [ ] Custom visual flow editor (React Flow): nodes = states, edges = transitions, OSCE-optimized
