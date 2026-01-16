# React Specialist - Technical Reference

## Behavioral Traits

### Performance Optimization
- Implements React.memo, useMemo, and useCallback strategically
- Optimizes re-renders with component composition and state placement
- Leverages React DevTools Profiler for performance analysis
- Implements virtual scrolling and lazy loading for large datasets
- Uses code splitting and dynamic imports for bundle optimization

### Component Architecture
- Designs composable, reusable component APIs with clear contracts
- Implements compound component patterns for complex UIs
- Creates headless UI components for maximum flexibility
- Establishes consistent prop interfaces and TypeScript types
- Implements render prop and children prop patterns effectively

### Data Flow Expertise
- Master of unidirectional data flow principles
- Implements complex state synchronization patterns
- Handles side effects cleanly with custom hooks
- Manages server and client state separation effectively
- Optimizes network requests with strategic caching strategies

## When to Use

### Ideal Scenarios
- **Modern Web Applications**: SPAs, PWAs, and complex interactive UIs
- **E-commerce Platforms**: Shopping carts, product catalogs, checkout flows
- **Dashboards**: Real-time data visualization and analytics
- **Social Media Applications**: Feeds, messaging, real-time updates
- **Admin Panels**: Complex forms, data tables, and management interfaces

### Problem Areas Addressed
- Performance bottlenecks in large React applications
- Complex state management challenges
- Server-client state synchronization issues
- Component re-render optimization
- Bundle size management and code splitting

## Development Workflow

### Project Setup
- Configures React 18+ with TypeScript and strict mode
- Sets up Next.js App Router or Vite for optimal development experience
- Implements testing with React Testing Library and MSW
- Configures linting with ESLint and formatting with Prettier
- Sets up Husky for pre-commit hooks and quality gates

### Component Development
- Uses component-driven development with Storybook
- Implements atomic design principles for scalable component architecture
- Creates comprehensive prop types and documentation
- Establishes consistent naming conventions and file organization
- Uses render props and compound patterns for flexible APIs

### Performance Optimization
- Implements React Profiler monitoring and analysis
- Uses code splitting and lazy loading strategically
- Optimizes bundle size with tree shaking and dynamic imports
- Implements virtual scrolling for large lists
- Monitors and optimizes re-render patterns

## Workflow: Implement Server State with TanStack Query

**Use case:** Fetch, cache, and synchronize server data efficiently

### 1. Setup TanStack Query
```bash
npm install @tanstack/react-query
```

```tsx
// App.tsx - Configure QueryClient
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,  // Consider data fresh for 1 minute
      cacheTime: 5 * 60 * 1000,  // Keep in cache for 5 minutes
      retry: 3,  // Retry failed requests 3 times
      refetchOnWindowFocus: false,  // Don't auto-refetch on window focus
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Dashboard />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### 2. Basic Query (GET Request)
```tsx
import { useQuery } from '@tanstack/react-query';

interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: number }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],  // Unique key for caching
    queryFn: async () => {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      return response.json() as Promise<User>;
    },
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!data) return null;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}

// Data automatically cached! Second mount uses cached data.
```

### 3. Mutation (POST/PUT/DELETE)
```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function UserForm() {
  const queryClient = useQueryClient();

  const createUser = useMutation({
    mutationFn: async (userData: Omit<User, 'id'>) => {
      const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData),
      });
      if (!response.ok) throw new Error('Failed to create user');
      return response.json();
    },
    onSuccess: () => {
      // Invalidate and refetch users list
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    createUser.mutate({
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit" disabled={createUser.isPending}>
        {createUser.isPending ? 'Creating...' : 'Create User'}
      </button>
      {createUser.isError && <p>Error: {createUser.error.message}</p>}
      {createUser.isSuccess && <p>User created!</p>}
    </form>
  );
}
```

### 4. Optimistic Updates
```tsx
const updateUser = useMutation({
  mutationFn: async ({ id, ...data }: Partial<User> & { id: number }) => {
    const response = await fetch(`/api/users/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    return response.json();
  },
  
  onMutate: async (newUser) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['user', newUser.id] });

    // Snapshot previous value
    const previousUser = queryClient.getQueryData(['user', newUser.id]);

    // Optimistically update UI
    queryClient.setQueryData(['user', newUser.id], newUser);

    // Return context for rollback
    return { previousUser };
  },
  
  onError: (err, newUser, context) => {
    // Rollback on error
    if (context?.previousUser) {
      queryClient.setQueryData(
        ['user', newUser.id],
        context.previousUser
      );
    }
  },
  
  onSettled: (data, error, variables) => {
    // Refetch to ensure sync
    queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
  },
});

// User sees instant update, rolls back if server fails
```
