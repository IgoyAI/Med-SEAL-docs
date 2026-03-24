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
| AI Backend | Python 3.11 / FastAPI / LangGraph | REST, SSE |
| AI Models | SEA-LION v4-32B (AI Singapore) |  - |
| Patient App | Expo / React Native |  - |
| Auth | PostgreSQL + custom SSO |  - |
| Infra | GCP Cloud Run + GKE |  - |

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

## Deployment Diagram

Infrastructure asset register showing all containers, databases, and volume mounts running on the Docker host.

```{uml}
@startuml
skinparam componentStyle uml2

node "Docker Host" as Host {

  node "medseal-net (bridge network)" as Net {

    package "Patient-Facing" {
      artifact "Patient Portal Native" as App <<Expo/RN>>
    }

    package "AI & Services" {
      component "ai-service\n:4003" as AISvc <<Node.js>>
      component "ai-frontend\n:3001" as AIUI <<Vite/React>>
    }

    package "Data & Interoperability" {
      component "medplum-server\n:8103" as MedSrv <<Java>>
      component "medplum-app\n:3000" as MedApp <<React>>
      database "medplum-db\n:5433" as MedDB <<PostgreSQL 16>>
      component "medplum-redis\n:6380" as Redis <<Redis 7>>
    }

    package "Clinical EMR" {
      component "openemr\n:8081/:8080" as OE <<PHP>>
      database "openemr-db\n:3307" as OEDB <<MariaDB 10.11>>
    }

    package "Auth" {
      database "sso-db\n:5434" as SSODB <<PostgreSQL 16>>
    }
  }

  storage "medplum-db-data" as MedVol
  storage "openemr-db-data" as OEVol
  storage "sso-db-data" as SSOVol
  storage "audit-data" as AuditVol
}

MedDB --> MedVol
OEDB --> OEVol
SSODB --> SSOVol
AISvc --> AuditVol

App ..> AISvc : REST / HTTPS
App ..> MedSrv : FHIR R4
AISvc --> MedSrv : FHIR R4
AISvc --> SSODB : SQL
AISvc --> OEDB : SQL (sync)
MedSrv --> MedDB : SQL
MedSrv --> Redis : Cache
OE --> OEDB : SQL
@enduml
```

## Data Flow Diagram

Shows how Protected Health Information (PHI) flows through the system and which data crosses trust boundaries. Required for HIPAA/ISO 27001 data mapping.

```{uml}
@startuml
skinparam actorStyle awesome

actor "Patient" as Patient
actor "Clinician" as Clinician
actor "Admin" as Admin

rectangle "External Trust Boundary" #line.dashed {

  rectangle "Internal Trust Boundary" #line.dashed {

    rectangle "AI & Services Zone" {
      entity "AI Service" as AISvc
      entity "SEA-LION Guard" as Guard
      entity "LLM (med-r1)" as LLM
      entity "SSO" as SSO
    }

    rectangle "Data Zone" {
      database "Medplum\n(FHIR R4)" as FHIR
      database "SSO DB" as SSODB
    }

    rectangle "Clinical Zone" {
      entity "OpenEMR" as OE
      database "OpenEMR DB" as OEDB
    }
  }
}

Patient --> AISvc : Chat, vitals,\nmed confirmations\n(PHI)
Patient --> FHIR : FHIR reads\n(PHI)
Clinician --> OE : Clinical docs,\norders, Rx\n(PHI)
Admin --> SSO : User CRUD\n(PII)

AISvc --> Guard : All input/output\n(PHI filtered)
AISvc --> LLM : Prompts\n(de-identified)
AISvc --> FHIR : FHIR read/write\n(PHI)
AISvc --> SSODB : Auth queries\n(PII)

FHIR <--> OE : Sync\n(PHI)
SSO --> OEDB : User sync\n(PII)

note right of Guard
  PII redaction before LLM
  Hallucination check on output
  Clinical harm filter
end note
@enduml
```

