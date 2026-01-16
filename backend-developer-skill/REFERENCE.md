# Backend Developer - Technical Reference

## Workflow 1: Setting Up Production-Ready Express + TypeScript API with JWT Auth

**Goal:** Bootstrap secure REST API with TypeScript, Prisma ORM, JWT auth, validation, error handling in <1 hour.

### Step 1: Initialize Project with TypeScript

```bash
mkdir my-backend-api && cd my-backend-api
npm init -y
npm install express cors helmet dotenv
npm install -D typescript @types/node @types/express ts-node-dev
npx tsc --init
```

### Step 2: Setup Prisma ORM

```bash
npm install prisma @prisma/client
npm install -D prisma
npx prisma init
```

```prisma
# prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

```bash
# Run migrations
npx prisma migrate dev --name init
npx prisma generate
```

### Step 3: Implement JWT Authentication

```typescript
// src/auth/jwt.ts
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';
const JWT_EXPIRES_IN = '15m';
const REFRESH_TOKEN_EXPIRES_IN = '7d';

export function generateTokens(userId: string) {
  const accessToken = jwt.sign({ userId }, JWT_SECRET, { 
    expiresIn: JWT_EXPIRES_IN 
  });
  
  const refreshToken = jwt.sign({ userId }, JWT_SECRET, { 
    expiresIn: REFRESH_TOKEN_EXPIRES_IN 
  });
  
  return { accessToken, refreshToken };
}

export function verifyToken(token: string) {
  try {
    return jwt.verify(token, JWT_SECRET) as { userId: string };
  } catch (error) {
    throw new Error('Invalid token');
  }
}

// Middleware
export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const payload = verifyToken(token);
    req.userId = payload.userId; // Attach to request
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Step 4: Setup Input Validation with Zod

```bash
npm install zod
```

```typescript
// src/validators/user.validator.ts
import { z } from 'zod';

export const registerSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  name: z.string().optional(),
});

export const loginSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(1, 'Password required'),
});

// Middleware
export function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({ 
          error: 'Validation failed', 
          details: error.errors 
        });
      }
      next(error);
    }
  };
}
```

### Step 5: Implement Auth Routes

```typescript
// src/routes/auth.routes.ts
import { Router } from 'express';
import bcrypt from 'bcrypt';
import { PrismaClient } from '@prisma/client';
import { validate } from '../validators/user.validator';
import { registerSchema, loginSchema } from '../validators/user.validator';
import { generateTokens } from '../auth/jwt';

const router = Router();
const prisma = new PrismaClient();

// Register
router.post('/register', validate(registerSchema), async (req, res, next) => {
  try {
    const { email, password, name } = req.body;
    
    // Check if user exists
    const existingUser = await prisma.user.findUnique({ where: { email } });
    if (existingUser) {
      return res.status(409).json({ error: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = await prisma.user.create({
      data: { email, password: hashedPassword, name },
    });
    
    // Generate tokens
    const { accessToken, refreshToken } = generateTokens(user.id);
    
    res.status(201).json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken,
      refreshToken,
    });
  } catch (error) {
    next(error);
  }
});

// Login
router.post('/login', validate(loginSchema), async (req, res, next) => {
  try {
    const { email, password } = req.body;
    
    // Find user
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Verify password
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate tokens
    const { accessToken, refreshToken } = generateTokens(user.id);
    
    res.json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken,
      refreshToken,
    });
  } catch (error) {
    next(error);
  }
});

export default router;
```

### Step 6: Global Error Handler

```typescript
// src/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error('Error:', error);
  
  res.status(500).json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? error.message : undefined,
  });
}
```

### Step 7: Main Server Setup

```typescript
// src/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import dotenv from 'dotenv';
import authRoutes from './routes/auth.routes';
import { errorHandler } from './middleware/error.middleware';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json());

// Routes
app.use('/api/auth', authRoutes);

// Error handling
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Workflow 2: Implementing Background Jobs with Bull Queue

**Goal:** Process long-running tasks (email sending, image processing) asynchronously.

### Step 1: Install Dependencies

```bash
npm install bull @types/bull
```

### Step 2: Setup Queue

```typescript
// src/queues/email.queue.ts
import Queue from 'bull';
import { sendEmail } from '../services/email.service';

