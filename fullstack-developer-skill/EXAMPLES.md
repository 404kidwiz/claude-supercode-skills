# Fullstack Developer - Code Examples & Patterns

This document contains real-world examples, common patterns, and anti-patterns for full-stack development.

## Example 1: E-Commerce Platform Development

**Scenario:** Building a full-featured e-commerce platform with cart, checkout, and user accounts.

**Tech Stack:**
- **Frontend**: React with TypeScript, Redux Toolkit, Material-UI
- **Backend**: Node.js with Express, PostgreSQL, Redis caching
- **Infrastructure**: Docker containers, Kubernetes on AWS

**Key Implementation:**
1. **Shopping Cart**: Persistent cart with optimistic updates
2. **Checkout Flow**: Multi-step wizard with payment integration
3. **User Accounts**: JWT authentication with refresh tokens
4. **Admin Dashboard**: Role-based access control with analytics

**Results:**
- Page load time: < 2 seconds
- API response time: < 100ms average
- 99.9% uptime with auto-scaling
- Mobile-responsive design

---

## Example 2: Real-Time Collaboration Tool

**Scenario:** Developing a collaborative document editor with live updates.

**Technical Architecture:**
1. **Real-Time Sync**: WebSocket connections with operational transformation
2. **Document Editor**: Rich text editor with collaborative cursors
3. **Presence**: Live user presence and activity feeds
4. **Comments**: Threaded comments with real-time updates

**Implementation Highlights:**
- Conflict-free replicated data types (CRDTs) for collaboration
- Optimistic UI updates for responsive experience
- WebSocket server with connection pooling
- Redis for pub/sub and session management

---

## Example 3: SaaS Dashboard Application

**Scenario:** Building a multi-tenant SaaS analytics dashboard.

**Multi-Tenant Architecture:**
1. **Database**: Row-level security with PostgreSQL
2. **API**: Tenant-aware routing and authentication
3. **Frontend**: Dashboard with configurable widgets
4. **Billing**: Stripe integration with subscription management

**Enterprise Features:**
- Role-based access control (RBAC)
- Audit logging and compliance reporting
- SSO integration with SAML/OIDC
- Custom branding and white-labeling

---

## Common Patterns

### Pattern 1: API Gateway with Authentication

```javascript
// API Gateway Setup
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';
import jwt from 'jsonwebtoken';

const app = express();

// Authentication middleware
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Route to user service
app.use('/api/users', authMiddleware, createProxyMiddleware({
  target: 'http://user-service:3001',
  changeOrigin: true,
  pathRewrite: { '^/api/users': '' }
}));

// Route to product service
app.use('/api/products', authMiddleware, createProxyMiddleware({
  target: 'http://product-service:3002',
  changeOrigin: true,
  pathRewrite: { '^/api/products': '' }
}));
```

### Pattern 2: Repository Pattern with TypeScript

```typescript
// Repository Interface
interface IRepository<T> {
  findAll(): Promise<T[]>;
  findById(id: string): Promise<T | null>;
  create(entity: Partial<T>): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

// User Repository Implementation
class UserRepository implements IRepository<User> {
  constructor(private db: Database) {}

  async findAll(): Promise<User[]> {
    return this.db.query('SELECT * FROM users');
  }

  async findById(id: string): Promise<User | null> {
    const [user] = await this.db.query('SELECT * FROM users WHERE id = ?', [id]);
    return user || null;
  }

  async create(userData: Partial<User>): Promise<User> {
    const result = await this.db.query(
      'INSERT INTO users (email, name) VALUES (?, ?)',
      [userData.email, userData.name]
    );
    return this.findById(result.insertId);
  }

  async update(id: string, userData: Partial<User>): Promise<User> {
    await this.db.query(
      'UPDATE users SET email = ?, name = ? WHERE id = ?',
      [userData.email, userData.name, id]
    );
    return this.findById(id);
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.db.query('DELETE FROM users WHERE id = ?', [id]);
    return result.affectedRows > 0;
  }
}
```

### Pattern 3: Custom React Hook for API Calls

