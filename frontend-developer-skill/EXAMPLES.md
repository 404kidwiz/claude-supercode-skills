# Frontend Developer - Code Examples & Patterns

## Anti-Pattern 1: useState for Server Data

### What it looks like (BAD):
```typescript
// ❌ BAD: Managing API data with useState
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);
  
  // ❌ Missing: cache, refetch, stale data handling, optimistic updates
  return <div>{user?.name}</div>;
};
```

### Why it fails:
- No automatic caching (refetches on every mount)
- No background refetching for stale data
- Manual loading/error state management
- No optimistic updates support
- Duplicated logic across components

### Correct approach:
```typescript
// ✅ GOOD: Use TanStack Query for server data
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

const UserProfile = () => {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user'],
    queryFn: () => fetch('/api/user').then(res => res.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
  
  const queryClient = useQueryClient();
  
  const updateMutation = useMutation({
    mutationFn: (updates) => fetch('/api/user', {
      method: 'PATCH',
      body: JSON.stringify(updates),
    }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
  
  // ✅ Automatic caching, refetching, optimistic updates
  return <div>{user?.name}</div>;
};
```

**Impact:** 90% less boilerplate, automatic caching, better UX.

## Anti-Pattern 2: Not Memoizing Expensive Computations

### What it looks like (BAD):
```typescript
// ❌ BAD: Expensive computation runs on every render
function ProductList({ products }) {
  // ❌ Runs EVERY render, even if products unchanged!
  const sortedProducts = products
    .map(p => ({ ...p, score: calculateComplexScore(p) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, 20);
  
  return <div>{sortedProducts.map(p => <ProductCard {...p} />)}</div>;
}
```

### Why it fails:
- Expensive calculation runs on every render
- Parent re-renders cause unnecessary work
- Poor performance with large datasets
- No dependency tracking

### Correct approach:
```typescript
// ✅ GOOD: Memoize expensive computation
import { useMemo, memo } from 'react';

function ProductList({ products }) {
  const sortedProducts = useMemo(() => {
    return products
      .map(p => ({ ...p, score: calculateComplexScore(p) }))
      .sort((a, b) => b.score - a.score)
      .slice(0, 20);
  }, [products]); // Only recalculate when products change
  
  return <div>{sortedProducts.map(p => <ProductCard key={p.id} {...p} />)}</div>;
}

// ✅ Even better: Memoize ProductCard too
const ProductCard = memo(({ id, name, score }) => {
  return <div>{name} - {score}</div>;
});
```

**Impact:** 10x+ faster renders, smoother UX, better performance.

## Zustand Store Pattern

```typescript
// src/store/useCartStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  totalPrice: () => number;
  totalItems: () => number;
}

export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],
      
      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id);
        if (existing) {
          return {
            items: state.items.map(i =>
              i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          };
        }
        return { items: [...state.items, { ...item, quantity: 1 }] };
      }),
      
      removeItem: (id) => set((state) => ({
        items: state.items.filter(i => i.id !== id),
      })),
      
      updateQuantity: (id, quantity) => set((state) => ({
        items: quantity > 0
          ? state.items.map(i => i.id === id ? { ...i, quantity } : i)
          : state.items.filter(i => i.id !== id),
      })),
      
      clearCart: () => set({ items: [] }),
      
      totalPrice: () => get().items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      ),
      
      totalItems: () => get().items.reduce(
        (sum, item) => sum + item.quantity, 0
      ),
    }),
    { name: 'cart-storage' }
  )
);

// Usage
const ShoppingCart = () => {
  const { items, addItem, removeItem, totalPrice } = useCartStore();
  
  return (
    <div>
      {items.map(item => (
        <div key={item.id}>
          {item.name} x{item.quantity}
          <button onClick={() => removeItem(item.id)}>Remove</button>
        </div>
      ))}
      <div>Total: ${totalPrice()}</div>
    </div>
  );
};
```

## Testing Examples

### Unit Test (Vitest)

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('renders user information', () => {
    render(<UserProfile userId={1} />);
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });
  
  it('handles edit mode toggle', async () => {
    const user = userEvent.setup();
    render(<UserProfile userId={1} />);
    
    await user.click(screen.getByRole('button', { name: /edit/i }));
    
    expect(screen.getByRole('textbox', { name: /name/i })).toBeInTheDocument();
  });
});
```

### E2E Test (Playwright)

```typescript
import { test, expect } from '@playwright/test';

test('user login flow', async ({ page }) => {
  await page.goto('http://localhost:3000');
  await page.click('text=Login');
  await page.fill('input[name="email"]', 'user@example.com');
  await page.fill('input[name="password"]', 'password');
  await page.click('button:has-text("Login")');
  
  await expect(page).toHaveURL(/dashboard/);
});

test('add item to cart', async ({ page }) => {
  await page.goto('http://localhost:3000/products');
  
  await page.click('[data-testid="product-1"] button:has-text("Add to Cart")');
  
  await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');
});
```

## Performance Patterns

### Code Splitting with React.lazy

```typescript
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));

const App = () => {
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  );
};
```

### Memoization with useMemo and useCallback

```typescript
import { memo, useMemo, useCallback } from 'react';

const ExpensiveComponent = memo(({ data, onAction }: Props) => {
  const processedData = useMemo(() => {
    return heavyComputation(data);
  }, [data]);

  const handleClick = useCallback(() => {
    onAction(processedData);
  }, [onAction, processedData]);

  return (
    <div onClick={handleClick}>
      {processedData.map(item => <Item key={item.id} {...item} />)}
    </div>
  );
});
```

### Virtualization for Long Lists

```typescript
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Error Boundary Pattern

```typescript
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Log to error reporting service
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="p-4 bg-red-50 text-red-800 rounded">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>
```

## Form Handling with React Hook Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    await login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>
      
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register('password')} />
        {errors.password && <span role="alert">{errors.password.message}</span>}
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```
