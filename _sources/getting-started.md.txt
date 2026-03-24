# Getting Started

This guide walks you through spinning up the entire Med-SEAL Suite locally using Docker Compose.

## Prerequisites

- **Docker Desktop** ≥ 4.25 (with Docker Compose v2)
- **16 GB RAM** recommended (multiple services run concurrently)
- **20 GB disk** free (images + volumes)
- **Node.js** ≥ 18 (for patient portal native development)
- **Git**

## Clone the Repository

```bash
git clone https://github.com/IgoyAI/Med-SEAL-Suite.git
cd Med-SEAL-Suite
```

## Start All Services

```bash
docker compose up -d
```

This brings up:
- OpenEMR + MariaDB
- Medplum Server + PostgreSQL + Redis + Admin App
- AI Service + SSO PostgreSQL
- AI Frontend

Monitor startup progress:

```bash
docker compose ps       # Check container status
docker compose logs -f  # Stream all logs
```

```{note}
First startup takes **5–10 minutes** as services initialise databases and run health checks. Wait until all containers show `healthy` status.
```

## Access Points

| Service | URL | Credentials |
|---|---|---|
| **OpenEMR** (Clinical EMR) | <http://localhost:8081> | `admin` / `pass` |
| **Medplum App** (FHIR Admin) | <http://localhost:3000> | Register new account |
| **Medplum API** (FHIR R4) | <http://localhost:8103> |  - |
| **AI Service** (API) | <http://localhost:4003> |  - |
| **AI Frontend** (Dashboard) | <http://localhost:3001> |  - |

## Seed Sample Data

Med-SEAL includes scripts to populate the system with realistic synthetic patient data:

```bash
# Generate and load Synthea patients into Medplum
node scripts/load-synthea.js

# Seed a specific Medplum patient with full clinical data
node scripts/seed-medplum-patient.js

# Sync patients between Medplum and OpenEMR
node scripts/sync-medplum-openemr.js
```

See the {doc}`scripts` page for full details on every utility script.

## Verify the Stack

1. **OpenEMR**  - Log in at `http://localhost:8081` with `admin` / `pass`. You should see the clinical dashboard.
2. **Medplum**  - Open `http://localhost:3000`, register an account, and browse FHIR resources.
3. **AI Service**  - Check the health endpoint:
   ```bash
   curl http://localhost:4003/health
   ```
4. **Patient Portal Native**  - See {doc}`components/patient-portal-native` for Expo setup.

## Stop & Reset

```bash
# Stop services (data preserved)
docker compose down

# Stop and delete all data volumes (full reset)
docker compose down -v
```
