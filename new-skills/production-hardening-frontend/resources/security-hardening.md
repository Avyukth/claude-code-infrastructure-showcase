# Security Hardening - SvelteKit Production Security (2024-2025)

Comprehensive security best practices for SvelteKit applications based on latest CVEs, OWASP guidelines, and real-world production deployments.

## Table of Contents

- [Recent Security Vulnerabilities (CVEs)](#recent-security-vulnerabilities-cves)
- [Content Security Policy (CSP)](#content-security-policy-csp)
- [XSS Prevention](#xss-prevention)
- [CSRF Protection](#csrf-protection)
- [Authentication Patterns](#authentication-patterns)
- [Input Validation](#input-validation)
- [Security Headers](#security-headers)
- [Dependency Security](#dependency-security)

---

## Recent Security Vulnerabilities (CVEs)

### CVE-2025-32388 (April 2025) - XSS via URL Search Parameters

**Critical**: XSS vulnerability when iterating over `event.url.searchParams` in load functions.

#### Vulnerable Code

```typescript
// ❌ VULNERABLE
export const load: PageServerLoad = async ({ url }) => {
  const params = {};
  for (const [key, value] of url.searchParams) {
    params[key] = value; // Unescaped!
  }
  return { params };
};
```

#### Fix with Zod Validation

```typescript
// ✅ SECURE
import { z } from 'zod';

const paramsSchema = z.object({
  query: z.string().max(100).trim(),
  filter: z.enum(['all', 'active', 'archived']),
  page: z.coerce.number().int().positive().optional()
});

export const load: PageServerLoad = async ({ url }) => {
  const rawParams = Object.fromEntries(url.searchParams);

  // Validate and sanitize
  const result = paramsSchema.safeParse(rawParams);

  if (!result.success) {
    throw error(400, 'Invalid parameters');
  }

  return { params: result.data };
};
```

### CVE-2024-53262 (November 2024) - XSS in Error Pages

**Critical**: Unescaped placeholders in `error.html` allow XSS attacks.

#### Vulnerable Pattern

```html
<!-- src/error.html - VULNERABLE -->
<h1>%sveltekit.status%</h1>
<p>%sveltekit.error.message%</p>
```

#### Fix

**Option 1**: Upgrade to SvelteKit 2.8.3+ (automatically escapes)

```bash
npm install @sveltejs/kit@latest
```

**Option 2**: Manual escaping in hooks

```typescript
// src/hooks.server.ts
export const handleError: HandleServerError = async ({ error, event }) => {
  const message = error instanceof Error ? error.message : 'Unknown error';

  return {
    message: escapeHtml(message),
    code: 'INTERNAL_ERROR'
  };
};

function escapeHtml(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

### CVE-2024-23641 - DoS via Request Body

**Impact**: GET/HEAD requests with body crash the application.

**Fix**: Upgrade to SvelteKit 2.4.3+ or 3.0.3+

```bash
npm install @sveltejs/kit@latest
```

### CVE-2023-29003 - CSRF Bypass

**Impact**: CSRF protection didn't validate `text/plain` content type.

**Fix**: Upgrade to SvelteKit 1.15.1+ (validates PUT, PATCH, DELETE methods)

---

## Content Security Policy (CSP)

### Built-in SvelteKit CSP

```javascript
// svelte.config.js
/** @type {import('@sveltejs/kit').Config} */
const config = {
  kit: {
    csp: {
      mode: 'auto', // 'hash' | 'nonce' | 'auto'
      directives: {
        'default-src': ['self'],
        'script-src': ['self'],
        // Svelte transitions require unsafe-inline for styles
        'style-src': ['self', 'unsafe-inline'],
        'img-src': ['self', 'data:', 'https:'],
        'font-src': ['self'],
        'connect-src': ['self'],
        'frame-ancestors': ['none'],
        'base-uri': ['self'],
        'form-action': ['self']
      },
      reportOnly: {
        'default-src': ['self'],
        'report-uri': ['/api/csp-report']
      }
    }
  }
};

export default config;
```

### CSP Modes Explained

| Mode | Best For | How It Works |
|------|----------|--------------|
| **hash** | Prerendered/static sites | Computes SHA256 hashes for all scripts |
| **nonce** | SSR sites | Random strings per request |
| **auto** | Hybrid apps (recommended) | Hashes for prerendered, nonces for SSR |

**⚠️ Important**: Using nonces with prerendered pages is insecure!

### Manual Nonce Usage

```html
<!-- src/app.html -->
<script nonce="%sveltekit.nonce%">
  // Your inline script
  console.log('Initialized');
</script>
```

### CSP Report Endpoint

```typescript
// src/routes/api/csp-report/+server.ts
import type { RequestHandler } from './$types';
import { logger } from '$lib/server/logger';

export const POST: RequestHandler = async ({ request }) => {
  const report = await request.json();

  logger.error({
    type: 'csp_violation',
    report
  });

  // Forward to monitoring service (Sentry, etc.)
  await sendToMonitoring('csp-violation', report);

  return new Response(null, { status: 204 });
};
```

---

## XSS Prevention

### Never Use `@html` Without Sanitization

```svelte
<script lang="ts">
  import DOMPurify from 'isomorphic-dompurify';

  export let userContent: string;

  // ❌ DANGEROUS
  // {@html userContent}

  // ✅ SAFE: Sanitize first
  $: sanitized = DOMPurify.sanitize(userContent, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'a'],
    ALLOWED_ATTR: ['href'],
    ALLOW_DATA_ATTR: false
  });
</script>

{@html sanitized}
```

### Best Practice: Avoid HTML Altogether

```svelte
<!-- ✅ BEST: Use text interpolation (automatically escaped) -->
<p>{userContent}</p>

<!-- ✅ GOOD: Use Markdown with sanitization -->
<script>
  import { marked } from 'marked';
  import DOMPurify from 'isomorphic-dompurify';

  $: html = DOMPurify.sanitize(marked(userContent));
</script>

{@html html}
```

### ESLint Rule to Block `@html`

```javascript
// .eslintrc.cjs
module.exports = {
  extends: ['plugin:svelte/recommended'],
  rules: {
    'svelte/no-at-html-tags': 'error' // Blocks {@html}
  }
};
```

### Server-Side Input Validation

```typescript
// src/routes/api/comment/+server.ts
import { z } from 'zod';
import type { RequestHandler } from './$types';

const commentSchema = z.object({
  text: z.string().min(1).max(1000).trim(),
  author: z.string().min(1).max(100).trim()
});

export const POST: RequestHandler = async ({ request }) => {
  const body = await request.json();

  const validated = commentSchema.safeParse(body);

  if (!validated.success) {
    return new Response(
      JSON.stringify({ error: 'Invalid input', details: validated.error }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    );
  }

  const { text, author } = validated.data;

  // Save to database (already sanitized)
  await db.comment.create({ data: { text, author } });

  return new Response(
    JSON.stringify({ success: true }),
    { status: 200, headers: { 'Content-Type': 'application/json' } }
  );
};
```

---

## CSRF Protection

### Built-in Protection (Enabled by Default)

SvelteKit automatically validates the `Origin` header for POST, PUT, PATCH, and DELETE requests.

```typescript
// svelte.config.js
const config = {
  kit: {
    csrf: {
      checkOrigin: true // Default: true
    }
  }
};
```

**⚠️ Important**: CSRF checks only apply in production, not development!

### Custom CSRF Token Implementation

```typescript
// src/lib/server/csrf.ts
import { randomBytes } from 'crypto';

export function generateCsrfToken(): string {
  return randomBytes(32).toString('hex');
}

export function validateCsrfToken(token: string, sessionToken: string): boolean {
  if (!token || !sessionToken) return false;

  // Constant-time comparison to prevent timing attacks
  let result = token.length === sessionToken.length ? 0 : 1;
  for (let i = 0; i < token.length && i < sessionToken.length; i++) {
    result |= token.charCodeAt(i) ^ sessionToken.charCodeAt(i);
  }
  return result === 0;
}
```

```typescript
// src/hooks.server.ts
import { generateCsrfToken, validateCsrfToken } from '$lib/server/csrf';
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  const method = event.request.method;

  // Generate token for GET requests
  if (method === 'GET') {
    const csrfToken = generateCsrfToken();
    event.locals.csrfToken = csrfToken;
    event.cookies.set('csrf_token', csrfToken, {
      httpOnly: true,
      sameSite: 'strict',
      secure: true,
      path: '/'
    });
  }

  // Validate token for state-changing requests
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(method)) {
    const csrfToken = event.request.headers.get('X-CSRF-Token');
    const cookieToken = event.cookies.get('csrf_token');

    if (!csrfToken || !validateCsrfToken(csrfToken, cookieToken || '')) {
      return new Response('CSRF validation failed', { status: 403 });
    }
  }

  return await resolve(event);
};
```

### Using CSRF Tokens in Forms

```svelte
<!-- +page.svelte -->
<script lang="ts">
  export let data;

  async function handleSubmit(event: Event) {
    event.preventDefault();
    const formData = new FormData(event.target as HTMLFormElement);

    await fetch('/api/submit', {
      method: 'POST',
      headers: {
        'X-CSRF-Token': data.csrfToken
      },
      body: formData
    });
  }
</script>

<form on:submit={handleSubmit}>
  <input type="hidden" name="csrf_token" value={data.csrfToken} />
  <!-- form fields -->
  <button type="submit">Submit</button>
</form>
```

---

## Authentication Patterns

### HttpOnly Cookies (Recommended)

**Benefits**:
- Cannot be accessed by JavaScript (XSS protection)
- Automatically sent with requests
- Secure with proper flags

```typescript
// src/routes/auth/login/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import bcrypt from 'bcryptjs';

export const actions: Actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData();
    const email = data.get('email')?.toString();
    const password = data.get('password')?.toString();

    if (!email || !password) {
      return fail(400, { error: 'Missing credentials' });
    }

    // Authenticate user
    const user = await db.user.findUnique({ where: { email } });

    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return fail(401, { error: 'Invalid credentials' });
    }

    // Create session
    const sessionToken = await createSession(user.id);

    // Set HttpOnly cookie
    cookies.set('session', sessionToken, {
      httpOnly: true,          // Cannot be accessed by JavaScript
      secure: true,            // Only sent over HTTPS
      sameSite: 'strict',      // CSRF protection
      path: '/',
      maxAge: 60 * 60 * 24 * 7 // 7 days
    });

    throw redirect(303, '/dashboard');
  }
};
```

### Authentication Hook

```typescript
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';
import { verifySession } from '$lib/server/auth';

export const handle: Handle = async ({ event, resolve }) => {
  const sessionToken = event.cookies.get('session');

  if (sessionToken) {
    const user = await verifySession(sessionToken);
    if (user) {
      event.locals.user = user;
    } else {
      // Invalid session, clear cookie
      event.cookies.delete('session');
    }
  }

  return await resolve(event);
};
```

### Protected Routes

```typescript
// src/routes/dashboard/+page.server.ts
import { redirect } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals }) => {
  if (!locals.user) {
    throw redirect(303, '/login');
  }

  return {
    user: locals.user
  };
};
```

### JWT Pattern (Less Recommended for Frontend)

```typescript
// src/lib/server/jwt.ts
import jwt from 'jsonwebtoken';
import { env } from '$env/dynamic/private';

