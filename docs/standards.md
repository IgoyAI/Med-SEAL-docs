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
