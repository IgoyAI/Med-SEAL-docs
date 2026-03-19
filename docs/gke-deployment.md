# GKE Cloud Deployment

Med-SEAL Suite is deployed to **Google Kubernetes Engine (GKE) Autopilot** in the `asia-southeast1` region. All infrastructure is provisioned via shell scripts and Kubernetes manifests located in the `gcp/` directory of the main repository.

---

## Infrastructure Overview

```{uml}
@startuml
skinparam componentStyle uml2

cloud "Google Cloud Platform" as GCP {

  node "GKE Autopilot Cluster\n(medseal-cluster)" as K8s {

    package "Namespace: medseal" {
      component "AI Service\n:4003" as AISvc
      component "AI Frontend\n:80" as AIUI
      component "OpenEMR\n:80" as OE
      component "Medplum Server\n:8103" as MedSrv
      component "Medplum App\n:3000" as MedApp
      component "Orthanc PACS\n:8042" as PACS
      component "OHIF Viewer\n:80" as OHIF
    }
  }

  database "Cloud SQL MySQL 8.0\n(medseal-openemr-db)" as MySQL
  database "Cloud SQL Postgres 16\n(medseal-pg-db)" as PG
  component "Memorystore Redis\n(medseal-redis)" as Redis
  storage "Filestore NFS\n(orthanc_data)" as NFS
  storage "Cloud Storage\n(medplum-binary)" as GCS

  component "GCE Ingress\nLoad Balancer\n(34.54.226.15)" as LB
}

actor "Users" as Users

Users --> LB : HTTPS
LB --> AISvc
LB --> AIUI
LB --> OE
LB --> MedSrv
LB --> MedApp
LB --> PACS
LB --> OHIF

OE --> MySQL : Private IP
MedSrv --> PG : Private IP
MedSrv --> Redis : Private IP
AISvc --> PG : SSO DB
PACS --> NFS : DICOM data
MedSrv --> GCS : Binary storage
@enduml
```

---

## Deployment Steps

### Step 1: Bootstrap Infrastructure

Creates the VPC, subnets, GKE Autopilot cluster, and Artifact Registry.

```bash
cd gcp
./setup.sh <PROJECT_ID> asia-southeast1
```

**Resources created:**
- VPC `medseal-vpc` with subnet `10.0.0.0/16`
- GKE Autopilot cluster `medseal-cluster`
- Artifact Registry `medseal` (Docker)

### Step 2: Configure Secrets

Stores passwords and API keys securely in Google Secret Manager.

```bash
./secrets.sh
```

**Secrets managed:**
| Secret | Purpose |
|---|---|
| `openemr-db-pass` | OpenEMR MySQL password |
| `medplum-db-pass` | Medplum PostgreSQL password |
| `sso-db-pass` | SSO PostgreSQL password |
| `orthanc-auth` | Orthanc PACS auth credentials |
| `llm-api-key` | LLM API key for AI agents |

### Step 3: Provision Databases

Creates all managed database services.

```bash
./databases.sh <PROJECT_ID> asia-southeast1
```

| Service | Type | Instance |
|---|---|---|
| OpenEMR | Cloud SQL MySQL 8.0 | `medseal-openemr-db` |
| Medplum + SSO | Cloud SQL PostgreSQL 16 | `medseal-pg-db` |
| Cache | Memorystore Redis | `medseal-redis` |
| DICOM Storage | Filestore NFS (1 TB) | `medseal-orthanc-nfs` |
| Binary Storage | Cloud Storage | `<project>-medplum-binary` |
| Audit Data | Cloud Storage | `<project>-audit-data` |

```{warning}
After running `databases.sh`, copy the **private IP addresses** printed by the script and update the Kubernetes manifests in `k8s/` before deploying.
```

### Step 4: Build and Push Custom Images

```bash
./push-images.sh <PROJECT_ID> asia-southeast1
```

Builds `linux/amd64` images for **ai-service** and **ai-frontend**, then pushes them to Artifact Registry.

### Step 5: Deploy to Kubernetes

```bash
gcloud container clusters get-credentials medseal-cluster --region asia-southeast1

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/ -n medseal

# Watch pods come up
kubectl get pods -n medseal -w
```

### Step 6: Verify Ingress

```bash
kubectl get ingress medseal-ingress -n medseal
```

The GCE Ingress controller provisions a Google Cloud Load Balancer exposing all services via `nip.io` subdomains.

---

## Kubernetes Manifests

All manifests are in `gcp/k8s/`:

| File | Resources |
|---|---|
| `namespace.yaml` | Namespace `medseal` |
| `openemr.yaml` | OpenEMR Deployment + Service + ConfigMap |
| `medplum.yaml` | Medplum Server + App Deployments + Services |
| `ai-service.yaml` | AI Service Deployment + Service |
| `ai-frontend.yaml` | AI Frontend Deployment + Service |
| `orthanc.yaml` | Orthanc PACS Deployment + Service |
| `ohif.yaml` | OHIF Viewer Deployment + Service |
| `sync-service.yaml` | Data Sync CronJob |
| `ingress.yaml` | GCE Ingress with 7 host rules |

---

## Ingress Routing

The Ingress controller maps subdomains to internal services:

| Subdomain | Service | Port |
|---|---|---|
| `app.medseal.34.54.226.15.nip.io` | ai-frontend | 80 |
| `api.medseal.34.54.226.15.nip.io` | ai-service | 4003 |
| `emr.medseal.34.54.226.15.nip.io` | openemr | 80 |
| `fhir.medseal.34.54.226.15.nip.io` | medplum-server | 8103 |
| `medplum.medseal.34.54.226.15.nip.io` | medplum-app | 3000 |
| `pacs.medseal.34.54.226.15.nip.io` | orthanc-proxy | 80 |
| `viewer.medseal.34.54.226.15.nip.io` | ohif-viewer | 80 |

---

## GKE Architecture Diagram

```{uml}
@startuml
skinparam componentStyle uml2

rectangle "medseal-vpc (10.0.0.0/16)" {

  rectangle "medseal-subnet\npods: 10.1.0.0/16\nservices: 10.2.0.0/16" {
    node "GKE Autopilot" {
      component "Pod: openemr" as OE
      component "Pod: medplum-server" as MedSrv
      component "Pod: medplum-app" as MedApp
      component "Pod: ai-service" as AISvc
      component "Pod: ai-frontend" as AIUI
      component "Pod: orthanc" as PACS
      component "Pod: ohif-viewer" as OHIF
      component "Pod: sync-service" as Sync
    }
  }

  rectangle "Private Services Access" {
    database "Cloud SQL MySQL\n10.195.0.3" as MySQL
    database "Cloud SQL Postgres\n10.195.0.5" as PG
    database "Memorystore Redis\n10.195.1.3" as Redis
    storage "Filestore NFS" as NFS
  }
}

rectangle "Internet" {
  component "GCE Load Balancer\n34.54.226.15" as LB
}

LB --> OE
LB --> MedSrv
LB --> MedApp
LB --> AISvc
LB --> AIUI
LB --> PACS
LB --> OHIF

OE --> MySQL
MedSrv --> PG
MedSrv --> Redis
AISvc --> PG : SSO
AISvc --> MySQL : OpenEMR sync
PACS --> NFS

Sync --> MedSrv
Sync --> MySQL
@enduml
```
