# AI Agent — Implementation & GCP Cloud Run Deployment

This page documents the implementation architecture and **GCP Cloud Run** deployment of the **Med-SEAL AI Agent** — the multi-agent clinical reasoning service that powers the platform's 18 smart healthcare features.

---

## 1. Implementation Overview

### 1.1 Technology Stack

| Component | Technology |
|---|---|
| **Runtime** | Python 3.11 |
| **Web framework** | FastAPI + Uvicorn |
| **Agent framework** | LangGraph (LangChain) |
| **Clinical LLM** | SEA-LION v4-32B (AI Singapore) |
| **Conversation model** | SEA-LION v4 (AI Singapore) |
| **Safety guard** | SEA-Guard (AI Singapore) |
| **FHIR client** | Medplum (custom `httpx`-based client) |
| **Session persistence** | SQLite (via `langgraph-checkpoint-sqlite`) |
| **Containerisation** | Docker (`python:3.11-slim`) |
| **Cloud hosting** | GCP Cloud Run (Singapore, `asia-southeast1`) |
| **Infrastructure** | GKE (`medseal-cluster`) for full stack |

### 1.2 Agent Roster

The system builds and registers 7 LangGraph agent graphs at startup:

| Agent ID | Module | Purpose |
|---|---|---|
| `companion-agent` | `agent.agents.companion` | Patient-facing conversational interface |
| `clinical-reasoning-agent` | `agent.agents.clinical` | Clinical reasoning (drug interactions, lab interpretation) |
| `doctor-cds-agent` | `agent.agents.doctor_cds` | OpenEMR clinician chat & decision support |
| `nudge-agent` | `agent.agents.nudge` | Proactive reminders and escalation |
| `lifestyle-agent` | `agent.agents.lifestyle` | Dietary and lifestyle coaching |
| `insight-synthesis-agent` | `agent.agents.insight` | Pre-visit brief synthesis |
| `previsit-summary-agent` | `agent.agents.previsit` | FHIR-based pre-visit data aggregation |

Each agent is a compiled LangGraph `StateGraph` with its own tools, system prompt, and checkpointer.

### 1.3 LLM Backend — Dual Mode

The system supports two LLM backends via the **LLM Factory** (`agent/core/llm_factory.py`):

```python
# config.py — defaults to SEA-LION
clinical_llm_backend: str = "azure"  # or "vllm" for Med-SEAL V1
```

| Mode | Backend | Model | When |
|---|---|---|---|
| `azure` *(current)* | SEA-LION v4-32B via API | Qwen-SEA-LION-v4-32B-IT | Production — no GPU needed |
| `vllm` | Self-hosted vLLM | `med-r1` (Med-SEAL V1) | Future — requires 2× H200 GPU |

Switching is a single env var change: `MEDSEAL_CLINICAL_LLM_BACKEND=vllm`.

### 1.4 API Endpoints

| Method | Path | Surface | Description |
|---|---|---|---|
| `POST` | `/sessions` | Patient | Create new chat session |
| `POST` | `/sessions/{id}/messages` | Patient | Send message (sync) |
| `POST` | `/sessions/{id}/messages/stream` | Patient | Send message (SSE streaming) |
| `GET` | `/sessions/{id}/messages` | Patient | Get conversation history |
| `DELETE` | `/sessions/{id}` | Patient | Delete session |
| `POST` | `/patients/{id}/previsit-summary` | Clinician | Generate pre-visit summary |
| `POST` | `/openemr/sessions/{id}/chat` | Clinician | Doctor chat (SSE) |
| `POST` | `/openemr/sessions/{id}/chat/sync` | Clinician | Doctor chat (sync) |
| `POST` | `/cds-services/patient-view` | CDS Hooks | Trigger insight synthesis |
| `POST` | `/triggers/{type}` | System | Fire nudge/measurement triggers |
| `GET` | `/agents` | Admin | List registered agents |
| `GET` | `/agents/{id}/health` | Admin | Agent health check |
| `GET` | `/health` | Admin | System health check |

### 1.5 External Dependencies

| Service | Endpoint | Purpose |
|---|---|---|
| **SEA-LION API** | `api.sea-lion.ai/v1` | Clinical reasoning + Conversation + Guard |
| **Medplum FHIR** | `fhir.medseal.34.54.226.15.nip.io/fhir/R4` | Patient health records (GKE) |

---

## 2. Project Structure

