# Med-SEAL V1 Technical Report

**Version:** 1.0  
**Date:** March 2026  
**Status:** Model trained — not deployed in demo (no GPU infrastructure)

---

## 1. What is Med-SEAL V1?

Med-SEAL V1 refers to **`med-r1`**, a **fine-tuned clinical large language model** based on [Qwen3-VL-8B-Thinking](https://huggingface.co/Qwen). It is the foundational AI model purpose-built for the Med-SEAL healthcare platform.

Med-SEAL V1 is **purely the base model** — not the multi-agent system, not the orchestrator, and not the patient/clinician application stack. The broader Med-SEAL agent architecture (A1–A6, SEA-LION Guard, smolagents) *consumes* this model as its clinical reasoning backbone.

> **Demo Note:** Med-SEAL V1 (`med-r1`) is **not deployed** in the live demonstration due to the absence of GPU inference infrastructure. In the demo, its role is fulfilled by **ChatGPT** (OpenAI API) using the same prompt architecture and FHIR context pipeline. Switching to `med-r1` in production requires only environment variable changes — no code changes.

---

## 2. Model Card

| Field | Value |
|---|---|
| **Model name** | `med-r1` |
| **Base model** | Qwen3-VL-8B-Thinking |
| **Parameters** | ~8 billion |
| **Modality** | Vision-Language (text + image) |
| **Fine-tuning method** | Supervised fine-tuning (SFT) on clinical reasoning tasks |
| **Domain** | Chronic disease management (diabetes, hypertension, hyperlipidemia) |
| **Region focus** | Singapore and Southeast Asia |
| **Languages** | English, Chinese (Simplified), Malay, Tamil |
| **Thinking mode** | Chain-of-thought reasoning enabled |
| **Temperature** | 0.3 (clinical precision) |
| **Max output tokens** | 1024–2048 |
| **Serving framework** | vLLM |
| **GPU requirement** | 2× NVIDIA H200 |
| **API compatibility** | OpenAI-compatible (`/v1/chat/completions`) |

---

## 3. Why Qwen3-VL-8B?

The base model was selected for:

- **Vision-Language capability** — can process medical images (lab reports, medication labels, wound photos) alongside text, enabling future multimodal clinical features
- **8B parameter sweet spot** — large enough for clinical reasoning quality, small enough to serve on 2× H200 GPUs with acceptable latency
- **Thinking mode** — built-in chain-of-thought reasoning produces transparent, auditable clinical reasoning traces
- **Multilingual foundation** — strong performance across English, Chinese, Malay, and Tamil — the four official languages of Singapore
- **Open weights** — allows fine-tuning, on-premise deployment, and compliance with healthcare data sovereignty requirements

---

## 4. Fine-Tuning: From Qwen3-VL to med-r1

### 4.1 Training Objective

Transform the general-purpose Qwen3-VL-8B into a **clinical reasoning specialist** that can:

1. Synthesise patient health records (FHIR-formatted) into evidence-based clinical assessments
2. Check drug–drug and drug–food interactions
3. Interpret lab results and vital sign trends over time
4. Generate structured clinical summaries for clinician pre-visit briefs
5. Reason about multi-condition patients (diabetes + hypertension + hyperlipidemia)
6. Produce outputs in a structured JSON format with explicit evidence citations

### 4.2 Training Data Domains

| Domain | Content |
|---|---|
| Clinical Q&A | Drug interactions, condition explanations, lab interpretation |
| FHIR-grounded reasoning | Responses that cite specific FHIR resources (Observation, Condition, MedicationRequest) |
| Southeast Asian clinical context | Singapore healthcare protocols, local medication names, cultural health practices |
| Structured output | JSON-formatted clinical responses with assessment, evidence, confidence, warnings |
| Safety alignment | Refusals for diagnosis, medication changes, emergency triage |

### 4.3 Output Format

`med-r1` is trained to produce structured JSON clinical responses:

```json
{
  "assessment": "Plain-text clinical summary...",
  "evidence": [
    {
      "resource_type": "Observation",
      "resource_id": "obs-001",
      "key_value": "HbA1c 7.2%",
      "date": "2026-02-15"
    }
  ],
  "confidence": "high | medium | low",
  "warnings": ["Any safety-relevant concerns"],
  "suggested_actions": ["Optional clinician follow-ups"]
}
```

- All evidence entries cite real FHIR resource IDs and dates
- Never fabricates data not present in the provided context
- States confidence level based on data completeness: `high` (clear evidence), `medium` (partial data), `low` (insufficient data)

---

## 5. Model Serving Configuration

### 5.1 Production Endpoint (Target)

```bash
LLM_API_URL=https://medseal-llm.ngrok-free.dev/v1/chat/completions
LLM_MODEL=med-r1
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048
```

Served via **vLLM** on 2× NVIDIA H200 GPUs, exposed as an OpenAI-compatible API.

### 5.2 Demo Substitute (Current)

```bash
LLM_API_URL=https://api.openai.com/v1/chat/completions
LLM_MODEL=gpt-4o
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048
```

Since `med-r1` requires dedicated GPU infrastructure that is not available for the demo, **ChatGPT** is used as a drop-in substitute. The same prompt templates, FHIR context injection, and safety guards are applied regardless of which model is behind the endpoint.

### 5.3 Infrastructure Requirements

| Resource | Requirement |
|---|---|
| GPU | 2× NVIDIA H200 (80 GB HBM3 each) |
| Serving framework | vLLM  |
| Model weights storage | ~16 GB (FP16) |
| Endpoint | Private, OpenAI-compatible `/v1/chat/completions` |
| Latency target | < 10 seconds for clinical reasoning responses |

---

## 6. How the Agent System Uses Med-SEAL V1

Med-SEAL V1 is consumed by the broader agent architecture as follows:

```
Patient / Clinician
        │
  SEA-LION Guard (input)
        │
  smolagents Orchestrator
        │
   ┌────┴─────┐
   │ A2 Agent │  ←── calls med-r1 for clinical reasoning
   │ A5 Agent │  ←── calls med-r1 for pre-visit briefs
   └────┬─────┘
        │
  ┌─────▼──────┐
  │  med-r1    │  ← THIS IS MED-SEAL V1
  │  (LLM)     │
  └─────┬──────┘
        │
  SEA-LION Guard (output)
        │
  FHIR R4 response
```

- **A2 (Clinical Reasoning Agent)** — routes clinical questions (drug interactions, lab interpretation, condition reasoning) to `med-r1`
- **A5 (Insight Synthesis Agent)** — uses `med-r1` to generate pre-visit clinical briefs from aggregated patient data
- **A1 (Companion Agent)** — indirectly uses `med-r1` via delegation to A2 for complex medical questions, then rephrases the response for patients

Other agents (A3 Nudge, A4 Lifestyle) use different models (MERaLiON, SEA-LION) and do **not** depend on Med-SEAL V1.

---

## 7. Clinical Capabilities of med-r1

### 7.1 Drug Interaction Checking
- Input: List of active medications (RxNorm-coded from FHIR `MedicationRequest`)
- Output: Interaction severity, affected drug pairs, clinical recommendation
- Checks both drug–drug and drug–food interactions

### 7.2 Lab Result Interpretation
- Input: FHIR `Observation` resources (LOINC-coded) with temporal data
- Output: Trend analysis across ≥ 3 data points over 90+ days
- Covers: HbA1c, blood pressure, fasting glucose, lipid panel, eGFR

### 7.3 Multi-Condition Clinical Reasoning
- Reasons across co-morbid conditions: Type 2 diabetes + hypertension + hyperlipidemia
- Identifies interactions between conditions and their respective treatment plans
- Produces holistic clinical assessments rather than single-condition responses

### 7.4 Pre-Visit Brief Generation
- Aggregates 30 days of patient-side data (adherence, biometrics, PRO scores, engagement)
- Outputs structured `FHIR Composition` sections
- Designed for delivery to clinicians via CDS Hooks ≥ 30 min before appointment

### 7.5 Safety Constraints (Built into the Model)
- **Never fabricates** clinical data not present in the provided FHIR context
- **Never recommends** starting, stopping, or changing medications
- **Never provides** a diagnosis — only explains existing conditions on record
- **Never triages** emergencies — defers to clinical escalation pathways
- All outputs pass through the external SEA-LION Guard before reaching any user surface

---

## 8. Why Not Deployed?

Med-SEAL V1 (`med-r1`) requires **2× NVIDIA H200 GPUs** running vLLM to serve the model at production-grade latency (< 10s per request). This infrastructure is not available in the current demo environment.

**What the demo preserves without the model:**

| Aspect | Preserved? | Detail |
|---|---|---|
| Prompt architecture | ✅ Yes | Same system prompts, few-shot examples, instruction templates |
| FHIR context injection | ✅ Yes | Same Patient, Condition, Observation, MedicationRequest loading |
| Safety guards | ✅ Yes | SEA-LION Guard input/output gates applied identically |
| Structured JSON output | ✅ Yes | ChatGPT follows the same output schema |
| Clinical fine-tuning quality | ❌ No | ChatGPT lacks domain-specific fine-tuning for SEA clinical context |
| Multimodal (image) support | ❌ No | ChatGPT does not replicate `med-r1`'s vision-language capabilities |
| Latency guarantees | ❌ No | Depends on OpenAI API availability |

**To switch to production:** update two environment variables (`LLM_API_URL`, `LLM_MODEL`). No code changes required.

---

## 9. Relationship to V2

Med-SEAL V1 is the **first-generation model**. In the V2 roadmap, the model may be updated or replaced, but V1 remains the baseline:

| V1 | V2 (Planned) |
|---|---|
| Single model (`med-r1`) serves all clinical reasoning | Potentially specialised models per agent |
| Qwen3-VL-8B base | May upgrade to larger or more specialised base |
| 2× H200 serving | Scaled serving with load balancing |
| SFT-only training | May add RLHF or DPO alignment stages |

---

## 10. References

- {doc}`ai-agents/overview` — Multi-agent architecture that consumes this model
- {doc}`ai-agents/clinical-agent` — Clinical agent detail page
- {doc}`architecture` — System architecture and deployment diagrams
- {doc}`developer-guide/fhir-data-model` — FHIR resource profiles used in model context
- {doc}`deployment` — Infrastructure and environment configuration
