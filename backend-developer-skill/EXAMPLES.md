# Backend Developer - Code Examples & Patterns

## Pattern 1: Repository Pattern for Data Access

```typescript
// src/repositories/user.repository.ts
import { PrismaClient, User } from '@prisma/client';

export class UserRepository {
  constructor(private prisma: PrismaClient) {}
  
  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }
  
  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }
  
  async create(data: { email: string; password: string; name?: string }): Promise<User> {
    return this.prisma.user.create({ data });
  }
  
  async update(id: string, data: Partial<User>): Promise<User> {
    return this.prisma.user.update({ where: { id }, data });
  }
  
  async delete(id: string): Promise<User> {
    return this.prisma.user.delete({ where: { id } });
  }
  
  async list(options?: { skip?: number; take?: number }): Promise<User[]> {
    return this.prisma.user.findMany(options);
  }
}

// Usage
const userRepo = new UserRepository(prisma);
const user = await userRepo.findByEmail('test@example.com');
```

## Pattern 2: Service Layer for Business Logic

```typescript
// src/services/user.service.ts
import bcrypt from 'bcrypt';
import { UserRepository } from '../repositories/user.repository';
import { generateTokens } from '../auth/jwt';

export class UserService {
  constructor(private userRepo: UserRepository) {}
  
  async register(email: string, password: string, name?: string) {
    // Business rule: Check if user exists
    const existing = await this.userRepo.findByEmail(email);
    if (existing) {
      throw new Error('User already exists');
    }
    
    // Business rule: Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = await this.userRepo.create({
      email,
      password: hashedPassword,
      name,
    });
    
    // Generate tokens
    const tokens = generateTokens(user.id);
    
    return { user, tokens };
  }
  
  async login(email: string, password: string) {
    const user = await this.userRepo.findByEmail(email);
    if (!user) {
      throw new Error('Invalid credentials');
    }
    
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      throw new Error('Invalid credentials');
    }
    
    const tokens = generateTokens(user.id);
    return { user, tokens };
  }
}
```

## Pattern 3: Custom Error Classes

```typescript
// src/errors/index.ts
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export class BadRequestError extends AppError {
  constructor(message: string) {
    super(400, message);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(404, message);
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(409, message);
  }
}

// Error handler middleware
export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.message,
    });
  }
  
  // Unexpected errors
  console.error('Unexpected error:', err);
  res.status(500).json({ error: 'Internal server error' });
}

// Usage
import { NotFoundError, ConflictError } from '../errors';

if (!user) {
  throw new NotFoundError('User not found');
}

if (existingUser) {
  throw new ConflictError('User already exists');
}
```

## Anti-Pattern 1: Not Validating Input

### What it looks like (BAD):
```typescript
// ❌ BAD: No validation, trusting client input
router.post('/users', async (req, res) => {
  const { email, password } = req.body;
  
  // ❌ No validation! What if email is invalid or password too short?
  const user = await prisma.user.create({
    data: { email, password }
  });
  
  res.json(user);
});
```

### Why it fails:
- SQL injection vulnerabilities
- Invalid data in database
- Poor user experience (unclear errors)
- Security risks (weak passwords accepted)

### Correct approach:
```typescript
// ✅ GOOD: Validate with Zod
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

router.post('/users', async (req, res) => {
  try {
    const validated = createUserSchema.parse(req.body);
    
    const hashedPassword = await bcrypt.hash(validated.password, 10);
    const user = await prisma.user.create({
      data: {
        email: validated.email,
        password: hashedPassword,
      },
    });
    
    res.json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ errors: error.errors });
    }
    throw error;
  }
});
```

## Anti-Pattern 2: Not Handling Async Errors in Express

### What it looks like (BAD):
```typescript
// ❌ BAD: Unhandled promise rejection crashes server
router.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id }
  });
  // ❌ If error occurs, promise rejection is unhandled!
  res.json(user);
});
```

### Why it fails:
- Server crashes on unhandled promise rejections
- No error response sent to client
- Poor debugging experience

### Correct approach:
```typescript
// ✅ GOOD: Wrap in try-catch OR use express-async-errors
import 'express-async-errors'; // Auto-catches async errors

router.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id }
  });
  
  if (!user) {
    throw new NotFoundError('User not found');
  }
  
  res.json(user);
});

// Global error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.statusCode || 500).json({
    error: err.message || 'Internal server error',
  });
});
```

## API Client Example (Axios)

