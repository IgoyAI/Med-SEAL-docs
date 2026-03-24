# GCP Deployment Guide

This guide covers deploying the Med-SEAL AI Agent system to **Google Cloud Platform** using **Cloud Run** (serverless) alongside the existing **GKE** infrastructure.

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                    GCP Project: gen-lang-client-0538005727      │
│                                                                │
│  ┌──────────────────────┐    ┌───────────────────────────────┐ │
│  │   Cloud Run           │    │   GKE (medseal-cluster)       │ │
│  │   (AI Agent Service)  │    │   Region: asia-southeast1     │ │
│  │                       │    │                               │ │
│  │   medseal-agent       │◄──►│   ai-frontend     (1 pod)    │ │
│  │   - 7 LangGraph agents│    │   ai-service      (1 pod)    │ │
│  │   - FastAPI + Uvicorn │    │   medplum-server   (1 pod)    │ │
│  │   - SEA-Guard safety  │    │   medplum-app      (1 pod)    │ │
│  │                       │    │   openemr           (1 pod)    │ │
│  │   Port: 8000          │    │   sync-service     (1 pod)    │ │
│  └──────────────────────┘    └───────────────────────────────┘ │
│           │                              │                     │
│           ▼                              ▼                     │
│  ┌──────────────────┐          ┌───────────────────┐           │
│  │ Artifact Registry │          │ Ingress: 34.54.226.15        │
│  │ (Docker images)   │          │  app.medseal.*.nip.io        │
│  └──────────────────┘          │  fhir.medseal.*.nip.io       │
│                                │  emr.medseal.*.nip.io        │
│                                │  api.medseal.*.nip.io        │
│                                └───────────────────┘           │
└────────────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
  ┌─────────────┐     ┌──────────────┐
  │ SEA-LION API │     │ Azure OpenAI │
  │ (5 agents +  │     │ (Clinical A2)│
  │  SEA-Guard)  │     │ GPT-5.3      │
  └─────────────┘     └──────────────┘
```

## Cloud Run Deployment

### Prerequisites

- Google Cloud SDK (`gcloud`) installed and authenticated
- GCP project with billing enabled
- Required APIs: Cloud Run, Cloud Build, Artifact Registry

### Step 1: Authenticate & Set Project

```bash
export PATH="$HOME/google-cloud-sdk/bin:$PATH"
gcloud auth login --no-launch-browser
gcloud config set project gen-lang-client-0538005727
```

### Step 2: Enable Required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

### Step 3: Deploy from Source

This command uploads the source, builds the Docker image via Cloud Build, and deploys to Cloud Run in one step:

```bash
gcloud run deploy medseal-agent \
  --source /path/to/Med-SEAL \
  --region asia-southeast1 \
  --port 8000 \
  --memory 1Gi \
  --cpu 1 \
  --timeout 60 \
  --allow-unauthenticated \
  --set-env-vars="\
MEDSEAL_SEALION_API_KEY=<your-sealion-key>,\
MEDSEAL_AZURE_OPENAI_ENDPOINT=<your-azure-endpoint>,\
MEDSEAL_AZURE_OPENAI_API_KEY=<your-azure-key>,\
MEDSEAL_AZURE_OPENAI_DEPLOYMENT=gpt-5.3,\
MEDSEAL_MEDPLUM_URL=http://fhir.medseal.34.54.226.15.nip.io/fhir/R4" \
  --quiet
```

### Step 4: Verify Deployment

```bash
# Service URL
gcloud run services describe medseal-agent \
  --region asia-southeast1 \
  --format="value(status.url)"

# Health check
curl https://medseal-agent-74997794842.asia-southeast1.run.app/health

# List registered agents
curl https://medseal-agent-74997794842.asia-southeast1.run.app/agents
```

### Updating Environment Variables

```bash
gcloud run services update medseal-agent \
  --region asia-southeast1 \
  --update-env-vars="MEDSEAL_MEDPLUM_URL=<new-url>"
```

## Current Deployment

### Cloud Run Service

| Field | Value |
|-------|-------|
| **Service URL** | `https://medseal-agent-74997794842.asia-southeast1.run.app` |
| **Swagger UI** | `https://medseal-agent-74997794842.asia-southeast1.run.app/docs` |
| **Region** | `asia-southeast1` (Singapore) |
| **CPU** | 1 vCPU |
| **Memory** | 1 GiB |
| **Agents** | 7 (companion, clinical, doctor-cds, nudge, lifestyle, insight, previsit) |

