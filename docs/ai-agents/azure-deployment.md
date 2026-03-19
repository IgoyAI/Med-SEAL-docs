# AI Agent — Implementation & Azure Deployment

This page documents the implementation architecture and Azure deployment of the **Med-SEAL AI Agent** — the multi-agent clinical reasoning service that powers the platform's 18 smart healthcare features.

---

## 1. Implementation Overview

### 1.1 Technology Stack

| Component | Technology |
|---|---|
| **Runtime** | Python 3.11 |
| **Web framework** | FastAPI + Uvicorn |
| **Agent framework** | LangGraph (LangChain) |
| **Clinical LLM** | Azure OpenAI — ChatGPT (GPT deployment) |
| **Conversation model** | SEA-LION v4 (AI Singapore) |
| **Safety guard** | SEA-Guard (AI Singapore) |
| **FHIR client** | Medplum (custom `httpx`-based client) |
| **Session persistence** | SQLite (via `langgraph-checkpoint-sqlite`) |
| **Containerisation** | Docker (`python:3.11-slim`) |
| **Cloud hosting** | Azure App Service (Linux, B1 plan) |

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
# config.py — defaults to Azure
clinical_llm_backend: str = "azure"  # or "vllm" for Med-SEAL V1
```

| Mode | Backend | Model | When |
|---|---|---|---|
| `azure` *(current)* | Azure OpenAI | ChatGPT (GPT) | Demo & deployment — no GPU needed |
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
| **Azure OpenAI** | `*.cognitiveservices.azure.com` | Clinical reasoning LLM |
| **SEA-LION API** | `api.sea-lion.ai/v1` | Conversation + Guard |
| **Medplum FHIR** | `medseal-fhir.ngrok-free.dev/fhir/R4` | Patient health records |

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
│   ├── llm_factory.py           # Azure / vLLM backend selector
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

## 3. Azure Deployment

### 3.1 Azure Resources

| Resource | Type | Configuration |
|---|---|---|
| **Resource Group** | `medseal-rg` | East US 2 |
| **App Service Plan** | `medseal-plan` | Linux, **B1** (Basic) |
| **Web App** | `medseal-agent` | Python 3.11, via ZIP deploy |
| **Azure OpenAI** | Cognitive Services | GPT deployment |

### 3.2 Environment Variables (App Settings)

Configure these in Azure Portal → Web App → Configuration → Application Settings:

```bash
# Clinical LLM (Azure OpenAI)
MEDSEAL_CLINICAL_LLM_BACKEND=azure
MEDSEAL_AZURE_OPENAI_ENDPOINT=https://<your-resource>.cognitiveservices.azure.com/
MEDSEAL_AZURE_OPENAI_API_KEY=<your-api-key>
MEDSEAL_AZURE_OPENAI_DEPLOYMENT=gpt-5.3
MEDSEAL_AZURE_OPENAI_API_VERSION=2025-04-01-preview

# SEA-LION (conversation + guard)
MEDSEAL_SEALION_API_URL=https://api.sea-lion.ai/v1
MEDSEAL_SEALION_API_KEY=<your-sea-lion-key>
MEDSEAL_SEALION_MODEL=aisingapore/Qwen-SEA-LION-v4-32B-IT
MEDSEAL_SEAGUARD_MODEL=aisingapore/SEA-Guard

# Medplum FHIR
MEDSEAL_MEDPLUM_URL=https://medseal-fhir.ngrok-free.dev/fhir/R4
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

Set in Azure Portal → Web App → Configuration → General Settings → Startup Command.

### 3.4 Deployment Steps

#### Prerequisites
- Azure CLI installed and authenticated (`az login`)
- Python 3.11+ locally
- Access to Azure OpenAI resource

#### Step 1: Create Azure Resources

```bash
# Login
az login

# Create resource group
az group create --name medseal-rg --location eastus2

# Create App Service plan (Linux, Basic tier)
az appservice plan create \
  --name medseal-plan \
  --resource-group medseal-rg \
  --is-linux \
  --sku B1

# Create web app
az webapp create \
  --name medseal-agent \
  --resource-group medseal-rg \
  --plan medseal-plan \
  --runtime "PYTHON:3.11"
```

#### Step 2: Configure Environment Variables

