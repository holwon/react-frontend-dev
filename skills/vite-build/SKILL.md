---
name: vite-build
description: Vite build tool — vite.config.ts configuration, environment variables (import.meta.env), path aliases, common plugins (@vitejs/plugin-react), code splitting strategies, build optimization, and dev server proxy. Use when configuring the build pipeline, handling environment variables, or optimizing bundle size.
---

# Vite Build Configuration

## Basic Configuration

```typescript
// Path: vite.config.ts
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [
      react(),
    ],

    // Path aliases
    resolve: {
      alias: {
        '@': resolve(__dirname, './src'),
        '@features': resolve(__dirname, './src/features'),
        '@shared': resolve(__dirname, './src/shared'),
      },
    },

    // Dev server
    server: {
      port: 3000,
      open: true,
      proxy: {
        '/api': {
          target: env.VITE_API_BASE_URL || 'http://localhost:8080',
          changeOrigin: true,
          rewrite: path => path.replace(/^\/api/, ''),
        },
      },
    },

    // Build configuration
    build: {
      target: 'es2022',
      outDir: 'dist',
      sourcemap: mode === 'development',
      rollupOptions: {
        output: {
          // Manual code splitting
          manualChunks: {
            // Third-party libraries bundled separately
            'vendor-react': ['react', 'react-dom', 'react-router-dom'],
            'vendor-query': ['@tanstack/react-query'],
            'vendor-form': ['react-hook-form', 'zod'],
          },
        },
      },
    },

    // Preview server
    preview: {
      port: 4173,
    },
  };
});
```

---

## Environment Variables

**Rule**: In Vite, only variables prefixed with `VITE_` are exposed to client-side code (via `import.meta.env`).

```bash
# .env                  — all environments
# .env.local            — local overrides (do not commit to git)
# .env.development      — development environment
# .env.production       — production environment
```

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:8080
VITE_APP_NAME=MyApp Dev
```

```typescript
// Path: src/lib/config.ts
// Type-safe environment variable access
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_NAME: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

// Centralized access (easy to mock and test)
export const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  appName: import.meta.env.VITE_APP_NAME,
  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
} as const;
```

---

## Code Splitting Strategies

```tsx
// Path: src/routes/index.tsx
import { lazy, Suspense } from 'react';

// ✅ Lazy-load each page/feature route
const UserList = lazy(() => import('@features/user/UserList'));
const ProductCatalog = lazy(() => import('@features/product/ProductCatalog'));
const AdminPanel = lazy(() => import('@features/admin/AdminPanel'));

// ✅ With prefetch comment (Vite magic comment)
const Dashboard = lazy(() => import(/* webpackChunkName: "dashboard" */ '@features/dashboard/Dashboard'));

// ✅ Component-level lazy loading (large editors, charts, etc.)
const RichEditor = lazy(() =>
  import('@shared/components/RichEditor').then(m => ({ default: m.RichEditor }))
);
```

---

## Common Plugins

```typescript
// vite.config.ts plugin examples

import react from '@vitejs/plugin-react';                // Required: React support (Babel)
import reactSwc from '@vitejs/plugin-react-swc';         // Alternative: faster SWC compiler
import { visualizer } from 'rollup-plugin-visualizer';  // Bundle size analysis
import svgr from 'vite-plugin-svgr';                    // Import SVGs as React components
import tsconfigPaths from 'vite-tsconfig-paths';        // Auto-read tsconfig paths

// Example: configuration with analyzer
plugins: [
  react(),
  svgr(),
  tsconfigPaths(),
  // Enable only during analyze builds
  process.env.ANALYZE && visualizer({
    open: true,
    gzipSize: true,
    brotliSize: true,
  }),
].filter(Boolean),
```

---

## Using Path Aliases

```typescript
// tsconfig.json — keep in sync with aliases in vite.config.ts
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@features/*": ["./src/features/*"],
      "@shared/*": ["./src/shared/*"]
    }
  }
}
```

```tsx
// ✅ Use aliases (clear and refactor-friendly)
import { Button } from '@shared/components/Button';
import { useAuthStore } from '@/lib/stores/useAuthStore';

// ❌ Relative path hell (hard to maintain)
import { Button } from '../../../shared/components/Button';
```

---

## Build Optimization Cheatsheet

| Optimization | Configuration |
|--------|------|
| Bundle size analysis | `rollup-plugin-visualizer` |
| Image compression | `vite-plugin-imagemin` |
| Remove console in production | `build.terserOptions.compress.drop_console: true` |
| Preload critical resources | `<link rel="modulepreload">` or Vite auto-injection |
| Enable gzip/brotli | Configured on the server (Nginx/CDN), not directly by Vite |
| CSS code splitting | Enabled by default (`build.cssCodeSplit: true`) |

---

## package.json Script Conventions

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .ts,.tsx --report-unused-disable-directives",
    "type-check": "tsc --noEmit",
    "test": "vitest",
    "test:run": "vitest run",
    "test:e2e": "playwright test",
    "analyze": "ANALYZE=true vite build"
  }
}
```
