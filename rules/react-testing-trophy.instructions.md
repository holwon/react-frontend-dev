---
name: React Testing Trophy & MSW Best Practices
description: Enforces modern React testing trophy guidelines with Vitest, React Testing Library, MSW network mocking, and Playwright E2E
applyTo: "**/*.test.tsx,**/*.test.ts,**/*.spec.ts"
---

# React Testing Trophy & MSW Best Practices

Hard rules — enforced for all unit, component, integration, and E2E test suites in React applications.

1. **Testing Trophy Philosophy**:
   - Focus heavily on **Component & Integration Tests** using Vitest + React Testing Library (RTL).
   - Test **user behavior and visible UI outcomes** (e.g. `screen.getByRole`, `userEvent.click`), NEVER internal state variables or private instance methods.
2. **Network Boundary API Mocking (MSW)**:
   - Use **MSW (Mock Service Worker)** to intercept HTTP requests at the network layer.
   - **DO NOT** mock internal hooks or utility functions unless absolutely necessary. Keep mocks focused on external boundaries (APIs, third-party SDKs).
3. **End-to-End (E2E) Verification**:
   - Use **Playwright** for critical end-to-end user flows (e.g., authentication, checkout, major CRUD operations).
   - Do not aim for 100% E2E test coverage; use Vitest + RTL for fast, comprehensive UI coverage.
