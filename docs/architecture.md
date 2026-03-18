# Architecture

Med-SEAL Suite is a multi-layer healthcare platform where each layer handles a distinct concern. All services communicate over a shared Docker network (`medseal-net`).

## System Layers

```{uml}
@startuml
skinparam componentStyle uml2

package "Patient-Facing Layer" as PatientLayer {
  component "Patient Portal Native (Expo / React Native)\n//iOS & Android - vitals, meds, appointments, chat//" as PortalApp
}

package "AI & Services Layer" as AILayer {
  component "AI Service (Node/TS)\n//6 agents · 18 features//\n//LLM orchestration//\n//Port 4003//" as AIService
  component "SSO Service\n//Unified auth//\n//OpenEMR sync//\n//Postgres-backed//" as SSO
}

package "Data & Interoperability Layer" as DataLayer {
  component "Medplum (FHIR R4 API)\n//FHIR store · Subscriptions · Terminology//\n//API: port 8103 · Admin UI: port 3000//" as Medplum
}

package "Clinical EMR Layer" as ClinicalLayer {
  component "OpenEMR v7.0.2\n//Patient records · Orders · Billing · Scheduling//\n//Web UI: port 8081 (HTTP) / 8080 (HTTPS)//" as OpenEMR
}

PatientLayer ..> AILayer : FHIR R4 / REST
AILayer ..> DataLayer : FHIR R4 / SQL
DataLayer ..> ClinicalLayer : HL7 / SQL
@enduml
```

## Data Flow

1. **Clinicians** interact with **OpenEMR** for clinical documentation, orders, and scheduling.
2. **Medplum** serves as the FHIR R4 data backbone  - syncing clinical data from OpenEMR and exposing it via standard FHIR APIs.
3. The **AI Service** reads FHIR data from Medplum and connects to the **LLM** (med-r1 model) for clinical reasoning, nudges, and patient insights.
4. **SSO** provides unified authentication across all services, with user data synced to OpenEMR.
5. The **Patient Portal Native** app connects to both the AI Service (for chat, nudges, and features) and Medplum (for FHIR data) to deliver a comprehensive patient experience.

## Technology Stack

| Layer | Technology | Standards |
|---|---|---|
| Clinical EMR | OpenEMR 7.0.2 | ICD-10, SNOMED CT, HL7 v2 |
| FHIR API | Medplum Server + App | HL7 FHIR R4 |
| AI Backend | Node.js / TypeScript | REST, WebSocket |
| AI Models | med-r1 (via custom LLM API) |  - |
| Patient App | Expo / React Native |  - |
| Auth | PostgreSQL + custom SSO |  - |
| Infra | Docker Compose |  - |

## Network Topology

All services run on the `medseal-net` Docker bridge network. External access is via mapped ports:

| Port | Service |
|---|---|
| `3000` | Medplum Admin App |
| `4003` | AI Service API |
| `5433` | Medplum PostgreSQL |
| `5434` | SSO PostgreSQL |
| `8080` | OpenEMR (HTTPS) |
| `8081` | OpenEMR (HTTP) |
| `8103` | Medplum FHIR API |