const JWT_SECRET = env.JWT_SECRET;

export function createJWT(userId: string): string {
  return jwt.sign(
    { userId },
    JWT_SECRET,
    { expiresIn: '15m' }
  );
}

export function verifyJWT(token: string): { userId: string } | null {
  try {
    return jwt.verify(token, JWT_SECRET) as { userId: string };
  } catch {
    return null;
  }
}
```

**⚠️ Warning**: Store JWTs in HttpOnly cookies, never in localStorage!

---

## Input Validation

### Zod Schema Validation

```typescript
import { z } from 'zod';

// User registration schema
const registerSchema = z.object({
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(50, 'Username must be at most 50 characters')
    .regex(/^[a-zA-Z0-9_-]+$/, 'Username can only contain letters, numbers, hyphens, and underscores'),

  email: z.string()
    .email('Invalid email address')
    .toLowerCase()
    .trim(),

  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character'),

  confirmPassword: z.string()
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword']
});

// Usage in form action
export const actions: Actions = {
  register: async ({ request }) => {
    const formData = await request.formData();
    const rawData = Object.fromEntries(formData);

    const result = registerSchema.safeParse(rawData);

    if (!result.success) {
      return fail(400, {
        errors: result.error.flatten().fieldErrors
      });
    }

    const { username, email, password } = result.data;

    // Create user
    await createUser({ username, email, password });

    throw redirect(303, '/login');
  }
};
```

### File Upload Validation

```typescript
// src/routes/api/upload/+server.ts
import type { RequestHandler } from './$types';
import path from 'path';

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'image/avif'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

