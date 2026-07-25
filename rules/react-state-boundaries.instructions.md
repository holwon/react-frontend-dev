---
name: React State Management & Layer Boundaries
description: Enforces strict state layer boundaries between Server State, Global Client State, and Local Component State
applyTo: "**/*.ts,**/*.tsx"
---

# React State Management & Layer Boundaries

Hard rules — enforced every time state management or data fetching is introduced.

1. **State Categorization & Strict Tooling Assignment**:
   - **Server State**: MUST be managed using TanStack Query (React Query) or SWR. Covers API responses, caching, polling, and invalidation.
   - **Global Client State**: MUST be managed using Zustand or Jotai. Covers app themes, active user session UI state, sidebar toggles, modal queues.
   - **Local Component State**: MUST use native `useState` / `useReducer` or React Context (for shallow subtree state).
2. **FORBIDDEN State Anti-Patterns**:
   - **NEVER** mirror server data inside Zustand / Redux stores. Query cache is the single source of truth for server state.
   - **NEVER** use global Redux / Redux Toolkit for pure server API data fetching.
   - **NEVER** put non-serializable objects (DOM nodes, class instances with methods) into Zustand or React Query state.
3. **Data Fetching Hooks**:
   - Custom Query hooks MUST follow naming pattern `use[Entity]Query` or `use[Action][Entity]Mutation`.
   - Always define query key factories (e.g. `userKeys.detail(id)`) to prevent key collisions during cache invalidation.
