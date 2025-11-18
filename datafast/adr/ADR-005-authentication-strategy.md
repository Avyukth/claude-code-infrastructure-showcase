# ADR-005: Authentication Strategy

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Engineering Team, Security Lead
**Technical Story:** [FR-4 - Quick Setup Flow, FR-10 - Event Limits & Billing, NFR-4 - Security]

## Context and Problem Statement

DataFa.st needs a secure, user-friendly authentication system that:
- Supports **email/password** and **OAuth** (Google, X/Twitter)
- Enables **quick sign-up** (<1 minute, no friction)
- Integrates with **Stripe** (customer portal for billing)
- Scales to **10K+ users** without performance issues
- Complies with **GDPR** (user data deletion, export)
- Prevents **account takeover** (brute force, credential stuffing)

The choice affects user onboarding conversion, security posture, and operational complexity.

## Decision Drivers

- **Onboarding Speed:** <1 minute from landing page to dashboard
- **Security:** Prevent account takeover, protect user data
- **User Experience:** Minimize friction (no CAPTCHA unless necessary)
- **OAuth Support:** Google and X/Twitter (indie maker audience)
- **Billing Integration:** Link to Stripe Customer Portal
- **Scalability:** 10K+ concurrent logins
- **GDPR Compliance:** User deletion, data export
- **Developer Experience:** Easy to implement, maintain

## Considered Options

1. **NextAuth.js** (OAuth + credentials for Next.js)
2. **Clerk** (Managed auth service)
3. **Supabase Auth** (PostgreSQL + JWT)
4. **Custom Auth** (Roll-your-own JWT + bcrypt)

## Decision Outcome

**Chosen option:** "NextAuth.js (with Database Sessions)"

**Rationale:**
NextAuth.js is the de-facto standard for Next.js auth, offering:
- **OAuth providers:** Google, X/Twitter built-in
- **Credentials provider:** Email/password with bcrypt
- **Database sessions:** Secure, revocable (vs. JWT-only)
- **Stripe integration:** Easy to link auth to Stripe customer ID
- **Free and open-source:** No per-user pricing
- **Active community:** 15K+ GitHub stars, frequent updates

For an indie/bootstrapped project, avoiding $0.02/MAU pricing (Clerk) is significant.

### Positive Consequences

