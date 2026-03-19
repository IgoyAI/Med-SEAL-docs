# Med-SEAL V1 Technical Report

**Version:** 1.0  
**Date:** March 2026  
**Status:** Research Prototype — Demo environment active; production deployment pending infrastructure provisioning

> **Demo Disclaimer:** The Med-SEAL V1 AI Agent (`med-r1`) is not deployed in the live demonstration due to the absence of a GPU inference server. All AI-powered clinical interactions in the demo are fulfilled by **ChatGPT** (OpenAI API) using an identical prompt architecture, FHIR context injection pipeline, and safety guard layer. No functional changes are required to switch to the production `med-r1` endpoint — only environment variable updates.

---

## 1. Executive Summary

Med-SEAL (Medical – Systems for Empowerment with AI & Learning) V1 is a **clinical-grade, FHIR-native AI agent platform** designed to empower chronic disease patients in Singapore and Southeast Asia. Built on top of OpenEMR and Medplum, the system delivers 18 smart healthcare features through a multi-agent AI architecture, covering patient engagement, clinical decision support, medication adherence, lifestyle coaching, and proactive care escalation.

The V1 system introduces the **Med-SEAL V1 Clinical Agent** — a fine-tuned large language model (`med-r1`, based on Qwen3-VL-8B) that consolidates clinical reasoning and patient-facing AI into a single architecture, later decomposed into specialised A2/A5 agents in the planned V2 roadmap.

**Key outcomes targeted by V1:**

| Outcome | Metric |
|---|---|
| Medication adherence (PDC) | ≥ 80% |
| Patient-reported outcomes collected | ≥ 1 per patient per 2 weeks |
| Clinician pre-visit brief delivery | ≥ 30 min before appointment |
| Clinical escalation latency | < 5 min from trigger |
| Safety guard block rate (inappropriate output) | < 0.1% of interactions |

---

## 2. System Architecture

### 2.1 Layer Model

Med-SEAL V1 is organised into four distinct layers:

```
┌─────────────────────────────────────────────────────┐
│           Patient-Facing Layer                      │
│   Expo / React Native (iOS & Android)               │
└────────────────────┬────────────────────────────────┘
                     │ REST / HTTPS
┌────────────────────▼────────────────────────────────┐
│           AI & Services Layer                       │
│   AI Service (Node.js/TypeScript, Port 4003)        │
│   6 agents · 18 features · LLM orchestration        │
│   SSO Service (PostgreSQL-backed, unified auth)     │
└────────────┬───────────────────────────┬────────────┘
             │ FHIR R4                   │ SQL (sync)
┌────────────▼──────────┐   ┌────────────▼────────────┐
│  Data & Interop Layer │   │  Clinical EMR Layer     │
│  Medplum Server:8103  │◄──►  OpenEMR 7.0.2          │
│  FHIR R4 API          │   │  Orders · Billing       │
│  PostgreSQL 16        │   │  Scheduling · Records   │
└───────────────────────┘   └─────────────────────────┘
```

### 2.2 Technology Stack

| Layer | Technology | Standards |
|---|---|---|
| Clinical EMR | OpenEMR 7.0.2 | ICD-10, SNOMED CT, HL7 v2 |
| FHIR API | Medplum Server + App | HL7 FHIR R4 |
| AI Backend | Node.js / TypeScript | REST, WebSocket |
| AI Models | `med-r1` (Qwen3-VL-8B, fine-tuned) / ChatGPT *(demo)* | — |
| Patient App | Expo / React Native | — |
| Auth | PostgreSQL + custom SSO | SMART on FHIR, OAuth 2.0 |
| Container Infra | Docker Compose | — |

### 2.3 Network Topology

All services run on the `medseal-net` Docker bridge network:

| Port | Service |
|---|---|
| `3000` | Medplum Admin App |
| `4003` | AI Service API |
| `5433` | Medplum PostgreSQL |
| `5434` | SSO PostgreSQL |
| `8080` | OpenEMR (HTTPS) |
| `8081` | OpenEMR (HTTP) |
| `8103` | Medplum FHIR API |

---

## 3. AI Agent Architecture

### 3.1 Multi-Agent Roster

