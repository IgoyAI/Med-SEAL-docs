# AI Service API Reference

The **AI Service** exposes a REST API on port `4003`. All endpoints (except `/health`) require a valid session token issued by the SSO service, passed as `Authorization: Bearer <token>`.

---

## Base URL

```
http://localhost:4003
```

---

## Authentication

Include the SSO token in all requests:

```text
Authorization: Bearer <sso-token>
```

---

## Health Check

### `GET /health`

Returns service health. No authentication required.

**Response 200**
```json
{ "status": "ok", "version": "1.0.0" }
```

---

## Chat (Companion Agent A1)

### `POST /chat`

Send a message to the AI Companion and receive a response.

**Request body**

```json
{
  "patientId": "patient-123",
  "message": "What are the side effects of metformin?",
  "language": "en",
  "sessionId": "session-abc"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `patientId` | string | Yes | FHIR Patient ID |
| `message` | string | Yes | Patient's message text |
| `language` | string | No | `en`, `zh`, `ms`, `ta` (default: `en`) |
| `sessionId` | string | No | Existing session ID for context continuity |

**Response 200**
```json
{
  "reply": "Metformin commonly causes nausea and diarrhoea, especially when first starting...",
  "sessionId": "session-abc",
  "agentsUsed": ["A1", "A2"],
  "guardDecision": "PASS",
  "language": "en"
}
```

---

## Nudges (Nudge Agent A3)

### `GET /nudges/:patientId`

Retrieve pending nudges for a patient.

**Response 200**
```json
{
  "nudges": [
    {
      "id": "nudge-001",
      "type": "medication-reminder",
      "message": "Your morning metformin dose is due.",
      "severity": "info",
      "createdAt": "2026-03-19T08:00:00+08:00"
    }
  ]
}
```

### `POST /nudges/:patientId/dismiss`

Dismiss a nudge after the patient has acknowledged it.

**Request body**
```json
{ "nudgeId": "nudge-001" }
```

**Response 200**
```json
{ "dismissed": true }
```

---

## Pre-Visit Summary (Insight Agent A5)

### `POST /patients/:id/previsit-summary`

Generate or retrieve a pre-visit summary for an appointment.

**Request body**
```json
{
  "appointmentId": "appt-456",
  "lookbackDays": 30
}
```

**Response 200**
```json
{
  "compositionId": "Composition/comp-789",
  "sections": [
    { "title": "Medication Adherence", "content": "..." },
    { "title": "Biometric Trends", "content": "..." },
    { "title": "Suggested Discussion Points", "content": "..." }
  ],
  "generatedAt": "2026-03-19T08:00:00+08:00"
}
```

---

## Adherence Metrics (Measurement Agent A6)

### `GET /metrics/:patientId/adherence`

Retrieve Proportion of Days Covered (PDC) for all active medications.

**Response 200**
```json
{
  "patientId": "patient-123",
  "period": { "start": "2026-02-17", "end": "2026-03-19" },
  "medications": [
    { "name": "Metformin 500mg", "pdc": 0.85, "status": "on-track" },
    { "name": "Lisinopril 10mg", "pdc": 0.60, "status": "needs-attention" }
  ],
  "overallPdc": 0.72
}
```

---

## Error Responses

| Status | Code | Description |
|---|---|---|
| 400 | `INVALID_REQUEST` | Missing or malformed request body |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 404 | `NOT_FOUND` | Patient or resource not found |
| 422 | `GUARD_BLOCKED` | Request blocked by SEA-LION Guard |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

**Error envelope**
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "patientId is required"
  }
}
```
