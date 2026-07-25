---
name: vitest-playwright-testing
description: "Procedural testing workflow for React frontends: step-by-step guide for Vitest + RTL component tests, MSW network mocking, error/loading state coverage, and Playwright E2E. Use when creating, reviewing, or debugging test files."
---

# Vitest & Playwright React Testing Workflow

Step-by-step procedure for building complete test suites in React + Vite + TypeScript projects. Constraints (testing philosophy, MSW, Playwright) are enforced by `rules/react-testing-trophy.instructions.md` — this skill covers **how** to apply them.

## 1. Test Utility Setup (`src/shared/testing/`)

### renderWithProviders — Base Utility

Every component test wraps in providers. Build the utility once, reuse everywhere.

```typescript
// Path: src/shared/testing/testUtils.tsx
import React, { ReactElement } from 'react';
import { render, RenderOptions, cleanup } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { afterEach } from 'vitest';

afterEach(() => cleanup());

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });

export function renderWithProviders(ui: ReactElement, options?: Omit<RenderOptions, 'wrapper'>) {
  const testQueryClient = createTestQueryClient();
  const result = render(
    <QueryClientProvider client={testQueryClient}>
      {ui}
    </QueryClientProvider>,
    options,
  );
  return { ...result, queryClient: testQueryClient };
}
```

**Key decisions:**
- `retry: false` + `gcTime: 0` — prevents flaky retries and stale cache leaking between tests.
- Returns `queryClient` — enables tests to pre-populate cache or assert cache state.

## 2. MSW Network Mocking

Mock at the network boundary. Never mock your own hooks or utilities.

### MSW Setup

```typescript
// Path: src/shared/testing/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// Path: src/shared/testing/setup.ts
import { beforeAll, afterEach, afterAll } from 'vitest';
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Register in `vitest.config.ts`:
```typescript
// Path: vitest.config.ts
export default defineConfig({
  test: {
    setupFiles: ['./src/shared/testing/setup.ts'],
    environment: 'jsdom',
  },
});
```

### MSW Handler Organization

Group handlers per feature. Export a shared `handlers` array for the server, individual arrays for tests that need custom responses.

```typescript
// Path: src/shared/testing/mocks/handlers/userHandlers.ts
import { http, HttpResponse } from 'msw';
import type { User } from '@/features/users/types';

const mockUser: User = { id: '1', email: 'test@example.com', name: 'Test User' };

export const userHandlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json(mockUser);
  }),

  http.get('/api/users', () => {
    return HttpResponse.json({ data: [mockUser], total: 1 });
  }),
];
```

```typescript
// Path: src/shared/testing/mocks/handlers/index.ts
import { userHandlers } from './userHandlers';
// Import other handler groups as features grow

export const handlers = [...userHandlers];
```

### MSW in Component Tests

```typescript
// Path: src/features/users/components/UserProfile.test.tsx
import { describe, it, expect } from 'vitest';
import { screen, waitFor } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { renderWithProviders } from '@/shared/testing/testUtils';
import { server } from '@/shared/testing/mocks/server';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('displays user data after successful fetch', async () => {
    renderWithProviders(<UserProfile userId="1" />);

    expect(await screen.findByText('Test User')).toBeInTheDocument();
    expect(screen.getByText('test@example.com')).toBeInTheDocument();
  });

  it('shows error state when API returns 500', async () => {
    server.use(
      http.get('/api/users/:id', () => {
        return new HttpResponse(null, { status: 500 });
      }),
    );

    renderWithProviders(<UserProfile userId="1" />);

    expect(await screen.findByRole('alert')).toBeInTheDocument();
    expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
  });
});
```

**Key decisions:**
- `server.use()` in test — overrides global handlers for that test only, reset automatically by `afterEach`.
- `onUnhandledRequest: 'error'` — fails tests that hit unmocked endpoints, catching regressions early.
- Test both happy path AND error state — UI must handle failure gracefully.

### Mocking Mutations (POST/PUT/DELETE)

```typescript
// Path: src/features/users/components/UserForm.test.tsx
import { http, HttpResponse } from 'msw';
import { server } from '@/shared/testing/mocks/server';

it('shows success message after creating user', async () => {
  server.use(
    http.post('/api/users', async ({ request }) => {
      const body = await request.json() as { email: string };
      return HttpResponse.json({ id: '2', ...body }, { status: 201 });
    }),
  );

  // ... fill form and submit ...
  expect(await screen.findByText(/user created/i)).toBeInTheDocument();
});
```

## 3. Component Test Patterns

### Happy Path — User Interactions

Focus on **user behavior** (`userEvent`) and **accessible selectors** (`getByRole`, `getByLabelText`).

```typescript
// Path: src/features/auth/components/LoginForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { renderWithProviders } from '@/shared/testing/testUtils';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits form with valid credentials', async () => {
    const handleSubmit = vi.fn();
    const user = userEvent.setup();

    renderWithProviders(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'user@example.com');
    await user.type(screen.getByLabelText(/password/i), 'Secret123!');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'Secret123!',
    });
  });
});
```

### Loading & Error States

Every data-fetching component MUST test its loading and error UI.

```typescript
// Path: src/features/users/components/UserList.test.tsx
it('shows skeleton while loading', () => {
  renderWithProviders(<UserList />);
  expect(screen.getByTestId('user-list-skeleton')).toBeInTheDocument();
});

it('shows error message on network failure', async () => {
  server.use(
    http.get('/api/users', () => {
      return new HttpResponse(null, { status: 500 });
    }),
  );

  renderWithProviders(<UserList />);
  expect(await screen.findByRole('alert')).toBeInTheDocument();
});
```

### Accessible Selectors Priority

```typescript
// ✅ GOOD — tests accessible behavior
screen.getByRole('button', { name: /submit/i });
screen.getByLabelText(/email/i);
screen.getByText(/welcome/i);

// ❌ BAD — tests implementation details
screen.getByTestId('submit-btn');  // avoid unless no accessible role exists
container.querySelector('.form-input');
```

## 4. E2E Testing with Playwright

Write E2E tests for **critical user journeys only** — authentication, checkout, major CRUD flows.

```typescript
// Path: e2e/auth-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('user can log in and view dashboard', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('testuser@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Log in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: /welcome back/i })).toBeVisible();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('badpassword');
    await page.getByRole('button', { name: 'Log in' }).click();

    await expect(page.getByRole('alert')).toContainText(/invalid credentials/i);
  });
});
```

## 5. Test File Organization

```
src/
├── shared/
│   └── testing/
│       ├── testUtils.tsx          # renderWithProviders
│       ├── setup.ts               # MSW server lifecycle
│       └── mocks/
│           ├── server.ts          # setupServer instance
│           └── handlers/
│               ├── index.ts       # aggregated handlers
│               ├── userHandlers.ts
│               └── orderHandlers.ts
├── features/
│   └── users/
│       ├── components/
│       │   ├── LoginForm.tsx
│       │   └── LoginForm.test.tsx  # co-located with component
│       └── api/
│           └── useUserQuery.ts
e2e/
├── auth-flow.spec.ts              # Playwright E2E
└── checkout.spec.ts
```

**Naming convention:** `test_{component}_{scenario}_{expected}` — e.g., `test_LoginForm_submits_with_valid_credentials`.

## 6. Execution Checklist

1. Unit/component tests: `npx vitest run`
2. Watch mode: `npx vitest`
3. Coverage: `npx vitest run --coverage`
4. Typecheck test code: `npx tsc --noEmit`
5. E2E: `npx playwright test`
6. Lint: `npx eslint src/ e2e/`