```
agent/
├── main.py                      # FastAPI app, lifespan, agent graph compilation
├── config.py                    # Pydantic Settings (env vars, MEDSEAL_ prefix)
├── agents/                      # LangGraph agent definitions
│   ├── companion.py             #   A1 — patient conversation
│   ├── clinical.py              #   A2 — clinical reasoning
│   ├── doctor_cds.py            #   Doctor chat + CDS
│   ├── nudge.py                 #   A3 — proactive nudges
│   ├── lifestyle.py             #   A4 — dietary coaching
│   ├── insight.py               #   A5 — insight synthesis
│   ├── previsit.py              #   Pre-visit summary
│   └── measurement.py           #   A6 — analytics
├── api/
│   └── routes.py                # All FastAPI route handlers
├── core/
│   ├── orchestrator.py          # Intent classification + agent routing
│   ├── guard.py                 # SEA-LION Guard (input/output safety)
│   ├── llm_factory.py           # SEA-LION / vLLM backend selector
│   ├── graph.py                 # Legacy agent graph
│   ├── identity.py              # Agent identity definitions
│   ├── language.py              # Language detection
│   └── router.py                # Message routing logic
└── tools/
    ├── fhir_client.py           # Medplum FHIR client (httpx)
    ├── fhir_tools_clinical.py   # FHIR tools for A2
    ├── fhir_tools_companion.py  # FHIR tools for A1
    ├── fhir_tools_nudge.py      # FHIR tools for A3
    ├── fhir_tools_lifestyle.py  # FHIR tools for A4
    ├── fhir_tools_insight.py    # FHIR tools for A5
    ├── fhir_tools_previsit.py   # FHIR tools for pre-visit
    ├── fhir_tools_measurement.py # FHIR tools for A6
    ├── fhir_tools_appointment.py # Appointment management
    └── medical_tools.py         # Medical knowledge tools
```

---

## 3. GCP Cloud Run Deployment

### 3.1 GCP Resources

| Resource | Type | Configuration |
|---|---|---|
| **Project** | `gen-lang-client-0538005727` | Gemini Project1 |
| **Cloud Run Service** | `medseal-agent` | `asia-southeast1` (Singapore) |
| **GKE Cluster** | `medseal-cluster` | 2× `ek-standard-8` nodes |
| **Artifact Registry** | `cloud-run-source-deploy` | Docker images |
| **Ingress IP** | `34.54.226.15` | GKE external load balancer |

### 3.2 Environment Variables

```bash
# Clinical LLM (SEA-LION)
MEDSEAL_CLINICAL_LLM_BACKEND=azure
MEDSEAL_SEALION_API_URL=https://api.sea-lion.ai/v1
MEDSEAL_SEALION_API_KEY=<your-sea-lion-key>
MEDSEAL_SEALION_MODEL=aisingapore/Qwen-SEA-LION-v4-32B-IT
MEDSEAL_SEAGUARD_MODEL=aisingapore/SEA-Guard

# Medplum FHIR (GKE)
MEDSEAL_MEDPLUM_URL=http://fhir.medseal.34.54.226.15.nip.io/fhir/R4
MEDSEAL_MEDPLUM_EMAIL=admin@example.com
MEDSEAL_MEDPLUM_PASSWORD=medplum_admin

# App behaviour
MEDSEAL_MAX_RECURSION=5
MEDSEAL_TEMPERATURE=0.6
```

### 3.3 Startup Command

```bash
uvicorn agent.main:app --host 0.0.0.0 --port 8000
```

### 3.4 Deployment Steps

#### Prerequisites
- Google Cloud SDK (`gcloud`) installed and authenticated
- GCP project with billing enabled
- Access to SEA-LION API (AI Singapore)

#### Step 1: Authenticate & Configure Project

```bash
export PATH="$HOME/google-cloud-sdk/bin:$PATH"
gcloud auth login --no-launch-browser
gcloud config set project gen-lang-client-0538005727
```

#### Step 2: Enable Required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

#### Step 3: Deploy from Source

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
MEDSEAL_SEALION_API_KEY=<key>,\
MEDSEAL_MEDPLUM_URL=http://fhir.medseal.34.54.226.15.nip.io/fhir/R4" \
  --quiet
```

#### Step 4: Verify

```bash
# Health check
curl https://medseal-agent-74997794842.asia-southeast1.run.app/health

