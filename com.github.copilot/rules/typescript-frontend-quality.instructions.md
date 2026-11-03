---
name: TypeScript & Frontend Code Quality Constraints
description: Enforces TypeScript strict mode rules, zero placeholder code, and safe dependencies
applyTo: "**/*.ts,**/*.tsx"
---

# TypeScript & Frontend Code Quality Constraints

Hard rules — enforced every time TypeScript code is generated, edited, or reviewed.

1. **Strict Type Safety**:
   - **NO `any` TYPE**: Use explicit TypeScript interfaces, generic constraints, or `unknown` with type guards.
   - **Explicit Return Types**: All exported functions, custom hooks, and async handlers MUST state explicit return types.
   - **Event Handler Typing**: Use React's exact event types (e.g. `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>`).
2. **100% Complete & Production-Ready Code**:
   - **NO Dummy / Placeholder Code**: Do NOT output truncated code containing `// TODO`, `// ... existing code`, or empty handler stubs.
   - All imports MUST be valid and resolved.
3. **Dependency & Package Safety**:
   - Verify package names and API compatibility before suggesting third-party npm packages.
   - Prefer modern standard packages (e.g., TanStack Query v5+, Zustand v4+, Zod v3+, Vitest v2+, Vite v6+).
