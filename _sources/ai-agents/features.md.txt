# Feature Specifications (F01-F18)

This page documents all 18 Med-SEAL features, organised into three tiers by priority. Each feature specifies its assigned agent(s), input sources, processing logic, output destinations, and **current implementation status**.

---

## Implementation Status Summary

| Feature | Name | Status | Patient App | AI Service | FHIR |
|---|---|---|---|---|---|
| **F01** | Chat Interface | **Live** | `chat.tsx` (64KB) | `/chat`, `/chat/stream` | AuditEvent |
| **F02** | Medication Management | **Live** | `medications.tsx` (25KB) | FHIR read | MedicationRequest, MedicationAdministration |
| **F03** | Smart Nudge Engine | **Built** | `index.tsx` nudge cards | Mock engine | -- |
| **F04** | Conversational PROs | **Live** | `checkin.tsx` (15KB) | FHIR write | QuestionnaireResponse |
| **F05** | Wearable Data | **Live** | `watch.tsx` (23KB), `health.tsx` | HealthKit bridge | Observation (vitals) |
| **F06** | Pre-Visit Summary | **Live** | `insights.tsx` (31KB) | `/patients/:id/previsit-summary` | Composition, 11-section FHIR aggregation |
| **F07** | Dietary/Lifestyle | **Live** | `lifestyle.tsx` (39KB) | AI diet plan + fallback | NutritionOrder, Goal |
| **F08** | Outcome Measurement | **Built** | `analytics.tsx` (12KB) | Dashboard metrics | Observation, MeasureReport |
| **F09** | Analytics Dashboard | **Built** | `analytics.tsx` | `getDashboardMetrics()` | Observation (vitals + labs) |
| **F10** | Escalation Pathways | **Live** | `escalations.tsx` (8KB) | FHIR read | Flag |
| **F11** | Behavioral Anticipation | **Built** | `analytics.tsx` badges | Mock risk model | -- |
| **F12** | Appointment Orchestrator | **Live** | `appointments.tsx` (53KB) | `/api/appointments/writeback` | Appointment, Encounter, DocumentReference |
| **F13** | Caregiver Mode | **Built** | `caregiver.tsx` (33KB) | FHIR read (consent-gated) | Consent, filtered vitals |
| **F14** | Health Education | **Built** | `education.tsx` (38KB) | Context-triggered cards | -- |
| **F15** | Data Fusion Timeline | **Built** | `timeline.tsx` (8KB) | Multi-source aggregation | -- |
| **F16** | Readmission Risk | **Built** | `analytics.tsx` | Mock risk scoring | RiskAssessment |
| **F17** | A/B Evaluation | **Planned** | -- | Synthea scripts exist | MeasureReport |
| **F18** | Patient Satisfaction | **Built** | `satisfaction.tsx` (11KB) | Mock NPS tracking | -- |

**Status definitions:**
- **Live** -- End-to-end working: UI + AI Service API + FHIR data (or real HealthKit bridge)
- **Built** -- UI screen is complete and functional; uses mock/fallback data when backend API is unavailable
- **Planned** -- Designed and specified; not yet implemented in code

---

## Tier 1: Must Be Functional in Prototype

### F01 - MERaLiON / SEA-LION Chat Interface

**Agent:** A1 (Companion) | **Status: Live**

The primary conversational interface supporting English, Chinese, Malay, and Tamil. The Companion Agent handles casual and general health queries directly, delegates medical queries to A2 (Clinical Reasoning) and dietary queries to A4 (Lifestyle), then rephrases responses empathetically via MERaLiON.

**Implementation:**
- **Patient app:** `chat.tsx` (64KB) -- full-featured chat UI with streaming responses, voice input, language picker, and suggested prompts
- **AI Service:** `POST /chat` and `POST /chat/stream` endpoints in `index.ts`
- **LLM:** Connected to med-r1 via `llm.ts` with system prompts from `prompts.ts`
- **FHIR:** Patient context built from live `Patient`, `Condition`, `AllergyIntolerance`, `MedicationRequest` resources