// File signature validation (magic bytes)
const FILE_SIGNATURES = {
  'image/jpeg': [0xFF, 0xD8, 0xFF],
  'image/png': [0x89, 0x50, 0x4E, 0x47],
  'image/webp': [0x52, 0x49, 0x46, 0x46]
};

export const POST: RequestHandler = async ({ request }) => {
  const formData = await request.formData();
  const file = formData.get('file') as File;

  if (!file) {
    return new Response('No file uploaded', { status: 400 });
  }

  // Validate MIME type
  if (!ALLOWED_TYPES.includes(file.type)) {
    return new Response('Invalid file type', { status: 400 });
  }

  // Validate file size
  if (file.size > MAX_SIZE) {
    return new Response('File too large', { status: 400 });
  }

  // Validate filename (prevent path traversal)
  const filename = path.basename(file.name);
  const sanitized = filename.replace(/[^a-zA-Z0-9.-]/g, '_');

  // Read file buffer
  const buffer = await file.arrayBuffer();
  const arr = new Uint8Array(buffer);

  // Verify file signature (magic bytes)
  const signature = FILE_SIGNATURES[file.type as keyof typeof FILE_SIGNATURES];
  if (signature) {
    const isValid = signature.every((byte, i) => arr[i] === byte);
    if (!isValid) {
      return new Response('Invalid file', { status: 400 });
    }
  }

  // Process file (save to storage, etc.)
  await saveFile(sanitized, buffer);

  return new Response(
    JSON.stringify({ success: true, filename: sanitized }),
    {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    }
  );
};
```

---

## Session Management with Redis

### Using svelte-kit-sessions with Upstash Redis

```bash
npm install svelte-kit-sessions svelte-kit-connect-upstash-redis @upstash/redis
```

```typescript
// src/hooks.server.ts
import { sequence } from '@sveltejs/kit/hooks';
import { sessions } from 'svelte-kit-sessions';
import UpstashRedisStore from 'svelte-kit-connect-upstash-redis';
import { Redis } from '@upstash/redis';
import { env } from '$env/dynamic/private';

