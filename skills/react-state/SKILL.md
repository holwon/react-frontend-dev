---
name: react-state
description: React State Management — State solution decision tree, Zustand / Jotai / Redux Toolkit usage, selector pattern, immutable updates, separation of server state and client state. Use when choosing state libraries, designing global state, or optimizing rendering.
---

# React State Management

## State Categorization and Selection Decision Tree

```
What type of state is it?
│
├─ Server State (Async data from APIs)
│   └─ → TanStack Query or SWR (see data-fetching skill)
│       Never manage server data with Redux/Zustand
│
├─ Client Global State (Shared across components, not from server)
│   ├─ Simple / Atomic state structure → Jotai
│   └─ Complex state structure / Needs actions → Zustand
│
├─ Cross-Component but Local (e.g., Modal toggles, themes)
│   └─ → Context + useContext
│
└─ Purely Local Component State
    └─ → useState / useReducer
```

**Core Principle**: Each category of state is managed by its designated tool; do not mix them.

---

## Zustand

Best for global client state where state structures are complex or require explicit actions.

### Basic Store

```tsx
// Path: src/lib/stores/useAuthStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface User {
  id: string;
  name: string;
  role: string;
}

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  login: (user: User) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set) => ({
        user: null,
        isAuthenticated: false,
        login: (user) => set({ user, isAuthenticated: true }, false, 'auth/login'),
        logout: () => set({ user: null, isAuthenticated: false }, false, 'auth/logout'),
      }),
      { name: 'auth-storage' }
    ),
    { name: 'AuthStore' }
  )
);
```

### Selector Pattern (Preventing Unnecessary Re-renders)

```tsx
// ❌ Subscribing to the entire store (any field change will trigger re-renders)
const { user, isAuthenticated } = useAuthStore();

// ✅ Subscribing only to required fields
const user = useAuthStore(state => state.user);
const isAuthenticated = useAuthStore(state => state.isAuthenticated);

// ✅ Derived data using a selector
const isAdmin = useAuthStore(state => state.user?.role === 'admin');
```

### Slice Pattern (Splitting Large Stores)

```tsx
// Path: src/lib/stores/slices/cartSlice.ts
import type { StateCreator } from 'zustand';

export interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
}

export const createCartSlice: StateCreator<CartSlice> = (set) => ({
  items: [],
  addItem: (item) => set(state => ({ items: [...state.items, item] })),
  removeItem: (id) => set(state => ({ items: state.items.filter(i => i.id !== id) })),
});
```

---

## Jotai

Best for atomic state where states have no strong logical dependencies on each other.

```tsx
// Path: src/lib/atoms/themeAtom.ts
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';
import { atomWithStorage } from 'jotai/utils';

// Base atom
export const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light');

// Derived atom (read-only)
export const isDarkAtom = atom(get => get(themeAtom) === 'dark');

// Writable atom (action)
export const toggleThemeAtom = atom(
  null,
  (get, set) => set(themeAtom, get(themeAtom) === 'light' ? 'dark' : 'light')
);

// Usage
const ThemeToggle = () => {
  const isDark = useAtomValue(isDarkAtom);
  const toggleTheme = useSetAtom(toggleThemeAtom);
  return <button onClick={toggleTheme}>{isDark ? '🌙' : '☀️'}</button>;
};
```

---

## Redux Toolkit

Only consider this for migrating existing Redux projects or for complex middleware requirements (e.g., time-travel debugging). **Prefer Zustand for new projects.**

```tsx
// Path: src/lib/stores/cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CartState {
  items: CartItem[];
}

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] } as CartState,
  reducers: {
    addItem: (state, action: PayloadAction<CartItem>) => {
      // RTK has built-in Immer, allowing direct mutation of draft state
      state.items.push(action.payload);
    },
    removeItem: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(i => i.id !== action.payload);
    },
  },
});

export const { addItem, removeItem } = cartSlice.actions;
export default cartSlice.reducer;
```

---

## useReducer

Best for complex local component state logic (centralizing state transition logic).

```tsx
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset'; payload: number };

const reducer = (state: number, action: Action): number => {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    case 'reset': return action.payload;
    default: return state;
  }
};

const Counter = () => {
  const [count, dispatch] = useReducer(reducer, 0);
  return (
    <>
      <span>{count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'reset', payload: 0 })}>Reset</button>
    </>
  );
};
```

---

## Immutable Update Cheat Sheet

```tsx
// Objects
setState(prev => ({ ...prev, name: 'new' }));

// Arrays - Add
setState(prev => [...prev, newItem]);

// Arrays - Delete
setState(prev => prev.filter(item => item.id !== targetId));

// Arrays - Modify
setState(prev => prev.map(item =>
  item.id === targetId ? { ...item, ...updates } : item
));

// Nested Objects
setState(prev => ({
  ...prev,
  address: { ...prev.address, city: 'Beijing' }
}));
```

---

## Common Anti-patterns

| Anti-pattern | Correct Approach |
|--------|---------|
| Storing API data in Redux/Zustand | Use TanStack Query |
| Modifying the store outside of components | Modify via store actions |
| Selectors returning a new object on every render | Selectors returning primitive values or stable references |
| One massive global store | Split into multiple stores by feature |
| Using Context for high-frequency updates | Replace with Zustand/Jotai |