- **Free:** No per-user costs (vs. Clerk: $0.02/MAU = $120/mo at 6K users)
- **Full control:** No vendor lock-in, can customize flows
- **Stripe-friendly:** Easy to link user → Stripe customer
- **Database sessions:** Revoke sessions server-side (vs. JWT where you can't)
- **Familiar:** Team knows Next.js, shallow learning curve
- **GDPR-ready:** Delete user → CASCADE deletes sessions

### Negative Consequences

- **More code:** Must implement password reset, email verification manually
- **Email sending:** Requires email service (Resend, SendGrid)
- **Security responsibility:** Team owns bcrypt config, rate limiting
- **UI complexity:** Must build login/signup forms (vs. Clerk's pre-built)
- **No MFA built-in:** Must add TOTP manually (Phase 2)

## Pros and Cons of the Options

### NextAuth.js

**Pros:**
- **Free and open-source:** No recurring costs
- **OAuth built-in:** Google, X/Twitter, GitHub, etc.
- **Database sessions:** Secure, revocable (store in PostgreSQL)
- **Stripe integration:** Custom callbacks to link user → customer_id
- **Flexible:** Customize login/signup flows
- **Active community:** 15K+ stars, well-maintained

**Cons:**
- More setup than managed services (Clerk)
- Must handle email sending (verification, password reset)
- No built-in MFA (TOTP, SMS)
- UI not included (must build forms)
- Security is team's responsibility (bcrypt, rate limiting)

---

### Clerk

**Pros:**
- **Managed service:** No infrastructure to maintain
- **Pre-built UI:** Beautiful login/signup components
- **MFA built-in:** TOTP, SMS out of the box
- **User management:** Admin dashboard for support
- **Compliance:** GDPR, SOC 2 handled by Clerk

**Cons:**
- **Expensive:** $0.02/MAU (= $120/mo at 6K users, $300/mo at 15K)
- **Vendor lock-in:** Hard to migrate away
- **Limited customization:** UI/UX controlled by Clerk
- **Stripe integration:** Requires webhooks (more complex)
- **Overkill for MVP:** Most features unused

**Verdict:** Great for well-funded startups, too expensive for bootstrapped indie.

---

### Supabase Auth

**Pros:**
- **Free tier:** 50K MAU (generous)
- **PostgreSQL-based:** Integrates with existing DB
- **OAuth + email:** Google, GitHub, magic links
- **Row-level security:** DB-level auth policies
- **Real-time:** WebSocket auth for live features

**Cons:**
- **Supabase lock-in:** Auth tied to Supabase platform
- **Learning curve:** RLS policies are complex
- **X/Twitter OAuth:** Not built-in (must configure manually)
- **Stripe integration:** Requires custom logic
- **Migration risk:** If leaving Supabase, must rewrite auth

**Verdict:** Good if using Supabase for DB, but we're using ClickHouse + PostgreSQL separately.

---

### Custom Auth (JWT + bcrypt)

**Pros:**
- **Full control:** No dependencies, total customization
- **Free:** No third-party costs
- **Lightweight:** Minimal code footprint

**Cons:**
- **Time-consuming:** Weeks to build securely (password reset, email verification, rate limiting)
- **Security risk:** Easy to introduce vulnerabilities (weak bcrypt rounds, JWT secret leaks)
- **Maintenance burden:** Team must handle all edge cases
- **No OAuth:** Must implement Google/X OAuth flows manually (complex)
- **Reinventing the wheel:** NextAuth solves this already

**Verdict:** Not worth the risk/time for a bootstrapped team.

---

## Technical Implementation

### Authentication Flow

**Sign-Up:**
```
1. User enters email/password OR clicks "Sign in with Google"
2. NextAuth creates user in PostgreSQL (users table)
3. Send verification email (Resend or SendGrid)
4. User clicks link → Email verified
5. Redirect to dashboard (14-day trial starts)
```

**Login:**
```
1. User enters credentials OR uses OAuth
2. NextAuth validates:
   - Email/password: bcrypt compare
   - OAuth: Token exchange with provider
3. Create session in DB (sessions table)
4. Set HTTP-only cookie with session token
5. Redirect to dashboard
```

**Logout:**
```
1. User clicks "Logout"
2. NextAuth deletes session from DB
3. Clear cookie
4. Redirect to homepage
```

---

### Database Schema (PostgreSQL)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    email_verified TIMESTAMP,
    password_hash VARCHAR(255), -- bcrypt, NULL if OAuth-only
    name VARCHAR(255),
    image VARCHAR(500), -- Avatar URL from OAuth
    oauth_provider VARCHAR(50), -- 'google', 'twitter', NULL if email/pass
    oauth_provider_id VARCHAR(255), -- Provider's user ID
    stripe_customer_id VARCHAR(255), -- Link to Stripe
    subscription_plan VARCHAR(50) DEFAULT 'trial', -- 'trial', 'starter', 'growth'
    subscription_status VARCHAR(50) DEFAULT 'trialing', -- 'trialing', 'active', 'canceled'
    trial_ends_at TIMESTAMP DEFAULT NOW() + INTERVAL '14 days',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    session_token VARCHAR(255) UNIQUE NOT NULL,
    expires TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sessions_token ON sessions(session_token);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe ON users(stripe_customer_id);
```

---

### NextAuth Configuration

**`/app/api/auth/[...nextauth]/route.ts`:**
```typescript
import NextAuth from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import TwitterProvider from "next-auth/providers/twitter";
import CredentialsProvider from "next-auth/providers/credentials";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import { prisma } from "@/lib/db";
import bcrypt from "bcrypt";

export const authOptions = {
    adapter: PrismaAdapter(prisma),
    providers: [
        GoogleProvider({
            clientId: process.env.GOOGLE_CLIENT_ID!,
            clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
        }),
        TwitterProvider({
            clientId: process.env.TWITTER_CLIENT_ID!,
            clientSecret: process.env.TWITTER_CLIENT_SECRET!,
            version: "2.0", // Use OAuth 2.0
        }),
        CredentialsProvider({
            name: "Email",
            credentials: {
                email: { label: "Email", type: "email" },
                password: { label: "Password", type: "password" },
            },
            async authorize(credentials) {
                const user = await prisma.user.findUnique({
                    where: { email: credentials.email },
                });
                if (!user || !user.password_hash) return null;

                const valid = await bcrypt.compare(
                    credentials.password,
                    user.password_hash
                );
                if (!valid) return null;

                return { id: user.id, email: user.email, name: user.name };
            },
        }),
    ],
    callbacks: {
        async session({ session, user }) {
            // Add user ID and Stripe customer ID to session
            session.user.id = user.id;
            session.user.stripeCustomerId = user.stripe_customer_id;
            session.user.plan = user.subscription_plan;
            return session;
        },
        async jwt({ token, user }) {
            if (user) {
                token.id = user.id;
            }
            return token;
        },
    },
    pages: {
        signIn: "/login",
        signOut: "/",
        error: "/login", // Error code passed in query string
        verifyRequest: "/verify-email",
    },
    session: {
        strategy: "database", // Use DB sessions (not JWT)
        maxAge: 30 * 24 * 60 * 60, // 30 days
    },
};

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

---

### Security Measures

**1. Password Hashing:**
- **Algorithm:** bcrypt (industry standard)
- **Cost factor:** 12 rounds (balance security vs. performance)
- **Salt:** Automatic per-password (built into bcrypt)

```typescript
import bcrypt from "bcrypt";

// Sign-up: Hash password
const passwordHash = await bcrypt.hash(password, 12);

// Login: Verify password
const valid = await bcrypt.compare(password, user.password_hash);
```

**2. Rate Limiting:**
- **Tool:** Upstash Rate Limit (Redis-based, Vercel-friendly)
- **Limits:**
  - Login: 5 attempts per 15 minutes per IP
  - Sign-up: 3 accounts per hour per IP
  - Password reset: 3 requests per hour per email

```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
    redis: Redis.fromEnv(),
    limiter: Ratelimit.slidingWindow(5, "15 m"),
});

export async function POST(req: Request) {
    const ip = req.headers.get("x-forwarded-for") ?? "127.0.0.1";
    const { success } = await ratelimit.limit(ip);

    if (!success) {
        return new Response("Too many requests", { status: 429 });
    }

    // Proceed with login
}
```

**3. CAPTCHA (Triggered):**
- **Tool:** Cloudflare Turnstile (privacy-friendly, free)
- **Trigger:** Only after 3 failed login attempts (reduce friction)

**4. Email Verification:**
- Send verification link on sign-up
- Users can view dashboard but see banner: "Verify email to prevent data loss"
- After 7 days unverified → Account paused (prevent spam)

**5. Session Security:**
- **HTTP-only cookies:** Prevent XSS attacks
- **SameSite=Lax:** CSRF protection
- **Secure flag:** HTTPS-only (production)
- **Session expiry:** 30 days (revoke on logout)

---

### Stripe Integration

**Link User to Stripe Customer:**
```typescript
// When user upgrades to paid plan
const customer = await stripe.customers.create({
    email: user.email,
    metadata: { userId: user.id },
});

await prisma.user.update({
    where: { id: user.id },
    data: { stripe_customer_id: customer.id },
});
```

**Stripe Customer Portal:**
```typescript
// User clicks "Manage Billing"
const session = await stripe.billingPortal.sessions.create({
    customer: user.stripe_customer_id,
    return_url: `${process.env.NEXT_PUBLIC_URL}/settings/billing`,
});

redirect(session.url);
```

---

## GDPR Compliance

**User Deletion (Right to Erasure):**
```typescript
// Delete user and all associated data
await prisma.$transaction([
    prisma.session.deleteMany({ where: { userId: user.id } }),
    prisma.website.deleteMany({ where: { userId: user.id } }),
    // Delete from ClickHouse (analytics events)
    clickhouse.query(`DELETE FROM events WHERE website_id IN (
        SELECT website_id FROM websites WHERE user_id = '${user.id}'
    )`),
    prisma.user.delete({ where: { id: user.id } }),
]);
```

**Data Export (Right to Access):**
```typescript
// Export user data as JSON
const data = {
    user: await prisma.user.findUnique({ where: { id: user.id } }),
    websites: await prisma.website.findMany({ where: { userId: user.id } }),
    sessions: await prisma.session.findMany({ where: { userId: user.id } }),
    // Events from ClickHouse (async export to CSV)
};
```

---

## Future Enhancements (Phase 2)

- [ ] **Multi-Factor Authentication (MFA):** TOTP via Authenticator apps
- [ ] **Magic Links:** Passwordless email login
- [ ] **SSO (Single Sign-On):** SAML for enterprise customers
- [ ] **Team Accounts:** Invite members, role-based access control (RBAC)
- [ ] **Account Recovery:** Security questions or backup codes

---

## Links

- [NextAuth.js Documentation](https://next-auth.js.org/)
- [Upstash Rate Limit](https://upstash.com/docs/redis/features/ratelimiting)
- [Cloudflare Turnstile](https://www.cloudflare.com/products/turnstile/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [PRD Section 4: NFR-4 Security](../prd/DATAFAST-PRD.md#nfr-4-security)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]
