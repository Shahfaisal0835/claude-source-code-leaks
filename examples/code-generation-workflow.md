# Code Generation Workflow: End-to-End Example

> **Note:** This document contains an illustrative, educational example of how an AI coding assistant might handle a realistic development task from start to finish. All code, file names, and tool calls are fictional and for educational purposes only.

---

## Scenario

A developer asks an AI coding assistant to implement a new feature: **"Add rate limiting to the REST API."**

This example traces the full workflow from the initial request through exploration, planning, implementation, testing, and verification.

---

## Step 1: Receiving and Parsing the Task

**User input:**
```
Add rate limiting to our REST API. Each user should be limited to 100 requests per 15 minutes. 
Return HTTP 429 with a Retry-After header when the limit is exceeded.
```

**Model's internal reasoning (thinking/scratchpad):**
```
The user wants rate limiting. I need to understand:
1. What framework/language is this API using?
2. Is there an existing rate-limiting library in use?
3. Where are API routes defined?
4. Is there an existing middleware pattern?

I'll start by exploring the project structure.
```

---

## Step 2: Codebase Exploration

The model issues a series of tool calls to understand the codebase:

**Tool call 1: List top-level files**
```json
{ "tool": "list_directory", "arguments": { "path": "." } }
```
Result: `src/, tests/, package.json, README.md, tsconfig.json`

**Tool call 2: Read package.json**
```json
{ "tool": "read_file", "arguments": { "path": "package.json" } }
```
Result:
```json
{
  "dependencies": {
    "express": "^4.18.2",
    "redis": "^4.6.0"
  }
}
```

**Tool call 3: Explore src directory**
```json
{ "tool": "list_directory", "arguments": { "path": "src" } }
```
Result: `app.ts, routes/, middleware/, services/`

**Tool call 4: Read middleware directory**
```json
{ "tool": "list_directory", "arguments": { "path": "src/middleware" } }
```
Result: `auth.ts, logging.ts, errorHandler.ts`

**Tool call 5: Read app.ts to understand middleware registration**
```json
{ "tool": "read_file", "arguments": { "path": "src/app.ts" } }
```
Result:
```typescript
import express from 'express';
import { authMiddleware } from './middleware/auth';
import { loggingMiddleware } from './middleware/logging';
import { errorHandler } from './middleware/errorHandler';
import { userRoutes } from './routes/users';
import { productRoutes } from './routes/products';

const app = express();

app.use(express.json());
app.use(loggingMiddleware);
app.use(authMiddleware);

app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);

app.use(errorHandler);

export { app };
```

**Model's updated reasoning:**
```
I can see:
- Express.js framework
- Redis is already a dependency (ideal for rate limiting)
- Middleware pattern is already established in src/middleware/
- Middleware is applied globally in app.ts

I'll create src/middleware/rateLimiter.ts and register it in app.ts.
I should use Redis for the rate limit store since it's already available.
No need to add a new dependency - I can implement this with the existing redis client.

However, I should check if there's already a redis client being used somewhere.
```

**Tool call 6: Search for Redis usage**
```json
{ "tool": "search_code", "arguments": { "query": "redis", "glob": "src/**/*.ts" } }
```
Result: `src/services/cache.ts` uses Redis.

**Tool call 7: Read cache.ts**
```json
{ "tool": "read_file", "arguments": { "path": "src/services/cache.ts" } }
```
Result:
```typescript
import { createClient } from 'redis';

export const redisClient = createClient({
  url: process.env.REDIS_URL ?? 'redis://localhost:6379'
});

redisClient.on('error', (err) => console.error('Redis error:', err));
```

---

## Step 3: Planning

**Model's plan (visible to user):**
```
Here's my plan:

1. Create `src/middleware/rateLimiter.ts`
   - Use the existing `redisClient` from `src/services/cache.ts`
   - Implement sliding window rate limiting: 100 requests per 15 minutes per user
   - Return HTTP 429 with `Retry-After` header when limit exceeded
   - Use the authenticated user's ID as the rate limit key

2. Register the middleware in `src/app.ts`
   - Add after `authMiddleware` (so we have the user ID available)

3. Add tests in `tests/middleware/rateLimiter.test.ts`

Shall I proceed?
```

**User:** "Yes, go ahead."

---

## Step 4: Implementation

**Tool call: Write rateLimiter.ts**
```json
{
  "tool": "write_file",
  "arguments": {
    "path": "src/middleware/rateLimiter.ts",
    "content": "..."
  }
}
```

