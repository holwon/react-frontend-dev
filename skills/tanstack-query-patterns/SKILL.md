---
name: tanstack-query-patterns
description: Procedural guide and patterns for TanStack Query (React Query) server state management, query key factories, mutations, optimistic updates, and caching strategies.
---

# TanStack Query Patterns & Best Practices

This skill provides step-by-step procedural workflows for implementing server state management using TanStack Query v5 in React applications.

## 1. Query Key Factory Pattern

Always construct structured query key factories to avoid string key collisions and simplify cache invalidation.

```typescript
// src/features/users/api/userKeys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: Record<string, unknown>) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};
```

## 2. Custom Query Hook Definition

Extract queries into dedicated hooks with explicit types and query key factory references.

```typescript
// src/features/users/api/useUserQuery.ts
import { useQuery } from '@tanstack/react-query';
import { fetchUserById } from './userApi';
import { userKeys } from './userKeys';
import type { User } from '../types';

export function useUserQuery(userId: string) {
  return useQuery<User, Error>({
    queryKey: userKeys.detail(userId),
    queryFn: () => fetchUserById(userId),
    enabled: Boolean(userId),
    staleTime: 1000 * 60 * 5, // 5 minutes stale time
  });
}
```

## 3. Mutation & Optimistic Updates Pattern

When implementing data updates with instant UI feedback:

```typescript
// src/features/users/api/useUpdateUserMutation.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { updateUserApi } from './userApi';
import { userKeys } from './userKeys';
import type { User, UpdateUserDto } from '../types';

export function useUpdateUserMutation(userId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (dto: UpdateUserDto) => updateUserApi(userId, dto),
    onMutate: async (newUserData) => {
      // Cancel outgoing refetches so they don't overwrite optimistic update
      await queryClient.cancelQueries({ queryKey: userKeys.detail(userId) });

      // Snapshot previous value
      const previousUser = queryClient.getQueryData<User>(userKeys.detail(userId));

      // Optimistically update to the new value
      if (previousUser) {
        queryClient.setQueryData<User>(userKeys.detail(userId), {
          ...previousUser,
          ...newUserData,
        });
      }

      return { previousUser };
    },
    onError: (_err, _newUserData, context) => {
      // Rollback on error
      if (context?.previousUser) {
        queryClient.setQueryData(userKeys.detail(userId), context.previousUser);
      }
    },
    onSettled: () => {
      // Invalidate to synchronize with server truth
      queryClient.invalidateQueries({ queryKey: userKeys.detail(userId) });
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}
```
