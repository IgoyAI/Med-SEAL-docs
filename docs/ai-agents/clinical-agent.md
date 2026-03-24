# Med-SEAL V1 Clinical Agent

Med-SEAL V1 refers to **`med-r1`** — the fine-tuned **base model** that powers the clinical reasoning capabilities of the Med-SEAL platform.

> **Note:** `med-r1` is **not deployed** in the current system (requires 2× NVIDIA H200 GPUs). The production system uses **SEA-LION v4-32B** (AI Singapore's National LLM) as the clinical reasoning backend with the same prompt architecture and safety guards. See {doc}`../technical-report-v1` for the full technical report.

---

## What is Med-SEAL V1?

| Field | Value |
|---|---|
| **Model name** | `med-r1` |
| **Base model** | Qwen3-VL-8B-Thinking |
| **Parameters** | ~8B |
| **Modality** | Vision-Language (text + image) |
| **Fine-tuning** | SFT on clinical reasoning tasks |
| **Domain** | Chronic disease management (diabetes, hypertension, hyperlipidemia) |
| **Languages** | English, Chinese, Malay, Tamil |
| **Temperature** | 0.3 |
| **GPU requirement** | 2× NVIDIA H200 (vLLM) |

Med-SEAL V1 is **purely the model** — not the agent system. The multi-agent architecture (A1–A6, SEA-LION Guard, smolagents Orchestrator) *consumes* this model as its clinical reasoning backend.

---

## Which Agents Use It?

| Agent | How it uses med-r1 |
|---|---|
| **A2** (Clinical Reasoning) | Calls `med-r1` for drug interactions, lab interpretation, condition reasoning |
| **A5** (Insight Synthesis) | Calls `med-r1` for pre-visit clinical brief generation |
| **A1** (Companion) | Indirectly — delegates complex questions to A2, which calls `med-r1` |

Other agents (A3, A4, A6) use different models (MERaLiON, SEA-LION, analytics engine) and do **not** use Med-SEAL V1.

---

## Current vs Target Deployment

| | Current (SEA-LION) | Target (Med-SEAL V1) |
|---|---|---|
| **Model** | SEA-LION v4-32B (AI Singapore) | `med-r1` (Qwen3-VL-8B fine-tuned) |
| **Endpoint** | `api.sea-lion.ai/v1` | Self-hosted vLLM |
| **GPU** | None (cloud API) | 2× H200 |
| **Clinical fine-tuning** | ❌ (general-purpose) | ✅ (chronic disease SFT) |
| **Multimodal** | ❌ | ✅ (vision-language) |
| **Platform** | GCP Cloud Run | GCP Cloud Run or GKE |

Switching requires only environment variable changes — no code changes.

---

## Related Pages

- {doc}`../technical-report-v1` — Full technical report on `med-r1`
- {doc}`overview` — Multi-agent roster and orchestration
- {doc}`features` — Feature-to-agent mapping
