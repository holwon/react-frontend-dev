---
name: code-standards
description: Frontend Coding Standards — ESLint configuration, Prettier settings, file and directory naming conventions, import order, component file structure, TypeScript strict mode conventions, Git commit standards. Use when establishing project standards, conducting code reviews, or configuring linting tools.
---

# Frontend Coding Standards

## ESLint Configuration

```javascript
// Path: eslint.config.js (ESLint v9 Flat Config)
import js from '@eslint/js';
import globals from 'globals';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['dist', 'coverage'] },
  {
    extends: [
      js.configs.recommended,
      ...tseslint.configs.strictTypeChecked,
      ...tseslint.configs.stylisticTypeChecked,
    ],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2022,
      globals: globals.browser,
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      // React Hooks rules (Required)
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': 'warn',

      // TypeScript strict rules
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/no-non-null-assertion': 'warn',
      '@typescript-eslint/prefer-nullish-coalescing': 'error',
      '@typescript-eslint/prefer-optional-chain': 'error',

      // Best practices
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
      'no-var': 'error',
    },
  }
);
```

---

## Prettier Configuration

```json
// Path: .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

---

## Naming Conventions

### Files and Directories

| Type | Convention | Example |
|------|------|------|
| Component File | PascalCase | `UserCard.tsx` |
| Hook File | camelCase | `useAuthStore.ts` |
| Utility Function | camelCase | `formatDate.ts` |
| Type Definition | camelCase | `userTypes.ts` |
| Test File | Base file name + `.test` | `UserCard.test.tsx` |
| Style File | Component name + `.module.css` | `UserCard.module.css` |
| Constant File | camelCase | `apiEndpoints.ts` |
| Directory Name | kebab-case | `user-management/` |

### Variables and Functions

```typescript
// Variables: camelCase
const userName = 'Alice';
const isLoading = false;

// Constants (immutable values): SCREAMING_SNAKE_CASE
const MAX_RETRY_COUNT = 3;
const API_BASE_URL = '/api/v1';

// Boolean variables: Start with is/has/can/should
const isVisible = true;
const hasPermission = false;
const canEdit = true;

// Event handlers: handle + Verb OR on + Noun
const handleSubmit = () => {};
const handleUserSelect = (id: string) => {};

// Async functions: Verb + Noun
const fetchUser = async (id: string): Promise<User> => {};
const createOrder = async (data: CreateOrderDto): Promise<Order> => {};
```

---

## Component File Structure (Standard Order)

```tsx
// Path: src/features/user/components/UserProfile.tsx

// 1. External library imports (alphabetical order)
import { memo, useState, useCallback } from 'react';
import { useNavigate } from 'react-router-dom';

// 2. Internal @/ aliased imports (furthest to closest)
import { useAuthStore } from '@/lib/stores/useAuthStore';
import { Button } from '@/shared/components/Button';
import { formatDate } from '@/shared/utils/formatDate';

// 3. Relative imports (sibling/child)
import { useUserPermissions } from '../hooks/useUserPermissions';
import type { User } from '../types/userTypes';
import styles from './UserProfile.module.css';

// 4. Type/Interface definitions
interface UserProfileProps {
  userId: string;
  onUpdate?: (user: User) => void;
}

// 5. Constants (static values used inside the component)
const AVATAR_SIZE = 64;

// 6. Component implementation
const UserProfile = ({ userId, onUpdate }: UserProfileProps) => {
  // 6a. Hooks (in order of usage)
  const navigate = useNavigate();
  const { user: currentUser } = useAuthStore();
  const { data: user, isLoading } = useUser(userId);
  const [isEditing, setIsEditing] = useState(false);

  // 6b. Derived data / computations
  const canEdit = currentUser?.id === userId;

  // 6c. Event handlers
  const handleEdit = useCallback(() => setIsEditing(true), []);
  const handleCancel = useCallback(() => setIsEditing(false), []);

  // 6d. Conditional rendering (loading/error/empty)
  if (isLoading) return <ProfileSkeleton />;
  if (!user) return null;

  // 6e. Main render
  return (
    <div className={styles.profile}>
      {/* JSX */}
    </div>
  );
};

// 7. displayName (when using memo)
UserProfile.displayName = 'UserProfile';

// 8. Default export (end of file)
export default memo(UserProfile);
```

---

## Import Order (Enforced by ESLint)

```typescript
// 1. Node.js built-in modules (rarely used in frontend)

// 2. External npm packages (react first)
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

// 3. Internal @/ aliases (in order: @/lib → @/shared → @/features)
import { queryClient } from '@/lib/queryClient';
import { Button } from '@/shared/components/Button';
import { UserCard } from '@/features/user/components/UserCard';

// 4. Relative paths (../parent → ./sibling → ./child)
import { useUser } from '../hooks/useUser';
import styles from './Component.module.css';

// 5. Type imports (last)
import type { User } from '@/types';
```

---

## Feature Module Internal Structure

```
src/features/user/
├── components/          ← UI components
│   ├── UserCard.tsx
│   ├── UserCard.module.css
│   ├── UserCard.test.tsx
│   ├── UserList.tsx
│   └── UserForm.tsx
├── hooks/               ← Custom Hooks
│   ├── useUser.ts
│   ├── useUser.test.ts
│   └── useUserPermissions.ts
├── api/                 ← API call layer
│   └── userApi.ts
├── types/               ← Module-specific type definitions
│   └── userTypes.ts
└── index.ts             ← Public API (barrel export)
```

**Barrel Export Convention**:
```typescript
// Path: src/features/user/index.ts
// Export only what is needed by other modules (avoid export * from)
export { UserCard } from './components/UserCard';
export { UserList } from './components/UserList';
export { useUser } from './hooks/useUser';
export type { User, CreateUserDto } from './types/userTypes';
// Do not export internal implementation details
```

---

## Git Commit Conventions (Conventional Commits)

```
<type>(<scope>): <subject>

feat(user): add user avatar upload feature
fix(auth): fix issue where expired token didn't redirect to login
refactor(product): split ProductList into presentational and container components
style(button): adjust hover color for Button component
test(user): add click test cases for UserCard
chore(deps): upgrade @tanstack/react-query to v5
docs(readme): update local development startup instructions
```

| Type | Description |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code refactoring (no functional changes) |
| `style` | Formatting and styling changes only |
| `test` | Add or modify tests |
| `chore` | Build, dependencies, tools, etc. |
| `docs` | Documentation changes |
| `perf` | Performance optimization |
