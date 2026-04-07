# Testing

This page describes the testing strategy for Med-SEAL Suite and how to run tests for each service.

---

## Overview

| Component | Test Type | Tool |
|---|---|---|
| AI Service | Unit + integration | Jest + Supertest |
| Patient Portal Native | Component + E2E | Jest + Detox |
| AI Frontend | Component | Vitest + React Testing Library |
| FHIR integration | Integration | Medplum test server |
| Full stack | Smoke tests | Docker Compose |

---

## AI Service Tests

### Run All Tests

```bash
cd apps/ai-service
npm test
```

### Run with Coverage

```bash
npm run test:coverage
```

### Unit Tests

Unit tests live in `apps/ai-service/src/**/__tests__/`. They mock FHIR calls and the LLM client:

```typescript
// handler.test.ts
import { myFeatureHandler } from '../handlers/my-feature';

jest.mock('../lib/medplum', () => ({
  medplum: {
    search: jest.fn().mockResolvedValue({ entry: [] }),
  },
}));

jest.mock('../lib/llm', () => ({
  callLLM: jest.fn().mockResolvedValue('Test response'),
}));

describe('myFeatureHandler', () => {
  it('returns a structured response', async () => {
    const result = await myFeatureHandler('patient-123', {});
    expect(result).toHaveProperty('result');
    expect(result.guardDecision).toBe('PASS');
  });
});
```

### Integration Tests

Integration tests in `apps/ai-service/src/__integration__/` run against the full Docker stack. They require the stack to be running:

```bash
# Start the stack first
docker compose up -d

# Then run integration tests
npm run test:integration
```

---

## Patient Portal Native Tests

### Unit and Component Tests

```bash
cd apps/patient-portal-native
npm test
```

Component tests use Jest and React Native Testing Library:

```typescript
import { render, fireEvent } from '@testing-library/react-native';
import { MedicationCard } from '../components/MedicationCard';

it('calls onConfirm when tapped', () => {
  const onConfirm = jest.fn();
  const { getByText } = render(
    <MedicationCard medication={mockMed} onConfirm={onConfirm} />
  );
  fireEvent.press(getByText('Mark as Taken'));
  expect(onConfirm).toHaveBeenCalledTimes(1);
});
```

### E2E Tests (Detox)

E2E tests require a running simulator:

```bash
# Build the test app
npx detox build --configuration ios.sim.debug

# Run E2E tests
npx detox test --configuration ios.sim.debug
```

---

## Testing FHIR Integrations

For isolated FHIR testing, use Medplum's in-memory test client:

```typescript
import { MockClient } from '@medplum/mock';

const medplum = new MockClient();

// Seed test data
await medplum.createResource({
  resourceType: 'Patient',
  id: 'test-patient',
  name: [{ family: 'Doe', given: ['John'] }],
});

// Your code under test
const result = await fetchPatientMedications(medplum, 'test-patient');
expect(result).toHaveLength(0);
```

`@medplum/mock` is an in-memory FHIR server that requires no external services.

---

## Smoke Testing the Full Stack

After `docker compose up -d`, run the built-in smoke check:

```bash
node scripts/smoke-test.js
```

This script checks:

1. Medplum FHIR API responds at `localhost:8103`
2. OpenEMR is reachable at `localhost:8081`
3. AI Service `/health` returns `ok`
4. SSO database is accessible
5. A test patient can be created and retrieved via FHIR

---

## Continuous Integration

The repository uses GitHub Actions for CI. The workflow runs on every pull request:

```yaml
# .github/workflows/ci.yml
- npm install && npm test   # AI Service
- npm install && npm test   # AI Frontend
- npm install && npm test   # Patient Portal Native
```

All tests must pass before a PR can be merged.
