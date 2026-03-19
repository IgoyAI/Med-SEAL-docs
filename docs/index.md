# Med-SEAL Suite Documentation

**Open-source healthcare platform built on international standards.**

Med-SEAL Suite integrates clinical EMR, FHIR-based interoperability, patient-facing mobile apps, AI-powered clinical agents, and single sign-on into a unified Docker-based deployment.

---

## Key Capabilities

- **Clinical EMR**  - OpenEMR for patient management, orders, billing
- **FHIR R4 Interoperability**  - Medplum as the standards-compliant data backbone
- **Patient Mobile App**  - Expo/React Native app for vitals, meds, appointments
- **AI Clinical Agents**  - 6 specialised agents powering 18 smart features
- **Single Sign-On**  - Unified auth across all Med-SEAL services

---

```{toctree}
:maxdepth: 2
:caption: Overview

demo
architecture
getting-started
```

```{toctree}
:maxdepth: 2
:caption: Components

components/openemr
components/medplum
components/patient-portal-native
components/sso
```

```{toctree}
:maxdepth: 2
:caption: AI Platform

ai-agents/overview
ai-agents/clinical-agent
ai-agents/azure-deployment
ai-agents/features
technical-report-v1
```

```{toctree}
:maxdepth: 2
:caption: Operations & Reference

deployment
gke-deployment
scripts
standards
fhir-resources
contributing
```

```{toctree}
:maxdepth: 2
:caption: User Guide

user-guide/index
user-guide/login-sso
user-guide/patient-portal
user-guide/openemr-clinician
user-guide/ai-chat
user-guide/medications
user-guide/appointments
user-guide/vitals
```

```{toctree}
:maxdepth: 2
:caption: Developer Guide

developer-guide/index
developer-guide/environment-setup
developer-guide/fhir-data-model
developer-guide/api-reference
developer-guide/adding-ai-features
developer-guide/testing
developer-guide/extending-openemr
```
