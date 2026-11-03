---
name: React Feature Directory & Encapsulation Constraints
description: Architectural constraints for feature-based directory structure (src/features/ and src/shared/)
applyTo: "**"
---

# React Feature Directory & Encapsulation Constraints

Hard rules — enforced for project directory layout, module imports, and architecture encapsulation.

1. **Workspace Directory Structure**:
   - `src/features/[feature-name]/`: Contains domain-specific components, hooks, api, and types.
   - `src/shared/`: Contains domain-agnostic UI primitives, generic hooks, utility functions, and global design tokens.
2. **Encapsulation Rules**:
   - **Cross-Feature Imports Prohibition**: Direct deep imports between features (e.g. `import ... from '../featureA/components/InternalCard'`) are STRICTLY FORBIDDEN.
   - **Public API Barrels**: Features MUST expose public interfaces and components via their root `index.ts` barrel file (`src/features/[feature-name]/index.ts`).
   - If two features require shared business logic or domain types, lift the shared code to `src/shared/` or create a shared domain module.
3. **Internal Feature Layering**:
   Within `src/features/[feature-name]/`:
   - `api/` — API request definitions & Query hooks
   - `components/` — Feature UI components
   - `hooks/` — Custom feature logic hooks
   - `types/` — Domain TypeScript interfaces/types
   - `index.ts` — Explicit public exports
