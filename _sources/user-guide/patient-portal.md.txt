# Patient Portal Native

The **Patient Portal Native** app is your personal health companion on iOS and Android. It gives you on-demand access to your health data, AI-powered support, and direct communication tools -- all connected securely to your clinical records.

---

## Installing the App

- **iOS**: Download from the App Store -- search "Med-SEAL Patient Portal".
- **Android**: Download from Google Play -- search "Med-SEAL Patient Portal".

For local development builds, see {doc}`/components/patient-portal-native`.

---

## App Overview

After logging in you land on the **Home** screen. The bottom navigation bar contains five sections:

| Tab | What you will find |
|---|---|
| **Home** | Today's summary: medications due, upcoming appointments, recent vitals, nudges |
| **Health** | Full medical records -- conditions, allergies, immunisations, past encounters |
| **Medications** | Medication schedule, dose history, adherence score (see {doc}`medications`) |
| **Appointments** | Upcoming and past visits (see {doc}`appointments`) |
| **Chat** | AI health assistant (see {doc}`ai-chat`) |

---

## Home Screen

The Home screen shows a personalised daily snapshot:

- **Medication reminders** -- doses due in the next 2 hours appear as cards. Tap a card to confirm the dose.
- **Upcoming appointments** -- the next scheduled visit with a countdown.
- **Recent vitals** -- your last logged blood pressure, glucose, or weight reading.
- **AI nudges** -- proactive suggestions from your care team's AI (e.g. "Your blood pressure has been elevated 3 days in a row").

---

## Health Records

Tap **Health** to view your clinical data, pulled directly from your FHIR record:

- **Conditions** -- active diagnoses with onset date and status.
- **Allergies** -- documented allergies and intolerances.
- **Immunisations** -- vaccination history with dates.
- **Encounters** -- past clinical visits and procedure notes.

All data is read-only. To update your records, contact your care team through the app chat or directly with your clinic.

---

## Notifications

The app sends push notifications for:

- Medication doses due
- Appointment reminders (48 hours and 1 hour before)
- Biometric threshold alerts (e.g. blood pressure above target)
- Messages from your care team

To manage notifications, go to **Settings > Notifications** in the app.

---

## Privacy and Data

Your health data is stored on your organisation's secure FHIR server (Medplum). The app never stores sensitive data locally beyond your active session. For data requests or deletions, contact your administrator.