```bash
az webapp config appsettings set \
  --name medseal-agent \
  --resource-group medseal-rg \
  --settings \
    MEDSEAL_CLINICAL_LLM_BACKEND=azure \
    MEDSEAL_AZURE_OPENAI_ENDPOINT="https://<your-resource>.cognitiveservices.azure.com/" \
    MEDSEAL_AZURE_OPENAI_API_KEY="<your-api-key>" \
    MEDSEAL_AZURE_OPENAI_DEPLOYMENT="gpt-5.3" \
    MEDSEAL_AZURE_OPENAI_API_VERSION="2025-04-01-preview" \
    MEDSEAL_SEALION_API_URL="https://api.sea-lion.ai/v1" \
    MEDSEAL_SEALION_API_KEY="<your-key>" \
    MEDSEAL_MEDPLUM_URL="https://medseal-fhir.ngrok-free.dev/fhir/R4"
```

#### Step 3: Set Startup Command

```bash
az webapp config set \
  --name medseal-agent \
  --resource-group medseal-rg \
  --startup-file "uvicorn agent.main:app --host 0.0.0.0 --port 8000"
```

#### Step 4: Deploy via ZIP

```bash
# Create deployment ZIP (slim — agent code + requirements only)
zip -r deploy.zip agent/ requirements_deploy.txt

# Deploy
az webapp deploy \
  --name medseal-agent \
  --resource-group medseal-rg \
  --src-path deploy.zip \
  --type zip
```

#### Step 5: Verify

```bash
# Check logs
az webapp log tail --name medseal-agent --resource-group medseal-rg

# Health check
curl https://medseal-agent.azurewebsites.net/health
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

> `vllm: unreachable` is expected — Med-SEAL V1 (`med-r1`) is not deployed. The system uses Azure OpenAI instead.

### 3.5 Dockerfile (Alternative Deployment)

The project also includes a Dockerfile for container-based deployment:

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

Deploy as a container via Azure Container Apps or Azure Web App for Containers:

```bash
# Build and push
docker build -t medseal-agent .
docker tag medseal-agent <your-acr>.azurecr.io/medseal-agent:latest
docker push <your-acr>.azurecr.io/medseal-agent:latest
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
│                    Azure App Service (B1)                    │
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
    │Azure OpenAI│    │  SEA-LION   │    │   Medplum   │
    │ ChatGPT    │    │ api.sea-lion│    │  FHIR R4    │
    │(clinical)  │    │ .ai/v1     │    │ (ngrok)     │
    └────────────┘    └─────────────┘    └─────────────┘
```

---

## 5. Monitoring & Troubleshooting

### 5.1 Logs

```bash
# Stream live logs
az webapp log tail --name medseal-agent --resource-group medseal-rg

# Download log files
az webapp log download --name medseal-agent --resource-group medseal-rg
```

### 5.2 Common Issues

| Issue | Cause | Fix |
|---|---|---|
| `vllm: unreachable` in health | Expected — Med-SEAL V1 not deployed | No action; Azure OpenAI handles clinical LLM |
| `STARTUP FAILED — running in degraded mode` | Missing env vars or API key issues | Check `MEDSEAL_*` app settings |
| `Azure OpenAI not configured` | Missing endpoint/key env vars | Set `MEDSEAL_AZURE_OPENAI_ENDPOINT` and `MEDSEAL_AZURE_OPENAI_API_KEY` |
| Deployment timeout | ZIP too large or slow build | Use `requirements_deploy.txt` (slim), not full `requirements.txt` |
| `SQLite checkpointer unavailable` | File permission or dependency issue | Falls back to in-memory MemorySaver; sessions won't persist across restarts |

### 5.3 Scaling

The B1 plan provides 1 core / 1.75 GB RAM. For higher load:

```bash
# Scale up to S1 (Standard)
az appservice plan update --name medseal-plan --resource-group medseal-rg --sku S1

# Scale out (multiple instances)
az webapp scale --name medseal-agent --resource-group medseal-rg --instance-count 2
```

> **Note:** When scaling to multiple instances, switch session persistence from SQLite to Redis (`MEDSEAL_REDIS_URL`) since SQLite is per-instance.

---

## 6. Related Pages

- {doc}`../technical-report-v1` — Med-SEAL V1 base model (`med-r1`) technical report
- {doc}`overview` — Multi-agent roster and orchestration
- {doc}`../architecture` — Full system architecture
- {doc}`../developer-guide/api-reference` — REST API reference
- {doc}`../developer-guide/environment-setup` — Local development setup
