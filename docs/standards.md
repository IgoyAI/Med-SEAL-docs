# Standards Compliance

Med-SEAL Suite is built on international healthcare interoperability standards to ensure data portability, clinical accuracy, and regulatory alignment.

## Standards Matrix

| Standard | Version | Component | Usage |
|---|---|---|---|
| **HL7 FHIR** | R4 (4.0.1) | Medplum | Primary data format for all clinical resources |
| **DICOM** | 3.0 | Orthanc | Medical imaging storage and transfer |
| **DICOMweb** |  - | Orthanc REST API | Web-based access to imaging data |
| **HL7 v2** | 2.x | OpenMRS / OpenEMR | Legacy messaging (ADT, ORU) |
| **ICD-10** | 2024 | OpenEMR | Diagnosis coding |
| **SNOMED CT** | International | OpenEMR, Medplum | Clinical terminology |
| **LOINC** | 2.77 | Medplum, AI Service | Lab and vital sign coding |
| **RxNorm** |  - | Medplum, AI Service | Medication terminology + drug interaction lookup |
| **CPT** | 2024 | OpenEMR | Procedure coding |
| **NDF-RT** |  - | AI Service (A4) | Drug-food interaction database |

## FHIR R4 Compliance

Med-SEAL uses FHIR R4 as its primary interoperability standard. All data exchange between services uses FHIR resources serialised as JSON.

### Capabilities

- **CRUD**  - create, read, update, delete on all supported resource types
- **Search**  - parameterised searches with chaining and includes
- **Transactions**  - atomic batch operations via Bundle resources
- **Subscriptions**  - event-driven triggers for real-time workflows
- **Terminology**  - `$validate-code`, `$translate`, `$lookup` operations
- **Validation**  - `$validate` against profiles

### SMART on FHIR

Agent access control follows SMART on FHIR scoping:

| Agent | Scope |
|---|---|
| A1 (Companion) | `patient/Patient.read`, `patient/Observation.read`, `patient/Communication.write` |
| A2 (Clinical Reasoning) | `patient/Condition.read`, `patient/MedicationRequest.read`, `patient/Observation.read` |
| A3 (Nudge) | `patient/Communication.write`, `patient/Flag.write`, `patient/CommunicationRequest.write` |
| A5 (Insight Synthesis) | `patient/*.read`, `patient/Composition.write` |
| A6 (Measurement) | `patient/Observation.read`, `patient/MeasureReport.write` |

## Terminology Bindings

Key LOINC codes used:

| Measurement | LOINC Code |
|---|---|
| Systolic blood pressure | 8480-6 |
| Diastolic blood pressure | 8462-4 |
| Blood glucose | 2345-7 |
| Heart rate | 8867-4 |
| Step count | 55423-8 |
| Body weight | 29463-7 |
| HbA1c | 4548-4 |

## Security Architecture

Network segmentation and trust boundary diagram for IT audit (ISO 27001, HIPAA). Shows data classification levels and access control checkpoints.

```{uml}
@startuml
skinparam componentStyle uml2

rectangle "Untrusted Zone\n(Public Internet)" #FDD {
  actor "Patient\n(Mobile)" as PatientActor
  actor "Clinician\n(Browser)" as ClinicianActor
  actor "Admin\n(Browser)" as AdminActor
}

rectangle "DMZ\n(Reverse Proxy / TLS Termination)" #FED {
  component "HTTPS Gateway\n(nginx / Caddy)" as Gateway
}

rectangle "Application Zone\n(Docker: medseal-net)" #DFD {

  rectangle "Services" {
    component "Patient Portal\nNative" as Portal
    component "AI Service\n:4003" as AISvc
    component "AI Frontend\n:3001" as AIUI
    component "OpenEMR\n:8081" as OE
    component "Medplum App\n:3000" as MedApp
  }

  rectangle "AI Safety" {
    component "SEA-LION Guard" as Guard
    component "LLM (med-r1)" as LLM
  }

  rectangle "Data Zone\n<<Confidential / Restricted>>" #EEF {
    component "Medplum Server\n:8103" as MedSrv
    database "Medplum DB\n<<PHI>>" as MedDB
    database "OpenEMR DB\n<<PHI>>" as OEDB
    database "SSO DB\n<<PII>>" as SSODB
    database "Audit Logs\n<<Internal>>" as AuditDB
  }
}

PatientActor --> Gateway : HTTPS (TLS 1.3)
ClinicianActor --> Gateway : HTTPS (TLS 1.3)
AdminActor --> Gateway : HTTPS (TLS 1.3)

Gateway --> Portal
Gateway --> OE
Gateway --> AIUI
Gateway --> MedApp
Gateway --> AISvc

AISvc --> Guard : All I/O
AISvc --> LLM : De-identified prompts
AISvc --> MedSrv : FHIR R4
AISvc --> SSODB : Auth
AISvc --> AuditDB : AuditEvent write

Guard ..> AuditDB : PII redaction log

MedSrv --> MedDB
OE --> OEDB
AISvc --> OEDB : User sync

note bottom of MedDB
  **Data Classification: RESTRICTED**
  Contains all PHI (Patient, Condition,
  Observation, MedicationRequest, etc.)
  Encrypted at rest. Access via FHIR
  scopes only.
end note

note bottom of SSODB
  **Data Classification: CONFIDENTIAL**
  Contains PII (usernames, emails,
  password hashes). Encrypted at rest.
end note

note bottom of AuditDB
  **Data Classification: INTERNAL**
  Immutable audit trail.
  Retained 7 years per policy.
end note
@enduml
```

