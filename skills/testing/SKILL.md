---
name: testing
description: Frontend Testing Strategy — Vitest unit test configuration and usage, React Testing Library component testing, Playwright E2E testing, mocking strategies (vi.mock/MSW), the testing pyramid, test coverage. Use when writing tests, configuring test environments, or debugging failed tests.
---

# Frontend Testing Strategy

## Testing Pyramid

```
         /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
        /   E2E (Playwright)   \   ← Low quantity, covers core user flows
       /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
      /  Component (RTL + Vitest)  \  ← Medium quantity, covers component interactions
     /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
    /     Unit (Vitest)              \  ← High quantity, covers pure logic
   /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
```

**Division of Responsibilities**:
- **Vitest Unit**: utils, custom Hooks, state stores, pure functions, utility classes.
- **RTL Component Testing**: component rendering, user interactions, Props changes.
- **Playwright E2E**: login flows, access control, critical business flows (ordering, payment, etc.).

---

## Vitest Configuration

```typescript
// Path: vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',         // Simulate browser environment
    globals: true,                // Global describe/it/expect (no import required)
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['src/test/**', 'src/**/*.d.ts', 'src/main.tsx'],
      thresholds: {
        lines: 70,
        functions: 70,
        branches: 60,
      },
    },
  },
});
```

```typescript
// Path: src/test/setup.ts
import '@testing-library/jest-dom'; // Extends matchers: toBeInTheDocument, etc.
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';

// Cleanup after each test
afterEach(cleanup);

// Mock global APIs
Object.defineProperty(window, 'matchMedia', {
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
  })),
});
```

---

## Vitest Unit Testing

```typescript
// Path: src/shared/utils/formatDate.test.ts
import { describe, it, expect } from 'vitest';
import { formatDate } from './formatDate';

describe('formatDate', () => {
  it('formats standard date', () => {
    expect(formatDate(new Date('2024-01-15'))).toBe('2024-01-15');
  });

  it('handles null input', () => {
    expect(formatDate(null)).toBe('—');
  });
});
```

```typescript
// Path: src/features/user/hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('defaults to 0', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('increment increases count', () => {
    const { result } = renderHook(() => useCounter());
    act(() => result.current.increment());
    expect(result.current.count).toBe(1);
  });
});
```

---

## React Testing Library (Component Testing)

```tsx
// Path: src/features/user/components/UserCard.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UserCard } from './UserCard';

const mockUser: User = { id: '1', name: 'Alice', email: 'alice@example.com' };

describe('UserCard', () => {
  it('renders user name', () => {
    render(<UserCard user={mockUser} onSelect={vi.fn()} />);
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });

  it('calls onSelect on click', async () => {
    const onSelect = vi.fn();
    const user = userEvent.setup();

    render(<UserCard user={mockUser} onSelect={onSelect} />);
    await user.click(screen.getByRole('button', { name: /alice/i }));

    expect(onSelect).toHaveBeenCalledWith('1');
    expect(onSelect).toHaveBeenCalledTimes(1);
  });

  it('displays skeleton when loading', () => {
    render(<UserCard user={mockUser} isLoading onSelect={vi.fn()} />);
    expect(screen.getByTestId('skeleton')).toBeInTheDocument();
    expect(screen.queryByText('Alice')).not.toBeInTheDocument();
  });
});
```

---

## Mocking Strategies

### vi.mock (Module Mocking)

```typescript
// Path: src/features/user/hooks/useUser.test.ts
import { vi } from 'vitest';

// Mock the entire module
vi.mock('@/lib/api/client', () => ({
  apiClient: {
    get: vi.fn(),
    post: vi.fn(),
  },
}));

// Mock a single function
vi.mock('./userApi', async importOriginal => ({
  ...(await importOriginal<typeof import('./userApi')>()),
  fetchUser: vi.fn().mockResolvedValue(mockUser),
}));
```

### MSW (Mock Service Worker, recommended for integration testing)

```typescript
// Path: src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Alice' });
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 'new-id', ...body }, { status: 201 });
  }),
];

// Path: src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// Path: src/test/setup.ts (Addition)
import { server } from './mocks/server';
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## Playwright E2E Testing

```typescript
// Path: e2e/auth/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Login', () => {
  test('successfully logs in with correct credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('admin@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Log in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome back')).toBeVisible();
  });

  test('displays error message with wrong credentials', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrongpassword');
    await page.getByRole('button', { name: 'Log in' }).click();

    await expect(page.getByText('Invalid email or password')).toBeVisible();
  });
});
```

```typescript
// Path: playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:3000',
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Test Naming Conventions

| Level | File Location | Naming |
|------|---------|------|
| Unit Test | Same directory as source file | `utils.test.ts` |
| Component Test | Same directory as component | `UserCard.test.tsx` |
| Hook Test | Same directory as Hook | `useUser.test.ts` |
| E2E Test | `e2e/` directory | `login.spec.ts` |

**describe naming**: The module/component name being tested.
**it naming**: `verb + expected behavior` (e.g., "renders user name", "calls onSelect on click").