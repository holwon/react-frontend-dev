---
name: typescript
description: TypeScript in React Projects — Component Props types, generic components, utility types (Pick/Omit/Partial/Required), type guards, discriminated union, satisfies operator, React built-in types. Use when defining types, using generics, or encountering type errors.
---

# TypeScript (React Projects)

## Basic Configuration

```json
// Path: tsconfig.json (Key configuration options)
{
  "compilerOptions": {
    "strict": true,              // Must be enabled
    "noUncheckedIndexedAccess": true,  // Array access returns T | undefined
    "exactOptionalPropertyTypes": true, // Optional properties cannot be assigned undefined
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "paths": {
      "@/*": ["./src/*"]         // Path alias
    }
  }
}
```

---

## Component Props Types

```tsx
// ✅ Use interface to define Props (Recommended, supports declaration merging)
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';       // Optional prop
  isLoading?: boolean;
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
  children: React.ReactNode;
}

// ✅ Inherit HTML element attributes
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
  // Override base type
  onChange: (value: string) => void;
}

const Input = ({ label, error, onChange, ...rest }: InputProps) => (
  <div>
    <label>{label}</label>
    <input {...rest} onChange={e => onChange(e.target.value)} />
    {error && <span className="error">{error}</span>}
  </div>
);
```

---

## Generic Components

```tsx
// ✅ Generic component (comma after <T,> in .tsx files to avoid JSX conflicts)
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getKey: (option: T) => string | number;
}

const Select = <T,>({ options, value, onChange, getLabel, getKey }: SelectProps<T>) => (
  <select
    value={value ? String(getKey(value)) : ''}
    onChange={e => {
      const selected = options.find(o => String(getKey(o)) === e.target.value);
      if (selected) onChange(selected);
    }}
  >
    {options.map(option => (
      <option key={getKey(option)} value={String(getKey(option))}>
        {getLabel(option)}
      </option>
    ))}
  </select>
);

// Usage (automatically infers T as User)
<Select<User>
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
  getLabel={u => u.name}
  getKey={u => u.id}
/>
```

---

## Utility Types Cheat Sheet

```tsx
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}

// Pick — Select specific fields
type UserSummary = Pick<User, 'id' | 'name'>;

// Omit — Exclude specific fields
type CreateUserDto = Omit<User, 'id' | 'createdAt'>;

// Partial — Make all fields optional
type UpdateUserDto = Partial<Omit<User, 'id' | 'createdAt'>>;

// Required — Make all fields required
type RequiredUser = Required<User>;

// Record — Key-value mapping
type RolePermissions = Record<User['role'], string[]>;

// ReturnType — Extract function return type
const fetchUser = async (id: string): Promise<User> => { /* ... */ return {} as User; };
type FetchUserResult = Awaited<ReturnType<typeof fetchUser>>; // User

// Parameters — Extract function parameter types
type FetchUserParams = Parameters<typeof fetchUser>; // [string]
```

---

## Discriminated Union

```tsx
// ✅ Represent a state machine with a discriminated union
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

const renderContent = (state: AsyncState<User[]>) => {
  switch (state.status) {
    case 'idle': return <p>Ready</p>;
    case 'loading': return <Spinner />;
    case 'success': return <UserList users={state.data} />; // state.data is type-safe
    case 'error': return <ErrorMessage error={state.error} />;
  }
};
```

---

## Type Guards

```tsx
// User-defined type guard
const isUser = (value: unknown): value is User => {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
};

// Usage
const processData = (data: unknown) => {
  if (isUser(data)) {
    console.log(data.name); // Type-safe
  }
};

// instanceof guard
const handleError = (error: unknown) => {
  if (error instanceof Error) {
    console.error(error.message); // Type-safe
  }
};
```

---

## satisfies Operator

Performs type checking while preserving the inferred type.

```tsx
// ❌ as const inference is too narrow to validate value types
const config = {
  theme: 'dark',
  timeout: 3000,
} as const;

// ❌ Type annotations lose inference (config.theme becomes string, losing the literal type)
const config: Record<string, unknown> = {
  theme: 'dark',
  timeout: 3000,
};

// ✅ satisfies: Validates structure while retaining inference
const config = {
  theme: 'dark',      // Inferred as 'dark' instead of string
  timeout: 3000,
} satisfies { theme: 'light' | 'dark'; timeout: number };
```

---

## React Built-in Types Cheat Sheet

```tsx
// Event types
onClick: React.MouseEventHandler<HTMLButtonElement>
onChange: React.ChangeEventHandler<HTMLInputElement>
onSubmit: React.FormEventHandler<HTMLFormElement>
onKeyDown: React.KeyboardEventHandler<HTMLInputElement>

// Child component types
children: React.ReactNode        // Most generic: any renderable content
children: React.ReactElement     // Must be a React element
children: JSX.Element            // Single JSX element (same as ReactElement)

// Ref types
ref: React.RefObject<HTMLDivElement>
ref: React.MutableRefObject<number>

// forwardRef
const Input = React.forwardRef<HTMLInputElement, InputProps>((props, ref) => (
  <input ref={ref} {...props} />
));

// CSS style types
style: React.CSSProperties
className: string

// Component types
const MyComponent: React.FC<Props> = () => { ... }  // Not recommended (implicit children)
const MyComponent = (props: Props): React.ReactElement => { ... } // Recommended
```

---

## Prohibited Patterns

| Prohibited | Reason | Alternative |
|------|------|------|
| `any` | Bypasses type checking | `unknown` + type guard |
| `// @ts-ignore` | Silences/hides errors | `// @ts-expect-error` + explanatory comment |
| Type assertions (`as`) without validation | Risk of runtime crashes | Type guards or `satisfies` |
| Async functions without explicit return types | Implicit `Promise<any>` | Explicit `Promise<User>` |
| Enums (`enum`) | Runtime overhead, contrary to TS design principles | `as const` object or union literal types |