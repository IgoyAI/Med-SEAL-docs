# AI Agents Overview

Med-SEAL Suite employs a **multi-agent AI architecture** where specialised agents collaborate to deliver 18 smart healthcare features. The agents are orchestrated by a rule-based router and protected by the SEA-LION Guard safety system.

## Agent Roster

| ID | Agent | Model | Surface |
|---|---|---|---|
| **V1** | **Med-SEAL V1 Clinical Agent** | `med-r1` / ChatGPT *(demo)* | Both |
| A1 | **Companion Agent** | MERaLiON + SEA-LION | Patient app |
| A2 | **Clinical Reasoning Agent** | Qwen3-VL-8B (Med-SEAL) | Both |
| A3 | **Nudge Agent** | MERaLiON + rule engine | Patient app |
| A4 | **Lifestyle Agent** | SEA-LION + nutrition KB | Patient app |
| A5 | **Insight Synthesis Agent** | Qwen3-VL-8B (Med-SEAL) | Both |
| A6 | **Measurement Agent** | Analytics engine (no LLM) | Both |
| SYS | **SEA-LION Guard** | SEA-LION Guard | System-wide |
| SYS | **smolagents Orchestrator** | Rule-based router | System-wide |

> **⚠️ Demo Note:** The **Med-SEAL V1 Clinical Agent** is the first-generation clinical agent and is **not live-deployed** in the demo environment due to infrastructure constraints (no dedicated GPU inference server). Its role in the interactive demo is fulfilled by **ChatGPT**. See {doc}`clinical-agent` for full details.

### Agent Class Diagram

```{uml}
@startuml
skinparam classAttributeIconSize 0
hide circle

abstract class BaseAgent {
  agentId : string
  model : string
  --
  +process(input: AgentInput): AgentOutput
  +readFHIR(query: FHIRQuery): Bundle
}

class CompanionAgent {
  language : string
  --
  +chat(message, patientId): Reply
  +rephrase(technicalText): string
  +deliverPRO(questionnaire): Response
}

class ClinicalReasoningAgent {
  --
  +analyse(conditions, meds, obs): ClinicalResponse
  +checkInteractions(meds): Warning[]
}

class NudgeAgent {
  ruleEngine : RuleEngine
  --
  +evaluate(triggers): Nudge[]
  +escalate(severity, patientId): Alert
}

class LifestyleAgent {
  nutritionKB : KnowledgeBase
  --
  +recommend(diet, conditions): Advice
  +checkFoodInteractions(meds, food): Warning[]
}

class InsightSynthesisAgent {
  lookbackDays : number
  --
  +generateBrief(patientId): Composition
  +synthesise(adherence, biometrics, PROs): Summary
}

class MeasurementAgent {
  --
  +computePDC(meds, admin): number
  +computeTrends(observations): TrendData
  +generateMeasureReport(): MeasureReport
}

class SEALIONGuard {
  --
  +checkInput(content): GuardDecision
  +checkOutput(content): GuardDecision
  +redactPII(text): string
}

class Orchestrator {
  --
  +classify(message): Intent
  +route(intent): BaseAgent
}

enum GuardDecision {
  PASS
  FLAG
  ESCALATE
  BLOCK
}

BaseAgent <|-- CompanionAgent : A1
BaseAgent <|-- ClinicalReasoningAgent : A2
BaseAgent <|-- NudgeAgent : A3
BaseAgent <|-- LifestyleAgent : A4
BaseAgent <|-- InsightSynthesisAgent : A5
BaseAgent <|-- MeasurementAgent : A6

Orchestrator --> BaseAgent : routes to
SEALIONGuard --> BaseAgent : wraps
SEALIONGuard ..> GuardDecision : returns
@enduml
```

## Agent Descriptions

### V1 — Med-SEAL V1 Clinical Agent
The **first-generation clinical agent** for Med-SEAL. A single fine-tuned model (`med-r1`, Qwen3-VL-8B) that consolidates clinical reasoning and insight synthesis. It reads FHIR patient data, answers clinical questions, checks drug interactions, and generates pre-visit briefs. In the V2 multi-agent design its responsibilities are split across A2 and A5.

> **Not deployed in demo** — ChatGPT is used as a substitute. See {doc}`clinical-agent` for architecture details and the infrastructure requirements needed to run V1 in production.

### A1  - Companion Agent
The patient's primary conversational interface. Powered by **MERaLiON** for empathetic, culturally-aware dialogue and **SEA-LION** for multilingual support (English, Chinese, Malay, Tamil). Delegates complex queries to specialised agents and rephrases technical responses in patient-friendly language.