const client = new Redis({
  url: env.UPSTASH_REDIS_URL,
  token: env.UPSTASH_REDIS_TOKEN
});

const sessionHandler = sessions({
  secret: env.SESSION_SECRET, // At least 32 characters
  store: new UpstashRedisStore({ client }),
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7 // 7 days
  },
  rolling: true, // Extend session on activity
  saveUninitialized: false // Don't create session until something stored
});

export const handle = sequence(sessionHandler, async ({ event, resolve }) => {
  // Session available at event.locals.session
  return resolve(event);
});
```

**Benefits:**
- Server-side session storage (secure)
- Automatic session rotation
- Redis provides fast, distributed storage
- Sessions survive server restarts
- Easy horizontal scaling

---

## Cryptographic Patterns

### Password Hashing with bcrypt

```bash
npm install bcryptjs
npm install -D @types/bcryptjs
```

```typescript
// src/lib/server/auth.ts
import bcrypt from 'bcryptjs';

/**
 * Hash password with bcrypt
 * Uses 12 rounds (recommended for 2024)
 */
export async function hashPassword(password: string): Promise<string> {
  const salt = await bcrypt.genSalt(12);
  return bcrypt.hash(password, salt);
}

/**
 * Verify password against hash
 * Constant-time comparison
 */
export async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

