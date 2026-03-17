# Scripts Reference

Med-SEAL Suite includes a collection of utility scripts in the `scripts/` directory for data generation, seeding, and synchronisation.

## Data Loading & Seeding

### `load-synthea.js`

Loads Synthea-generated synthetic patient bundles into Medplum.

```bash
node scripts/load-synthea.js
```

Synthea bundles are processed as FHIR transaction Bundles and uploaded to the Medplum FHIR API.

---

### `seed-medplum-patient.js`

Seeds a comprehensive patient record in Medplum with clinical data including demographics, conditions, medications, observations, encounters, and care plans.

```bash
node scripts/seed-medplum-patient.js
```

---

### `generate-clinical-notes.js`

Generates realistic clinical notes (progress notes, discharge summaries) for seeded patients.

```bash
node scripts/generate-clinical-notes.js
```

---

### `clean-patient-names.js`

Normalises and cleans patient names across Medplum records for consistent display.

```bash
node scripts/clean-patient-names.js
```

## Synchronisation

### `sync-medplum-openemr.js`

Synchronises patient data between Medplum (FHIR) and OpenEMR (SQL):

```bash
node scripts/sync-medplum-openemr.js
```

Maps FHIR Patient resources to OpenEMR database records, maintaining referential integrity.

---

### `sync-medplum-openmrs.js`

Synchronises patient data between Medplum and OpenMRS:

```bash
node scripts/sync-medplum-openmrs.js
```

---

### `sync-patients-full.js`

Runs a complete patient synchronisation across all connected systems:

```bash
node scripts/sync-patients-full.js
```

---

### `sync-batch.js`

Batch synchronisation tool for processing multiple patients:

```bash
node scripts/sync-batch.js
```

---

### `sync-remaining.js`

Syncs patients that were missed or failed in previous sync runs:

```bash
node scripts/sync-remaining.js
```

## Radiology & DICOM

### `integrate-radiology.js`

Creates ImagingStudy and DiagnosticReport FHIR resources linked to Orthanc DICOM studies:

```bash
node scripts/integrate-radiology.js
```

---

### `generate-dicom-3d.py`

Python script that generates synthetic 3D DICOM datasets for testing:

```bash
python scripts/generate-dicom-3d.py
```

**Requires:** Python 3.x with `pydicom` and `numpy`.

## Simulation

### `simulate-busy-hospital.js`

Simulates a busy hospital environment by generating a stream of clinical events (admissions, discharges, orders, observations) over time:

```bash
node scripts/simulate-busy-hospital.js
```

Useful for testing system performance under load and demonstrating real-time data flows.

## Synthea

The `scripts/` directory also includes the Synthea JAR file (`synthea-with-dependencies.jar`) for generating synthetic patient data:

```bash
java -jar scripts/synthea-with-dependencies.jar \
  -p 100 \
  --exporter.fhir.export true \
  -o scripts/output
```

This generates 100 synthetic patients as FHIR bundles in `scripts/output/fhir/`.
