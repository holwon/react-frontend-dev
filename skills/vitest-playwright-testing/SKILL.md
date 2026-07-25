---
name: vitest-playwright-testing
description: Procedural guide and patterns for testing React applications using Vitest + React Testing Library (unit/component) and Playwright (E2E).
---

# Vitest & Playwright React Testing Workflow

This skill outlines testing standards and patterns for React frontend applications using Vitest and Playwright.

## 1. Unit & Component Testing with Vitest + React Testing Library

### Setup & Render Utility Pattern
Always render components wrapped in required providers (QueryClient, Router, Theme).

```typescript
// src/shared/testing/testUtils.tsx
import React, { ReactElement } from 'react';
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });

export function renderWithProviders(ui: ReactElement, options?: Omit<RenderOptions, 'wrapper'>) {
  const testQueryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={testQueryClient}>
      {ui}
    </QueryClientProvider>,
    options
  );
}
```

### Component Test Case Example
Focus on user interactions (`userEvent`) and accessible query selectors (`getByRole`).

```typescript
// src/features/auth/components/LoginForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { renderWithProviders } from '@/shared/testing/testUtils';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits form with valid user credentials', async () => {
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

## 2. E2E Testing with Playwright

Write user-centric end-to-end tests for critical user journeys.

```typescript
// e2e/auth-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('user can log in successfully and view dashboard', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('testuser@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Log in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: /welcome back/i })).toBeVisible();
  });
});
```
