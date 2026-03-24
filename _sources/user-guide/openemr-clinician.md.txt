# OpenEMR Clinician Guide

**OpenEMR** is the electronic medical record (EMR) system used by clinicians in the Med-SEAL Suite. It handles patient management, clinical documentation, prescribing, scheduling, and billing.

---

## Accessing OpenEMR

Navigate to your OpenEMR instance (default: `http://localhost:8081`) and log in with your SSO credentials. First-time users should see the main clinical dashboard.

---

## Dashboard Overview

After login, the OpenEMR dashboard displays:

- **Messages** -- internal messages and alerts from staff
- **Reminder mail** -- upcoming patient tasks
- **Upcoming appointments** -- today's and tomorrow's schedule
- **Patient finder** -- quick search by name or ID

---

## Patient Management

### Searching for a Patient

1. From the top menu, click **Patient > Patient Finder**.
2. Enter the patient's name, date of birth, or patient ID.
3. Click **Search**, then select the patient from the results list.

### Registering a New Patient

1. Click **Patient > New/Search**.
2. Click **New Patient**.
3. Fill in the **Demographics** tab: name, date of birth, gender, contact details.
4. Click **Create New Patient**.

---

## Clinical Documentation

### Recording an Encounter

1. Open a patient's chart.
2. Click **Encounters > New Encounter**.
3. Select the **Encounter date**, **Facility**, and **Provider**.
4. Click **Save**.
5. Add encounter notes in the **SOAP** or **History & Physical** form.

### Ordering Labs and Procedures

1. Inside an open encounter, click **Clinical > Procedures**.
2. Select the procedure type and enter order details.
3. Click **Save and Transmit**.

---

## Prescribing Medications

1. Open the patient's chart and navigate to **Prescriptions**.
2. Click **Add**.
3. Search for the drug by name or NDC code.
4. Set dosage, frequency, quantity, and refills.
5. Click **Save**.

The prescription is saved to the patient's FHIR record via the Medplum sync.

---

## Scheduling Appointments

1. Click **Calendar** in the top menu.
2. Click an available time slot for the provider.
3. Search for and select the patient.
4. Choose appointment type and add notes.
5. Click **Save**.

A FHIR `Appointment` resource is created automatically and visible in the Patient Portal.

---

## AI Clinical Brief (Pre-Visit Summary)

Before a scheduled appointment the AI Insight Synthesis Agent (A5) generates a **Pre-Visit Summary** -- a structured brief covering:

- Medication adherence over the past 30 days
- Biometric trends (blood pressure, glucose, weight)
- Patient-Reported Outcomes (PRO scores)
- Open flags and escalation alerts
- AI-suggested talking points

To view the brief, open the patient's chart and click **AI Insights** in the left sidebar.

---

## Billing and Coding

1. After documenting an encounter, click **Billing > Add New Bill**.
2. Assign ICD-10 diagnosis codes and CPT procedure codes.
3. Submit the claim or save for later review.

---

## Logging Out

Click your username in the top-right corner and select **Logout**. For shared workstations, always log out after each session.
