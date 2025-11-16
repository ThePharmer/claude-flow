# Secure Coding Patterns for claude-flow

This guide provides secure coding patterns based on vulnerabilities found in the security review.

---

## 1. Command Execution Security

### ❌ VULNERABLE
```typescript
// DON'T: String concatenation with shell execution
const command = `gh issue create --title "${title}" --body "${body}"`;
execSync(command);

// DON'T: User input in shell strings
execSync(`cp "${userPath}" "${backupPath}"`);
```

### ✅ SECURE
```typescript
// DO: Use spawn with array arguments
import { spawn } from 'child_process';

const child = spawn('gh', ['issue', 'create', '--title', title, '--body', body], {
  shell: false, // CRITICAL: Prevents shell injection
  stdio: 'pipe'
});

// DO: Use Node.js APIs instead of shell commands
import { copyFileSync } from 'fs';
copyFileSync(userPath, backupPath);

// DO: Use the github-cli-safety-wrapper pattern
import { GitHubCliSafe } from './utils/github-cli-safety-wrapper.js';
const cli = new GitHubCliSafe();
await cli.createIssue({ title, body });
```

---

## 2. SQL Injection Prevention

### ❌ VULNERABLE
```typescript
// DON'T: String interpolation in SQL
this.db.prepare(`
  DELETE FROM memory
  WHERE metadata LIKE '%"swarmId":"${swarmId}"%'
`).run();

// DON'T: Template literals with user data
const query = `SELECT * FROM users WHERE id = ${userId}`;
```

### ✅ SECURE
```typescript
// DO: Use parameterized queries
this.db.prepare(`
  DELETE FROM memory
  WHERE json_extract(metadata, '$.swarmId') = ?
`).run(swarmId);

// DO: Prepared statements with placeholders
const stmt = this.db.prepare('SELECT * FROM users WHERE id = ?');
const user = stmt.get(userId);

// DO: Whitelist column names for dynamic queries
const ALLOWED_COLUMNS = ['name', 'status', 'created_at'];

async updateAgent(id: string, updates: Record<string, any>) {
  const setClauses: string[] = [];
  const values: any[] = [];

  for (const [key, value] of Object.entries(updates)) {
    if (!ALLOWED_COLUMNS.includes(key)) {
      throw new Error(`Invalid column: ${key}`);
    }
    setClauses.push(`${key} = ?`);
    values.push(value);
  }

  const stmt = this.db.prepare(`
    UPDATE agents SET ${setClauses.join(', ')} WHERE id = ?
  `);
  stmt.run(...values, id);
}
```

---

## 3. Authentication & Password Security

### ❌ VULNERABLE
```typescript
// DON'T: Weak hashing (SHA256)
import { createHash } from 'crypto';
const hash = createHash('sha256').update(password + 'salt').digest('hex');

// DON'T: Hardcoded credentials
const adminPassword = 'admin123';

// DON'T: Static salt
const salt = 'hardcoded_salt';
```

### ✅ SECURE
```typescript
// DO: Use bcrypt with proper rounds
import bcrypt from 'bcrypt';

async function hashPassword(password: string): Promise<string> {
  const saltRounds = 12; // Recommended: 10-12
  return await bcrypt.hash(password, saltRounds);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}

// DO: Generate random credentials
import { randomBytes } from 'crypto';

function generateSecurePassword(length: number = 32): string {
  return randomBytes(length).toString('base64url');
}

// DO: Force credential change on first login
if (user.mustChangePassword) {
  throw new AuthenticationError('Password change required');
}
```

---

## 4. XSS Prevention

### ❌ VULNERABLE
```typescript
// DON'T: Use innerHTML with user data
element.innerHTML = userInput;
element.innerHTML = `<div>${data}</div>`;

// DON'T: Directly insert unescaped content
div.innerHTML += text;
```

### ✅ SECURE
```typescript
// DO: Use textContent for text
element.textContent = userInput;

// DO: Create elements programmatically
const div = document.createElement('div');
div.textContent = data;
parent.appendChild(div);

// DO: Use sanitization library for rich content
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userHtml);

// DO: Use Content Security Policy
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    }
  }
}));
```

---

## 5. Path Traversal Prevention

