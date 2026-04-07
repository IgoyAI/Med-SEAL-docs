# Environment Setup

This page describes how to configure a full local development environment for all Med-SEAL Suite services.

---

## Prerequisites

| Tool | Minimum Version | Notes |
|---|---|---|
| Docker Desktop | 4.25 | Enable Docker Compose v2 |
| Node.js | 18 LTS | Use `nvm` for version management |
| npm | 9 | Bundled with Node 18 |
| Git | 2.40 | Any modern version |
| Python | 3.10 | For docs only |
| Xcode | 15 (macOS) | iOS simulator for mobile dev |
| Android Studio | Latest | Android emulator for mobile dev |

---

## Repository Clone

```bash
git clone https://github.com/IgoyAI/Med-SEAL-Suite.git
cd Med-SEAL-Suite
```

---

## Docker Compose Stack

Start the full backend:

```bash
docker compose up -d
```

This starts all infrastructure services. Wait for all containers to show `healthy`:

```bash
docker compose ps
```

Expected healthy services after first boot (5-10 minutes):

- `medseal-openemr`
- `medseal-openemr-db`
- `medseal-medplum-server`
- `medseal-medplum-app`
- `medseal-medplum-db`
- `medseal-medplum-redis`
- `medseal-ai-service`
- `medseal-ai-frontend`
- `medseal-sso-db`

---

## AI Service (Local Development)

For hot-reload development of the AI backend:

```bash
cd apps/ai-service
npm install
npm run dev
```

The service starts on port `4003`. Environment variables are loaded from `.env` in the project root. Copy the example:

```bash
cp .env.example .env
```

Key variables to configure:

```bash
LLM_API_URL=https://your-llm-endpoint/v1/chat/completions
LLM_MODEL=med-r1
MEDPLUM_BASE_URL=http://localhost:8103
MEDPLUM_CLIENT_ID=your-client-id
MEDPLUM_CLIENT_SECRET=your-client-secret
SSO_DB_URL=postgres://sso:sso_secret@localhost:5434/medseal_sso
```

---

## Patient Portal Native (Mobile Dev)

```bash
cd apps/patient-portal-native
npm install
```

Run on iOS simulator:

```bash
npx expo run:ios
```

Run on Android emulator:

```bash
npx expo run:android
```

Start the CORS proxy for local API calls:

```bash
node cors-proxy.js
```

The proxy forwards requests from the mobile app to avoid browser CORS restrictions in the development Expo environment.

---

## AI Frontend (Admin Dashboard)

```bash
cd apps/ai-frontend
npm install
npm run dev
```

Dashboard is available at `http://localhost:3001`.

---

## Seeding Data

After the stack is healthy, load sample data:

```bash
# Load Synthea-generated patients into Medplum FHIR store
node scripts/load-synthea.js

# Seed a single patient with full clinical data
node scripts/seed-medplum-patient.js

# Sync Medplum patients to OpenEMR
node scripts/sync-medplum-openemr.js
```

See {doc}`/scripts` for all available utility scripts.

---

## Connecting to Services

| Service | Internal URL (Docker) | External URL (Dev) |
|---|---|---|
| Medplum FHIR API | `http://medplum-server:8103` | `http://localhost:8103` |
| Medplum Admin App | -- | `http://localhost:3000` |
| OpenEMR | `http://openemr:80` | `http://localhost:8081` |
| AI Service | `http://ai-service:4003` | `http://localhost:4003` |
| SSO DB | `postgres://sso-db:5432` | `postgres://localhost:5434` |

---

## Rebuilding Custom Images

After making changes to `ai-service` or `ai-frontend` Dockerfiles:

```bash
docker compose up -d --build ai-service ai-frontend
```