**Usage in registration:**

```typescript
// src/routes/auth/register/+page.server.ts
import { hashPassword } from '$lib/server/auth';
import type { Actions } from './$types';

export const actions: Actions = {
  default: async ({ request, locals }) => {
    const data = await request.formData();
    const password = data.get('password')?.toString();

    if (!password || password.length < 12) {
      return fail(400, { error: 'Password must be at least 12 characters' });
    }

    // Hash password before storing
    const passwordHash = await hashPassword(password);

    // Store user with hashed password
    await db.user.create({
      data: {
        email: data.get('email')?.toString(),
        passwordHash
      }
    });

    return { success: true };
  }
};
```

### Encryption for Sensitive Data

```typescript
// src/lib/server/crypto.ts
import crypto from 'crypto';
import { env } from '$env/dynamic/private';

const algorithm = 'aes-256-cbc';
const key = Buffer.from(env.ENCRYPTION_KEY, 'hex'); // 32 bytes hex (64 chars)

/**
 * Encrypt sensitive data
 * Returns: iv:encrypted_data
 */
export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(algorithm, key, iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  return iv.toString('hex') + ':' + encrypted;
}

/**
 * Decrypt sensitive data
 */
export function decrypt(text: string): string {
  const parts = text.split(':');
  const iv = Buffer.from(parts.shift()!, 'hex');
  const encrypted = parts.join(':');

  const decipher = crypto.createDecipheriv(algorithm, key, iv);

  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}
```

**Generate encryption key:**

```bash
# Generate 32-byte key for AES-256
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**Usage example:**

```typescript
// Storing encrypted PII
const encryptedSSN = encrypt(userSSN);
await db.user.update({
  where: { id: userId },
  data: { encryptedSSN }
});

// Retrieving
const user = await db.user.findUnique({ where: { id: userId } });
const ssn = decrypt(user.encryptedSSN);
```

---

## NoSQL Injection Prevention

### Prisma Query Injection

**Vulnerable Code:**

```typescript
// ❌ DANGEROUS: User input directly in query
export const load: PageServerLoad = async ({ url }) => {
  const userInput = Object.fromEntries(url.searchParams);

  // If userInput contains operators like {$gt, $lt}, can manipulate query
  const users = await prisma.user.findMany({
    where: userInput // VULNERABLE!
  });

  return { users };
};
```

**Secure Code with Zod Validation:**

```typescript
// ✅ SAFE: Validate and sanitize input
import { z } from 'zod';

const searchSchema = z.object({
  email: z.string().email().optional(),
  name: z.string().max(100).optional(),
  role: z.enum(['admin', 'user', 'moderator']).optional()
});

export const load: PageServerLoad = async ({ url }) => {
  const rawInput = Object.fromEntries(url.searchParams);

  // Validate - throws if invalid
  const validated = searchSchema.parse(rawInput);

  // Now safe to use in query
  const users = await prisma.user.findMany({
    where: validated
  });

  return { users };
};
```

### MongoDB-Specific Injection

If using MongoDB directly (not Prisma):

```typescript
// ❌ DANGEROUS
const username = req.body.username; // {$ne: null}
const user = await db.collection('users').findOne({ username });

// ✅ SAFE: Ensure strings
const username = String(req.body.username);
const user = await db.collection('users').findOne({ username });

// ✅ BETTER: Validate with Zod
const userSchema = z.object({
  username: z.string().min(3).max(50)
});

