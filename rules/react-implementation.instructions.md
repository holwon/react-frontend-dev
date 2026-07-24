---
name: React Implementation
description: 实现质量约束 — 完整代码、路径标注、TypeScript 严格性、验证与依赖引入
applyTo: "**/*.{ts,tsx,js,jsx}"
---

# React Implementation Standards

## Completeness

- Deliver **100% complete, runnable** code for the requested change.
- **Forbidden**: `// TODO`, `// ... existing code`, stub returns, placeholder handlers that do nothing when a real behavior is required.
- Prefer editing real files in the workspace over dumping large unrelated rewrites.

## File path headers

When showing code for the IDE to apply, start the block with an exact path comment:

```ts
// Path: src/features/user/components/UserCard.tsx
```

Use the repository’s real path style (`@/` aliases only if the project already has them).

## TypeScript

- Strict mindset: no `any`. Prefer `unknown` + narrowing when input is untrusted.
- Async functions that return data should have **explicit** `Promise<...>` (or inferred equivalent that remains precise).
- Use React’s event and node types (`React.MouseEvent`, `React.ReactNode`, etc.) instead of loose `Function` / `object`.
- Prefer `interface` for object props shapes; use `type` for unions, intersections, and mapped utilities — or match the file’s existing style.

## Align with the repository

- Mirror **import order**, naming, test layout, and folder conventions already present in the touched feature.
- Reuse existing components, hooks, and utilities before adding new abstractions.
- New npm dependencies: only when necessary; verify package name/usage (e.g. via docs research) and justify briefly. Prefer packages already in `package.json`.

## Errors, loading, empty

For user-facing data views and forms, handle **loading / error / empty** (or disabled/submitting) states unless the surrounding pattern intentionally omits them.

## Verification

After non-trivial changes, state how to verify:

- Project test commands (unit / component / e2e as applicable)
- Typecheck / lint if the repo uses them
- Manual UI path for interactive behavior

Do not claim tests pass unless they were actually run (via TestRunner or equivalent).

## Language in code

- Code, identifiers, and comments in source files: follow the **repository’s** language convention (usually English for code).
- Chat replies to the user: Simplified Chinese per the agent persona, with English technical terms unchanged.
