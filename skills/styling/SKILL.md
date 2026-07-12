---
name: styling
description: Frontend Styling Solutions — CSS Modules, Tailwind CSS, CSS-in-JS (styled-components/emotion), CSS variables and design tokens, responsive layout, animations/transitions. Use when implementing UI styles, building design systems, or handling responsive layouts.
---

# Frontend Styling Solutions

## Selection Decision Tree

```
Which component library does the project use?
│
├─ No component library (Building a design system from scratch)
│   └─ CSS Modules + CSS Variables (Recommended)
│       Or Tailwind CSS (For rapid development)
│
├─ MUI / Ant Design / Chakra UI (With built-in theme system)
│   └─ Use the component library's built-in theme/styling override mechanism
│       Do not mix CSS Modules to override component library styles
│
└─ shadcn/ui (Headless + Tailwind)
    └─ Tailwind CSS (Built-in to the project)
```

---

## CSS Modules (Default Recommendation)

Zero runtime overhead, automatically scoped class names, and works excellently with TypeScript.

```css
/* Path: src/features/user/components/UserCard.module.css */
.card {
  padding: var(--spacing-4);
  border-radius: var(--radius-md);
  background: var(--color-surface);
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.2s ease;
}

.card:hover {
  box-shadow: var(--shadow-md);
}

.card__header {
  display: flex;
  align-items: center;
  gap: var(--spacing-2);
  margin-bottom: var(--spacing-3);
}

.card--active {
  border: 2px solid var(--color-primary);
}
```

```tsx
// Path: src/features/user/components/UserCard.tsx
import styles from './UserCard.module.css';
import { clsx } from 'clsx'; // Recommended: Conditional class name merging

interface UserCardProps {
  user: User;
  isActive?: boolean;
}

const UserCard = ({ user, isActive }: UserCardProps) => (
  <div className={clsx(styles.card, isActive && styles['card--active'])}>
    <div className={styles.card__header}>
      <Avatar src={user.avatar} />
      <h3>{user.name}</h3>
    </div>
  </div>
);
```

---

## CSS Variables and Design Tokens

```css
/* Path: src/styles/tokens.css */
:root {
  /* Color Palette */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-primary-light: #eff6ff;
  --color-danger: #dc2626;
  --color-success: #16a34a;
  --color-warning: #d97706;

  /* Semantic Colors */
  --color-text-primary: #111827;
  --color-text-secondary: #6b7280;
  --color-surface: #ffffff;
  --color-border: #e5e7eb;
  --color-background: #f9fafb;

  /* Spacing (4px base) */
  --spacing-1: 0.25rem;   /* 4px */
  --spacing-2: 0.5rem;    /* 8px */
  --spacing-3: 0.75rem;   /* 12px */
  --spacing-4: 1rem;      /* 16px */
  --spacing-6: 1.5rem;    /* 24px */
  --spacing-8: 2rem;      /* 32px */

  /* Border Radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);

  /* Fonts */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;

  /* Font Sizes */
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-base: 250ms ease;
}

/* Dark Theme */
[data-theme='dark'] {
  --color-text-primary: #f9fafb;
  --color-text-secondary: #9ca3af;
  --color-surface: #1f2937;
  --color-border: #374151;
  --color-background: #111827;
}
```

---

## Tailwind CSS

Use when the project utilizes shadcn/ui or Tailwind CSS is explicitly chosen.

```tsx
// ✅ Use the cn() utility function to merge class names (tailwind-merge + clsx)
import { cn } from '@/lib/utils';

interface ButtonProps {
  variant?: 'primary' | 'secondary';
  size?: 'sm' | 'md' | 'lg';
  className?: string;
  children: React.ReactNode;
}

const Button = ({ variant = 'primary', size = 'md', className, children }: ButtonProps) => {
  return (
    <button
      className={cn(
        // Base styles
        'inline-flex items-center justify-center rounded-md font-medium transition-colors',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2',
        // Variants
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'secondary' && 'border border-gray-300 bg-white hover:bg-gray-50',
        // Sizes
        size === 'sm' && 'h-8 px-3 text-sm',
        size === 'md' && 'h-10 px-4 text-sm',
        size === 'lg' && 'h-12 px-6 text-base',
        // External overrides
        className
      )}
    >
      {children}
    </button>
  );
};
```

```typescript
// Path: src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export const cn = (...inputs: ClassValue[]) => twMerge(clsx(inputs));
```

---

## Responsive Layout

```css
/* Breakpoints (Mobile First) */
/* sm: 640px  md: 768px  lg: 1024px  xl: 1280px */

.grid {
  display: grid;
  grid-template-columns: 1fr;        /* Mobile: Single column */
  gap: var(--spacing-4);
}

@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr); /* Tablet: Two columns */
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr); /* Desktop: Three columns */
  }
}
```

---

## Animation and Transitions

```css
/* Recommended: Only animate transform and opacity (GPU accelerated, does not trigger reflow) */
.fade-in {
  animation: fadeIn var(--transition-base) ease forwards;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Respect the user's preference for reduced motion */
@media (prefers-reduced-motion: reduce) {
  .fade-in {
    animation: none;
  }
}
```

---

## Style Guidelines

| Rule | Description |
|------|------|
| Use CSS Variables | Do not hardcode values for colors, spacing, and border-radii |
| BEM Naming (CSS Modules) | `.block__element--modifier` |
| Prohibit `!important` | Resolve override issues by increasing selector specificity |
| Avoid Inline styles (except for dynamic values) | Use CSS classes or CSS variables instead |
| Mobile First | Write styles for mobile first, then extend using @media |
| `prefers-reduced-motion` | This media query must be considered for all animations |