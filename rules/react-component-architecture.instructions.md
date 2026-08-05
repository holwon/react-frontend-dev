---
name: React Component Architecture & Design Constraints
description: Architectural constraints for React components, Hooks extraction, props boundaries, and React 19 standards
applyTo: "**/*.tsx,**/*.jsx"
---

# React Component Architecture & Design Constraints

Hard rules — enforced every time you design, write, or refactor React components.

1. **Component Separation**: Distinguish strictly between Presentational (UI-only, stateless/local UI state) and Container (smart, hook-integrated) components.
2. **Custom Hooks Extraction**: Extract complex interaction, async, or form logic into custom Hooks (e.g., `use[Feature]Form`, `use[Feature]Actions`). Component files MUST focus primarily on JSX layout.
3. **Props Boundary**: Props interface SHOULD NOT exceed 7 properties. If a component requires more props, group related fields into composite object types or use React Context for deeply nested trees.
4. **React 19 Standards**:
   - Prefer React 19 native primitives (`use`, `useActionState`, `useOptimistic`) where appropriate.
   - Do NOT use legacy lifecycle methods or class components.
   - Access ref as a normal prop where appropriate in React 19, avoiding unnecessary `forwardRef` boilerplate.
5. **Render Performance**:
   - Avoid defining inline object literals or functions as props inside render loops if passed to heavy child components.
   - ALWAYS provide stable, unique `key` props (never use array index as `key` for dynamic re-ordered lists).
