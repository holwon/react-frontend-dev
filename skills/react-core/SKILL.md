---
name: react-core
description: React Core Concepts — Functional components, built-in Hooks (useState/useEffect/useRef/useMemo/useCallback/useContext), Context API, Error Boundaries, Concurrent Features (Suspense, useTransition, useDeferredValue). Use when designing component structures, using built-in Hooks, or handling async rendering.
---

# React Core

React is a declarative UI library. Components are the fundamental building blocks of a React application.

## Functional Components

Use functional components for all components. Do not use class components.

```tsx
// ✅ Correct: Functional Component
interface UserCardProps {
  userId: string;
  onSelect: (id: string) => void;
}

const UserCard = ({ userId, onSelect }: UserCardProps) => {
  return <div onClick={() => onSelect(userId)}>{userId}</div>;
};
```

**Naming Conventions**: Component names use PascalCase. Props interfaces are named `${ComponentName}Props`.

---

## useState

Manage local component state.

```tsx
// ✅ Typed useState
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// ✅ Functional update (must be used when relying on the previous state)
setCount(prev => prev + 1);

// ❌ Prohibited: Directly mutating state
user.name = 'new name'; // Error!
setUser({ ...user, name: 'new name' }); // ✅ Correct
```

**Array Updates**:
```tsx
// ✅ Add
setItems(prev => [...prev, newItem]);
// ✅ Delete
setItems(prev => prev.filter(item => item.id !== id));
// ✅ Modify
setItems(prev => prev.map(item => item.id === id ? { ...item, ...updates } : item));
```

---

## useEffect

Handle side effects (data fetching, subscriptions, manual DOM manipulation).

```tsx
// ✅ useEffect with cleanup
useEffect(() => {
  const subscription = eventBus.subscribe(handler);
  return () => subscription.unsubscribe(); // Cleanup function
}, [handler]);

// ✅ Dependency array rules
useEffect(() => {
  fetchUser(userId); // Depends on userId
}, [userId]); // Must list all dependencies
```

**Key Rules**:
- Dependency array cannot be omitted (enforced by `eslint-plugin-react-hooks`).
- Directly using `async` inside `useEffect` is prohibited. Use an internal IIFE or named function instead.

```tsx
// ❌ Prohibited
useEffect(async () => { ... }, []);

// ✅ Correct
useEffect(() => {
  const load = async () => {
    const data = await fetchUser(id);
    setUser(data);
  };
  load();
}, [id]);
```

---

## useRef

Hold mutable values or access DOM nodes without triggering re-renders.

```tsx
// DOM reference
const inputRef = useRef<HTMLInputElement>(null);
// Check for null before accessing
inputRef.current?.focus();

// Persist mutable value (does not trigger render)
const timerRef = useRef<ReturnType<typeof setTimeout>>();
timerRef.current = setTimeout(...);
```

---

## useMemo / useCallback

**Only use when there are measured performance issues.** Overuse does more harm than good.

```tsx
// ✅ useMemo: Memoize expensive calculation results
const sortedList = useMemo(
  () => items.sort((a, b) => b.score - a.score),
  [items]
);

// ✅ useCallback: Stabilize function reference (passed to memoized child or used as a useEffect dependency)
const handleSubmit = useCallback(async (data: FormData) => {
  await api.submit(data);
}, [api]);
```

**Decision criteria**: Profile first, then optimize. Do not preventatively memoize everything.

---

## Context API

Share data across the component tree to avoid prop drilling. **Not suitable for highly frequent updates of global state** (use Zustand instead).

```tsx
// 1. Create Context (typed)
interface ThemeContextValue {
  theme: 'light' | 'dark';
  toggle: () => void;
}
const ThemeContext = createContext<ThemeContextValue | null>(null);

// 2. Custom Hook wrapper (forces Provider check)
export const useTheme = (): ThemeContextValue => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
};

// 3. Provider Component
export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const toggle = useCallback(() => setTheme(t => t === 'light' ? 'dark' : 'light'), []);
  return <ThemeContext.Provider value={{ theme, toggle }}>{children}</ThemeContext.Provider>;
};
```

---

## Error Boundaries

Capture rendering errors in the child component tree. **Must be implemented using class components** (React does not provide a functional API for this yet).

```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError(): { hasError: boolean } {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo): void {
    console.error('ErrorBoundary caught:', error, info);
  }

  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}
```

We recommend using the `react-error-boundary` package to simplify this pattern.

---

## Concurrent Features

### Suspense

```tsx
// Code splitting with lazy
const Dashboard = lazy(() => import('./features/dashboard/Dashboard'));

<Suspense fallback={<Spinner />}>
  <Dashboard />
</Suspense>
```

### useTransition

Mark non-urgent updates as interruptible, keeping the UI responsive.

```tsx
const [isPending, startTransition] = useTransition();

const handleSearch = (query: string) => {
  // Urgent: Update input immediately
  setInputValue(query);
  // Non-urgent: Interruptible search result update
  startTransition(() => setSearchQuery(query));
};
```

### useDeferredValue

Delay the update of non-urgent values.

```tsx
const deferredQuery = useDeferredValue(searchQuery);
// Render list using deferredQuery to avoid blocking user input
```

---

## Component Design Principles

| Principle | Description |
|------|------|
| Single Responsibility | Each component should do only one thing |
| Props Count | Do not exceed 7; consider object props or splitting the component if surpassed |
| Side-Effect-Free Render | Render function must be pure; side effects belong in useEffect |
| Downward Data Flow | Data flows down from parent to child; events flow up from child to parent |
| Key Stability | List keys must be stable and unique (disable array indexes unless the list is static) |