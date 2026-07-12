---
name: react-routing
description: React Routing — React Router v6 Data API (loader/action), TanStack Router, nested routes, route lazy loading, authorization route guards, programmatic navigation. Use when configuring routing, implementing access control, or utilizing route data prefetching.
---

# React Routing

## React Router v6 (Recommended)

### Route Configuration (Data Router Mode)

```tsx
// Path: src/main.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { lazy, Suspense } from 'react';

// Route lazy loading
const Dashboard = lazy(() => import('./features/dashboard/Dashboard'));
const UserList = lazy(() => import('./features/user/UserList'));

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      {
        index: true,
        element: <Suspense fallback={<Spinner />}><Dashboard /></Suspense>,
        loader: dashboardLoader, // Data prefetching
      },
      {
        path: 'users',
        element: <Suspense fallback={<Spinner />}><UserList /></Suspense>,
        loader: usersLoader,
      },
      {
        path: 'users/:userId',
        element: <UserDetail />,
        loader: userDetailLoader,
      },
    ],
  },
  {
    path: '/login',
    element: <LoginPage />,
  },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

### Loader (Data Prefetching)

```tsx
// Path: src/features/user/UserList.loader.ts
import type { LoaderFunctionArgs } from 'react-router-dom';

export const usersLoader = async ({ request }: LoaderFunctionArgs) => {
  const url = new URL(request.url);
  const page = url.searchParams.get('page') ?? '1';
  const data = await api.users.list({ page: parseInt(page) });
  return data; // Return value is retrieved via useLoaderData()
};

// Usage in component
import { useLoaderData } from 'react-router-dom';

const UserList = () => {
  const users = useLoaderData() as User[];
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
};
```

### Action (Form Submission / Data Mutation)

```tsx
// Path: src/features/user/CreateUser.action.ts
import type { ActionFunctionArgs } from 'react-router-dom';

export const createUserAction = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData();
  const name = formData.get('name') as string;
  try {
    await api.users.create({ name });
    return redirect('/users');
  } catch (error) {
    return { error: 'Failed to create user' };
  }
};
```

---

## Access Control Route Guards

```tsx
// Path: src/shared/components/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '@/lib/stores/useAuthStore';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: string;
}

const ProtectedRoute = ({ children, requiredRole }: ProtectedRouteProps) => {
  const location = useLocation();
  const { isAuthenticated, user } = useAuthStore();

  if (!isAuthenticated) {
    // Save the attempted path to redirect after login
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/403" replace />;
  }

  return <>{children}</>;
};

// Usage in route configuration
{
  path: 'admin',
  element: (
    <ProtectedRoute requiredRole="admin">
      <AdminPanel />
    </ProtectedRoute>
  ),
}
```

---

## Nested Layouts

```tsx
// Path: src/layouts/DashboardLayout.tsx
import { Outlet, NavLink } from 'react-router-dom';

const DashboardLayout = () => {
  return (
    <div className="dashboard-layout">
      <nav>
        <NavLink to="/" end>Home</NavLink>
        <NavLink to="/users">Users</NavLink>
      </nav>
      <main>
        <Outlet /> {/* Sub-routes render here */}
      </main>
    </div>
  );
};
```

---

## Programmatic Navigation

```tsx
import { useNavigate, useSearchParams } from 'react-router-dom';

const SearchPage = () => {
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();

  // Navigation
  const goToUser = (id: string) => navigate(`/users/${id}`);
  const goBack = () => navigate(-1);

  // URL search parameters
  const query = searchParams.get('q') ?? '';
  const setQuery = (q: string) => setSearchParams({ q });

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
    />
  );
};
```

---

## Route Parameters and Path Matching

```tsx
import { useParams, useMatch } from 'react-router-dom';

const UserDetail = () => {
  // Route parameters
  const { userId } = useParams<{ userId: string }>();

  // Check if current route matches
  const isExactMatch = useMatch('/users/:userId');

  return <div>User: {userId}</div>;
};
```

---

## TanStack Router (Type-Safe Routing, Alternative)

Suitable for scenarios requiring full type safety for route parameters and search parameters.

```tsx
// Path: src/routes/users.$userId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

const userSearchSchema = z.object({
  tab: z.enum(['profile', 'activity']).default('profile'),
});

export const Route = createFileRoute('/users/$userId')({
  validateSearch: userSearchSchema,
  loader: ({ params: { userId } }) => api.users.get(userId),
  component: UserDetail,
});

function UserDetail() {
  const user = Route.useLoaderData();
  const { tab } = Route.useSearch(); // Fully type-safe
  const { userId } = Route.useParams(); // Fully type-safe
  return <div>{user.name}</div>;
}
```

---

## Common Anti-patterns

| Anti-pattern | Correct Approach |
|--------|---------|
| Fetching route data in component `useEffect` | Use loaders to prefetch data |
| Hardcoded route path strings scattered everywhere | Centralize route constants or use TanStack Router's type-safe routes |
| Directly inlining large amounts of logic in route components | Keep route components thin, placing logic in Hooks and loaders |
| No route lazy loading (all pages bundled together) | Split each page/feature route using `lazy()` |