### A2  - Clinical Reasoning Agent
The medical reasoning engine powered by **Qwen3-VL-8B** (fine-tuned as med-r1). Reads FHIR clinical data (conditions, medications, observations) and synthesises evidence-based clinical responses. Used by both patient-facing features (via A1) and clinician-facing features.

### A3  - Nudge Agent
The proactive engagement engine combining **MERaLiON** for message generation with a configurable **rule engine** for trigger evaluation. Handles medication reminders, appointment alerts, biometric threshold warnings, and tiered escalation to clinicians.

### A4  - Lifestyle Agent
Specialises in dietary and lifestyle recommendations using **SEA-LION** with a **nutrition knowledge base**. Provides culturally-appropriate suggestions grounded in Southeast Asian food (hawker centre dishes, festive food alternatives) with drug-food interaction checking.

### A5  - Insight Synthesis Agent
Generates **pre-visit clinical briefs** for clinicians using **Qwen3-VL-8B**. Synthesises 30 days of patient-side data (adherence, biometrics, PROs, engagement, flags) into structured FHIR Compositions delivered via CDS Hooks.

### A6  - Measurement Agent
A **non-LLM analytics engine** that computes clinical metrics: medication adherence (PDC), biometric trends, PRO score changes, engagement frequency, readmission rates. Produces FHIR MeasureReports for both individual patients and population cohorts.

## Orchestration

The **smolagents Orchestrator** classifies incoming requests and routes them:

```{uml}
@startuml
skinparam handwritten false

rectangle "Patient message" as Message
hexagon "SEA-LION Guard\n//(input validation)//" as Guard
hexagon "Orchestrator\n//(intent classification)//" as Orchestrator

rectangle "A1 (Companion)" as A1
rectangle "A2 (Clinical Reasoning)" as A2
rectangle "A4 (Lifestyle)" as A4
rectangle "A3 (Nudge)" as A3
rectangle "clinician alert" as Alert

Message --> Guard
Guard --> Orchestrator

Orchestrator --> A1 : casual / general health
Orchestrator --> A2 : medical query
Orchestrator --> A4 : dietary / lifestyle
Orchestrator --> A1 : PRO questionnaire
Orchestrator --> A3 : escalation

A2 ..> A1 : rephrase
A4 ..> A1 : rephrase

A3 --> Alert
@enduml
```

## SEA-LION Guard

A system-wide safety layer that wraps all agent interactions:

### Input Gate
- Prompt injection detection
- Multilingual toxicity filter (EN, ZH, MS, TA)
- PII redaction
- FHIR reference validation

### Output Gate
- FHIR `$validate` (profile conformance)
- `$validate-code` (terminology binding)
- Hallucination check (findings vs source data)
- Clinical harm filter

### Decisions
- **PASS**  - content delivered as-is
- **FLAG**  - content delivered with warning annotation
- **ESCALATE**  - content requires clinician review before delivery
- **BLOCK**  - content rejected entirely

## Feature-to-Agent Matrix

| Feature | A1 | A2 | A3 | A4 | A5 | A6 |
|---|---|---|---|---|---|---|
| F01 Chat | PRIMARY | delegate | | | | |
| F02 Medication | display | PRIMARY | alert | | | compute |
| F03 Nudge | | | PRIMARY | | | |
| F04 PROs | PRIMARY | | trigger | | consume | |
| F05 Wearables | context | | alert | | | PRIMARY |
| F06 Clinician summary | | | | | PRIMARY | data |
| F07 Dietary | relay | | | PRIMARY | | |
| F08 Outcome framework | | | | | | PRIMARY |
| F09 Dashboard | | | | | | PRIMARY |
| F10 Escalation | | | PRIMARY | | enrich | |
| F11 Anticipation | | | PRIMARY | | | data |
| F12 Appointments | PRIMARY | | trigger | | pre-visit | |
| F13 Caregiver | PRIMARY | | forward | | | data |
| F14 Education | PRIMARY | | trigger | assist | | |
| F15 Timeline | | | | | | PRIMARY |
| F16 Readmission | | | flag | | consume | PRIMARY |
| F17 A/B framework | | | | | | PRIMARY |
| F18 Satisfaction | PRIMARY | | trigger | | | compute |

## LLM Configuration

The AI Service connects to a self-hosted LLM via:

```bash
LLM_API_URL=https://medseal-llm.ngrok-free.dev/v1/chat/completions
LLM_MODEL=med-r1
LLM_TEMPERATURE=0.3
LLM_MAX_TOKENS=2048
```

See {doc}`features` for the full I/O specification of all 18 features.
