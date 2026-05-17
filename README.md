# OSCE Voice Simulation Platform

> An OSCE Simulation Operating System — not just a voice app.

A real-time, ultra-low latency platform where medical students interact with AI patients over phone or web, and professors can create OSCE scenarios in under 1 minute.

---

## Vision

- Students call (or receive a call from) an AI patient and practice clinical skills
- Professors upload a PDF or write a scenario — the system generates a structured OSCE spec automatically
- Conversations feel natural: interruptible, emotionally realistic, sub-800ms latency
- Two modes: **Training** (free, fast) and **Exam** (premium, high-realism)

---

## System Layers

```
1. Authoring Layer   — Professor UX (PDF/text → OSCE Spec → Visual Editor)
2. Runtime Layer     — Dograh (agent orchestration, state, interrupts, tools)
3. Model Layer       — Pluggable STT / LLM / TTS per mode
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

- **Do not fork Dograh** — use it as the backend runtime, build the authoring + OSCE spec layer on top
- Colocate TTS/LLM services for latency
- GPU recommended for self-hosted TTS (Chatterbox) and LLM
- Stream at every step — no full-sentence waits

---

## MVP Phases

### Phase 1 — Core Loop
- [ ] Text → OSCE Spec (LLM extraction)
- [ ] Dograh integration + patient agent
- [ ] Basic voice loop (STT → LLM → TTS)
- [ ] Voice model evaluation: Chatterbox vs Kokoro vs ElevenLabs

### Phase 2 — Authoring
- [ ] PDF ingestion + parsing
- [ ] OSCE Spec validation
- [ ] Test mode for professors

### Phase 3 — Polish
- [ ] Visual flow editor
- [ ] Evaluator agent + scoring
- [ ] Narrator agent + SMS
- [ ] Topic-triggered interruptions