| ID | Agent | Model | Surface | LLM |
|---|---|---|---|---|
| **V1** | **Med-SEAL V1 Clinical Agent** | `med-r1` / ChatGPT *(demo)* | Both | Yes |
| A1 | Companion Agent | MERaLiON + SEA-LION | Patient app | Yes |
| A2 | Clinical Reasoning Agent | Qwen3-VL-8B (Med-SEAL) | Both | Yes |
| A3 | Nudge Agent | MERaLiON + rule engine | Patient app | Yes |
| A4 | Lifestyle Agent | SEA-LION + nutrition KB | Patient app | Yes |
| A5 | Insight Synthesis Agent | Qwen3-VL-8B (Med-SEAL) | OpenEMR | Yes |
| A6 | Measurement Agent | Analytics engine (no LLM) | Both | No |
| G1 | SEA-LION Guard | SEA-LION Guard v1.0 | System-wide | Yes |
| O1 | smolagents Orchestrator | Rule-based router | System-wide | No |

### 3.2 Orchestration Flow

```
Patient App / OpenEMR
        │
SEA-LION Guard (input gate) ── PII redaction, toxicity filter, injection detection
        │
smolagents Orchestrator ── intent classification (rule-based, no LLM)
     /  │  │  │  │  \
   A1  A2  A3  A4  A5  A6
     \  │  │  │  │  /
SEA-LION Guard (output gate) ── FHIR validation, hallucination check, harm filter
        │
  Medplum FHIR R4 ──► OpenEMR sync
```

### 3.3 Model Serving (Production Target)

| Model | Framework | Endpoint | GPU |
|---|---|---|---|
| MERaLiON | vLLM | `http://meralion.medseal.internal:8000/v1` | 1× H200 |
| SEA-LION (chat) | vLLM | `http://sealion.medseal.internal:8000/v1` | 1× H200 |
| SEA-LION Guard | vLLM | `http://guard.medseal.internal:8000/v1` | 1× H200 |
| Qwen3-VL-8B (`med-r1`) | vLLM | `http://medseal-vlm.medseal.internal:8000/v1` | 2× H200 |

> **Note:** None of these endpoints are provisioned in the current demo. ChatGPT (`api.openai.com`) is used as the LLM backend for all clinical interactions.

---

## 4. Med-SEAL V1 Clinical Agent — Deep Dive

### 4.1 Identity & Configuration

| Field | Value |
|---|---|
| Agent ID | `clinical-reasoning-agent` |
| FHIR Device ID | `Device/medseal-clinical-agent` |
| Model (production) | Qwen3-VL-8B-Thinking fine-tuned as `med-r1` |
| Model (demo) | ChatGPT (OpenAI API) |
| Temperature | 0.3 (clinical precision) |
| Thinking mode | Chain-of-thought reasoning enabled |
| Max tokens | 1024 |
| Surfaces | Patient app (via A1 delegation) · OpenEMR clinician portal (via A5) |

### 4.2 Clinical Capabilities

The V1 Clinical Agent covers the full spectrum of clinical reasoning tasks:

#### Drug Interaction Checking
- Reads `MedicationRequest` (active prescriptions) and `AllergyIntolerance` resources
- Queries the RxNorm-coded terminology service for drug–drug interactions
- Flags contraindications and severity (mild / moderate / severe) for clinician review
- Checks drug–food interactions (e.g., grapefruit + statin, warfarin + vitamin K)

#### Lab Result Interpretation
- Retrieves `Observation` resources filtered by LOINC code and time window
- Interprets HbA1c trends (`4548-4`), blood pressure (`55284-4`), fasting glucose (`2345-7`), lipid panel
- Compares across ≥ 3 data points over 90+ days to assess trajectory
- Returns confidence level: `high` (clear evidence) / `medium` (partial) / `low` (insufficient)

#### Condition & Risk Reasoning
- Reads `Condition` resources (SNOMED CT coded, `clinicalStatus=active`)
- Synthesises multi-condition context (diabetes + hypertension + hyperlipidemia)
- Produces evidence-based risk commentary with explicit EHR citations
- Flags anomalies: worsening trends, missed follow-ups, unaddressed conditions

#### Pre-Visit Clinical Brief Generation
- Aggregates 30 days of patient-side data: medication adherence (PDC), biometrics, PRO scores, engagement, escalation flags
- Produces structured `FHIR Composition` delivered to clinicians via CDS Hooks
- Supports clinician portal (OpenEMR) integration

### 4.3 FHIR Scopes

