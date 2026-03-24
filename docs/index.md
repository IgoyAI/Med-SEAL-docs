# Med-SEAL Suite Documentation

**Empowering 1.8 million Singaporeans with chronic diseases to thrive between clinic visits.**

---

## The Problem

In Singapore, **1 in 3 adults** lives with a chronic condition — diabetes, hypertension, or hyperlipidemia. They see their doctor for 15 minutes every 3–6 months. That leaves **4,300+ hours a year** where patients manage medications, diet, symptoms, and anxiety **alone**.

During those hours, patients:
- 💊 Forget doses — **50% of chronic disease patients** are non-adherent to medication
- 📉 Miss warning signs — a rising HbA1c or blood pressure trend goes unnoticed until the next visit
- 😰 Feel unsupported — questions go unanswered, motivation fades, conditions worsen
- 🏥 End up in A&E — preventable complications from poor self-management cost **SGD 2.5B annually**

The HealthierSG initiative asks patients to take ownership of their health. But ownership without support is just burden.

## The Solution

**Med-SEAL** (Medical — Safe Empowerment through AI-assisted Living) is an **AI-powered health companion** that walks beside the patient in those 4,300 hours.

It speaks their language — English, Mandarin, Malay, or Tamil. It knows their medications, conditions, and lab results. It reminds them gently, explains clearly, escalates urgently, and never pretends to be a doctor.

Behind the scenes, 7 specialised AI agents collaborate to deliver proactive, empathetic, culturally-aware chronic disease management — all built on Singapore's own **SEA-LION** National LLM and interoperable **HL7 FHIR R4** healthcare standards.

> *"Uncle Lim, 68, Type 2 diabetic. He missed his Metformin twice this week. Med-SEAL noticed, sent a gentle nudge in Mandarin. His blood glucose was trending up — Med-SEAL flagged it to Dr. Tan before Friday's appointment. At the visit, Dr. Tan had a pre-visit brief ready with 30 days of adherence data, biometric trends, and PRO scores. The consultation was focused, informed, and 10 minutes shorter."*
>
> — This is the experience Med-SEAL enables.

---

## What Med-SEAL Does

| For Patients | For Clinicians |
|---|---|
| 🗣️ Chat in 4 languages about health concerns | 📋 Auto-generated pre-visit briefs with 30 days of data |
| 💊 Proactive medication reminders | 🚨 Tiered escalation alerts (low → medium → high) |
| 📊 Track vitals, labs, and trends over time | 📈 Population-level adherence and outcome dashboards |
| 🥗 Culturally-aware dietary advice (hawker food, festive meals) | 🩺 CDS Hooks integration in OpenEMR |
| 📅 Appointment booking, reminders, and preparation | 🔒 Full FHIR R4 audit trail |
| ⚠️ Emergency detection → immediate 995 guidance | 📱 Caregiver mode with consent-gated access |

## How It's Built

| Layer | Technology |
|---|---|
| **AI Agents** | 7 LangGraph agents orchestrated by a safety-guarded rule-based router |
| **LLM** | SEA-LION v4-32B (AI Singapore) + [Med-SEAL-V1](https://huggingface.co/aagdeyogipramana/Med-SEAL-V1) (fine-tuned clinical model) |
| **Safety** | Dual-layer guard: 21 regex patterns + SEA-Guard LLM classification |
| **Data** | HL7 FHIR R4 via Medplum — interoperable with any hospital system |
| **EMR** | OpenEMR integration with CDS Hooks |
| **Patient App** | React Native (iOS & Android) |
| **Infrastructure** | GCP Cloud Run + GKE (Singapore region) |

---

```{toctree}
:maxdepth: 2
:caption: Overview

demo
architecture
getting-started
```

```{toctree}
:maxdepth: 2
:caption: Components

components/openemr
components/medplum
components/patient-portal-native
components/sso
```

```{toctree}
:maxdepth: 2
:caption: AI Platform

ai-agents/overview
ai-agents/clinical-agent
ai-agents/azure-deployment
ai-agents/gcp-deployment

ai-agents/features
technical-report-v1
```

```{toctree}
:maxdepth: 2
:caption: Operations & Reference

deployment
gke-deployment
scripts
standards
fhir-resources
contributing
```

```{toctree}
:maxdepth: 2
:caption: User Guide

user-guide/index
user-guide/login-sso
user-guide/patient-portal
user-guide/openemr-clinician
user-guide/ai-chat
user-guide/medications
user-guide/appointments
user-guide/vitals
```

```{toctree}
:maxdepth: 2
:caption: Developer Guide

developer-guide/index
developer-guide/environment-setup
developer-guide/fhir-data-model
developer-guide/api-reference
developer-guide/adding-ai-features
developer-guide/testing
developer-guide/extending-openemr
```