const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

// Process jobs
emailQueue.process(async (job) => {
  const { to, subject, body } = job.data;
  console.log(`Sending email to ${to}...`);
  
  await sendEmail(to, subject, body);
  
  console.log(`Email sent to ${to}`);
});

// Add job to queue
export function queueEmail(to: string, subject: string, body: string) {
  return emailQueue.add({
    to,
    subject,
    body,
  }, {
    attempts: 3, // Retry 3 times on failure
    backoff: {
      type: 'exponential',
      delay: 2000, // 2s, 4s, 8s
    },
  });
}

export { emailQueue };
```

### Step 3: Use in API Route

```typescript
// src/routes/user.routes.ts
import { queueEmail } from '../queues/email.queue';

router.post('/send-welcome-email', authMiddleware, async (req, res) => {
  const user = await prisma.user.findUnique({ where: { id: req.userId } });
  
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  // Queue email (non-blocking)
  await queueEmail(
    user.email,
    'Welcome!',
    `Hello ${user.name}, welcome to our platform!`
  );
  
  res.json({ message: 'Welcome email queued' });
});
```

## Integration Patterns

### api-designer
- **Handoff:** backend-developer implements API routes → api-designer validates OpenAPI spec compliance
- **Collaboration:** backend-developer creates endpoints → api-designer ensures RESTful conventions, HTTP status codes
- **Tools:** backend-developer uses Express/FastAPI; api-designer validates with Swagger/Postman

### database-administrator
- **Handoff:** backend-developer implements ORM models → database-administrator optimizes database schema, indexes
- **Collaboration:** backend-developer writes queries → database-administrator tunes query performance
- **Tools:** backend-developer uses Prisma/TypeORM; database-administrator uses EXPLAIN ANALYZE, pg_stat_statements

### frontend-developer
- **Handoff:** backend-developer creates API endpoints → frontend-developer consumes via Axios/Fetch
- **Collaboration:** backend-developer defines API contracts → frontend-developer implements TypeScript types
- **Tools:** Both use TypeScript; backend-developer provides OpenAPI spec for frontend code generation

### devops-engineer
- **Handoff:** backend-developer creates Dockerfile → devops-engineer sets up CI/CD pipeline
- **Collaboration:** backend-developer implements health checks → devops-engineer configures Kubernetes probes
- **Tools:** backend-developer uses Docker; devops-engineer uses Kubernetes, GitHub Actions

### security-auditor
- **Handoff:** backend-developer implements auth → security-auditor audits for vulnerabilities (SQL injection, XSS)
- **Collaboration:** backend-developer adds input validation → security-auditor verifies secure coding practices
- **Tools:** backend-developer uses Zod/Joi; security-auditor uses OWASP ZAP, Burp Suite

## Scripts Reference

### API Scaffolding
```bash
python scripts/scaffold_api.py <framework> <project_name>
# Frameworks: express, fastapi, django, spring
```

### Database Model Generation
```bash
python scripts/generate_model.py <orm> --schema <schema_file> --output <output_dir>
# ORMs: sequelize, typeorm, sqlalchemy, django, jpa
```

### Authentication Setup
```bash
python scripts/setup_auth.py <framework> <auth_type>
# Auth types: jwt, oauth2, session
```

### Middleware Generation
```bash
python scripts/create_middleware.py <framework> --output <output_dir>
```

### Error Handler Setup
```bash
python scripts/error_handler.py <framework> --output <output_dir>
```

### Deployment Script
```bash
./scripts/deploy.sh [OPTIONS]
# Options:
# --skip-tests: Skip test execution
# --platform <kubernetes|aws|gcp>: Deployment platform
# --rollback: Rollback deployment
# --health-check: Run health check only
```
