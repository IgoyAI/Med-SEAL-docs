# Patient Portal Native

The Patient Portal Native app is a cross-platform mobile application built with **Expo / React Native**, providing patients with direct access to their health data, AI-powered chat, medication tracking, and appointment management.

## Role in Med-SEAL

The mobile app is the primary **patient-facing interface**:

- **AI Chat**  - conversational interface powered by the Med-SEAL AI agents
- **Medication Tracking**  - view schedules, confirm doses, track adherence
- **Vitals & Biometrics**  - log and visualise health measurements
- **Appointments**  - view upcoming visits, receive reminders
- **Medical Records**  - access conditions, allergies, immunisations, encounters

## Technology Stack

| Component | Technology |
|---|---|
| Framework | Expo (React Native) |
| Language | TypeScript |
| Platforms | iOS, Android |
| Data layer | FHIR R4 via Medplum SDK |
| Navigation | Expo Router (file-based) |

## Project Structure

```
apps/patient-portal-native/
├── app/                   # Expo Router pages
├── assets/                # Images, fonts, icons
├── components/            # Reusable UI components
├── lib/                   # Utilities, API clients, helpers
├── scripts/               # Build and dev scripts
├── ios/                   # Native iOS project
├── android/               # Native Android project
├── app.json               # Expo config
├── cors-proxy.js          # Dev proxy for CORS issues
├── package.json
└── tsconfig.json
```

## Getting Started

### Prerequisites

- Node.js ≥ 18
- Expo CLI (`npx expo`)
- iOS Simulator (macOS) or Android emulator
- Med-SEAL backend services running (see {doc}`/getting-started`)

### Install Dependencies

```bash
cd apps/patient-portal-native
npm install
```

### Run on iOS Simulator

```bash
npx expo run:ios
```

### Run on Android Emulator

```bash
npx expo run:android
```

### Development Server (Expo Go)

```bash
npx expo start
```

Then scan the QR code with Expo Go on your physical device.

## CORS Proxy

For local development, the app includes a CORS proxy (`cors-proxy.js`) to handle cross-origin requests to the Medplum and AI Service APIs:

```bash
node cors-proxy.js
```

## FHIR Integration

The patient portal connects to Medplum's FHIR R4 API to read and write:

| Feature | FHIR Resources |
|---|---|
| Medical records | `Patient`, `Condition`, `AllergyIntolerance` |
| Medications | `MedicationRequest`, `MedicationAdministration` |
| Vitals | `Observation` (BP, glucose, heart rate, weight) |
| Appointments | `Appointment` |
| Immunisations | `Immunization` |
| Encounters | `Encounter`, `Procedure` |

## Key Features

### AI Chat
Conversational interface connecting to the Companion Agent (A1) for:
- Health questions in multiple languages (EN, ZH, MS, TA)
- Medication reminders and explanations
- Dietary advice (SEA-culturally aware)
- PRO questionnaire delivery

### Medication Management
- Daily dose schedule with tap-to-confirm
- Missed dose reminders (via Nudge Agent A3)
- Drug interaction warnings
- Adherence tracking (PDC calculation)

### Vitals Dashboard
- Manual entry for blood pressure, glucose, weight
- Wearable data ingestion (Apple Health / Google Health Connect)
- Trend visualisation with goal lines
- Threshold alerts

### Appointments
- Upcoming visit list with countdown
- Pre-visit preparation prompts
- Post-visit summaries

## Building for Production

### iOS

```bash
npx expo run:ios --configuration Release
```

### Android

```bash
npx expo run:android --variant release
```