**Key outputs:** Chat responses, suggested follow-up actions, FHIR Communication + AuditEvent records.

---

### F02 - Medication Management + Adherence Tracking

**Agent:** A2 (Clinical Reasoning) + A3 (Nudge) + A1 (Companion) | **Status: Live**

Reads active MedicationRequests from Medplum, generates daily medication schedules, tracks dose confirmations as MedicationAdministration resources, checks drug interactions via RxNorm, and computes weekly PDC (proportion of days covered) adherence metrics.

**Implementation:**
- **Patient app:** `medications.tsx` (25KB) -- daily schedule view with timing inference, dose logging UI, refill tracking
- **Services:** `medication-service.ts` (17KB) -- FHIR `MedicationRequest` read, `MedicationAdministration` write, adherence computation
- **FHIR integration:** Full read/write -- `MedicationRequest` (active meds), `MedicationAdministration` (dose logs)
- **Mock fallback:** 4 medications (Metformin, Lisinopril, Atorvastatin, Vitamin D3) when FHIR unavailable

**Key outputs:** Daily schedule, interaction warnings, MedicationAdministration, adherence Observation.

---

### F03 - Smart Nudge Engine

**Agent:** A3 (Nudge) | **Status: Built**

Proactive engagement system triggered by: missed doses, high biometric readings, daily engagement checks, and appointment reminders. Nudges are delivered in the patient's language via MERaLiON. Tiered escalation:

- **Low** -- Patient nudge only (e.g., gentle medication reminder)
- **Medium** -- Patient nudge + next-day clinician flag
- **High** -- Immediate clinician alert (e.g., BP >180/120, suicidal ideation)

**Implementation:**
- **Patient app:** `index.tsx` home screen displays nudge cards with severity colours and action buttons
- **Services:** `nudge-service.ts` (3KB) -- nudge display and dismissal
- **Backend:** Nudge generation uses mock data; rule engine not yet wired to live triggers
- **Note:** UI fully functional; backend rule engine for automated trigger evaluation is planned

---

### F04 - Conversational PROs (Patient-Reported Outcomes)

**Agent:** A1 (Companion) | **Status: Live**

Delivers standardised questionnaires (PHQ-9, DDS-17) conversationally rather than as forms. The Companion Agent maps free-text patient responses to structured answers, computes derived scores, and triggers escalation when thresholds are exceeded.

**Implementation:**
- **Patient app:** `checkin.tsx` (15KB) -- conversational wellness check-in with emoji-based scales
- **Services:** `pro-service.ts` (7KB) -- PRO questionnaire delivery and scoring
- **FHIR write:** Completed check-ins are saved as `QuestionnaireResponse` resources
- **Scoring guide:** Built-in threshold detection (low/moderate/high distress)

**Key outputs:** FHIR QuestionnaireResponse, derived score Observations, threshold breach alerts.

---

### F05 - Wearable Data Ingestion

**Agent:** A6 (Measurement) + A3 (Nudge) | **Status: Live**

Ingests vitals from Apple Health, Google Health Connect, Bluetooth devices, and manual entry. Maps readings to FHIR Observations with LOINC codes (BP: 8480-6/8462-4, glucose: 2345-7, heart rate: 8867-4, steps: 55423-8, weight: 29463-7). Batch uploaded as FHIR transaction Bundles.

**Implementation:**
- **Patient app:** `watch.tsx` (23KB) -- wearable device management, sync status, data visualization; `health.tsx` (15KB) -- consolidated health dashboard
- **Services:** `health-kit-service.ts` (7KB) -- Apple HealthKit bridge for iOS; `vitals-service.ts` (8KB) -- FHIR Observation read/write
- **FHIR integration:** Reads `Observation` (vital-signs category) for heart rate, BP, glucose; writes new readings
- **Mock fallback:** 3 devices (Apple Watch, Accu-Chek glucometer, Omron BP monitor) and 7-day trend data when no FHIR data available

