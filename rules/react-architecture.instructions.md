---
name: React Architecture
description: Feature 模块边界、依赖方向与状态分层原则（栈无关，适用于 React 前端实现与规划）
applyTo: "**/*.{ts,tsx,js,jsx}"
---

# React Architecture Principles

这些是**架构原则**，不绑定具体状态库或请求库。具体技术选型以目标仓库的 project-stack instructions 与现有代码为准。

## Module boundaries

- Prefer **feature-based** layout when the project already uses it (e.g. `src/features/*`, `src/shared/*`). Match the repository’s real structure; do not invent a new layout without reason.
- **No direct cross-feature imports.** A feature may only depend on:
  - its own internals
  - shared / common layers
  - third-party packages
- Cross-feature reuse: extract to `shared/` (or the project’s shared layer), or expose a **narrow public API** (barrel / index) and import only that surface — never deep-import another feature’s private files.
- New code should live next to the owning feature. Shared UI/utils only when truly cross-cutting.

## Dependency direction

```
routes/pages → feature public API → shared → third-party
```

- Do not reverse this direction (shared must not import features).
- Prefer existing patterns in neighboring features over introducing a parallel architecture.

## Component design

- Separate **presentational** UI from **container** logic when it improves testability or reuse.
- Extract complex interaction or data orchestration into **custom hooks**.
- Avoid “god components”: split by responsibility (display, form, data load, layout).
- Keep props focused; if props explode, introduce composition, context at the right level, or a dedicated hook — not a grab-bag props object.

## State ownership (categories, not libraries)

Classify every piece of state before choosing a tool:

| Category | Meaning | Typical home |
|----------|---------|--------------|
| **Server state** | Async remote data, cache, invalidation | Project’s data-fetching library (Query/SWR/RTK Query/etc.) |
| **Client global** | Cross-route UI/session state not owned by the server cache | Project’s global store or app-level context |
| **Cross-cutting local** | Shared by a small tree (tabs, wizard, modal host) | Scoped context / composition |
| **Pure local** | One component’s UI | `useState` / `useReducer` |

**Hard rules:**

- Do **not** mirror server data into a global client store as the source of truth.
- Do **not** put ephemeral UI flags into a global store without a clear multi-consumer need.
- Prefer the **simplest** category that works; escalate only when reuse or performance requires it.

## Types and public contracts

- Public feature exports should include stable **types** for consumers.
- Prefer explicit props and return types on public hooks/components.
- Discriminated unions for multi-state UI (idle / loading / success / error) when it clarifies control flow.

## Performance (architecture-level)

- Stable `key`s for lists (identity, not array index, when order can change).
- Code-split heavy routes/features when the project already does so.
- Memoize only with a reason (measured re-renders, expensive derived data, or required referential stability for children/effects). Prefer better state placement over blanket `memo`/`useMemo`.
