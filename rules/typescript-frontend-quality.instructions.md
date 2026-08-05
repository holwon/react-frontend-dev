---
name: TypeScript & Frontend Code Quality Constraints
description: Enforces TypeScript strict mode rules, zero placeholder code, and safe dependencies
applyTo: "**/*.ts,**/*.tsx"
---

# TypeScript & Frontend Code Quality Constraints

Hard rules — enforced every time TypeScript code is generated, edited, or reviewed.

1. **Strict Type Safety**:
   - **NO `any` TYPE**: Use explicit TypeScript interfaces, generic constraints, or `unknown` with type guards. `any` disables type checking entirely, silently hiding bugs until runtime.
   - **Explicit Return Types**: All exported functions, custom hooks, and async handlers MUST state explicit return types. This is the public contract other modules compile against.
   - **Event Handler Typing**: Use React's exact event types (e.g. `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>`). Native event types are too broad and force unsafe casts at call sites.
2. **Dependency & Package Safety**:
   - Verify package names and API compatibility before suggesting third-party npm packages. A wrong name or incompatible major version wastes an entire install-fix loop.
   - Prefer modern standard packages (e.g., TanStack Query v5+, Zustand v4+, Zod v3+, Vitest v2+, Vite v6+) — older majors miss performance fixes and React 19 compatibility.