```
patient/Patient.read
patient/Patient.$everything
patient/Condition.read
patient/Observation.read
patient/MedicationRequest.read
patient/AllergyIntolerance.read
patient/Encounter.read
patient/CarePlan.read
patient/Procedure.read
patient/Immunization.read
patient/DocumentReference.read
patient/Composition.write
patient/ServiceRequest.write
```

### 4.4 Tool Registry (smolagents)

The agent operates via a structured tool registry. Key tools:

```python
patient_everything(patient_id)          # Full FHIR Bundle for patient compartment
search_conditions(patient_id, snomed)   # Active conditions by SNOMED CT
search_observations(patient_id, loinc, period_days)  # Labs/vitals by LOINC
search_medications(patient_id)          # Active MedicationRequests
search_allergies(patient_id)            # AllergyIntolerance records
check_drug_interaction(medication_codes)  # RxNorm-based DDI check
write_composition(patient_id, title, sections)  # FHIR Composition output
search_encounters(patient_id, status, period_days)  # Encounter history
```

### 4.5 Structured Output Format

The agent returns JSON-structured clinical responses (never raw prose sent directly to patients):

```json
{
  "assessment": "Clinical summary in plain text...",
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
  "suggested_actions": ["Optional follow-ups for clinician review"]
}
```

Responses are never fabricated — all `evidence` entries cite real FHIR resource IDs and dates.

### 4.6 Error Handling

| Error Condition | Handling |
|---|---|
| `med-r1` / LLM timeout (> 10s) | Return partial answer with `confidence=low`; log timeout |
| FHIR search returns empty | Explicitly state: `"No {resource} records found"` — never infer |
| Drug interaction service unavailable | Proceed without DDI data; add `warning: "Drug interaction check unavailable"` |
| SEA-LION Guard BLOCK on output | Regenerate with explicit safety constraints; log for audit |

---

## 5. Safety Architecture — SEA-LION Guard

The SEA-LION Guard is a mandatory safety wrapper applied to **all agent inputs and outputs**.

### 5.1 Input Gate
- **Prompt injection detection** — rejects attempts to hijack agent instructions
- **Multilingual toxicity filter** — covers EN, ZH, MS, TA
- **PII redaction** — strips identifiers before reaching the LLM
- **FHIR reference validation** — ensures referenced resources are valid and accessible

### 5.2 Output Gate
- **FHIR `$validate`** — profile conformance check on all written resources
- **`$validate-code`** — terminology binding verification (SNOMED, LOINC, RxNorm)
- **Hallucination check** — cross-references clinical findings against source FHIR data
- **Clinical harm filter** — blocks outputs that advise medication changes, diagnoses, or emergency self-treatment

### 5.3 Decision Taxonomy

| Decision | Meaning |
|---|---|
| `PASS` | Content delivered as-is |
| `FLAG` | Content delivered with a warning annotation |
| `ESCALATE` | Requires clinician review before delivery |
| `BLOCK` | Content rejected; agent regenerates or returns safe fallback |

---

## 6. FHIR Data Model

Med-SEAL V1 uses **HL7 FHIR R4** as its primary data standard. Key resource usage:

| Resource | Usage | Standard |
|---|---|---|
| `Patient` | Demographics, language, care team references | FHIR R4 |
| `Condition` | Active diagnoses (diabetes, hypertension, HLD) | SNOMED CT |
| `MedicationRequest` | Prescriptions | RxNorm |
| `MedicationAdministration` | Patient-confirmed dose records | RxNorm |
| `Observation` | Vitals, labs, wearable readings, PRO scores | LOINC |
| `AllergyIntolerance` | Drug/food allergies | SNOMED CT |
| `Composition` | Pre-visit clinical briefs, summaries | LOINC section codes |
| `Communication` | Chat messages, nudges | Med-SEAL CodeSystem |
| `Flag` | Clinical escalation markers | Med-SEAL CodeSystem |
| `CommunicationRequest` | Clinician notification requests | — |
| `RiskAssessment` | Behavioral risk scores | Med-SEAL CodeSystem |
| `Goal` | Patient wellness goals | — |
| `NutritionOrder` | Dietary recommendations | — |
| `QuestionnaireResponse` | PRO responses | — |

---

## 7. Feature Coverage

Med-SEAL V1 delivers **18 AI-powered features** across both patient and clinician surfaces:

| Feature | Description | Primary Agent |
|---|---|---|
| F01 Chat | Conversational health assistant | A1 (Companion) |
| F02 Medication | Adherence tracking, interaction checking | A2 / A6 |
| F03 Nudge | Proactive reminders and alerts | A3 |
| F04 PROs | Patient-reported outcome collection | A1 / A3 |
| F05 Wearables | Biometric data ingestion and alerting | A6 |
| F06 Clinician Summary | Pre-visit brief via CDS Hooks | A5 |
| F07 Dietary | Culturally-appropriate nutrition guidance | A4 |
| F08 Outcome Framework | Population-level clinical metrics | A6 |
| F09 Dashboard | Clinician and patient analytics views | A6 |
| F10 Escalation | Tiered alert escalation to care team | A3 |
| F11 Anticipation | Behavioral risk modeling | A3 |
| F12 Appointments | Scheduling, reminders, pre-visit prep | A1 / A3 |
| F13 Caregiver | Caregiver-aware communication | A1 |
| F14 Education | Personalised health education delivery | A1 / A4 |
| F15 Timeline | Longitudinal patient journey visualization | A6 |
| F16 Readmission | Readmission risk flagging | A6 / A3 |
| F17 A/B Framework | Clinical trial engagement randomisation | A6 |
| F18 Satisfaction | Patient satisfaction measurement | A1 / A6 |

---

## 8. Deployment Context

### 8.1 Current State (Demo)

| Component | Status |
|---|---|
| OpenEMR | ✅ Running (Docker, port 8080/8081) |
| Medplum FHIR | ✅ Running (Docker, port 8103) |
| AI Service | ✅ Running (Node.js, port 4003) |
| SSO | ✅ Running |
| `med-r1` LLM | ❌ Not deployed — no GPU server |
| MERaLiON | ❌ Not deployed |
| SEA-LION | ❌ Not deployed |
| **ChatGPT (demo substitute)** | ✅ Active via OpenAI API |

### 8.2 Production Requirements

To deploy Med-SEAL V1 with the full native model stack:

| Resource | Requirement |
|---|---|
| GPU (MERaLiON, SEA-LION, Guard) | 3× NVIDIA H200 (1 per model) |
| GPU (`med-r1` / Qwen3-VL-8B) | 2× NVIDIA H200 |
| LLM serving framework | vLLM |
| Storage | ≥ 500 GB NVMe (model weights + FHIR data) |
| Network | Private inference endpoints (`medseal.internal`) |

### 8.3 Switching from Demo to Production

Only environment variable changes are required — no code changes:

```bash
# Demo (current)
LLM_API_URL=https://api.openai.com/v1/chat/completions
LLM_MODEL=gpt-4o

# Production (Med-SEAL V1 native)
LLM_API_URL=https://medseal-llm.ngrok-free.dev/v1/chat/completions
LLM_MODEL=med-r1
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048
```

---

## 9. Standards Compliance

| Standard | Usage |
|---|---|
| HL7 FHIR R4 | All clinical data exchange |
| SMART on FHIR | OAuth 2.0 scoped access to patient resources |
| SNOMED CT | Diagnoses (`Condition.code`) |
| LOINC | Lab results and observations (`Observation.code`) |
| RxNorm | Medications (`MedicationRequest.medication`) |
| ICD-10 | Billing and diagnostic codes (OpenEMR layer) |
| CDS Hooks | Clinician-facing alerts and pre-visit briefs (F06) |

---

## 10. Roadmap: V1 → V2

The V1 Clinical Agent is a monolithic fine-tuned model covering all clinical reasoning tasks. The planned V2 architecture decomposes this into specialised agents for scalability and independent model updates:

| V1 Component | V2 Equivalent |
|---|---|
| V1 Clinical Agent (reasoning) | A2 Clinical Reasoning Agent |
| V1 Clinical Agent (pre-visit brief) | A5 Insight Synthesis Agent |
| V1 integrated safety layer | G1 SEA-LION Guard (standalone) |
| V1 single orchestrator | O1 smolagents Orchestrator (extended) |

---

## 11. References

- {doc}`ai-agents/overview` — Agent roster and orchestration diagram
- {doc}`ai-agents/clinical-agent` — V1 Clinical Agent detail page
- {doc}`architecture` — System layer and deployment diagrams
- {doc}`developer-guide/api-reference` — REST API reference
- {doc}`developer-guide/fhir-data-model` — FHIR resource profiles
- {doc}`developer-guide/adding-ai-features` — Extension guide
- {doc}`standards` — Coding standards and conventions
- {doc}`deployment` — Infrastructure and environment configuration