### ❌ VULNERABLE
```typescript
// DON'T: Use user input directly in file paths
const filePath = `/uploads/${userFileName}`;
fs.readFileSync(filePath);

// DON'T: Unvalidated path concatenation
execSync(`node deploy.js "${targetPath}"`);
```

### ✅ SECURE
```typescript
import path from 'path';
import fs from 'fs';

// DO: Validate and normalize paths
function getSecurePath(basePath: string, userPath: string): string {
  const normalizedPath = path.normalize(userPath);
  const fullPath = path.resolve(basePath, normalizedPath);

  // Ensure path doesn't escape base directory
  if (!fullPath.startsWith(path.resolve(basePath))) {
    throw new Error('Invalid path: attempted directory traversal');
  }

  return fullPath;
}

// DO: Use allowlist for file operations
const ALLOWED_EXTENSIONS = ['.txt', '.json', '.md'];

function validateFileExtension(filename: string): void {
  const ext = path.extname(filename).toLowerCase();
  if (!ALLOWED_EXTENSIONS.includes(ext)) {
    throw new Error(`File type not allowed: ${ext}`);
  }
}
```

---

## 6. Input Validation

### ❌ VULNERABLE
```typescript
// DON'T: Trust user input
function processInput(data: any) {
  return data; // No validation
}

// DON'T: Minimal validation
if (email.includes('@')) {
  // Not sufficient
}
```

### ✅ SECURE
```typescript
// DO: Comprehensive validation with allowlists
function validateEmail(email: string): boolean {
  const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

  if (!emailRegex.test(email)) {
    return false;
  }

  // Additional checks
  if (email.length > 254) return false; // RFC 5321
  const [local, domain] = email.split('@');
  if (local.length > 64) return false; // RFC 5321

  return true;
}

// DO: Use validation libraries
import Ajv from 'ajv';

const ajv = new Ajv();
const schema = {
  type: 'object',
  properties: {
    username: { type: 'string', minLength: 3, maxLength: 50 },
    email: { type: 'string', format: 'email' },
    age: { type: 'integer', minimum: 0, maximum: 150 }
  },
  required: ['username', 'email'],
  additionalProperties: false
};

const validate = ajv.compile(schema);

function validateUser(data: unknown): boolean {
  if (!validate(data)) {
    console.error('Validation errors:', validate.errors);
    return false;
  }
  return true;
}
```

---

## 7. Secrets Management

### ❌ VULNERABLE
```typescript
// DON'T: Hardcode secrets
const apiKey = 'sk-1234567890abcdef';
const dbPassword = 'mypassword123';

// DON'T: Commit .env files
// .env file in git repository

// DON'T: Log sensitive data
console.log('User password:', password);
```

### ✅ SECURE
```typescript
// DO: Use environment variables
const apiKey = process.env.ANTHROPIC_API_KEY;
if (!apiKey) {
  throw new Error('ANTHROPIC_API_KEY environment variable not set');
}

// DO: Use secrets management services
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

async function getSecret(secretName: string): Promise<string> {
  const client = new SecretsManager({ region: 'us-east-1' });
  const response = await client.getSecretValue({ SecretId: secretName });
  return response.SecretString!;
}

// DO: Redact sensitive data in logs
function redactSensitive(data: any): any {
  const redacted = { ...data };
  const sensitiveFields = ['password', 'apiKey', 'token', 'secret'];

  for (const field of sensitiveFields) {
    if (redacted[field]) {
      redacted[field] = '***REDACTED***';
    }
  }

  return redacted;
}

console.log('User data:', redactSensitive(userData));

// DO: Add .env to .gitignore
// .gitignore:
// .env
// .env.local
// .env.*.local
```

---

## 8. Error Handling

### ❌ VULNERABLE
```typescript
// DON'T: Expose internal errors
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack,
    query: req.query
  });
});

// DON'T: Different errors for enumeration
if (!user) throw new Error('User not found');
if (!validPassword) throw new Error('Invalid password');
```

### ✅ SECURE
```typescript
// DO: Generic errors in production
app.use((err, req, res, next) => {
  // Log full error internally
  logger.error('Request error', {
    error: err.message,
    stack: err.stack,
    requestId: req.id
  });

  // Return generic error to client
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({
      error: 'Internal server error',
      requestId: req.id // For support tracking
    });
  } else {
    // Detailed errors in development
    res.status(500).json({
      error: err.message,
      stack: err.stack
    });
  }
});

// DO: Consistent error messages
if (!user || !validPassword) {
  // Same error for both cases - prevents enumeration
  throw new AuthenticationError('Invalid credentials');
}
```

