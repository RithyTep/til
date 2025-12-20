# API Security Best Practices

<div align="center">

![Security](https://img.shields.io/badge/Security-API_Security-red?style=for-the-badge&logo=owasp&logoColor=white)
![Auth](https://img.shields.io/badge/Auth-JWT_|_OAuth2-yellow?style=for-the-badge)

*Production-grade security for modern APIs.*

![API Security](https://upload.wikimedia.org/wikipedia/commons/4/45/Security_lock_blue.svg)

</div>

## Authentication & Authorization

```typescript
// JWT with refresh token rotation
import { SignJWT, jwtVerify, JWTPayload } from 'jose';

interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

class TokenService {
  private readonly accessSecret = new TextEncoder().encode(process.env.JWT_ACCESS_SECRET);
  private readonly refreshSecret = new TextEncoder().encode(process.env.JWT_REFRESH_SECRET);

  async generateTokenPair(userId: string, roles: string[]): Promise<TokenPair> {
    const tokenFamily = crypto.randomUUID();

    const accessToken = await new SignJWT({ sub: userId, roles })
      .setProtectedHeader({ alg: 'HS256' })
      .setIssuedAt()
      .setExpirationTime('15m')
      .sign(this.accessSecret);

    const refreshToken = await new SignJWT({ sub: userId, family: tokenFamily })
      .setProtectedHeader({ alg: 'HS256' })
      .setIssuedAt()
      .setExpirationTime('7d')
      .sign(this.refreshSecret);

    // Store refresh token hash for rotation detection
    await this.redis.set(
      `refresh:${userId}:${tokenFamily}`,
      crypto.createHash('sha256').update(refreshToken).digest('hex'),
      'EX',
      7 * 24 * 60 * 60
    );

    return { accessToken, refreshToken };
  }

  async rotateRefreshToken(oldRefreshToken: string): Promise<TokenPair> {
    const payload = await jwtVerify(oldRefreshToken, this.refreshSecret);
    const { sub: userId, family } = payload.payload as JWTPayload & { family: string };

    // Check if token was already used (replay attack detection)
    const storedHash = await this.redis.get(`refresh:${userId}:${family}`);
    const currentHash = crypto.createHash('sha256').update(oldRefreshToken).digest('hex');

    if (storedHash !== currentHash) {
      // Token reuse detected - revoke entire family
      await this.redis.del(`refresh:${userId}:${family}`);
      throw new SecurityError('Token reuse detected - session invalidated');
    }

    // Generate new pair with same family
    return this.generateTokenPair(userId!, []);
  }

  async verifyAccessToken(token: string): Promise<JWTPayload> {
    const { payload } = await jwtVerify(token, this.accessSecret);
    return payload;
  }
}

// RBAC + ABAC hybrid authorization
interface AuthContext {
  userId: string;
  roles: string[];
  permissions: string[];
  attributes: Record<string, unknown>;
}

class PolicyEngine {
  private policies: Policy[] = [];

  async evaluate(context: AuthContext, resource: Resource, action: string): Promise<boolean> {
    for (const policy of this.policies) {
      const result = await policy.evaluate(context, resource, action);
      if (result === 'deny') return false;
      if (result === 'allow') return true;
    }
    return false; // Default deny
  }
}

// Policy example
const documentPolicy: Policy = {
  name: 'document-access',
  evaluate: async (ctx, resource, action) => {
    if (resource.type !== 'document') return 'not-applicable';

    // Admins can do anything
    if (ctx.roles.includes('admin')) return 'allow';

    // Owners have full access
    if (resource.ownerId === ctx.userId) return 'allow';

    // Check explicit permissions
    if (ctx.permissions.includes(`document:${action}`)) {
      // ABAC: Check department match
      if (resource.department === ctx.attributes.department) {
        return 'allow';
      }
    }

    return 'deny';
  },
};
```

## Rate Limiting & DDoS Protection

```typescript
// Sliding window rate limiter
class SlidingWindowRateLimiter {
  constructor(
    private redis: Redis,
    private windowSize: number, // in seconds
    private maxRequests: number
  ) {}

  async isAllowed(key: string): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
    const now = Date.now();
    const windowStart = now - this.windowSize * 1000;

    const pipeline = this.redis.pipeline();

    // Remove old entries
    pipeline.zremrangebyscore(key, 0, windowStart);

    // Add current request
    pipeline.zadd(key, now, `${now}:${crypto.randomUUID()}`);

    // Count requests in window
    pipeline.zcard(key);

    // Set expiry
    pipeline.expire(key, this.windowSize);

    const results = await pipeline.exec();
    const count = results?.[2]?.[1] as number;

    return {
      allowed: count <= this.maxRequests,
      remaining: Math.max(0, this.maxRequests - count),
      resetAt: now + this.windowSize * 1000,
    };
  }
}

// Adaptive rate limiting based on user behavior
class AdaptiveRateLimiter {
  private readonly baseLimits = {
    anonymous: { requests: 100, window: 60 },
    authenticated: { requests: 1000, window: 60 },
    premium: { requests: 10000, window: 60 },
  };

  async getLimit(userId: string | null): Promise<RateLimit> {
    if (!userId) return this.baseLimits.anonymous;

    const user = await this.userService.get(userId);
    const baseLimit = this.baseLimits[user.tier];

    // Adjust based on historical behavior
    const trustScore = await this.calculateTrustScore(userId);

    return {
      requests: Math.floor(baseLimit.requests * trustScore),
      window: baseLimit.window,
    };
  }

  private async calculateTrustScore(userId: string): Promise<number> {
    const metrics = await this.getRecentMetrics(userId);

    let score = 1.0;

    // Reduce for high error rates
    if (metrics.errorRate > 0.1) score *= 0.5;

    // Reduce for unusual patterns
    if (metrics.requestsPerSecondVariance > 100) score *= 0.7;

    // Increase for long-term good behavior
    if (metrics.accountAge > 365 && metrics.lifetimeErrorRate < 0.01) {
      score *= 1.5;
    }

    return Math.max(0.1, Math.min(2.0, score));
  }
}

// Circuit breaker for downstream protection
class CircuitBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failures = 0;
  private lastFailure = 0;

  constructor(
    private readonly threshold: number = 5,
    private readonly timeout: number = 30000,
    private readonly halfOpenRequests: number = 3
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = 'half-open';
      } else {
        throw new CircuitBreakerOpenError();
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    if (this.state === 'half-open') {
      this.state = 'closed';
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailure = Date.now();

    if (this.failures >= this.threshold) {
      this.state = 'open';
    }
  }
}
```

## Input Validation & Sanitization

```typescript
import { z } from 'zod';
import xss from 'xss';
import sqlstring from 'sqlstring';

// Strict input schemas
const CreateUserSchema = z.object({
  email: z.string()
    .email()
    .max(255)
    .transform(v => v.toLowerCase().trim()),

  password: z.string()
    .min(12)
    .regex(/[A-Z]/, 'Must contain uppercase')
    .regex(/[a-z]/, 'Must contain lowercase')
    .regex(/[0-9]/, 'Must contain number')
    .regex(/[^A-Za-z0-9]/, 'Must contain special character'),

  name: z.string()
    .min(1)
    .max(100)
    .regex(/^[\p{L}\p{N}\s\-']+$/u, 'Invalid characters')
    .transform(v => xss(v.trim())),

  phone: z.string()
    .regex(/^\+?[1-9]\d{6,14}$/)
    .optional(),

  metadata: z.record(z.unknown())
    .optional()
    .transform(v => v ? sanitizeObject(v) : undefined),
});

// Deep sanitization
function sanitizeObject(obj: Record<string, unknown>): Record<string, unknown> {
  const sanitized: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(obj)) {
    // Prevent prototype pollution
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      continue;
    }

    if (typeof value === 'string') {
      sanitized[key] = xss(value);
    } else if (typeof value === 'object' && value !== null) {
      sanitized[key] = sanitizeObject(value as Record<string, unknown>);
    } else {
      sanitized[key] = value;
    }
  }

  return sanitized;
}

// SQL injection prevention (parameterized queries)
class SafeQueryBuilder {
  private query = '';
  private params: unknown[] = [];

  select(columns: string[]): this {
    // Whitelist column names
    const safeColumns = columns.filter(c => /^[a-zA-Z_][a-zA-Z0-9_]*$/.test(c));
    this.query = `SELECT ${safeColumns.join(', ')}`;
    return this;
  }

  from(table: string): this {
    if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(table)) {
      throw new Error('Invalid table name');
    }
    this.query += ` FROM ${table}`;
    return this;
  }

  where(column: string, value: unknown): this {
    if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(column)) {
      throw new Error('Invalid column name');
    }
    this.query += ` WHERE ${column} = $${this.params.length + 1}`;
    this.params.push(value);
    return this;
  }

  build(): { query: string; params: unknown[] } {
    return { query: this.query, params: this.params };
  }
}
```

## Security Headers & CORS

```typescript
// Security middleware
function securityHeaders(req: Request, res: Response, next: NextFunction) {
  // Prevent XSS
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-XSS-Protection', '1; mode=block');

  // Prevent clickjacking
  res.setHeader('X-Frame-Options', 'DENY');

  // Content Security Policy
  res.setHeader('Content-Security-Policy', [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' https://cdn.example.com",
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "font-src 'self' https://fonts.gstatic.com",
    "connect-src 'self' https://api.example.com",
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'",
  ].join('; '));

  // HSTS
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');

  // Permissions Policy
  res.setHeader('Permissions-Policy', [
    'camera=()',
    'microphone=()',
    'geolocation=(self)',
    'payment=(self)',
  ].join(', '));

  next();
}

// CORS configuration
const corsOptions: CorsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://admin.example.com',
    ];

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS not allowed'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-RateLimit-Remaining', 'X-RateLimit-Reset'],
  credentials: true,
  maxAge: 86400,
};
```

## Secrets Management

```typescript
// HashiCorp Vault integration
import Vault from 'node-vault';

class SecretManager {
  private vault: Vault.client;
  private cache: Map<string, { value: unknown; expiresAt: number }> = new Map();

  constructor() {
    this.vault = Vault({
      endpoint: process.env.VAULT_ADDR,
      token: process.env.VAULT_TOKEN,
    });
  }

  async getSecret(path: string, ttl = 300): Promise<unknown> {
    const cached = this.cache.get(path);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }

    const { data } = await this.vault.read(`secret/data/${path}`);
    const value = data.data;

    this.cache.set(path, {
      value,
      expiresAt: Date.now() + ttl * 1000,
    });

    return value;
  }

  async getDatabaseCredentials(): Promise<{ username: string; password: string }> {
    // Dynamic database credentials with lease
    const { data, lease_id, lease_duration } = await this.vault.read('database/creds/my-role');

    // Schedule renewal before expiry
    setTimeout(
      () => this.vault.renew(lease_id),
      (lease_duration - 60) * 1000
    );

    return {
      username: data.username,
      password: data.password,
    };
  }
}

// Encryption at rest
import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from 'crypto';

class FieldEncryption {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;

  constructor(masterKey: string) {
    this.key = scryptSync(masterKey, 'salt', 32);
  }

  encrypt(plaintext: string): string {
    const iv = randomBytes(16);
    const cipher = createCipheriv(this.algorithm, this.key, iv);

    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
  }

  decrypt(ciphertext: string): string {
    const [ivHex, authTagHex, encrypted] = ciphertext.split(':');

    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');

    const decipher = createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Security, API, Authentication, Authorization, Rate Limiting*