Content written:
```typescript
import { Request, Response, NextFunction } from 'express';
import { redisClient } from '../services/cache';

const WINDOW_SECONDS = 15 * 60; // 15 minutes
const MAX_REQUESTS = 100;

export async function rateLimiter(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  const userId = (req as Request & { userId?: string }).userId;

  if (!userId) {
    next();
    return;
  }

  const key = `rate_limit:${userId}`;

  const current = await redisClient.incr(key);

  if (current === 1) {
    await redisClient.expire(key, WINDOW_SECONDS);
  }

  if (current > MAX_REQUESTS) {
    const ttl = await redisClient.ttl(key);
    res.set('Retry-After', String(ttl));
    res.status(429).json({
      error: 'Too Many Requests',
      retryAfter: ttl
    });
    return;
  }

  res.set('X-RateLimit-Limit', String(MAX_REQUESTS));
  res.set('X-RateLimit-Remaining', String(MAX_REQUESTS - current));

  next();
}
```

**Tool call: Update app.ts**
```json
{
  "tool": "write_file",
  "arguments": {
    "path": "src/app.ts",
    "content": "..."
  }
}
```

Updated content:
```typescript
import express from 'express';
import { authMiddleware } from './middleware/auth';
import { loggingMiddleware } from './middleware/logging';
import { errorHandler } from './middleware/errorHandler';
import { rateLimiter } from './middleware/rateLimiter';
import { userRoutes } from './routes/users';
import { productRoutes } from './routes/products';

const app = express();

app.use(express.json());
app.use(loggingMiddleware);
app.use(authMiddleware);
app.use(rateLimiter);  // Added after auth so userId is available

app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);

app.use(errorHandler);

export { app };
```

---

## Step 5: Writing Tests

**Tool call: Write test file**
```typescript
// tests/middleware/rateLimiter.test.ts
import { rateLimiter } from '../../src/middleware/rateLimiter';
import { redisClient } from '../../src/services/cache';

jest.mock('../../src/services/cache', () => ({
  redisClient: {
    incr: jest.fn(),
    expire: jest.fn(),
    ttl: jest.fn()
  }
}));

describe('rateLimiter', () => {
  let req: any, res: any, next: jest.Mock;

  beforeEach(() => {
    req = { userId: 'user-123' };
    res = {
      set: jest.fn(),
      status: jest.fn().mockReturnThis(),
      json: jest.fn()
    };
    next = jest.fn();
    jest.clearAllMocks();
  });

  it('calls next() when under the rate limit', async () => {
    (redisClient.incr as jest.Mock).mockResolvedValue(1);
    (redisClient.expire as jest.Mock).mockResolvedValue(1);

    await rateLimiter(req, res, next);

    expect(next).toHaveBeenCalled();
    expect(res.status).not.toHaveBeenCalledWith(429);
  });

  it('returns 429 when over the rate limit', async () => {
    (redisClient.incr as jest.Mock).mockResolvedValue(101);
    (redisClient.ttl as jest.Mock).mockResolvedValue(300);

    await rateLimiter(req, res, next);

    expect(res.status).toHaveBeenCalledWith(429);
    expect(res.set).toHaveBeenCalledWith('Retry-After', '300');
    expect(next).not.toHaveBeenCalled();
  });

  it('skips rate limiting for unauthenticated requests', async () => {
    req.userId = undefined;

    await rateLimiter(req, res, next);

    expect(next).toHaveBeenCalled();
    expect(redisClient.incr).not.toHaveBeenCalled();
  });
});
```

---

## Step 6: Running Tests

**Tool call: Run tests**
```json
{ "tool": "run_command", "arguments": { "cmd": "npm test tests/middleware/rateLimiter.test.ts" } }
```

Result:
```
PASS tests/middleware/rateLimiter.test.ts
  rateLimiter
    ✓ calls next() when under the rate limit (12ms)
    ✓ returns 429 when over the rate limit (3ms)
    ✓ skips rate limiting for unauthenticated requests (2ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
```

---

## Step 7: Final Summary

**Model's response to user:**
```
I've implemented rate limiting for your REST API. Here's a summary of changes:

**Files changed:**
- `src/middleware/rateLimiter.ts` (new) – rate limiter middleware using Redis sliding window
- `src/app.ts` (updated) – registered rateLimiter after authMiddleware
- `tests/middleware/rateLimiter.test.ts` (new) – 3 passing tests

**Behavior:**
- 100 requests per 15-minute window per authenticated user
- Returns HTTP 429 with `Retry-After: <seconds>` header when exceeded
- Unauthenticated requests are not rate limited (handled by authMiddleware)
- Adds `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers to all responses

**Note:** Make sure your Redis connection is active in production (`REDIS_URL` env var).
```

---

## Key Takeaways from This Workflow

1. **Exploration precedes action** – The model reads the codebase before writing a single line of code.
2. **Reuse over reinvention** – The model uses the existing Redis client instead of adding a new dependency.
3. **Plan before executing** – The model presents its plan and waits for confirmation on risky changes.
4. **Tests are part of the deliverable** – A complete implementation includes tests.
5. **Self-verification** – The model runs the tests itself to confirm correctness before reporting completion.
6. **Concise summary** – The final response is a clear, actionable summary of what changed and why.
