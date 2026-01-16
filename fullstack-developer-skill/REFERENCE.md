# Fullstack Developer - Technical Reference

This document contains detailed technical specifications, workflows, and configuration examples for full-stack development.

## Integration Patterns

### Frontend-Backend Communication

```javascript
// API Client Configuration
const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request/Response Interceptors
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Error Handling Middleware
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle authentication errors
      redirectToLogin();
    }
    return Promise.reject(error);
  }
);
```

### Component Integration

```javascript
// React Component with Backend Integration
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        const response = await apiClient.get('/api/users/profile');
        setUser(response.data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, []);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  
  return <UserProfileCard user={user} />;
};
```

### Backend Route Handler

```javascript
// Express.js Route Handler
router.get('/api/users/profile', authenticate, async (req, res) => {
  try {
    const userId = req.user.id;
    const user = await User.findById(userId)
      .select('-password')
      .populate('profile');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Cache user profile
    await cache.set(`user:${userId}`, user, 300); // 5 minutes

    res.json({
      success: true,
      data: {
        id: user.id,
        email: user.email,
        profile: user.profile,
        createdAt: user.createdAt
      }
    });
  } catch (error) {
    logger.error('Error fetching user profile:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

## End-to-End Feature Development

### Database Schema Design

```javascript
// User Model (Mongoose)
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    validate: [validator.isEmail, 'Invalid email']
  },
  password: {
    type: String,
    required: true,
    minlength: 8
  },
  profile: {
    firstName: String,
    lastName: String,
    avatar: String,
    bio: String
  },
  preferences: {
    theme: { type: String, enum: ['light', 'dark'], default: 'light' },
    notifications: { type: Boolean, default: true }
  }
}, { timestamps: true });

// Indexes for performance
userSchema.index({ email: 1 });
userSchema.index({ 'profile.firstName': 1, 'profile.lastName': 1 });
```

### Frontend Form Handling

```javascript
// Form Component with Validation
const UserSettingsForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm();
  const [updateUser, { isLoading }] = useUpdateUserMutation();

  const onSubmit = async (data) => {
    try {
      await updateUser(data).unwrap();
      toast.success('Settings updated successfully');
    } catch (error) {
      toast.error('Failed to update settings');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div className="form-group">
        <label>First Name</label>
        <input
          {...register('firstName', { required: 'First name is required' })}
        />
        {errors.firstName && (
          <span className="error">{errors.firstName.message}</span>
        )}
      </div>
      
      <div className="form-group">
        <label>Last Name</label>
        <input
          {...register('lastName', { required: 'Last name is required' })}
        />
        {errors.lastName && (
          <span className="error">{errors.lastName.message}</span>
        )}
      </div>
      
      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Saving...' : 'Save Changes'}
      </button>
    </form>
  );
};
```

## State Management Integration

### Global State Setup

```javascript
// Redux Store Configuration
const store = configureStore({
  reducer: {
    auth: authSlice,
    user: userSlice,
    products: productsSlice,
    ui: uiSlice
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER]
      }
    }).concat(apiSlice.middleware)
});

// Auth Slice
const authSlice = createSlice({
  name: 'auth',
  initialState: {
    user: null,
    token: null,
    isAuthenticated: false,
    loading: false
  },
  reducers: {
    loginStart: (state) => {
      state.loading = true;
    },
    loginSuccess: (state, action) => {
      state.user = action.payload.user;
      state.token = action.payload.token;
      state.isAuthenticated = true;
      state.loading = false;
    },
    loginFailure: (state) => {
      state.loading = false;
    },
    logout: (state) => {
      state.user = null;
      state.token = null;
      state.isAuthenticated = false;
    }
  }
});
```

### Backend Authentication Middleware

```javascript
// JWT Authentication Middleware
const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Access denied. No token provided.' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid token.' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token.' });
  }
};
```

## Real-time Features

### WebSocket Integration

```javascript
// Frontend WebSocket Hook
const useWebSocket = (url) => {
  const [socket, setSocket] = useState(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const ws = new WebSocket(url);

    ws.onopen = () => {
      setConnected(true);
      console.log('WebSocket connected');
    };

    ws.onclose = () => {
      setConnected(false);
      console.log('WebSocket disconnected');
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    setSocket(ws);

    return () => {
      ws.close();
    };
  }, [url]);

  const sendMessage = (message) => {
    if (socket && connected) {
      socket.send(JSON.stringify(message));
    }
  };

  return { socket, connected, sendMessage };
};

