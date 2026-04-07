# OpenEMR

[OpenEMR](https://www.open-emr.org/) is the clinical EMR at the heart of Med-SEAL Suite. It provides full electronic health record management including patient intake, clinical documentation, orders, scheduling, and billing.

## Role in Med-SEAL

OpenEMR serves as the **clinician-facing interface** where doctors, nurses, and administrators manage day-to-day clinical operations. It connects to the rest of the stack via:

- **Medplum**  - patient data is synced to the FHIR R4 layer for interoperability
- **AI Service**  - the AI chat widget is injected into OpenEMR's interface for clinical decision support
- **SSO**  - user accounts are synchronised between the SSO database and OpenEMR

## Access

| Property | Value |
|---|---|
| HTTP URL | <http://localhost:8081> |
| HTTPS URL | <https://localhost:8080> |
| Default credentials | `admin` / `pass` |
| Container name | `medseal-openemr` |
| Database | MariaDB 10.11 (`medseal-openemr-db`, port `3307`) |

## Docker Configuration

OpenEMR runs as a containerised instance (`openemr/openemr:7.0.2`) with the following customisations:

```yaml
openemr:
  image: openemr/openemr:7.0.2
  ports:
    - "8081:80"   # HTTP
    - "8080:443"  # HTTPS
  environment:
    MYSQL_HOST: openemr-db
    OE_USER: admin
    OE_PASS: pass
```

## Med-SEAL Customisations

Med-SEAL injects several customisations into the OpenEMR container:

### AI Chat Widget
An embedded chat interface that connects to the AI Service, allowing clinicians to query patient data and receive AI-powered insights directly within OpenEMR.

- **JavaScript**: `openemr/custom/ai-chat-widget.js`
- **Stylesheet**: `openemr/custom/ai-chat-widget.css`

### Med-SEAL Theme
A custom CSS theme that applies Med-SEAL branding to the OpenEMR interface.

- **Stylesheet**: `openemr/custom/medseal-theme.css`
- **Injection script**: `openemr/custom/inject-theme.sh`

## Standards Compliance

| Standard | Usage |
|---|---|
| **ICD-10** | Diagnosis coding |
| **SNOMED CT** | Clinical terminology |
| **HL7 v2** | Messaging and ADT |
| **CPT** | Procedure coding |

## Key Features

- Patient demographics and registration
- Clinical encounter documentation
- Medication management and e-prescribing
- Lab orders and results
- Appointment scheduling
- Billing and claims (CMS-1500, UB-04)
- Clinical decision rules
- Document management
- Multi-language support