**Key outputs:** FHIR Observations, computed 7/30-day averages, threshold breach events, trend visualisation data.

---

### F06 - Patient Insight Summary for Clinician

**Agent:** A5 (Insight Synthesis) | **Status: Live**

Generates a pre-visit brief as a FHIR Composition with 11 sections: active conditions, latest biometrics, lab results, current medications, medication adherence, allergies, upcoming appointments, recent encounters, health goals, escalation flags, and clinical summary. Triggered via API or 24h before appointments.

**Implementation:**
- **Patient app:** `insights.tsx` (31KB) -- enterprise-grade summary viewer with collapsible sections, data cards, and loading states
- **Services:** `feature-services.ts` -- `fetchPreVisitFromAgent()` calls `POST /patients/:id/previsit-summary`; `getPreVisitSummary()` provides local FHIR aggregation fallback
- **AI Service:** Dedicated endpoint with 11-section structured response
- **FHIR integration:** Full -- reads `Condition`, `Observation`, `MedicationRequest`, `MedicationAdministration`, `AllergyIntolerance`, `Appointment`, `Encounter`, `Goal`, `Flag`
- **Auto-generation:** Pre-visit summaries are automatically generated when a new appointment is booked (fire-and-forget) and saved as `DocumentReference`

---

## Tier 2: Strengthens Personalisation + Impact

### F07 - Dietary + Lifestyle Engine (SEA-Culturally Aware)

**Agent:** A4 (Lifestyle) | **Status: Live**

Culturally-aware dietary recommendations grounded in Southeast Asian food context (hawker dishes, festive foods). Checks drug-food interactions and generates FHIR NutritionOrders with specific, actionable advice.

**Implementation:**
- **Patient app:** `lifestyle.tsx` (39KB) -- meal logging, dietary recommendations, AI diet plan generation, exercise recommendations, festive food alerts (e.g., Hari Raya)
- **Services:** `feature-services.ts` -- `logMeal()` (FHIR `NutritionOrder` write), `getDietaryRecommendations()` (AI agent with fallback), `generateAIDietPlan()` (condition-aware + medication interaction), `getHealthGoals()` (FHIR `Goal` read), `getMealHistory()` (FHIR `NutritionOrder` read)
- **FHIR integration:** Full read/write -- `NutritionOrder`, `Goal`
- **Notable:** Localised recommendations for Singaporean food (nasi lemak, chicken rice, teh tarik alternatives, hawker centre tips)

---

### F08 - Outcome Measurement Framework

**Agent:** A6 (Measurement) | **Status: Built**

Computes population-level clinical metrics: medication adherence PDC, biometric improvement deltas, PRO score changes, engagement frequency, readmission rates, and time-to-intervention. Produces FHIR MeasureReports (individual + summary).

**Implementation:**
- **Patient app:** `analytics.tsx` (12KB) -- metrics dashboard with trend charts and intervention period markers
- **Services:** `feature-services.ts` -- `getDashboardMetrics()` reads FHIR `Observation` (vital-signs + laboratory) for BP, glucose, heart rate, HbA1c
- **FHIR integration:** Reads `Observation` for real metrics; time-series and population views use mock data
- **Note:** Individual metrics are live from FHIR; population-level aggregation uses mock data

---

### F09 - Adherence + Biometric Analytics Dashboard

**Agent:** A6 (Measurement) | **Status: Built**

Provides three view modes:
- **Patient** -- 90-day time-series with goal lines and intervention markers
- **Clinician** -- per-patient panel sortable by risk level
- **Population** -- cohort-level aggregated trends

**Implementation:**
- **Patient app:** `analytics.tsx` -- combined with F08 dashboard; chart data from `getChartData()` which calls `vitals-service.ts` for FHIR trend data
- **FHIR integration:** Reads vital trend data from `Observation` resources
- **Note:** Patient view is live; clinician and population views are planned

