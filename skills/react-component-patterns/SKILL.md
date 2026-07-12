---
name: react-component-patterns
description: React Component Design Patterns — Compound Components, Controlled/Uncontrolled Components, Custom Hook extraction, Render Props, Slot Pattern, Higher-Order Components (HOC), Component Composition. Use when designing reusable component libraries or refactoring complex components.
---

# React Component Design Patterns

## Compound Components

Shares implicit state between parent and child components via Context, exposing a self-descriptive sub-component API.

```tsx
// Path: src/shared/components/Tabs/Tabs.tsx
import { createContext, useContext, useState } from 'react';

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

const useTabs = () => {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs subcomponents must be used within <Tabs>');
  return ctx;
};

// Root component
const Tabs = ({ children, defaultTab }: { children: React.ReactNode; defaultTab: string }) => {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
};

// Sub-components
const TabList = ({ children }: { children: React.ReactNode }) => (
  <div role="tablist">{children}</div>
);

const Tab = ({ value, children }: { value: string; children: React.ReactNode }) => {
  const { activeTab, setActiveTab } = useTabs();
  return (
    <button
      role="tab"
      aria-selected={activeTab === value}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
};

const TabPanel = ({ value, children }: { value: string; children: React.ReactNode }) => {
  const { activeTab } = useTabs();
  return activeTab === value ? <div role="tabpanel">{children}</div> : null;
};

// Attach sub-components
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage
<Tabs defaultTab="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
    <Tabs.Tab value="activity">Activity</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="profile"><ProfileContent /></Tabs.Panel>
  <Tabs.Panel value="activity"><ActivityContent /></Tabs.Panel>
</Tabs>
```

---

## Custom Hooks

Extract reusable stateful logic from components.

**Rules**:
- Name must start with `use`.
- Only return data and methods, never return JSX.
- Each custom Hook should be responsible for a single concern.

```tsx
// Path: src/shared/hooks/useLocalStorage.ts
import { useState, useCallback } from 'react';

const useLocalStorage = <T,>(key: string, initialValue: T) => {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    setStoredValue(prev => {
      const resolved = value instanceof Function ? value(prev) : value;
      window.localStorage.setItem(key, JSON.stringify(resolved));
      return resolved;
    });
  }, [key]);

  const removeValue = useCallback(() => {
    window.localStorage.removeItem(key);
    setStoredValue(initialValue);
  }, [key, initialValue]);

  return [storedValue, setValue, removeValue] as const;
};

export default useLocalStorage;
```

---

## Controlled vs Uncontrolled Components

```tsx
// ✅ Controlled Component (state managed by parent)
interface ControlledInputProps {
  value: string;
  onChange: (value: string) => void;
}

const ControlledInput = ({ value, onChange }: ControlledInputProps) => (
  <input value={value} onChange={e => onChange(e.target.value)} />
);

// ✅ Uncontrolled Component (internal state management, optionally exposing default value)
interface UncontrolledInputProps {
  defaultValue?: string;
  onBlur?: (value: string) => void;
}

const UncontrolledInput = ({ defaultValue = '', onBlur }: UncontrolledInputProps) => {
  const ref = useRef<HTMLInputElement>(null);
  return (
    <input
      ref={ref}
      defaultValue={defaultValue}
      onBlur={() => onBlur?.(ref.current?.value ?? '')}
    />
  );
};
```

**Selection Criteria**: Need real-time validation/formatting/linkage? → Controlled component. Simple form, no real-time validation needed? → Uncontrolled component.

---

## Render Props

Pass rendering logic as a prop to achieve logic reuse. (Custom Hooks are generally a better choice in modern React.)

```tsx
// Path: src/shared/components/DataFetcher.tsx
interface DataFetcherProps<T> {
  url: string;
  render: (data: T | null, isLoading: boolean, error: Error | null) => React.ReactNode;
}

const DataFetcher = <T,>({ url, render }: DataFetcherProps<T>) => {
  const { data, isLoading, error } = useQuery({ queryKey: [url], queryFn: () => fetch(url).then(r => r.json()) });
  return <>{render(data ?? null, isLoading, error ?? null)}</>;
};

// Usage
<DataFetcher<User[]>
  url="/api/users"
  render={(users, loading, error) => {
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <UserList users={users ?? []} />;
  }}
/>
```

---

## Slot Pattern (Composition Slots)

Achieve flexible content slots via named props, similar to Vue's slots.

```tsx
// Path: src/shared/components/Card.tsx
interface CardProps {
  header?: React.ReactNode;
  footer?: React.ReactNode;
  children: React.ReactNode;
  className?: string;
}

const Card = ({ header, footer, children, className }: CardProps) => (
  <div className={cn('card', className)}>
    {header && <div className="card__header">{header}</div>}
    <div className="card__body">{children}</div>
    {footer && <div className="card__footer">{footer}</div>}
  </div>
);

// Usage
<Card
  header={<h2>User Profile</h2>}
  footer={<Button>Save</Button>}
>
  <ProfileForm />
</Card>
```

---

## Higher-Order Components (HOC)

Wrap components to add cross-cutting concerns. (Custom Hooks are usually cleaner in modern React.)

```tsx
// Path: src/shared/hocs/withAuth.tsx
const withAuth = <P extends object>(Component: React.ComponentType<P>) => {
  const WithAuth = (props: P) => {
    const isAuthenticated = useAuthStore(s => s.isAuthenticated);
    if (!isAuthenticated) return <Navigate to="/login" replace />;
    return <Component {...props} />;
  };
  WithAuth.displayName = `withAuth(${Component.displayName ?? Component.name})`;
  return WithAuth;
};

// Usage
const ProtectedDashboard = withAuth(Dashboard);
```

---

## Component Composition Decision Tree

```
Need to share implicit state across components?
├─ Yes → Compound Components
└─ No
    Need to reuse stateful logic?
    ├─ Yes → Custom Hook
    └─ No
        Need flexible rendering logic?
        ├─ Yes → Render Props or children as function
        └─ No
            Need flexible content areas?
            ├─ Yes → Slot Pattern (named props)
            └─ No → Simple Composition (props.children)
```