const { username } = userSchema.parse(req.body);
const user = await db.collection('users').findOne({ username });
```

---

## Client-Side Storage Security

### LocalStorage vs Cookies

| Storage | Security | Use Case |
|---------|----------|----------|
| **LocalStorage** | ❌ Vulnerable to XSS | Non-sensitive UI state only |
| **SessionStorage** | ❌ Vulnerable to XSS | Temporary, non-sensitive data |
| **HttpOnly Cookies** | ✅ Secure (not accessible via JS) | Authentication tokens, sessions |
| **Regular Cookies** | ⚠️ Accessible via JS | Non-sensitive data |

### LocalStorage Security Issues

**Problems:**
1. Accessible by any JavaScript (XSS vulnerability)
2. No automatic expiration
3. Data persists until explicitly deleted
4. No protection against script injection
5. Not sent with HTTP requests

**❌ NEVER Store in LocalStorage:**
- Authentication tokens
- Session IDs
- API keys
- Personal Identifiable Information (PII)
- Credit card data
- Passwords

**✅ Acceptable for LocalStorage:**
- User preferences (theme, language)
- Non-sensitive UI state
- Draft content (auto-save)
- Analytics data

### Secure LocalStorage Usage

```typescript
// src/lib/stores/theme.ts
import { browser } from '$app/environment';
import { writable } from 'svelte/store';
import { z } from 'zod';

const themeSchema = z.enum(['light', 'dark', 'auto']);
type Theme = z.infer<typeof themeSchema>;

function createThemeStore() {
  // Load from localStorage with validation
  const getInitialTheme = (): Theme => {
    if (!browser) return 'light';

    try {
      const stored = localStorage.getItem('theme');
      const result = themeSchema.safeParse(stored);
      return result.success ? result.data : 'light';
    } catch {
      return 'light';
    }
  };

  const { subscribe, set } = writable<Theme>(getInitialTheme());

  return {
    subscribe,
    set: (value: Theme) => {
      // Validate before storing
      const validated = themeSchema.parse(value);

      if (browser) {
        localStorage.setItem('theme', validated);
      }

      set(validated);
    }
  };
}

export const theme = createThemeStore();
```

### Authentication Token Storage (Correct Approach)

```typescript
// ❌ INSECURE: LocalStorage
localStorage.setItem('authToken', token);

// ✅ SECURE: HttpOnly cookies (server-side only)
// src/routes/auth/login/+page.server.ts
export const actions: Actions = {
  default: async ({ request, cookies }) => {
    // ... authenticate user ...

    const token = generateToken(user.id);

    // Set HttpOnly cookie
    cookies.set('auth_token', token, {
      httpOnly: true,      // Cannot be accessed by JavaScript
      secure: true,        // Only sent over HTTPS
      sameSite: 'strict',  // CSRF protection
      path: '/',
      maxAge: 60 * 60 * 24 * 7 // 7 days
    });

    throw redirect(303, '/dashboard');
  }
};
```

---

## Production Anti-Patterns

### 1. Hydration Errors (SSR/Client Mismatch)

**❌ BAD: Different values on server vs client**

```svelte
<script>
  // Date will differ between server and client!
  const now = new Date();
</script>

<p>Current time: {now.toLocaleString()}</p>
```

**✅ GOOD: Use onMount for client-only values**

```svelte
<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';

  let now = $state<Date | null>(null);

  onMount(() => {
    now = new Date();

    // Update every second
    const interval = setInterval(() => {
      now = new Date();
    }, 1000);

    return () => clearInterval(interval);
  });
</script>