```typescript
// src/api/client.ts
import axios, { AxiosInstance, AxiosError } from 'axios';

interface ApiClientConfig {
  baseURL: string;
  timeout?: number;
}

export function createApiClient(config: ApiClientConfig): AxiosInstance {
  const client = axios.create({
    baseURL: config.baseURL,
    timeout: config.timeout || 10000,
    headers: {
      'Content-Type': 'application/json',
    },
  });

  // Request interceptor - add auth token
  client.interceptors.request.use((config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  });

  // Response interceptor - handle errors
  client.interceptors.response.use(
    (response) => response,
    async (error: AxiosError) => {
      if (error.response?.status === 401) {
        // Token expired - try refresh
        const refreshToken = localStorage.getItem('refreshToken');
        if (refreshToken) {
          try {
            const { data } = await axios.post(`${config.baseURL}/auth/refresh`, {
              refreshToken,
            });
            localStorage.setItem('accessToken', data.accessToken);
            // Retry original request
            return client.request(error.config!);
          } catch {
            // Refresh failed - redirect to login
            localStorage.clear();
            window.location.href = '/login';
          }
        }
      }
      return Promise.reject(error);
    }
  );

  return client;
}

// Usage
const api = createApiClient({ baseURL: 'http://localhost:3000/api' });

// GET request
const users = await api.get('/users');

// POST request
const newUser = await api.post('/users', { email: 'test@example.com', name: 'Test' });

// PUT request
const updatedUser = await api.put('/users/123', { name: 'Updated Name' });

// DELETE request
await api.delete('/users/123');
```

## Testing Examples

### Unit Test (Jest)

```typescript
// src/services/__tests__/user.service.test.ts
import { UserService } from '../user.service';
import { UserRepository } from '../../repositories/user.repository';
import bcrypt from 'bcrypt';

jest.mock('bcrypt');
jest.mock('../../auth/jwt', () => ({
  generateTokens: jest.fn().mockReturnValue({
    accessToken: 'mock-access-token',
    refreshToken: 'mock-refresh-token',
  }),
}));

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockUserRepo = {
      findByEmail: jest.fn(),
      create: jest.fn(),
    } as any;
    
    userService = new UserService(mockUserRepo);
  });

  describe('register', () => {
    it('should create user with hashed password', async () => {
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.create.mockResolvedValue({
        id: '123',
        email: 'test@example.com',
        name: 'Test',
        password: 'hashed',
        createdAt: new Date(),
        updatedAt: new Date(),
      });
      (bcrypt.hash as jest.Mock).mockResolvedValue('hashed');

      const result = await userService.register('test@example.com', 'password123', 'Test');

      expect(result.user.email).toBe('test@example.com');
      expect(result.tokens.accessToken).toBe('mock-access-token');
      expect(bcrypt.hash).toHaveBeenCalledWith('password123', 10);
    });

    it('should throw error if user already exists', async () => {
      mockUserRepo.findByEmail.mockResolvedValue({ id: '123' } as any);

      await expect(
        userService.register('test@example.com', 'password123')
      ).rejects.toThrow('User already exists');
    });
  });
});
```

### Integration Test (Supertest)

```typescript
// src/routes/__tests__/auth.routes.test.ts
import request from 'supertest';
import { app } from '../../server';
import { prisma } from '../../db';

describe('Auth Routes', () => {
  beforeEach(async () => {
    // Clean up database
    await prisma.user.deleteMany();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  describe('POST /api/auth/register', () => {
    it('should register new user', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
          name: 'Test User',
        });

      expect(response.status).toBe(201);
      expect(response.body.user.email).toBe('test@example.com');
      expect(response.body.accessToken).toBeDefined();
      expect(response.body.refreshToken).toBeDefined();
    });

    it('should return 400 for invalid email', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'invalid-email',
          password: 'password123',
        });

      expect(response.status).toBe(400);
      expect(response.body.error).toBe('Validation failed');
    });

    it('should return 409 for existing user', async () => {
      // Create user first
      await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      // Try to register again
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      expect(response.status).toBe(409);
      expect(response.body.error).toBe('User already exists');
    });
  });

  describe('POST /api/auth/login', () => {
    beforeEach(async () => {
      // Register user for login tests
      await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      expect(response.status).toBe(200);
      expect(response.body.accessToken).toBeDefined();
    });

    it('should return 401 for invalid password', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'wrong-password',
        });

      expect(response.status).toBe(401);
      expect(response.body.error).toBe('Invalid credentials');
    });
  });
});
```

## Verification Commands

```bash
# Start dev server
npm run dev

# Test registration
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123","name":"Test User"}'

# Test login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Test protected route
curl http://localhost:3000/api/users/me \
  -H "Authorization: Bearer <access_token>"
```