### GKE Cluster (`medseal-cluster`)

| Field | Value |
|-------|-------|
| **Cluster** | `medseal-cluster` |
| **Region** | `asia-southeast1` |
| **Kubernetes** | v1.34.4-gke.1047000 |
| **Nodes** | 2 × `ek-standard-8` |
| **Master IP** | `34.124.151.168` |
| **Ingress IP** | `34.54.226.15` |

### GKE Services (namespace: `medseal`)

| Service | Host | Status |
|---------|------|--------|
| AI Frontend | `app.medseal.34.54.226.15.nip.io` | ✅ Running |
| AI Service | `api.medseal.34.54.226.15.nip.io` | ✅ Running |
| Medplum FHIR | `fhir.medseal.34.54.226.15.nip.io` | ✅ Running |
| Medplum App | `medplum.medseal.34.54.226.15.nip.io` | ✅ Running |
| OpenEMR | `emr.medseal.34.54.226.15.nip.io` | ✅ Running |
| OHIF Viewer | `viewer.medseal.34.54.226.15.nip.io` | Scaled to 0 |
| PACS/Orthanc | `pacs.medseal.34.54.226.15.nip.io` | Scaled to 0 |

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `MEDSEAL_SEALION_API_KEY` | SEA-LION API key (powers 5 agents + SEA-Guard) | ✅ |
| `MEDSEAL_SEALION_API_URL` | SEA-LION API endpoint | Default: `https://api.sea-lion.ai/v1` |
| `MEDSEAL_SEALION_MODEL` | SEA-LION model name | Default: `aisingapore/Qwen-SEA-LION-v4-32B-IT` |
| `MEDSEAL_SEAGUARD_MODEL` | SEA-Guard safety model | Default: `aisingapore/SEA-Guard` |
| `MEDSEAL_AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint URL | ✅ |
| `MEDSEAL_AZURE_OPENAI_API_KEY` | Azure OpenAI API key | ✅ |
| `MEDSEAL_AZURE_OPENAI_DEPLOYMENT` | Azure deployment name | Default: `gpt-5.3` |
| `MEDSEAL_CLINICAL_LLM_BACKEND` | `azure` or `vllm` | Default: `azure` |
| `MEDSEAL_MEDPLUM_URL` | Medplum FHIR R4 base URL | ✅ |
| `MEDSEAL_MEDPLUM_EMAIL` | Medplum admin email | Default: `admin@example.com` |
| `MEDSEAL_MEDPLUM_PASSWORD` | Medplum admin password | Default: `medplum_admin` |

## API Endpoints

### Patient App Surface

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/sessions` | Create a new chat session |
| `POST` | `/sessions/{id}/messages` | Send a message (sync) |
| `POST` | `/sessions/{id}/messages/stream` | Send a message (SSE stream) |
| `GET` | `/sessions/{id}/messages` | Conversation history |
| `DELETE` | `/sessions/{id}` | Delete session |
| `POST` | `/patients/{id}/previsit-summary` | Pre-visit summary |

### Clinician Surface (OpenEMR)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/openemr/sessions/{id}/chat` | Doctor CDS (SSE) |
| `POST` | `/openemr/sessions/{id}/chat/sync` | Doctor CDS (sync) |
| `POST` | `/cds-services/patient-view` | CDS Hooks |

### System & Admin

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/triggers/{type}` | System triggers |
| `GET` | `/agents` | List agents |
| `GET` | `/agents/{id}/health` | Agent health |
| `GET` | `/health` | System health |

## Troubleshooting

### Health Check Shows `medplum: unreachable`

The Cloud Run service cannot reach the Medplum FHIR server. Ensure:
- The GKE ingress IP (`34.54.226.15`) is accessible from Cloud Run
- The `MEDSEAL_MEDPLUM_URL` env var points to the correct FHIR endpoint

### Health Check Shows `vllm: unreachable`

This is **expected** when using Azure OpenAI as the clinical LLM backend. The vLLM endpoint is only used when `MEDSEAL_CLINICAL_LLM_BACKEND=vllm` (local Med-R1 GPU inference).

### Redeploying After Code Changes

```bash
# Re-deploy from source (rebuilds Docker image)
gcloud run deploy medseal-agent \
  --source /path/to/Med-SEAL \
  --region asia-southeast1 \
  --quiet
```

### Viewing Logs

```bash
# Cloud Run logs
gcloud run services logs read medseal-agent --region asia-southeast1 --limit 50

# GKE logs
kubectl logs -n medseal deployment/ai-service --tail=50
```
