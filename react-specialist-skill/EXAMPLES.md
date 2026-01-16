# React Specialist - Code Examples & Patterns

## Advanced Custom Hook Pattern

```typescript
// Optimized data fetching hook with TanStack Query
function useUserPreferences() {
  const queryClient = useQueryClient();
  
  return useQuery({
    queryKey: ['userPreferences'],
    queryFn: fetchUserPreferences,
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    onSuccess: (data) => {
      // Update related queries when preferences change
      queryClient.invalidateQueries(['userDashboard']);
    },
  });
}

// Compound component with state management
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function Tabs({ children, defaultValue }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultValue);
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

Tabs.Tab = function Tab({ value, children }: TabProps) {
  const { activeTab, setActiveTab } = useContext(TabsContext)!;
  const isActive = activeTab === value;
  
  return (
    <button
      className={isActive ? 'active' : ''}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
};
```

## Next.js App Router Pattern

```typescript
// Server Component with data fetching
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  const relatedProducts = await getRelatedProducts(params.id);
  
  return (
    <div>
      <ProductDetails product={product} />
      <ClientProductActions productId={product.id} />
      <ProductGrid 
        products={relatedProducts}
        title="Related Products"
      />
    </div>
  );
}

// Client Component for interactivity
'use client';

function ClientProductActions({ productId }: { productId: string }) {
  const [isLiked, setIsLiked] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  
  const { mutate: toggleLike } = useMutation({
    mutationFn: () => toggleProductLike(productId),
    onMutate: () => {
      setIsLoading(true);
      setIsLiked(prev => !prev);
    },
    onError: () => {
      setIsLiked(prev => !prev); // Revert on error
    },
    onSettled: () => {
      setIsLoading(false);
    },
  });
  
  return (
    <div>
      <Button 
        onClick={() => toggleLike()}
        disabled={isLoading}
        variant={isLiked ? "solid" : "outline"}
      >
        {isLiked ? "Liked" : "Like"}
      </Button>
    </div>
  );
}
```

## Zustand State Management

```typescript
// Zustand store with TypeScript
interface UserStore {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
}

const useUserStore = create<UserStore>((set, get) => ({
  user: null,
  isAuthenticated: false,
  
  login: async (credentials) => {
    const user = await api.login(credentials);
    set({ user, isAuthenticated: true });
    // Persist to localStorage
    localStorage.setItem('user', JSON.stringify(user));
  },
  
  logout: () => {
    set({ user: null, isAuthenticated: false });
    localStorage.removeItem('user');
  },
  
  updateUser: (updates) => {
    const currentUser = get().user;
    if (currentUser) {
      const updatedUser = { ...currentUser, ...updates };
      set({ user: updatedUser });
      localStorage.setItem('user', JSON.stringify(updatedUser));
    }
  },
}));

// Usage in components
function UserProfile() {
  const { user, logout, updateUser } = useUserStore();
  
  const handleNameChange = (newName: string) => {
    updateUser({ name: newName });
  };
  
  return (
    <div>
      <h1>Welcome, {user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

## Anti-Patterns & Fixes

### Anti-Pattern 1: Derived State Instead of Computation

**What it looks like (BAD):**
```jsx
function UserList({ users, searchTerm }) {
  const [filteredUsers, setFilteredUsers] = useState([]);

  // Sync state with effect - WRONG
  useEffect(() => {
    setFilteredUsers(
      users.filter(u => u.name.includes(searchTerm))
    );
  }, [users, searchTerm]);

  return filteredUsers.map(user => <UserCard key={user.id} user={user} />);
}
```

**Why it fails:**
- Extra render (effect runs after initial render)
- State can become out of sync
- More complex state management

**Correct approach:**
```jsx
function UserList({ users, searchTerm }) {
  // Compute during render - CORRECT
  const filteredUsers = users.filter(u => u.name.includes(searchTerm));

  return filteredUsers.map(user => <UserCard key={user.id} user={user} />);
}

// If expensive, use useMemo
function UserList({ users, searchTerm }) {
  const filteredUsers = useMemo(
    () => users.filter(u => u.name.includes(searchTerm)),
    [users, searchTerm]
  );

  return filteredUsers.map(user => <UserCard key={user.id} user={user} />);
}
```

### Anti-Pattern 2: Missing useCallback for Event Handlers

**What it looks like (BAD):**
```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // Creates new function every render - breaks React.memo
  const handleClick = () => {
    console.log('clicked');
  };

  return <MemoizedChild onClick={handleClick} />;
}
```

**Correct approach:**
```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // Stable reference - React.memo works correctly
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <MemoizedChild onClick={handleClick} />;
}
```

### Anti-Pattern 3: Prop Drilling Through Many Levels

**What it looks like (BAD):**
```jsx
function App() {
  const [user, setUser] = useState(null);
  return <Layout user={user} setUser={setUser} />;
}

function Layout({ user, setUser }) {
  return <Sidebar user={user} setUser={setUser} />;
}

function Sidebar({ user, setUser }) {
  return <UserMenu user={user} setUser={setUser} />;
}

function UserMenu({ user, setUser }) {
  // Finally uses the prop!
  return <div>{user?.name}</div>;
}
```

**Correct approach with Context:**
```jsx
const UserContext = createContext<UserContextValue>(null);

function App() {
  const [user, setUser] = useState(null);
  
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserMenu() {
  const { user, setUser } = useContext(UserContext);
  return <div>{user?.name}</div>;
}
```

**Or with Zustand (simpler):**
```jsx
const useUserStore = create((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));

function UserMenu() {
  const user = useUserStore((state) => state.user);
  return <div>{user?.name}</div>;
}
```