---

### F10 - Escalation Pathways (Tiered Clinician Alerts)

**Agent:** A3 (Nudge) + A5 (Insight Synthesis) | **Status: Live**

Structured clinical alerts with three severity tiers. High-priority alerts (severe symptoms, suicidal ideation) generate immediate FHIR Flags + CommunicationRequests. Clinicians receive one-click actions: Review, Call, Adjust Medication, Dismiss.

**Implementation:**
- **Patient app:** `escalations.tsx` (8KB) -- alert list with severity badges, clinician escalation status
- **Services:** `feature-services.ts` -- `getEscalationFlags()` reads FHIR `Flag` resources with severity parsing
- **FHIR integration:** Full read -- `Flag` (active alerts by patient)

---

### F11 - Behavioral Anticipation Model

**Agent:** A3 (Nudge) | **Status: Built**

Predictive model computing: disengagement risk, non-adherence risk, and clinical deterioration risk. Uses logistic model / rule-based thresholds over 30-day adherence, engagement, and biometric patterns. Produces FHIR RiskAssessments.

**Implementation:**
- **Patient app:** `analytics.tsx` -- engagement badges and streak tracking
- **Services:** `api.ts` -- `getMockEngagementData()` with risk level, streaks, badges, weekly activity
- **Note:** UI complete with gamification elements (badges, streaks); predictive model uses mock risk scores

---

## Tier 3: Polish + Differentiation

### F12 - Appointment Orchestrator (Pre/Post Visit)

**Agent:** A1 (Companion) + A5 (Insight Synthesis) | **Status: Live**

Pre-visit: preparation prompts (what to bring, questions to ask). Post-visit: plain-language summaries of medication changes, care plan updates, and follow-up tasks.

**Implementation:**
- **Patient app:** `appointments.tsx` (53KB) -- the largest screen; full appointment management with booking, cancellation, doctor/specialty selection, available slots, SOAP note viewing
- **Services:** `feature-services.ts` -- `getAppointments()` (FHIR `Appointment` + `Encounter` merge), `bookAppointment()` (FHIR `Appointment` write + OpenEMR writeback + auto pre-visit summary)
- **AI Service:** `POST /api/appointments/writeback` syncs to OpenEMR; `POST /api/admin/sync-appointments` for bulk sync
- **FHIR integration:** Full read/write -- `Appointment`, `Encounter`, `DocumentReference` (SOAP notes), `Practitioner` (doctor names)
- **Post-visit:** Extracts and displays SOAP note summaries from `DocumentReference` base64-encoded attachments

---

### F13 - Caregiver Mode (Linked Family View with Consent)

**Agent:** A1 (Companion) + A6 (Measurement) | **Status: Built**

Consent-gated caregiver access via FHIR Consent resources. Caregivers see a read-only filtered dashboard and receive forwarded alerts (medium/high severity). Full audit trail via AuditEvents.

**Implementation:**
- **Patient app:** `caregiver.tsx` (33KB) -- caregiver invitation, consent management, filtered health summary, alert forwarding
- **Services:** `api.ts` -- `getMockCaregiverView()` with adherence status, latest vitals, engagement level, and alerts
- **Note:** Full UI with consent management workflow; data uses mock source

---

### F14 - Adaptive Health Education (Teachable Moments)

**Agent:** A1 (Companion) + A4 (Lifestyle) | **Status: Built**

Event-triggered educational content (e.g., high glucose reading -> diabetes management tips). Adapted to patient health literacy level and preferred language.

**Implementation:**
- **Patient app:** `education.tsx` (38KB) -- educational content cards with categories (medication, nutrition, exercise, condition, lab), read time estimates, save/bookmark functionality
- **Services:** `api.ts` -- `getMockEducationContent()` with 6 contextual articles (HbA1c explained, Metformin mechanism, glucose spikes, walking benefits, BP basics, Hari Raya food guide)
- **Note:** UI complete with context triggers shown; content is currently static mock data

