# Adding a New AI Feature

This guide walks through the end-to-end process of adding a new feature to the Med-SEAL AI Service, from designing the agent interaction to wiring up the API endpoint and connecting it to the patient app.

---

## Overview

Each AI feature in Med-SEAL follows a standard pattern:

1. A **REST endpoint** in the AI Service receives a request.
2. The **Orchestrator** routes it to one or more **agents**.
3. Agents call the **LLM** and/or read FHIR data from Medplum.
4. The **SEA-LION Guard** validates the output.
5. The response is returned to the caller (app or OpenEMR).

---

## Step 1: Define Your Feature

Before writing code, document the feature:

- **What does it do?** One sentence.
- **Which agent(s) are responsible?** (A1-A6, or a new agent)
- **What FHIR data does it need?** List the resource types.
- **What does it return?** Describe the response shape.
- **Does it need LLM reasoning?** Or is it pure analytics?

---

## Step 2: Create the Route

Add a new Express route in `apps/ai-service/src/routes/`:

```typescript
// apps/ai-service/src/routes/my-feature.ts
import { Router, Request, Response } from 'express';
import { myFeatureHandler } from '../handlers/my-feature';

export const myFeatureRouter = Router();

myFeatureRouter.post('/patients/:id/my-feature', async (req: Request, res: Response) => {
  const { id } = req.params;
  try {
    const result = await myFeatureHandler(id, req.body);
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: (err as Error).message } });
  }
});
```

Register the router in `apps/ai-service/src/index.ts`:

```typescript
import { myFeatureRouter } from './routes/my-feature';
app.use('/api', myFeatureRouter);
```

---

## Step 3: Implement the Handler

Create `apps/ai-service/src/handlers/my-feature.ts`:

```typescript
import { medplum } from '../lib/medplum';
import { callLLM } from '../lib/llm';
import { sealionGuard } from '../lib/guard';

export async function myFeatureHandler(patientId: string, body: unknown) {
  // 1. Fetch relevant FHIR data
  const conditions = await medplum.search('Condition', {
    patient: `Patient/${patientId}`,
    'clinical-status': 'active',
  });

  // 2. Build the LLM prompt
  const prompt = buildPrompt(conditions, body);

  // 3. Call the LLM
  const rawResponse = await callLLM(prompt);

  // 4. Run the SEA-LION Guard output check
  const guardResult = await sealionGuard.checkOutput(rawResponse, { patientId });
  if (guardResult.decision === 'BLOCK') {
    throw new Error('Response blocked by SEA-LION Guard');
  }

  // 5. Return structured response
  return {
    patientId,
    result: guardResult.content,
    guardDecision: guardResult.decision,
    generatedAt: new Date().toISOString(),
  };
}

function buildPrompt(conditions: fhir4.Bundle, body: unknown): string {
  // Build a structured prompt from FHIR data + request parameters
  return `...`;
}
```

---

## Step 4: Add Guard Configuration

If your feature has specific safety requirements, add a Guard profile in `apps/ai-service/src/lib/guard-profiles.ts`:

```typescript
export const MY_FEATURE_GUARD: GuardProfile = {
  inputChecks: ['injection', 'toxicity', 'pii'],
  outputChecks: ['clinical-harm', 'hallucination'],
  escalationThreshold: 'medium',
};
```

Pass the profile when calling `sealionGuard.checkOutput(response, { profile: MY_FEATURE_GUARD })`.

---

## Step 5: Update the Feature Matrix

Add your feature to {doc}`/ai-agents/features` following the existing format (feature ID, description, input, output, agents used).

---

## Step 6: Connect to the Patient App

In `apps/patient-portal-native/lib/api.ts`, add a typed function:

```typescript
export async function fetchMyFeature(patientId: string, params: MyFeatureParams) {
  const res = await apiClient.post(`/patients/${patientId}/my-feature`, params);
  return res.data as MyFeatureResponse;
}
```

Then call it from a React component or hook in the app:

```typescript
const { data, isLoading } = useQuery({
  queryKey: ['my-feature', patientId],
  queryFn: () => fetchMyFeature(patientId, params),
});
```

---

## Step 7: Test

See {doc}`testing` for the full testing strategy. At minimum:

1. Run the AI service locally and call your endpoint with `curl` or Postman.
2. Verify the FHIR data is fetched correctly.
3. Verify the Guard passes or blocks appropriately.
4. Test the mobile app integration in the Expo simulator.
