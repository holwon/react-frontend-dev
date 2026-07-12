---
name: data-fetching
description: Frontend Data Fetching — TanStack Query (useQuery/useMutation/useInfiniteQuery), SWR, caching strategies, optimistic updates, error boundaries, request cancellation, data prefetching. Use when handling API requests, managing server state, or implementing optimistic UI.
---

# Data Fetching

## TanStack Query (Recommended)

### Basic Setup

```tsx
// Path: src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // Data is considered fresh for 5 minutes
      gcTime: 10 * 60 * 1000,      // GC after 10 minutes (formerly cacheTime)
      retry: 1,                     // Retry 1 time on failure
      refetchOnWindowFocus: false,  // Do not automatically refetch on window focus
    },
    mutations: {
      retry: 0,                     // Do not retry mutations
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
      {import.meta.env.DEV && <ReactQueryDevtools />}
    </QueryClientProvider>
  );
}
```

### Query Key Conventions

```typescript
// Path: src/lib/queryKeys.ts
// Centralize all query keys to avoid scattered strings
export const queryKeys = {
  users: {
    all: ['users'] as const,
    list: (params: UserListParams) => ['users', 'list', params] as const,
    detail: (id: string) => ['users', 'detail', id] as const,
  },
  products: {
    all: ['products'] as const,
    list: (filters: ProductFilters) => ['products', 'list', filters] as const,
    detail: (id: string) => ['products', 'detail', id] as const,
  },
} as const;
```

### useQuery

```tsx
// Path: src/features/user/hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';

const useUser = (userId: string) => {
  return useQuery({
    queryKey: queryKeys.users.detail(userId),
    queryFn: async () => {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) throw new Error('Failed to fetch user');
      return res.json() as Promise<User>;
    },
    enabled: !!userId,   // Do not fetch when userId is empty
    staleTime: Infinity, // User details are valid for a long time
  });
};

// Usage in component
const UserDetail = ({ userId }: { userId: string }) => {
  const { data: user, isLoading, error } = useUser(userId);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!user) return null;

  return <div>{user.name}</div>;
};
```

### useMutation (Data Mutation)

```tsx
// Path: src/features/user/hooks/useUpdateUser.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

const useUpdateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, data }: { id: string; data: UpdateUserDto }): Promise<User> => {
      const res = await fetch(`/api/users/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      if (!res.ok) throw new Error('Update failed');
      return res.json();
    },

    // Optimistic update
    onMutate: async ({ id, data }) => {
      // Cancel in-flight queries
      await queryClient.cancelQueries({ queryKey: queryKeys.users.detail(id) });
      // Save old data (for rollback)
      const previousUser = queryClient.getQueryData<User>(queryKeys.users.detail(id));
      // Optimistically update cache
      queryClient.setQueryData<User>(queryKeys.users.detail(id), old =>
        old ? { ...old, ...data } : old
      );
      return { previousUser };
    },

    onError: (err, { id }, context) => {
      // Rollback on error
      if (context?.previousUser) {
        queryClient.setQueryData(queryKeys.users.detail(id), context.previousUser);
      }
    },

    onSettled: (_, __, { id }) => {
      // Invalidate cache on success/failure to refetch the latest data
      queryClient.invalidateQueries({ queryKey: queryKeys.users.detail(id) });
      queryClient.invalidateQueries({ queryKey: queryKeys.users.all });
    },
  });
};
```

### Infinite Scrolling (useInfiniteQuery)

```tsx
const useInfiniteUsers = (filters: UserFilters) => {
  return useInfiniteQuery({
    queryKey: queryKeys.users.list(filters),
    queryFn: async ({ pageParam }) => {
      const res = await fetch(`/api/users?page=${pageParam}&...`);
      return res.json() as Promise<PaginatedResult<User>>;
    },
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.hasNext ? lastPage.currentPage + 1 : undefined,
  });
};
```

---

## API Layer Encapsulation

```typescript
// Path: src/lib/api/client.ts
// Unified fetch wrapper handling authentication, errors, and base URL
const apiClient = {
  async get<T>(path: string, options?: RequestInit): Promise<T> {
    const res = await fetch(`/api${path}`, {
      ...options,
      headers: {
        Authorization: `Bearer ${getToken()}`,
        ...options?.headers,
      },
    });
    if (!res.ok) {
      const error = await res.json().catch(() => ({ message: res.statusText }));
      throw new ApiError(res.status, error.message);
    }
    return res.json();
  },

  async post<T>(path: string, body: unknown): Promise<T> {
    return apiClient.get<T>(path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
  },
};

// Path: src/features/user/api/userApi.ts
export const userApi = {
  list: (params: UserListParams): Promise<PaginatedResult<User>> =>
    apiClient.get(`/users?${new URLSearchParams(params as any)}`),
  get: (id: string): Promise<User> =>
    apiClient.get(`/users/${id}`),
  create: (data: CreateUserDto): Promise<User> =>
    apiClient.post('/users', data),
};
```

---

## Caching Strategy Cheat Sheet

| Scenario | staleTime | Description |
|------|-----------|------|
| Real-time data (prices, inventory) | `0` | Refetch on every render |
| User info | `5 * 60 * 1000` | 5 minutes cache |
| Static config (permissions, menus) | `Infinity` | Unchanged during session |
| List pages | `30 * 1000` | 30 seconds, data changes relatively frequently |

---

## Error Handling

```tsx
// Path: src/shared/components/QueryErrorBoundary.tsx
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

const QueryErrorBoundary = ({ children }: { children: React.ReactNode }) => (
  <QueryErrorResetBoundary>
    {({ reset }) => (
      <ErrorBoundary
        onReset={reset}
        fallbackRender={({ error, resetErrorBoundary }) => (
          <div>
            <p>Something went wrong: {error.message}</p>
            <button onClick={resetErrorBoundary}>Retry</button>
          </div>
        )}
      >
        {children}
      </ErrorBoundary>
    )}
  </QueryErrorResetBoundary>
);
```

---

## Common Anti-patterns

| Anti-pattern | Correct Approach |
|--------|---------|
| Manually fetching in `useEffect` and storing in Zustand | Use TanStack Query to manage server state |
| Each component independently fetching the same data | Same queryKey automatically shares cache |
| Writing fetch URLs directly inside components | Encapsulate into `src/features/*/api/` modules |
| Forgetting to handle loading and error states | Always handle `isLoading` and `isError` |
| Not invalidating cache after a mutation | Call `invalidateQueries` in `onSettled` |