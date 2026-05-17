# OSCE Voice Simulation Platform

## Product + Architecture Spec (Revised)

---

# 1. Core Vision

A real-time, ultra-low latency OSCE simulation platform where:

* Students interact with AI patients over phone or web
* Professors can create OSCEs in under 1 minute (PDF or text)
* Scenarios are powered by an agent orchestration layer
* Conversations feel natural, interruptible, and emotionally realistic

---

# 2. Key Principles

* ⚡ Ultra-fast (<800ms response latency)
* 🧠 Agent-driven (not just TTS)
* 🎭 Emotionally realistic
* 🔁 Fully interruptible (barge-in)
* 🧩 Modular model support (swap providers easily)

---

# 3. System Overview

This system has 3 major layers:

1. Authoring Layer (Professor UX)
2. Runtime Layer (Voice + Agents)
3. Model Layer (LLMs, STT, TTS)

Dograh is used as the **agent orchestration runtime**.

---

# 4. Voice Pipeline (Real-Time System)

Core real-time loop:

Mic Input
→ VAD (interrupt detection)
→ Streaming STT (Whisper / Deepgram)
→ Dograh (agent orchestration)
→ LLM (patient reasoning)
→ TTS (Voxtral / Chatterbox / ElevenLabs)
→ Streaming Audio Output (interruptible)

---

## Key Requirements

* Streaming at every step
* Cancelable TTS (for interruptions)
* Partial decoding (no full sentence waits)
* Sub-second response target

---

# 5. Dograh Architecture

Dograh is deployed as a **hosted orchestration layer**.

It manages:

* Agent coordination
* State transitions
* Tool execution (speak, text, interrupt)
* Scenario logic

---

## Agents

### Patient Agent

* Role-playing logic
* Emotion + state transitions

### Narrator Agent

* Sends contextual cues (SMS or voice)
* Can interrupt conversation

### Evaluator Agent

* Scores student performance
* Generates feedback

---

# 6. Model Layer (Pluggable)

The system supports multiple models:

## STT

* Whisper (self-hosted)
* Deepgram (API)

## LLM

* Open-source (Llama, Mistral)
* API models (GPT-class)

## TTS

* Voxtral (fast, default)
* Chatterbox (emotional)
* ElevenLabs (premium fallback)

---

## Model Routing Strategy

### Training Mode (cheap)

* Small LLM
* Fast TTS (Voxtral / Kokoro)

### Exam Mode (premium)

* High-quality LLM
* Chatterbox or ElevenLabs

---

# 7. OSCE Spec (Core Representation)

All scenarios are converted into a structured OSCE Spec.

This drives:

* agent behavior
* voice simulation
* evaluation

---

# 8. Authoring System (Professor UX)

## Goal: Create OSCE in < 1 minute

### Input Methods

1. Upload PDF
2. Write scenario (few paragraphs)

---

## Pipeline

Input (PDF or Text)
→ Parsing
→ LLM Extraction
→ OSCE Spec
→ Visual Flow
→ Editable UI
→ Test Mode
→ Publish

---

# 9. PDF → Voice Pipeline

## Steps

1. PDF Parsing
2. Section Detection
3. Semantic Extraction
4. OSCE Spec Generation
5. Validation

---

## Output

* Persona
* Emotional states
* Transitions
* Narrator events
* Evaluation criteria

---

# 10. Visual Editor

Custom UI (not Dograh UI)

* Nodes = states
* Edges = transitions
* Triggers = conditions

Features:

* drag + edit
* click to modify behavior
* real-time preview

---

# 11. Test Mode (Critical)

Professors can:

* simulate full call
* see state transitions live
* observe narrator triggers
* debug scenario behavior

---

# 12. Web Platform

## Users

### Students

* choose OSCE
* start call or receive call
* interact with AI patient

### Professors

* create/edit OSCEs
* test scenarios
* publish

---

# 13. Telephony + Messaging

* Twilio (calls + SMS)
* supports concurrent users

Features:

* inbound/outbound calls
* narrator SMS
* multi-channel interaction

---

# 14. Latency Optimization

To ensure speed:

* streaming everywhere
* colocate services
* GPU for TTS + LLM
* minimize network hops

Target:

* <800ms end-to-end response

---

# 15. Should You Fork Dograh?

## Recommendation: DO NOT fork

Instead:

* Use Dograh as backend runtime
* Build your own:

  * authoring UI
  * OSCE spec layer
  * orchestration wrapper

### Why:

Forking = maintenance burden + slows you down

You only need to extend Dograh, not replace it

---

# 16. MVP Plan

## Phase 1

* Text → OSCE Spec
* Dograh integration
* Basic voice loop

## Phase 2

* PDF ingestion
* Test mode

## Phase 3

* Visual editor
* evaluator agent

---

# 17. Key Insight

This is not a voice app.

This is:

"An OSCE Simulation Operating System"

---

# End
