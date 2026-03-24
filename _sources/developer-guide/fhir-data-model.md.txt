# FHIR Data Model

The Med-SEAL Suite uses **HL7 FHIR R4** as its canonical data standard. All clinical data is stored in Medplum's FHIR store and accessed through its REST API. This page explains the key resources, naming conventions, and read/write patterns used across the platform.

---

## Key FHIR Resources

| Resource | Used For |
|---|---|
| `Patient` | Patient demographics and identifiers |
| `Practitioner` | Clinician identity |
| `Encounter` | Clinical visits and interactions |
| `Condition` | Diagnoses and problem list entries |
| `MedicationRequest` | Prescriptions |
| `MedicationAdministration` | Dose confirmations (from app) |
| `Observation` | Vitals, lab results, biometric readings |
| `Appointment` | Scheduled visits |
| `AllergyIntolerance` | Documented allergies |
| `Immunization` | Vaccination records |
| `Procedure` | Procedures performed during encounters |
| `Composition` | Structured clinical documents (e.g. pre-visit briefs) |
| `MeasureReport` | Population and individual adherence metrics |
| `QuestionnaireResponse` | PRO (Patient-Reported Outcome) submissions |

### FHIR Domain Model (Class Diagram)

```{uml}
@startuml
skinparam classAttributeIconSize 0
hide circle

class Patient {
  id : string
  name : HumanName[]
  birthDate : date
  gender : code
  telecom : ContactPoint[]
  --
  <<PHI - Restricted>>
}

class Practitioner {
  id : string
  name : HumanName[]
  qualification : code[]
}

class Encounter {
  id : string
  status : code
  class : Coding
  period : Period
  reasonCode : CodeableConcept[]
}

class Condition {
  id : string
  clinicalStatus : code
  code : CodeableConcept
  onsetDateTime : dateTime
}

class MedicationRequest {
  id : string
  status : code
  medicationCodeableConcept : CodeableConcept
  dosageInstruction : Dosage[]
}

class MedicationAdministration {
  id : string
  status : code
  effectiveDateTime : dateTime
}

class Observation {
  id : string
  status : code
  category : CodeableConcept[]
  code : CodeableConcept
  valueQuantity : Quantity
  effectiveDateTime : dateTime
}

class Appointment {
  id : string
  status : code
  start : instant
  end : instant
  appointmentType : CodeableConcept
}

class Composition {
  id : string
  status : code
  type : CodeableConcept
  date : dateTime
  section : Section[]
  --
  <<AI-generated tag>>
}

class MeasureReport {
  id : string
  status : code
  type : code
  period : Period
  group : Group[]
}

Patient "1" --> "0..*" Encounter : subject
Patient "1" --> "0..*" Condition : subject
Patient "1" --> "0..*" MedicationRequest : subject
Patient "1" --> "0..*" Observation : subject
Patient "1" --> "0..*" Appointment : participant
Patient "1" --> "0..*" Composition : subject

Encounter "1" --> "0..*" Condition : context
Encounter "1" --> "0..*" Observation : encounter
Encounter "1" --> "1" Practitioner : participant

MedicationRequest "1" --> "0..*" MedicationAdministration : request
MedicationRequest "1" --> "1" Practitioner : requester

Composition "1" --> "0..*" MeasureReport : references

Appointment "0..*" --> "1" Practitioner : participant
@enduml
```

---

## Reading FHIR Data

All reads go through the Medplum FHIR API. The base URL is configured via `MEDPLUM_BASE_URL`.

### Using the Medplum TypeScript SDK

```typescript
import { MedplumClient } from '@medplum/core';

const medplum = new MedplumClient({
  baseUrl: process.env.MEDPLUM_BASE_URL,
});

await medplum.startClientLogin(
  process.env.MEDPLUM_CLIENT_ID!,
  process.env.MEDPLUM_CLIENT_SECRET!
);

// Search for a patient by ID
const patient = await medplum.readResource('Patient', patientId);

// Search for active medications
const bundle = await medplum.search('MedicationRequest', {
  patient: `Patient/${patientId}`,
  status: 'active',
});
```

### Raw FHIR REST

```bash
# Get a patient
GET /fhir/R4/Patient/{id}

# Search for observations (vitals) for a patient
GET /fhir/R4/Observation?patient=Patient/{id}&category=vital-signs

# Search for active medications
GET /fhir/R4/MedicationRequest?patient=Patient/{id}&status=active
```

All requests require a Bearer token obtained via the Medplum OAuth2 flow.

---

## Writing FHIR Data

### Creating a Resource

```typescript
const observation = await medplum.createResource({
  resourceType: 'Observation',
  status: 'final',
  category: [
    {
      coding: [
        {
          system: 'http://terminology.hl7.org/CodeSystem/observation-category',
          code: 'vital-signs',
        },
      ],
    },
  ],
  code: {
    coding: [
      {
        system: 'http://loinc.org',
        code: '55284-4',  // Blood pressure
        display: 'Blood pressure systolic and diastolic',
      },
    ],
  },
  subject: { reference: `Patient/${patientId}` },
  effectiveDateTime: new Date().toISOString(),
  component: [
    {
      code: { coding: [{ system: 'http://loinc.org', code: '8480-6', display: 'Systolic BP' }] },
      valueQuantity: { value: 120, unit: 'mmHg', system: 'http://unitsofmeasure.org', code: 'mm[Hg]' },
    },
    {
      code: { coding: [{ system: 'http://loinc.org', code: '8462-4', display: 'Diastolic BP' }] },
      valueQuantity: { value: 80, unit: 'mmHg', system: 'http://unitsofmeasure.org', code: 'mm[Hg]' },
    },
  ],
});
```

### Updating a Resource

```typescript
await medplum.updateResource({
  ...existingResource,
  status: 'completed',
});
```

---

## Coding Systems

Always use the correct terminology system for coded fields:

| Concept | System | Example code |
|---|---|---|
| Vital sign types | `http://loinc.org` | `8867-4` (Heart rate) |
| Diagnoses | `http://snomed.info/sct` or ICD-10 | `44054006` (Diabetes) |
| Medications | RXNORM or NZulm | `860975` (Metformin 500mg) |
| Observation categories | FHIR category CS | `vital-signs`, `laboratory` |
| Units | UCUM (`http://unitsofmeasure.org`) | `kg`, `mm[Hg]`, `/min` |

---

## Med-SEAL Conventions

- **Patient reference format**: always `Patient/{id}` (never bare ID strings).
- **Timestamps**: always ISO 8601 with timezone (`2026-03-19T08:00:00+08:00`).
- **Dose confirmations**: use `MedicationAdministration` with `status: 'completed'` and `request` pointing to the originating `MedicationRequest`.
- **AI-generated content**: tag `Composition` and `Observation` resources with the extension `http://medseal.io/fhir/StructureDefinition/ai-generated: true` so downstream systems can distinguish AI output from clinician-entered data.

---

## FHIR Subscriptions

The AI Service subscribes to Medplum FHIR Subscriptions for real-time event processing:

| Subscription | Trigger | Handler |
|---|---|---|
| New `Observation` (vital) | Patient logs a reading | Threshold check, nudge evaluation |
| New `QuestionnaireResponse` | PRO submission | Score computation, escalation |
| Updated `Appointment` | Status change | Pre-visit brief generation |

Subscriptions are defined in `apps/ai-service/src/subscriptions/`.
