# Medplum

[Medplum](https://www.medplum.com/) is the **FHIR R4 interoperability layer** of Med-SEAL Suite. It acts as the central data backbone, storing and exposing all clinical data via standards-compliant FHIR APIs.

## Role in Med-SEAL

Medplum bridges the gap between the clinical EMR (OpenEMR) and the rest of the platform:

- **FHIR R4 API**  - all patient data accessible via standard FHIR endpoints
- **Data store**  - canonical source for synced clinical records
- **Subscriptions**  - real-time event triggers for the AI agent pipeline
- **Terminology**  - FHIR-based terminology lookups (RxNorm, LOINC, SNOMED CT)

## Access

| Property | Value |
|---|---|
| FHIR API | <http://localhost:8103> |
| Admin App | <http://localhost:3000> |
| Container (Server) | `medseal-medplum-server` |
| Container (App) | `medseal-medplum-app` |
| Database | PostgreSQL 16 (`medseal-medplum-db`, port `5433`) |
| Cache | Redis 7 (`medseal-medplum-redis`, port `6380`) |

## Docker Configuration

Medplum consists of three containers:

```yaml
medplum-server:
  image: medplum/medplum-server:latest
  ports:
    - "8103:8103"
  volumes:
    - ./medplum/medplum.config.json:/usr/src/medplum/medplum.config.json:ro

medplum-app:
  image: medplum/medplum-app:latest
  ports:
    - "3000:3000"
  environment:
    - MEDPLUM_BASE_URL=http://localhost:8103/
```

## Configuration

The Medplum server is configured via `medplum/medplum.config.json`:

```json
{
  "port": 8103,
  "baseUrl": "http://localhost:8103/",
  "database": {
    "host": "medplum-db",
    "port": 5432,
    "dbname": "medplum",
    "username": "medplum",
    "password": "medplum"
  },
  "redis": {
    "host": "medplum-redis",
    "port": 6379
  }
}
```

## FHIR API Usage

### Common Endpoints

```bash
# Search patients
curl http://localhost:8103/fhir/R4/Patient

# Get a specific patient
curl http://localhost:8103/fhir/R4/Patient/{id}

# Search observations for a patient
curl http://localhost:8103/fhir/R4/Observation?patient=Patient/{id}

# Create a resource (example: Observation)
curl -X POST http://localhost:8103/fhir/R4/Observation \
  -H "Content-Type: application/fhir+json" \
  -d '{"resourceType": "Observation", ...}'
```

### Bulk Operations

```bash
# Transaction bundle
curl -X POST http://localhost:8103/fhir/R4 \
  -H "Content-Type: application/fhir+json" \
  -d '{"resourceType": "Bundle", "type": "transaction", ...}'
```

## Admin App Features

The Medplum Admin App (`http://localhost:3000`) provides:

- **Resource browser**  - view, create, edit, and delete any FHIR resource
- **Patient timeline**  - chronological view of a patient's clinical data
- **Batch operations**  - bulk import/export FHIR resources
- **Bot management**  - configure FHIR Subscriptions and event-driven workflows
- **Access policies**  - define RBAC and SMART on FHIR scopes

## Resource Types Used

Med-SEAL uses **32 distinct FHIR resource types** across the platform. See {doc}`/fhir-resources` for the complete inventory.
