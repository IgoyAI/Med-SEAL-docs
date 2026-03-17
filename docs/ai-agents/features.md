# Feature Specifications (F01–F18)

This page documents all 18 Med-SEAL features, organised into three tiers by priority. Each feature specifies its assigned agent(s), input sources, processing logic, and output destinations.

---

## Tier 1: Must Be Functional in Prototype

### F01  - MERaLiON / SEA-LION Chat Interface

**Agent:** A1 (Companion) &nbsp;|&nbsp; **Impact:** KQ1 +15, KQ2 +10

The primary conversational interface supporting English, Chinese, Malay, and Tamil. The Companion Agent handles casual and general health queries directly, delegates medical queries to A2 (Clinical Reasoning) and dietary queries to A4 (Lifestyle), then rephrases responses empathetically via MERaLiON.

**Key outputs:** Chat responses, suggested follow-up actions, FHIR Communication + AuditEvent records.

---

### F02  - Medication Management + Adherence Tracking

**Agent:** A2 (Clinical Reasoning) + A3 (Nudge) + A1 (Companion) &nbsp;|&nbsp; **Impact:** KQ1 +10, KQ4 +15

Reads active MedicationRequests from Medplum, generates daily medication schedules, tracks dose confirmations as MedicationAdministration resources, checks drug interactions via RxNorm, and computes weekly PDC (proportion of days covered) adherence metrics.

**Key outputs:** Daily schedule, interaction warnings, MedicationAdministration, adherence Observation.

---

### F03  - Smart Nudge Engine

**Agent:** A3 (Nudge) &nbsp;|&nbsp; **Impact:** KQ1 +15, KQ4 +5

Proactive engagement system triggered by: missed doses, high biometric readings, daily engagement checks, and appointment reminders. Nudges are delivered in the patient's language via MERaLiON. Tiered escalation:

- **Low**  - Patient nudge only (e.g., gentle medication reminder)
- **Medium**  - Patient nudge + next-day clinician flag
- **High**  - Immediate clinician alert (e.g., BP >180/120, suicidal ideation)

---

### F04  - Conversational PROs (Patient-Reported Outcomes)

**Agent:** A1 (Companion) &nbsp;|&nbsp; **Impact:** KQ2 +5, KQ3 +5, KQ4 +10

Delivers standardised questionnaires (PHQ-9, DDS-17) conversationally rather than as forms. The Companion Agent maps free-text patient responses to structured answers, computes derived scores, and triggers escalation when thresholds are exceeded.

**Key outputs:** FHIR QuestionnaireResponse, derived score Observations, threshold breach alerts.

---

### F05  - Wearable Data Ingestion

**Agent:** A6 (Measurement) + A3 (Nudge) &nbsp;|&nbsp; **Impact:** KQ2 +10, KQ4 +10

Ingests vitals from Apple Health, Google Health Connect, Bluetooth devices, and manual entry. Maps readings to FHIR Observations with LOINC codes (BP: 8480-6/8462-4, glucose: 2345-7, heart rate: 8867-4, steps: 55423-8, weight: 29463-7). Batch uploaded as FHIR transaction Bundles.

**Key outputs:** FHIR Observations, computed 7/30-day averages, threshold breach events, trend visualisation data.

---

### F06  - Patient Insight Summary for Clinician

**Agent:** A5 (Insight Synthesis) &nbsp;|&nbsp; **Impact:** KQ3 +10

Generates a pre-visit brief as a FHIR Composition with 7 sections: adherence summary, biometric trends, PRO scores, engagement level, flagged concerns, goal progress, and recommended actions. Triggered via CDS Hooks `patient-view` or 24h before appointments.

---

## Tier 2: Strengthens Personalisation + Impact

### F07  - Dietary + Lifestyle Engine (SEA-Culturally Aware)

**Agent:** A4 (Lifestyle) &nbsp;|&nbsp; **Impact:** KQ2 +10

Culturally-aware dietary recommendations grounded in Southeast Asian food context (hawker dishes, festive foods). Checks drug-food interactions and generates FHIR NutritionOrders with specific, actionable advice.

---

### F08  - Outcome Measurement Framework

**Agent:** A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ4 +10

Computes population-level clinical metrics: medication adherence PDC, biometric improvement deltas, PRO score changes, engagement frequency, readmission rates, and time-to-intervention. Produces FHIR MeasureReports (individual + summary).

---

### F09  - Adherence + Biometric Analytics Dashboard

**Agent:** A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ4 +10

Provides three view modes:
- **Patient**  - 90-day time-series with goal lines and intervention markers
- **Clinician**  - per-patient panel sortable by risk level
- **Population**  - cohort-level aggregated trends

---

### F10  - Escalation Pathways (Tiered Clinician Alerts)

**Agent:** A3 (Nudge) + A5 (Insight Synthesis) &nbsp;|&nbsp; **Impact:** KQ3 +10

Structured clinical alerts with three severity tiers. High-priority alerts (severe symptoms, suicidal ideation) generate immediate FHIR Flags + CommunicationRequests. Clinicians receive one-click actions: Review, Call, Adjust Medication, Dismiss.

---

### F11  - Behavioral Anticipation Model

**Agent:** A3 (Nudge) &nbsp;|&nbsp; **Impact:** KQ1 +5, KQ4 +5

Predictive model computing: disengagement risk, non-adherence risk, and clinical deterioration risk. Uses logistic model / rule-based thresholds over 30-day adherence, engagement, and biometric patterns. Produces FHIR RiskAssessments.

---

## Tier 3: Polish + Differentiation

### F12  - Appointment Orchestrator (Pre/Post Visit)

**Agent:** A1 (Companion) + A5 (Insight Synthesis) &nbsp;|&nbsp; **Impact:** KQ1 +5

Pre-visit: preparation prompts (what to bring, questions to ask). Post-visit: plain-language summaries of medication changes, care plan updates, and follow-up tasks.

---

### F13  - Caregiver Mode (Linked Family View with Consent)

**Agent:** A1 (Companion) + A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ1 +5

Consent-gated caregiver access via FHIR Consent resources. Caregivers see a read-only filtered dashboard and receive forwarded alerts (medium/high severity). Full audit trail via AuditEvents.

---

### F14  - Adaptive Health Education (Teachable Moments)

**Agent:** A1 (Companion) + A4 (Lifestyle) &nbsp;|&nbsp; **Impact:** KQ2 +5

Event-triggered educational content (e.g., high glucose reading → diabetes management tips). Adapted to patient health literacy level and preferred language.

---

### F15  - Multi-Source Data Fusion Timeline

**Agent:** A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ2 +5

Unified chronological timeline merging vitals, medications, encounters, nudges, goals, and escalation events. Available in both patient and clinician views.

---

### F16  - Readmission Risk + Event Tracking

**Agent:** A6 (Measurement) + A3 (Nudge) &nbsp;|&nbsp; **Impact:** KQ4 +5

Readmission risk scoring using encounter history, adherence, biometric trends, PRO scores, and condition complexity. High-risk patients receive increased monitoring frequency.

---

### F17  - A/B Evaluation Framework (Synthetic Cohorts)

**Agent:** A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ4 +5

Compares Med-SEAL intervention cohort against Synthea-generated control cohort. Produces comparative FHIR MeasureReports with effect sizes and p-values for impact demonstration.

---

### F18  - Patient Satisfaction + NPS Tracking

**Agent:** A1 (Companion) + A6 (Measurement) &nbsp;|&nbsp; **Impact:** KQ4 +5

Monthly NPS + CSAT surveys delivered conversationally (via F04 pattern). Computes promoter/detractor ratios and satisfaction trends over time.