---

## 9. Rate Limiting

### ❌ VULNERABLE
```typescript
// DON'T: In-memory rate limiting only
const attempts = new Map<string, number>();

function checkLimit(userId: string) {
  const count = attempts.get(userId) || 0;
  if (count > 10) throw new Error('Too many requests');
  attempts.set(userId, count + 1);
}
```

### ✅ SECURE
```typescript
// DO: Distributed rate limiting with Redis
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT || '6379')
});

const rateLimiter = new RateLimiterRedis({
  storeClient: redis,
  points: 10, // Number of requests
  duration: 60, // Per 60 seconds
  blockDuration: 300, // Block for 5 minutes if exceeded
});

async function checkRateLimit(identifier: string): Promise<void> {
  try {
    await rateLimiter.consume(identifier);
  } catch (error) {
    if (error instanceof Error) {
      throw new RateLimitError('Too many requests, please try again later');
    }
    throw error;
  }
}

// DO: Multiple rate limit tiers
const strictLimiter = new RateLimiterRedis({ points: 3, duration: 60 });
const normalLimiter = new RateLimiterRedis({ points: 100, duration: 60 });

// Apply strict limiting to sensitive endpoints
app.post('/api/login', async (req, res) => {
  await checkRateLimit(req.ip, strictLimiter);
  // ... authentication logic
});
```

---

## 10. Session Management

### ❌ VULNERABLE
```typescript
// DON'T: Predictable session IDs
const sessionId = Date.now().toString();

// DON'T: Long session timeouts
const SESSION_TIMEOUT = 30 * 24 * 60 * 60 * 1000; // 30 days

// DON'T: Store sessions client-side only
localStorage.setItem('session', sessionId);
```

### ✅ SECURE
```typescript
// DO: Cryptographically random session IDs
import { randomBytes } from 'crypto';

function generateSessionId(): string {
  return randomBytes(32).toString('base64url');
}

// DO: Appropriate timeouts
const SESSION_TIMEOUT = 60 * 60 * 1000; // 1 hour
const ABSOLUTE_TIMEOUT = 8 * 60 * 60 * 1000; // 8 hours max

// DO: Secure session storage
interface Session {
  id: string;
  userId: string;
  createdAt: Date;
  lastActivity: Date;
  absoluteExpiry: Date;
  ipAddress: string;
  userAgent: string;
}

// DO: Regenerate session on privilege change
async function promoteUser(userId: string, newRole: string) {
  // Invalidate all existing sessions
  await invalidateAllUserSessions(userId);

  // Update user role
  await updateUserRole(userId, newRole);

  // User must re-authenticate
}

// DO: Implement session fixation protection
app.post('/login', async (req, res) => {
  // Destroy old session
  req.session.destroy();

  // Create new session after authentication
  const newSession = await createSession(user.id);
  res.cookie('sessionId', newSession.id, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: SESSION_TIMEOUT
  });
});
```

---

## Quick Reference

| Vulnerability | Prevention |
|---------------|------------|
| Command Injection | Use spawn with arrays, not shell strings |
| SQL Injection | Parameterized queries, whitelist columns |
| XSS | textContent over innerHTML, CSP headers |
| Path Traversal | Validate paths, check they don't escape base |
| Weak Crypto | bcrypt for passwords, strong random IDs |
| Hardcoded Secrets | Environment variables, secrets manager |
| Information Disclosure | Generic errors, redact sensitive data |
| Session Attacks | Random IDs, regenerate on privilege change |
| Rate Limiting | Redis-based distributed limiting |
| Missing Security Headers | Helmet middleware with strict CSP |

---

## Security Testing Checklist

Before deploying:

- [ ] All user inputs validated
- [ ] SQL queries use parameterized statements
- [ ] No shell execution with user data
- [ ] Passwords hashed with bcrypt
- [ ] No hardcoded secrets
- [ ] Security headers configured
- [ ] Rate limiting enabled
- [ ] Sessions secure and time-limited
- [ ] Error messages generic in production
- [ ] Dependencies updated and audited
- [ ] HTTPS enforced
- [ ] CORS properly configured

---

Last Updated: 2025-11-16