{#if now}
  <p>Current time: {now.toLocaleString()}</p>
{:else}
  <p>Loading time...</p>
{/if}
```

### 2. Trusting Client-Side Authorization

**❌ BAD: Client-only checks**

```svelte
<!-- src/routes/admin/+page.svelte -->
<script>
  import { user } from '$lib/stores/user';
</script>

{#if $user.role === 'admin'}
  <button on:click={deleteAllUsers}>Delete All Users</button>
{/if}
```

**Problem:** User can modify store or bypass check via DevTools.

**✅ GOOD: Server-side authorization**

```typescript
// src/routes/admin/+page.server.ts
export const load: PageServerLoad = async ({ locals }) => {
  // Check authentication
  if (!locals.user) {
    throw redirect(303, '/login');
  }

  // Check authorization
  if (locals.user.role !== 'admin') {
    throw error(403, 'Forbidden');
  }

  return {
    users: await db.user.findMany()
  };
};

export const actions: Actions = {
  deleteUser: async ({ request, locals }) => {
    // Re-check authorization in action!
    if (locals.user?.role !== 'admin') {
      throw error(403, 'Forbidden');
    }

    const data = await request.formData();
    const userId = data.get('userId');

    await db.user.delete({ where: { id: Number(userId) } });

    return { success: true };
  }
};
```

### 3. Exposing Secrets in Client Bundle

**❌ BAD: API keys in client code**

```typescript
// Will be visible in browser bundle!
const API_KEY = 'sk_live_abc123';

async function fetchData() {
  const response = await fetch('https://api.example.com/data', {
    headers: { 'Authorization': `Bearer ${API_KEY}` }
  });
  return response.json();
}
```

**✅ GOOD: Proxy through server route**

```typescript
// src/routes/api/data/+server.ts
import { PRIVATE_API_KEY } from '$env/static/private';

export const GET = async ({ fetch }) => {
  const response = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${PRIVATE_API_KEY}`
    }
  });

  return response;
};
```

```svelte
<!-- Client code -->
<script>
  async function fetchData() {
    // Calls your server proxy, not external API
    const response = await fetch('/api/data');
    return response.json();
  }
</script>
```

### 4. Missing Error Handling

**❌ BAD: Unhandled promise rejections**

```typescript
export const load: PageServerLoad = async () => {
  // Will crash if fetch fails!
  const response = await fetch('https://api.example.com/data');
  const data = await response.json();
  return { data };
};
```

**✅ GOOD: Proper error handling**

```typescript
export const load: PageServerLoad = async () => {
  try {
    const response = await fetch('https://api.example.com/data');

    if (!response.ok) {
      throw error(response.status, 'API request failed');
    }

    const data = await response.json();
    return { data };
  } catch (err) {
    console.error('Failed to load data:', err);

    // Return fallback or throw error
    throw error(500, 'Failed to load data');
  }
};
```

---

## Security Headers

### Using hooks.server.ts

```typescript
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);

  // Prevent clickjacking
  response.headers.set('X-Frame-Options', 'DENY');

  // Prevent MIME sniffing
  response.headers.set('X-Content-Type-Options', 'nosniff');

  // Referrer policy
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');

  // Permissions policy (disable unused features)
  response.headers.set(
    'Permissions-Policy',
    'geolocation=(), microphone=(), camera=(), payment=()'
  );

  // HSTS (only in production)
  if (import.meta.env.PROD) {
    response.headers.set(
      'Strict-Transport-Security',
      'max-age=63072000; includeSubDomains; preload'
    );
  }

  return response;
};
```

### Using Third-Party Libraries

**nosecone** (Recommended):

```bash
npm install @nosecone/sveltekit
```

```typescript
// src/hooks.server.ts
import { nosecone, defaults } from '@nosecone/sveltekit';

export const handle = nosecone({
  ...defaults,
  csp: {
    'default-src': ['self'],
    'script-src': ['self', 'strict-dynamic'],
    'style-src': ['self', 'unsafe-inline']
  }
});
```

---

## Dependency Security

### Regular Audits

```bash
# Check for vulnerabilities
npm audit

# Fix automatically (non-breaking)
npm audit fix

# Fix all (may break)
npm audit fix --force

# Generate detailed report
npm audit --json > audit-report.json
```

### CI/CD Integration

```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run npm audit
        run: npm audit --audit-level=moderate

      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

### Automated Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: 'npm'
    directory: '/'
    schedule:
      interval: 'weekly'
    open-pull-requests-limit: 10
    labels:
      - 'dependencies'
      - 'security'
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [Performance Optimization](performance-optimization.md)
- [Deployment Operations](deployment-operations.md)