# List agents
curl https://medseal-agent-74997794842.asia-southeast1.run.app/agents
```

Expected health response:
```json
{
  "status": "ok",
  "vllm": "unreachable",
  "redis": "ok",
  "medplum": "ok",
  "agents": {}
}
```

> `vllm: unreachable` is expected — Med-SEAL V1 (`med-r1`) is not deployed. The system uses SEA-LION via API instead.

### 3.5 Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt agent/requirements_agent.txt ./
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install --no-cache-dir -r requirements_agent.txt

COPY agent /app/agent

ENV PYTHONPATH=/app
EXPOSE 8000

CMD ["uvicorn", "agent.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 3.6 Requirements (Deployment)

The slim deployment requirements (`requirements_deploy.txt`) include only runtime dependencies:

```
fastapi>=0.115
uvicorn[standard]
pydantic-settings
langchain>=0.3
langchain-openai
langchain-community
langgraph>=0.2.70
langgraph-checkpoint-redis
langdetect
ddgs
httpx
sqlalchemy
aiosqlite
```

Training dependencies (PyTorch, transformers, datasets, etc.) are excluded to keep the deployment package small.

---

## 4. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│              GCP Cloud Run (asia-southeast1)                 │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              FastAPI (uvicorn :8000)                    │ │
│  │                                                        │ │
│  │  /sessions/*          → Orchestrator → Agent Router    │ │
│  │  /openemr/sessions/*  → Orchestrator → Doctor CDS      │ │
│  │  /cds-services/*      → Orchestrator → Insight Agent   │ │
│  │  /triggers/*          → Orchestrator → Nudge Agent     │ │
│  │  /patients/*/previsit → Previsit Agent                 │ │
│  │                                                        │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │ │
│  │  │ Companion A1 │  │ Clinical A2  │  │ Doctor CDS  │  │ │
│  │  │ Lifestyle A4 │  │  Nudge A3    │  │ Insight A5  │  │ │
│  │  │ Previsit     │  │ Measurement  │  │   Guard     │  │ │
│  │  └──────────────┘  └──────────────┘  └─────────────┘  │ │
│  │                                                        │ │
│  │  SQLite (medseal_sessions.db) — session checkpointing  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────┬──────────────────┬──────────────────┬─────────────┘
          │                  │                  │
    ┌─────▼─────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │ SEA-LION   │    │  SEA-Guard  │    │   Medplum   │
    │ v4-32B     │    │ Safety LLM  │    │  FHIR R4    │
    │(clinical + │    │ api.sea-lion│    │ (GKE)       │
    │ companion) │    │ .ai/v1      │    │             │
    └────────────┘    └─────────────┘    └─────────────┘
```

---

## 5. Monitoring & Troubleshooting

### 5.1 Logs

```bash
# Cloud Run logs
gcloud run services logs read medseal-agent --region asia-southeast1 --limit 50

# GKE pod logs
kubectl logs -n medseal deployment/ai-service --tail=50
```

### 5.2 Common Issues

| Issue | Cause | Fix |
|---|---|---|
| `vllm: unreachable` in health | Expected — Med-SEAL V1 not deployed | No action; SEA-LION handles clinical LLM |
| `medplum: unreachable` | GKE FHIR server down or URL misconfigured | Check GKE pods: `kubectl get pods -n medseal` |
| `STARTUP FAILED — running in degraded mode` | Missing env vars or API key issues | Check `MEDSEAL_*` env vars in Cloud Run |
| `SEA-LION API timeout` | Rate limit or network issue | Retry; check SEA-LION API status |
| `SQLite checkpointer unavailable` | File permission or dependency issue | Falls back to in-memory MemorySaver; sessions won't persist across restarts |

### 5.3 Scaling

Cloud Run auto-scales based on traffic. To adjust limits:

```bash
# Update CPU/memory
gcloud run services update medseal-agent \
  --region asia-southeast1 \
  --cpu 2 \
  --memory 2Gi

# Set max instances
gcloud run services update medseal-agent \
  --region asia-southeast1 \
  --max-instances 10
```

> **Note:** When scaling to multiple instances, switch session persistence from SQLite to Redis (`MEDSEAL_REDIS_URL`) since SQLite is per-instance.

---

## 6. Live Deployment URLs

| Service | URL | Platform |
|---|---|---|
| **AI Agent (API)** | `https://medseal-agent-74997794842.asia-southeast1.run.app` | Cloud Run |
| **Swagger UI** | `https://medseal-agent-74997794842.asia-southeast1.run.app/docs` | Cloud Run |
| **Patient App** | `app.medseal.34.54.226.15.nip.io` | GKE |
| **FHIR Server** | `fhir.medseal.34.54.226.15.nip.io` | GKE |
| **OpenEMR** | `emr.medseal.34.54.226.15.nip.io` | GKE |
| **Medplum Admin** | `medplum.medseal.34.54.226.15.nip.io` | GKE |

---

## 7. Related Pages

- {doc}`../technical-report-v1` — Med-SEAL V1 base model (`med-r1`) technical report
- {doc}`overview` — Multi-agent roster and orchestration
- {doc}`../architecture` — Full system architecture
- {doc}`gcp-deployment` — Detailed GCP deployment guide
- {doc}`../developer-guide/api-reference` — REST API reference
- {doc}`../developer-guide/environment-setup` — Local development setup
