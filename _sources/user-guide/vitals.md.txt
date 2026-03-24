# Vitals and Biometrics

The Patient Portal lets you log health measurements manually or sync them from wearables. Your care team can see these readings and the AI uses them to generate personalised insights.

---

## Supported Measurements

| Measurement | Unit | Normal Range (typical adult) |
|---|---|---|
| Blood pressure | mmHg (systolic/diastolic) | <120/80 |
| Blood glucose | mmol/L or mg/dL | 4.0-7.8 mmol/L (fasting) |
| Heart rate | bpm | 60-100 |
| Weight | kg | Varies by individual |
| Body temperature | degC | 36.1-37.2 |
| Oxygen saturation | % | 95-100 |

---

## Logging a Reading Manually

1. Tap the **Health** tab, then select **Vitals**.
2. Tap the **+ Add Reading** button.
3. Choose the measurement type (e.g. "Blood Pressure").
4. Enter the value(s) and confirm the date and time.
5. Tap **Save**.

The reading is saved to your FHIR record as an `Observation` resource.

---

## Viewing Trends

Tap any measurement type to see a **trend chart** showing your readings over the past 30 days. The chart includes:

- **Goal line** -- your target range set by your care team
- **Average** -- your average over the selected period
- **Colour coding** -- green (in range), amber (borderline), red (out of range)

Pinch to zoom in or out on the chart. Swipe to pan across dates.

---

## Wearable Integration

### Apple Health (iOS)

1. Go to **Settings > Wearables > Apple Health**.
2. Tap **Connect** and grant permissions for the data types you want to share.
3. Data syncs automatically every 15 minutes when the app is open.

### Google Health Connect (Android)

1. Go to **Settings > Wearables > Health Connect**.
2. Tap **Connect** and follow the on-screen permissions.
3. Data syncs automatically when the app is in the foreground or background.

Supported wearable data: steps, heart rate, blood pressure, blood glucose, weight, sleep duration, oxygen saturation.

---

## Alerts and Notifications

If a reading falls outside the thresholds set by your care team, the app will:

1. Show an **alert banner** on the Home screen.
2. Send a **push notification**.
3. Optionally escalate to your care team (for severe readings).

Thresholds are configured by your clinician and may differ from the typical ranges shown above.

---

## Deleting a Reading

If you entered a value incorrectly:

1. Tap the reading in the trend list.
2. Tap **Delete Reading**.
3. Confirm the deletion.

Deleted readings are removed from the FHIR record. Inform your care team if a deleted reading was clinically significant.