```typescript
// useApi Hook with Error Handling and Caching
function useApi<T>(
  endpoint: string,
  options: { immediate?: boolean; cache?: boolean } = {}
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(options.immediate !== false);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async (params?: Record<string, any>) => {
    setLoading(true);
    setError(null);

    try {
      const queryString = params 
        ? '?' + new URLSearchParams(params).toString() 
        : '';
      const response = await fetch(`/api${endpoint}${queryString}`);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const result = await response.json();
      setData(result);
      return result;
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Unknown error'));
      throw err;
    } finally {
      setLoading(false);
    }
  }, [endpoint]);

  useEffect(() => {
    if (options.immediate !== false) {
      fetchData();
    }
  }, [fetchData, options.immediate]);

  return { data, loading, error, refetch: fetchData };
}

// Usage
function ProductList() {
  const { data: products, loading, error, refetch } = useApi<Product[]>('/products');
  
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} retry={refetch} />;
  
  return <ProductGrid products={products} />;
}
```

### Pattern 4: Error Boundary with Fallback UI

```typescript
// Error Boundary Component
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Send to error tracking service
    trackError(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-container">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => window.location.reload()}>
            Refresh Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

## Anti-Patterns and Fixes

### Anti-Pattern 1: N+1 Query Problem

**Problem:**
```javascript
// BAD: N+1 queries
async function getOrdersWithItems() {
  const orders = await Order.findAll();
  
  // This creates N additional queries!
  for (const order of orders) {
    order.items = await OrderItem.findAll({ 
      where: { orderId: order.id } 
    });
  }
  
  return orders;
}
```

**Solution:**
```javascript
// GOOD: Single query with join/include
async function getOrdersWithItems() {
  return Order.findAll({
    include: [{
      model: OrderItem,
      as: 'items'
    }]
  });
}
```

### Anti-Pattern 2: Prop Drilling

**Problem:**
```jsx
// BAD: Passing props through many levels
function App({ user }) {
  return <Layout user={user} />;
}

function Layout({ user }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  return <UserInfo user={user} />;
}

function UserInfo({ user }) {
  return <span>{user.name}</span>;
}
```

**Solution:**
```jsx
// GOOD: Use Context for deeply nested data
const UserContext = createContext(null);

function App({ user }) {
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserInfo() {
  const user = useContext(UserContext);
  return <span>{user.name}</span>;
}
```

### Anti-Pattern 3: Missing Error Handling in API Calls

**Problem:**
```javascript
// BAD: No error handling
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}
```

**Solution:**
```javascript
// GOOD: Comprehensive error handling
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    
    if (!response.ok) {
      if (response.status === 404) {
        throw new NotFoundError(`User ${id} not found`);
      }
      if (response.status === 401) {
        throw new UnauthorizedError('Authentication required');
      }
      throw new ApiError(`API error: ${response.status}`);
    }
    
    return response.json();
  } catch (error) {
    if (error instanceof TypeError) {
      throw new NetworkError('Network connection failed');
    }
    throw error;
  }
}
```

### Anti-Pattern 4: Hardcoded Configuration

**Problem:**
```javascript
// BAD: Hardcoded values
const API_URL = 'http://localhost:3000/api';
const DB_HOST = 'localhost';
const JWT_SECRET = 'mysecret123';
```

**Solution:**
```javascript
// GOOD: Environment variables with validation
const config = {
  apiUrl: process.env.API_URL || 'http://localhost:3000/api',
  database: {
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || '5432'),
    name: process.env.DB_NAME,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '1d',
  }
};

// Validate required config
const required = ['DB_HOST', 'DB_NAME', 'JWT_SECRET'];
for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
}

export default config;
```

---

## Integration Checklist

### Before Deploying Full-Stack Feature

- [ ] Frontend and backend API contracts match
- [ ] Error handling covers all edge cases
- [ ] Authentication/authorization tested
- [ ] Database migrations applied
- [ ] Environment variables configured
- [ ] CORS settings verified
- [ ] Rate limiting in place
- [ ] Logging and monitoring enabled
- [ ] Unit and integration tests passing
- [ ] Performance tested under load
