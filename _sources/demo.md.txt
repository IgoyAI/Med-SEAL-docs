# Live Demo

This page provides direct access to all Med-SEAL Suite services deployed on Google Cloud (GKE). Use the credentials listed below to log in and explore the platform.

---

## Quick Access

| Service | URL | Description |
|---|---|---|
| **AI Dashboard** | [app.medseal.34.54.226.15.nip.io](http://app.medseal.34.54.226.15.nip.io) | SSO login portal, admin panel, AI chat |
| **OpenEMR** | [emr.medseal.34.54.226.15.nip.io](http://emr.medseal.34.54.226.15.nip.io) | Clinical EMR - accessed via SSO |
| **Medplum Admin** | [medplum.medseal.34.54.226.15.nip.io](http://medplum.medseal.34.54.226.15.nip.io) | FHIR R4 resource browser |
| **FHIR API** | [fhir.medseal.34.54.226.15.nip.io](http://fhir.medseal.34.54.226.15.nip.io) | Medplum FHIR R4 REST API |
| **AI Service API** | [api.medseal.34.54.226.15.nip.io](http://api.medseal.34.54.226.15.nip.io) | AI Service REST API |
| **Orthanc PACS** | [pacs.medseal.34.54.226.15.nip.io](http://pacs.medseal.34.54.226.15.nip.io) | DICOM medical imaging server |
| **OHIF Viewer** | [viewer.medseal.34.54.226.15.nip.io](http://viewer.medseal.34.54.226.15.nip.io) | Zero-footprint radiology viewer |

---

## Login Credentials

### AI Dashboard / SSO Portal

The primary entry point. Log in here to access AI chat, the admin panel, and OpenEMR.

| Role | Username | Password |
|---|---|---|
| **Admin** | `admin` | `pass` |
| **Clinician (Dr. Nana)** | `drnana` | `pass` |
| **Nurse** | `nurse1` | `pass` |

### OpenEMR (Clinical EMR)

OpenEMR is launched through the SSO Portal automatically after login. You can also access it directly:

| Role | Username | Password |
|---|---|---|
| **Administrator** | `admin` | `pass` |

### Medplum Admin Console

| Field | Value |
|---|---|
| **Email** | `admin@example.com` |
| **Password** | `medplum_admin` |

### Orthanc PACS

| Field | Value |
|---|---|
| **Username** | `admin` |
| **Password** | `pass` |

---

## Recommended Demo Walkthrough

### For Clinicians / Judges

1. **Go to** [AI Dashboard](http://app.medseal.34.54.226.15.nip.io) and log in as `admin` / `pass`.
2. **Launch OpenEMR** from the dashboard to view patient records, clinical documentation, and scheduling.
3. Open the **AI Chat** panel inside OpenEMR to ask clinical questions (e.g., "Summarise Siti Nurhaliza's medications").
4. Return to the AI Dashboard and navigate to **Pre-Visit Summaries** to view AI-generated clinical briefs.

### For Technical Reviewers

1. Open [Medplum Admin](http://medplum.medseal.34.54.226.15.nip.io) and log in with `admin@example.com` / `medplum_admin`.
2. Browse FHIR R4 resources (Patient, Observation, MedicationRequest, Encounter).
3. Test the FHIR API directly: `curl http://fhir.medseal.34.54.226.15.nip.io/fhir/R4/Patient`.
4. Open [Orthanc PACS](http://pacs.medseal.34.54.226.15.nip.io) and view uploaded DICOM studies.
5. Launch the [OHIF Viewer](http://viewer.medseal.34.54.226.15.nip.io) for zero-footprint radiology image viewing.

### Patient Portal (iOS)

The Patient Portal Native app is available on iOS devices. To connect to the live backend:
- **FHIR Base URL**: `http://fhir.medseal.34.54.226.15.nip.io`
- **AI Service URL**: `http://api.medseal.34.54.226.15.nip.io`

---

## Service Health

Check if the AI Service is running:

```bash
curl http://api.medseal.34.54.226.15.nip.io/health
# Expected: {"status":"ok","version":"1.0.0"}
```

Check Medplum FHIR API:

```bash
curl http://fhir.medseal.34.54.226.15.nip.io/healthcheck
# Expected: {"ok":true}
```
