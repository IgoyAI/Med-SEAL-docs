# Deployment

This guide covers the Docker Compose deployment of Med-SEAL Suite in detail, including environment configuration, volume management, networking, and production considerations.

## Docker Compose Overview

Med-SEAL Suite is deployed as a single `docker-compose.yml` defining all services. The stack consists of:

| Service | Image | Port |
|---|---|---|
| `openemr` | `openemr/openemr:7.0.2` | 8081 (HTTP), 8080 (HTTPS) |
| `openemr-db` | `mariadb:10.11` | 3307 |
| `medplum-server` | `medplum/medplum-server:latest` | 8103 |
| `medplum-app` | `medplum/medplum-app:latest` | 3000 |
| `medplum-db` | `postgres:16-alpine` | 5433 |
| `medplum-redis` | `redis:7-alpine` | 6380 |
| `ai-service` | Custom build | 4003 |
| `ai-frontend` | Custom build | 3001 |
| `sso-db` | `postgres:16-alpine` | 5434 |

## Starting the Stack

```bash
# Start all services in detached mode
docker compose up -d

# Start specific services only
docker compose up -d openemr medplum-server

# Rebuild custom images (ai-service, ai-frontend)
docker compose up -d --build
```

## Health Checks

All services include Docker health checks. Monitor them with:

```bash
docker compose ps
```

| Service | Health Check | Start Period |
|---|---|---|
| OpenEMR | HTTP GET `/` | 60s |
| Medplum Server | HTTP GET `/healthcheck` | 120s |
| AI Service | HTTP GET `/health` | 15s |
| MariaDB | `healthcheck.sh --connect` | 30s |
| PostgreSQL | `pg_isready` | 10s |
| Redis | `redis-cli ping` |  - |

## Environment Variables

### AI Service

| Variable | Description | Default |
|---|---|---|
| `LLM_API_URL` | LLM inference endpoint | `https://medseal-llm.ngrok-free.dev/v1/chat/completions` |
| `LLM_MODEL` | Model identifier | `med-r1` |
| `LLM_TEMPERATURE` | Generation temperature | `0.3` |
| `LLM_MAX_TOKENS` | Max response tokens | `2048` |
| `AI_SERVICE_PORT` | Service port | `4003` |
| `SSO_DB_URL` | SSO database connection string | `postgres://sso:sso_secret@sso-db:5432/medseal_sso` |
| `MEDPLUM_BASE_URL` | Internal Medplum URL | `http://medplum-server:8103` |
| `ORTHANC_URL` | Orthanc internal URL | `http://admin:pass@orthanc:8042` |
| `OPENEMR_DB_HOST` | OpenEMR database host | `openemr-db` |

### OpenEMR

| Variable | Description | Default |
|---|---|---|
| `MYSQL_HOST` | Database host | `openemr-db` |
| `MYSQL_ROOT_PASS` | Root password | `root` |
| `OE_USER` | Admin username | `admin` |
| `OE_PASS` | Admin password | `pass` |

## Volumes

Data is persisted across restarts via Docker named volumes:

| Volume | Service | Mount Point |
|---|---|---|
| `orthanc-data` | Orthanc | `/var/lib/orthanc/db` |
| `medplum-db-data` | Medplum DB | `/var/lib/postgresql/data` |
| `medplum-binary` | Medplum Server | Binary storage |
| `openemr-db-data` | OpenEMR DB | `/var/lib/mysql` |
| `openemr-sites` | OpenEMR | `/var/www/.../sites` |
| `openemr-logs` | OpenEMR | `/var/log` |
| `audit-data` | AI Service | `/data` |
| `sso-db-data` | SSO DB | `/var/lib/postgresql/data` |

## Networking

All services share the `medseal-net` Docker bridge network. Services reference each other by container name:

```
openemr-db    →  medseal-openemr-db
medplum-server → medseal-medplum-server
sso-db        →  medseal-sso-db
```

## Stopping & Resetting

```bash
# Stop all services (data preserved)
docker compose down

# Stop and delete all volumes (full reset)
docker compose down -v

# Remove unused images
docker image prune
```

## Cloud Deployment (GCP)

For cloud deployment using **Google Cloud Run** and **GKE**, see the dedicated guide:

- **[GCP Deployment Guide](ai-agents/gcp-deployment.md)** — Cloud Run serverless deployment, GKE cluster management, environment configuration

### Current Live Deployments

| Platform | URL | Region |
|----------|-----|--------|
| **Cloud Run** (AI Agent) | `https://medseal-agent-74997794842.asia-southeast1.run.app` | Singapore |
| **GKE** (Full Stack) | `*.medseal.34.54.226.15.nip.io` | Singapore |

## Production Considerations

```{warning}
The default configuration is for **development only**. For production deployments:
```

- **Secrets management**  - use Docker secrets or environment-specific `.env` files, never hardcoded passwords
- **HTTPS**  - terminate TLS at a reverse proxy (nginx/Caddy/Traefik) in front of all services
- **Resource limits**  - set `deploy.resources.limits` for CPU and memory per service
- **Backup**  - implement automated backup for all database volumes
- **Monitoring**  - add Prometheus + Grafana for service health and performance metrics
- **Logging**  - centralise logs via ELK/Loki stack
- **Scaling**  - consider Kubernetes for horizontal scaling of AI Service and Medplum
