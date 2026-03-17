# FHIR Resource Inventory

Med-SEAL Suite uses **32 distinct FHIR R4 resource types** across its 18 features. This page lists every resource type, grouped by category, with its usage context.

## Patient Identity

| Resource | Usage |
|---|---|
| `Patient` | Core patient demographics (name, DOB, gender, language, contact info) |
| `RelatedPerson` | Caregiver / family member linked to patient (for Caregiver Mode F13) |
| `Consent` | Consent grants for caregiver data access (scope, provision, expiry) |

## Clinical Data (Read from EMR)

| Resource | Usage |
|---|---|
| `Condition` | Active diagnoses (diabetes, hypertension, etc.)  - used by A2, A3, A4 for clinical context |
| `Observation` | Vitals, lab results, PRO scores, adherence metrics, computed averages |
| `MedicationRequest` | Active prescriptions with dosage instructions  - source for medication schedules |
| `AllergyIntolerance` | Patient allergies and adverse reactions |
| `Encounter` | Clinical visits (inpatient, outpatient, ED)  - used for readmission tracking |
| `Procedure` | Clinical procedures performed |
| `CarePlan` | Active care plans with goals and activities |
| `CareTeam` | Patient's assigned care team (practitioners, roles) |
| `Immunization` | Vaccination records |

## Medication Tracking (Write)

| Resource | Usage |
|---|---|
| `MedicationAdministration` | Dose confirmations (taken/skipped/delayed) from patient app |
| `MedicationDispense` | Dispensed medication records |

## Communication

| Resource | Usage |
|---|---|
| `Communication` | Chat messages, nudge deliveries, education content, audit records |
| `CommunicationRequest` | Scheduled nudges, clinician notifications, follow-up reminders |

## Assessments (Write)

| Resource | Usage |
|---|---|
| `Questionnaire` | PRO instrument templates (PHQ-9, DDS-17, NPS) |
| `QuestionnaireResponse` | Patient responses to PRO questionnaires |
| `RiskAssessment` | Behavioral anticipation scores, readmission risk predictions |
| `DetectedIssue` | Escalation triggers (medication interactions, threshold breaches) |

## Clinical Output (Write)

| Resource | Usage |
|---|---|
| `Composition` | Pre-visit patient insight summaries (7-section clinical briefs) |
| `DiagnosticReport` | AI-generated radiology and clinical reports |
| `Flag` | Active clinical alerts (tiered escalation: low/medium/high) |
| `NutritionOrder` | Dietary recommendations with specific nutrient targets |

## Goals & Planning

| Resource | Usage |
|---|---|
| `Goal` | Patient health goals (weight, glucose, dietary targets) with progress tracking |
| `Task` | Follow-up actions from encounters, system-generated todos |

## Measurement

| Resource | Usage |
|---|---|
| `Measure` | Metric definitions (adherence PDC, biometric improvement, engagement) |
| `MeasureReport` | Computed metric results (individual + population summary) |

## Devices & Wearables

| Resource | Usage |
|---|---|
| `Device` | Registered wearable devices, agent identities (one Device per agent) |
| `DeviceMetric` | Device-specific measurement metadata |

## Appointments

| Resource | Usage |
|---|---|
| `Appointment` | Upcoming and past clinical appointments |

## Audit & Provenance

| Resource | Usage |
|---|---|
| `AuditEvent` | Guard decisions, caregiver access logs, agent action audit trail |
| `Provenance` | Source tracking for AI-generated content (which agent, which source data) |

## Education

| Resource | Usage |
|---|---|
| `DocumentReference` | Health education content library, condition-specific materials |

## System

| Resource | Usage |
|---|---|
| `Device` (agent) | One Device resource per AI agent for identity and scope management |

---

**Total: 32 resource types across 18 features.**
