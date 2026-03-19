# Med-SEAL V1 Clinical Agent

The **Med-SEAL V1 Clinical Agent** is the first-generation agentic AI system built for the Med-SEAL platform. It is designed as a **clinical-grade conversational agent** capable of reasoning over patient health records, supporting both patient-facing interactions and clinician-facing decision support.

> **Demo note:** The V1 Clinical Agent is **not deployed in the live demo environment** due to infrastructure constraints. In the interactive demonstration, its role is fulfilled by **ChatGPT** to showcase the intended conversational and clinical reasoning capabilities. A production deployment would require a dedicated GPU-backed inference server running the fine-tuned `med-r1` model.

---

## Overview

| Attribute | Value |
|---|---|
| **Agent ID** | V1 |
| **Version** | Med-SEAL V1 |
| **Category** | Clinical Agent |
| **Underlying Model** | Med-SEAL fine-tuned LLM (`med-r1`, based on Qwen3-VL-8B) |
| **Demo Substitute** | ChatGPT (OpenAI) |
| **Deployment Status** | ⚠️ Not deployed — infrastructure unavailable |
| **Target Surface** | Patient app · OpenEMR clinician portal |

---

## Capabilities

The V1 Clinical Agent is designed to handle the full range of clinical interaction tasks in the Med-SEAL suite:

### 1. Patient-Facing Clinical Chat
- Answers patient questions about their conditions, medications, and lab results
- Explains clinical findings in plain language appropriate for the patient's health literacy level
- Supports multilingual interactions (English, Chinese, Malay, Tamil) via MERaLiON integration

### 2. Medication Reasoning
- Checks for drug–drug and drug–food interactions
- Provides adherence coaching and answers patient medication queries
- Reads FHIR `MedicationRequest` and `MedicationAdministration` resources

### 3. Clinical Decision Support
- Generates evidence-based responses grounded in the patient's FHIR record
- Flags clinical anomalies (abnormal vitals, missed medications, worsening trends)
- Supports the pre-visit clinical brief generation pipeline (A5 / Insight Synthesis Agent)

### 4. Symptom & Condition Reasoning
- Processes free-text symptom descriptions and maps them to structured FHIR `Condition` entries
- Provides differential reasoning pathways for clinician review

---

## Architecture

The V1 Clinical Agent wraps the core `med-r1` LLM with:

- **FHIR Context Loader** — reads relevant resources (Patient, Condition, MedicationRequest, Observation, AllergyIntolerance) to construct a grounded prompt context
- **SEA-LION Guard** — input validation, output safety checking, and PII redaction
- **smolagents Orchestrator** — routes clinical-intent messages from the Companion Agent (A1) to V1

```text
Patient message
      │
      ▼
SEA-LION Guard (input)
      │
      ▼
Orchestrator ──► V1 Clinical Agent
                      │
              FHIR Context Loader
                      │
              med-r1 LLM / ChatGPT (demo)
                      │
              SEA-LION Guard (output)
                      │
                      ▼
             Response to patient / clinician
```

---

## Why Not Deployed?

The V1 Clinical Agent requires a dedicated GPU inference server to serve the `med-r1` model at acceptable latency. For the current demonstration phase, this infrastructure is not available. As a placeholder:

- The demo environment routes clinical queries to **ChatGPT** via the OpenAI API
- The prompt structure, FHIR context injection, and safety guard layers remain identical to the production design
- This allows the user experience and workflow to be validated without the GPU infrastructure cost

Once infrastructure is provisioned, the only change required is to update the `LLM_API_URL` and `LLM_MODEL` environment variables to point to the self-hosted `med-r1` endpoint:

```bash
# Production (Med-SEAL V1 Clinical Agent)
LLM_API_URL=https://medseal-llm.ngrok-free.dev/v1/chat/completions
LLM_MODEL=med-r1
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048

# Demo substitute (ChatGPT)
LLM_API_URL=https://api.openai.com/v1/chat/completions
LLM_MODEL=chatgpt-4o-latest   # or equivalent GPT model available at demo time
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048
```

---

## Relation to Other Agents

The V1 Clinical Agent consolidates the responsibilities of **A2 (Clinical Reasoning Agent)** and parts of **A5 (Insight Synthesis Agent)** from the multi-agent roster. In the planned V2 architecture, these are split into specialised agents for scalability. In V1, a single fine-tuned model handles both roles.

| V1 Agent | V2 Equivalent Agent(s) |
|---|---|
| V1 Clinical Agent | A2 Clinical Reasoning + A5 Insight Synthesis |

---

## Related Pages

- {doc}`overview` — Full multi-agent roster and architecture
- {doc}`features` — Feature-to-agent mapping
- {doc}`../deployment` — Infrastructure and deployment configuration
- {doc}`../developer-guide/adding-ai-features` — How to extend the agent with new capabilities