---

### F15 - Multi-Source Data Fusion Timeline

**Agent:** A6 (Measurement) | **Status: Built**

Unified chronological timeline merging vitals, medications, encounters, nudges, goals, and escalation events. Available in both patient and clinician views.

**Implementation:**
- **Patient app:** `timeline.tsx` (8KB) -- chronological event list with type-based icons and colour coding
- **Services:** `api.ts` -- `getMockTimeline()` with 10 sample events from EHR, wearable, PRO, lifestyle, and flag sources
- **Note:** UI complete; data aggregation pipeline uses mock data

---

### F16 - Readmission Risk + Event Tracking

**Agent:** A6 (Measurement) + A3 (Nudge) | **Status: Built**

Readmission risk scoring using encounter history, adherence, biometric trends, PRO scores, and condition complexity. High-risk patients receive increased monitoring frequency.

**Implementation:**
- **Patient app:** `analytics.tsx` -- risk score visualization with contributing factors
- **Services:** `api.ts` -- `getMockReadmissionRisk()` with 6 risk factors (adherence, BP trend, HbA1c, PHQ-9, engagement, ED visits)
- **Note:** UI complete with factor breakdown; risk algorithm uses mock scoring

---

### F17 - A/B Evaluation Framework (Synthetic Cohorts)

**Agent:** A6 (Measurement) | **Status: Planned**

Compares Med-SEAL intervention cohort against Synthea-generated control cohort. Produces comparative FHIR MeasureReports with effect sizes and p-values for impact demonstration.

**Implementation:**
- **Scripts:** Synthea patient generation scripts exist in `scripts/` for control cohort creation
- **Note:** Framework designed; backend analysis and reporting not yet implemented

---

### F18 - Patient Satisfaction + NPS Tracking

**Agent:** A1 (Companion) + A6 (Measurement) | **Status: Built**

Monthly NPS + CSAT surveys delivered conversationally (via F04 pattern). Computes promoter/detractor ratios and satisfaction trends over time.

**Implementation:**
- **Patient app:** `satisfaction.tsx` (11KB) -- NPS score display, history trends, weekly pulse, recent feedback, message helpfulness ratio
- **Services:** `api.ts` -- `getMockSatisfactionData()` with NPS history, weekly pulse, feedback, and message helpfulness metrics
- **Note:** UI fully functional; data collection uses mock source

---

## Code Map

| App | Directory | Key Files |
|---|---|---|
| **Patient Portal Native** | `apps/patient-portal-native/` | `app/(tabs)/chat.tsx`, `app/(tabs)/medications.tsx`, `app/(tabs)/health.tsx`, `app/(tabs)/index.tsx`, `app/(tabs)/more.tsx` |
| | | `app/appointments.tsx`, `app/lifestyle.tsx`, `app/education.tsx`, `app/caregiver.tsx`, `app/analytics.tsx`, `app/timeline.tsx`, `app/satisfaction.tsx`, `app/escalations.tsx`, `app/checkin.tsx`, `app/watch.tsx`, `app/insights.tsx`, `app/records.tsx`, `app/profile.tsx` |
| **AI Service** | `apps/ai-service/src/` | `index.ts` (API routes), `llm.ts` (LLM client), `prompts.ts` (system prompts), `db.ts` (SSO DB), `openemr-db.ts` (OpenEMR sync) |
| **AI Frontend** | `apps/ai-frontend/src/` | `App.tsx` (SPA), `services.ts` (auth), `pages/` (admin UI), `components/` |
| **Services** | `apps/patient-portal-native/lib/services/` | `chat-service.ts`, `feature-services.ts`, `health-kit-service.ts`, `medication-service.ts`, `nudge-service.ts`, `pro-service.ts`, `records-service.ts`, `vitals-service.ts` |