// Backend WebSocket Handler
const setupWebSocket = (server) => {
  const wss = new WebSocketServer({ server });

  wss.on('connection', (ws, req) => {
    // Authenticate WebSocket connection
    const token = req.url.split('token=')[1];
    const user = jwt.verify(token, process.env.JWT_SECRET);
    
    ws.userId = user.id;
    ws.send(JSON.stringify({ type: 'connection', success: true }));

    ws.on('message', async (data) => {
      const message = JSON.parse(data);
      
      switch (message.type) {
        case 'join_room':
          await handleJoinRoom(ws, message.roomId);
          break;
        case 'send_message':
          await handleSendMessage(ws, message);
          break;
      }
    });
  });
};
```

## Performance Optimization

### Frontend Optimization

```javascript
// Code Splitting and Lazy Loading
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// Image Optimization
const OptimizedImage = ({ src, alt, ...props }) => {
  const [loaded, setLoaded] = useState(false);

  return (
    <div className="image-container">
      <img
        src={src}
        alt={alt}
        loading="lazy"
        onLoad={() => setLoaded(true)}
        className={`image ${loaded ? 'loaded' : 'loading'}`}
        {...props}
      />
    </div>
  );
};

// API Response Caching
const useApiCache = (key, fetcher, options = {}) => {
  const { data, error, isLoading } = useSWR(key, fetcher, {
    revalidateOnFocus: false,
    dedupingInterval: 60000, // 1 minute
    ...options
  });

  return { data, error, isLoading };
};
```

### Backend Optimization

```javascript
// Database Query Optimization
const getProducts = async (filters = {}, pagination = {}) => {
  const { limit = 20, offset = 0 } = pagination;
  const { category, minPrice, maxPrice, search } = filters;

  // Build efficient query
  const query = {};
  
  if (category) query.category = category;
  if (minPrice || maxPrice) {
    query.price = {};
    if (minPrice) query.price.$gte = minPrice;
    if (maxPrice) query.price.$lte = maxPrice;
  }
  if (search) {
    query.$text = { $search: search };
  }

  // Parallel execution with aggregation pipeline
  const [products, totalCount] = await Promise.all([
    Product.find(query)
      .select('name price category images')
      .limit(limit)
      .skip(offset)
      .sort({ createdAt: -1 }),
    Product.countDocuments(query)
  ]);

  return { products, totalCount };
};

// Response Compression
app.use(compression());
app.use(express.json({ limit: '10mb' }));

// Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP'
});
app.use('/api/', limiter);
```

## Testing Strategies

### Frontend Testing

```javascript
// Component Testing
describe('UserProfile', () => {
  test('renders user profile correctly', async () => {
    const mockUser = { id: 1, name: 'John Doe', email: 'john@example.com' };
    
    render(<UserProfile user={mockUser} />);
    
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  test('handles loading state', () => {
    render(<UserProfile loading={true} />);
    expect(screen.getByTestId('spinner')).toBeInTheDocument();
  });
});

// API Testing
import { renderHook, act } from '@testing-library/react-hooks';
import { useUserApi } from './api';

test('fetches user data successfully', async () => {
  const mockUser = { id: 1, name: 'John Doe' };
  axios.get.mockResolvedValue({ data: mockUser });

  const { result, waitForNextUpdate } = renderHook(() => useUserApi());

  expect(result.current.loading).toBe(true);

  await waitForNextUpdate();

  expect(result.current.user).toEqual(mockUser);
  expect(result.current.loading).toBe(false);
});
```

### Backend Testing

```javascript
// Integration Testing
describe('User API', () => {
  beforeEach(async () => {
    await User.deleteMany({});
  });

  test('POST /api/users creates new user', async () => {
    const userData = {
      email: 'test@example.com',
      password: 'password123',
      firstName: 'John',
      lastName: 'Doe'
    };

    const response = await request(app)
      .post('/api/users')
      .send(userData)
      .expect(201);

    expect(response.body.success).toBe(true);
    expect(response.body.data.email).toBe(userData.email);
  });

  test('GET /api/users/:id returns user data', async () => {
    const user = await User.create(userData);
    
    const response = await request(app)
      .get(`/api/users/${user.id}`)
      .expect(200);

    expect(response.body.data.id).toBe(user.id);
  });
});
```

## Deployment and DevOps

### Container Configuration

```dockerfile
# Frontend Dockerfile
FROM node:16-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```dockerfile
# Backend Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```
