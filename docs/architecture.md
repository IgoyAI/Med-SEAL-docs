# Architecture

Med-SEAL Suite is a multi-layer healthcare platform where each layer handles a distinct concern. All services communicate over a shared Docker network (`medseal-net`).

## System Layers

```{mermaid}
flowchart TD
    subgraph PatientLayer [Patient-Facing Layer]
        PortalApp["Patient Portal Native (Expo / React Native)<br/>iOS & Android - vitals, meds, appointments, chat"]
    end

    subgraph AILayer [AI & Services Layer]
        direction LR
        AIService["AI Service (Node/TS)<br/>6 agents · 18 features<br/>LLM orchestration<br/>Port 4003"]
        SSO["SSO Service<br/>Unified auth<br/>OpenEMR sync<br/>Postgres-backed"]
    end

    subgraph DataLayer [Data & Interoperability Layer]
        Medplum["Medplum (FHIR R4 API)<br/>FHIR store · Subscriptions · Terminology<br/>API: port 8103 · Admin UI: port 3000"]
    end

    subgraph ClinicalLayer [Clinical EMR Layer]
        OpenEMR["OpenEMR v7.0.2<br/>Patient records · Orders · Billing · Scheduling<br/>Web UI: port 8081 (HTTP) / 8080 (HTTPS)"]
    end

    PatientLayer -- "FHIR R4 / REST" --> AILayer
    AILayer -- "FHIR R4 / SQL" --> DataLayer
    DataLayer -- "HL7 / SQL" --> ClinicalLayer

    classDef layerBox fill:transparent,stroke:#666,stroke-width:2px,stroke-dasharray: 5 5;
    class PatientLayer,AILayer,DataLayer,ClinicalLayer layerBox;
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
