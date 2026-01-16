# Frontend Developer - Technical Reference

## Workflow 1: Setting Up New React Project with Best Practices

**Goal:** Bootstrap production-ready React app with TypeScript, testing, linting, and state management in <30 minutes.

### Step 1: Initialize Project with Vite

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
```

### Step 2: Setup State Management (Zustand)

```bash
npm install zustand
```

```typescript
// src/store/useAppStore.ts
import { create } from 'zustand';

interface AppState {
  user: User | null;
  setUser: (user: User | null) => void;
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  theme: 'light',
  toggleTheme: () => set((state) => ({ 
    theme: state.theme === 'light' ? 'dark' : 'light' 
  })),
}));
```

### Step 3: Setup TanStack Query for Server State

```bash
npm install @tanstack/react-query
```

```typescript
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 minute
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

### Step 4: Setup Testing (Vitest + Testing Library)

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @vitest/ui jsdom
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
  },
});
```

### Step 5: Setup Linting (ESLint + Prettier)

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D prettier eslint-config-prettier eslint-plugin-prettier
```

### Step 6: Add Tailwind CSS

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

**Expected Result:**
- TypeScript-ready React project with Vite (fast HMR)
- Zustand for client state, TanStack Query for server state
- Vitest configured for testing
- ESLint + Prettier for code quality
- Tailwind CSS for rapid styling

**Verification:**
```bash
npm run dev        # Dev server starts
npm run test       # Tests pass
npm run lint       # No errors
npm run build      # Production build succeeds
```

## Workflow 2: Optimizing Bundle Size for Production

**Goal:** Reduce production bundle from 500KB+ to <200KB (gzipped).

### Step 1: Analyze Current Bundle

```bash
npm install -D rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true })
  ],
});
```

### Step 2: Implement Code Splitting

```typescript
// src/App.tsx - Route-based code splitting
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

### Step 3: Replace Heavy Dependencies

```bash
# Before: moment.js (70KB) → After: date-fns (2KB per function)
npm uninstall moment
npm install date-fns

# Before: lodash (entire lib) → After: lodash-es (tree-shakeable)
npm uninstall lodash
npm install lodash-es
```

```typescript
// Before
import _ from 'lodash';
_.debounce(fn, 300);

// After (tree-shakeable)
import { debounce } from 'lodash-es';
debounce(fn, 300);
```

### Step 4: Configure Build Optimization

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
        },
      },
    },
    chunkSizeWarningLimit: 600,
  },
});
```

**Expected Result:**
- Main bundle: <100KB (gzipped)
- Vendor chunks: <150KB total
- Route chunks: 20-50KB each
- Total initial load: <200KB

## Scripts Reference

### React Component Scaffolding
```bash
ts-node scripts/scaffold_component.tsx <ComponentName> [OPTIONS]
# Options:
# --hooks=<hooks>: Include hooks (useState,useEffect,useCallback)
# --props=<json>: Component props as JSON
# --styles=<type>: CSS type (css, styled-components, emotion, module)
# --no-test: Skip test file generation
```

### State Management Setup
```bash
ts-node scripts/setup_state.ts <stateName> <type>
# Types: redux, zustand, context, jotai, recoil
```

### API Client Generation
```bash
ts-node scripts/create_api_client.ts <clientType>
# Client types: axios, fetch
```

### Testing Setup
```bash
ts-node scripts/setup_testing.ts <framework>
# Frameworks: jest, vitest, playwright
```

### Build Optimization
```bash
ts-node scripts/optimize_build.ts <bundler>
# Bundlers: vite, webpack
```

### Deployment Script
```bash
./scripts/deploy.sh [OPTIONS]
# Options:
# --skip-tests: Skip test execution
# --skip-quality: Skip linting/formatting
# --platform <vercel|netlify|s3|github>: Deployment platform
```

## References

### React Patterns (`references/react_patterns.md`)
- Functional components with hooks
- Container/Presentational pattern
- Higher-Order Components (HOC)
- Custom hooks
- State management patterns
- Performance optimization
- Testing patterns
- Error handling (Error Boundaries)
- Form handling
- Accessibility

### State Management (`references/state_management.md`)
- Redux Toolkit
- Zustand
- Context API
- Jotai
- Recoil
- Comparison table
- When to use what
- Best practices
- Code examples

### Performance Guide (`references/performance_guide.md`)
- Core Web Vitals (LCP, FID, CLS)
- Code splitting (route-based, component-based)
- Tree shaking
- Bundle optimization
- Memory management
- Image optimization
- Font optimization
- Rendering optimization (virtualization, memoization)
- Network optimization
- Performance monitoring
