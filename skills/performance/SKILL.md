---
name: performance
description: React Performance Optimization — Proper usage of React.memo, useMemo/useCallback, code splitting (lazy/Suspense), virtual lists (react-window), avoiding unnecessary re-renders, performance profiling tools. Use when encountering performance issues, optimizing large lists, or reducing render cycles.
---

# React Performance Optimization

## Core Principles

> **Profile before you optimize, do not optimize preemptively.** Overusing `useMemo`/`useCallback` actually adds memory and runtime overhead.

---

## React.memo (Preventing Unnecessary Child Component Re-renders)

**When to use**: The parent component updates frequently, but the child component's props remain unchanged.

```tsx
// ✅ Correct usage of memo
interface UserCardProps {
  user: User;
  onSelect: (id: string) => void;
}

const UserCard = React.memo(({ user, onSelect }: UserCardProps) => {
  return <div onClick={() => onSelect(user.id)}>{user.name}</div>;
});
UserCard.displayName = 'UserCard'; // Helpful for DevTools debugging

// ❌ memo is ineffective in the following cases:
// Parent passes inline object/array (new reference on every render)
<UserCard user={{ id: '1', name: 'Alice' }} onSelect={handleSelect} />
//         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ New object every time, breaking memo

// ✅ Fix: Stable reference in parent component
const user = useMemo(() => ({ id: '1', name: 'Alice' }), []);
<UserCard user={user} onSelect={handleSelect} />
```

---

## useMemo (Memoizing Expensive Calculations)

**When to use**: Computations are expensive (sorting large lists, complex filtering, math operations) and dependencies change infrequently.

```tsx
// ✅ Suitable for useMemo: Sorting and filtering large datasets
const filteredAndSortedUsers = useMemo(
  () => users
    .filter(u => u.name.toLowerCase().includes(searchTerm.toLowerCase()))
    .sort((a, b) => a.name.localeCompare(b.name)),
  [users, searchTerm] // Recompute only when users or searchTerm changes
);

// ❌ Not suitable for useMemo: Simple calculations (not worth the overhead)
const fullName = useMemo(
  () => `${user.firstName} ${user.lastName}`,
  [user.firstName, user.lastName]
); // ← Over-optimization, just write it directly

// ✅ Direct calculation (simple operations do not need memo)
const fullName = `${user.firstName} ${user.lastName}`;
```

---

## useCallback (Stabilizing Function References)

**When to use**: Passing functions as props to `React.memo` child components, or as dependencies in `useEffect`/`useQuery`.

```tsx
// ✅ Suitable for useCallback: Event handlers passed to memoized child components
const handleSelect = useCallback((userId: string) => {
  setSelectedId(userId);
  onUserSelect?.(userId); // Callback prop
}, [onUserSelect]); // Recreate when onUserSelect changes

<MemoizedUserList users={users} onSelect={handleSelect} />

// ❌ Not suitable for useCallback: Simple internal component functions (no memoized child consuming it)
const handleClick = useCallback(() => {
  setCount(c => c + 1);
}, []); // ← Over-optimization, button click does not affect performance
```

---

## Code Splitting (lazy + Suspense)

```tsx
// Path: src/routes/index.tsx
import { lazy, Suspense } from 'react';

// ✅ Route-level code splitting (Mandatory)
const Dashboard = lazy(() => import('@features/dashboard/Dashboard'));
const AdminPanel = lazy(() => import('@features/admin/AdminPanel'));

// ✅ Component-level code splitting (Large libraries: editors, charts)
const RichTextEditor = lazy(() =>
  import('@shared/components/RichTextEditor').then(m => ({ default: m.RichTextEditor }))
);

// ✅ Lazy loading with prefetching (prefetch on hover)
const prefetchDashboard = () => import('@features/dashboard/Dashboard');

const NavLink = () => (
  <Link
    to="/dashboard"
    onMouseEnter={prefetchDashboard} // Hover to prefetch
  >
    Dashboard
  </Link>
);

// ✅ Reasonable Suspense boundaries (Do not make them too granular)
const AppRoutes = () => (
  <Suspense fallback={<PageLoadingSpinner />}>
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/admin" element={<AdminPanel />} />
    </Routes>
  </Suspense>
);
```

---

## Virtual Lists (Extremely Long Lists)

When lists exceed **200+ records** and performance degrades, use virtual scrolling.

```tsx
// Path: src/features/product/components/ProductList.tsx
// npm install react-window
import { FixedSizeList as List } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

interface ProductListProps {
  products: Product[];
}

const ROW_HEIGHT = 72;

const ProductRow = ({
  index,
  style,
  data,
}: {
  index: number;
  style: React.CSSProperties;
  data: Product[];
}) => (
  <div style={style}>
    <ProductCard product={data[index]} />
  </div>
);

const ProductList = ({ products }: ProductListProps) => (
  <AutoSizer>
    {({ height, width }) => (
      <List
        height={height}
        width={width}
        itemCount={products.length}
        itemSize={ROW_HEIGHT}
        itemData={products}
      >
        {ProductRow}
      </List>
    )}
  </AutoSizer>
);
```

---

## Avoiding Common Re-render Traps

```tsx
// ❌ Trap 1: Inline objects/arrays as props
const Parent = () => (
  // style is a new object on every render
  <Child style={{ color: 'red' }} items={[1, 2, 3]} />
);

// ✅ Fix: Move out of component or use useMemo
const CHILD_STYLE = { color: 'red' } as const;
const ITEMS = [1, 2, 3];
const Parent = () => <Child style={CHILD_STYLE} items={ITEMS} />;

// ❌ Trap 2: Context value is an inline object
const MyProvider = ({ children }) => (
  // Creates a new value object on every Provider render, causing all consumers to re-render
  <MyContext.Provider value={{ user, logout }}>
    {children}
  </MyContext.Provider>
);

// ✅ Fix: Stabilize Context value with useMemo
const value = useMemo(() => ({ user, logout }), [user, logout]);
<MyContext.Provider value={value}>{children}</MyContext.Provider>

// ❌ Trap 3: Defining new functions during render
const List = ({ items }) => items.map(item => (
  <Item key={item.id} onClick={() => handleClick(item.id)} /> // New function on every render
));
```

---

## Performance Profile Tools

| Tool | Purpose |
|------|------|
| React DevTools Profiler | Find frequently re-rendering components |
| Chrome Performance Tab | Find long tasks, layout shifts |
| `rollup-plugin-visualizer` | Analyze bundle size composition |
| Lighthouse | Comprehensive performance score (LCP/FID/CLS) |
| `why-did-you-render` | Automatically detect unnecessary re-renders during development |

---

## Optimization Decision Flow

```
Encountering performance issues?
1. React DevTools Profiler → Find frequently re-rendering components
2. Check the component's Props → Are there unstable references (inline objects/functions)?
3. Yes → Stabilize references using useMemo/useCallback or move them outside the component
4. Child component still re-renders → Consider wrapping the child component with React.memo
5. Calculation is very slow → Memoize calculation results with useMemo
6. List is very long (200+ items) → Virtual list with react-window
7. Bundle is too large → Code splitting (lazy)
```