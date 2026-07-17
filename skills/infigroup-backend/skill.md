---
name: infigroup-backend
description: InfiGroup backend infrastructure skill. Security, architecture decisions, performance, observability, database patterns for the InfiGroup Next.js stack (withApi pipeline, Drizzle ORM, Supabase, Arcjet, Upstash, Inngest, Resend). Use when building API routes, database schemas, auth flows, background jobs, or making architecture decisions.
argument-hint: "[security | architecture | performance | database | auth | email | observability]"
---

# InfiGroup Backend Infrastructure

## Role

Senior backend engineer who:
- Enforces security at every boundary: validation, auth, rate limiting, CSRF, RLS
- Makes correct architecture decisions: knows when Server Actions vs API Routes vs tRPC
- Prevents N+1s, implements proper caching, uses connection pooling correctly
- Instruments everything: errors, performance, structured logs

---

## Architecture Decision Tree

**What are you building?**

```
Form mutation (create/update/delete) → Server Actions
  └─ With progressive enhancement → Server Actions + useFormState

Public API (webhooks, external integrations, third-party consumers) → REST Route Handlers
  └─ With withApi pipeline (auto CSRF + rate limit + Zod + error handling)

Internal type-safe client-server calls → tRPC
  └─ When: complex query/mutation logic, tight type safety needed across client/server

Data fetching (reads only) → Server Components directly
  └─ No API layer needed — fetch from DB in the component

Real-time subscriptions → Supabase Realtime
Background/async work → Inngest
```

**When NOT to use each:**
- Server Actions: NOT for public APIs, NOT for webhooks (no auth header access)
- tRPC: NOT when external consumers need the API
- REST Routes without `withApi`: NOT for mutation endpoints (missing CSRF/rate limiting)

---

## withApi Pipeline (base_system standard)

Every mutation endpoint uses this. Auto-handles: CSRF → Rate Limit → Auth → Zod Validation → Handler → Error Handling.

```typescript
export const POST = withApi(
  {
    schema: zodSchema,          // Zod v3 schema for request body
    rateLimit: 'contact',       // preset (see below)
    csrf: true,                 // enabled by default for mutations
    auth: false,                // set true to require authenticated user
  },
  async ({ data, request, user }) => {
    return successResponse(result);
  },
);
```

### Rate Limit Presets
| Preset | Limit |
|--------|-------|
| `contact` | 5 req/min |
| `api` | 60 req/min |
| `auth` | 10 req/5min |
| `strict` | 3 req/min |

### API Response Envelope (universal)
```typescript
interface ApiResponse<T = null> {
  success: boolean;
  data: T;
  error: string | null;
  meta?: { total: number; page: number; limit: number };
}
```
Always use `successResponse()` and `errorResponse()` helpers — never raw `Response.json()`.

---

## Security Requirements

### Pre-code Security Checklist
Before writing any endpoint:
- [ ] Is this a mutation? → `withApi` with `csrf: true`
- [ ] Does it accept user input? → Zod schema defined
- [ ] Does it need auth? → `auth: true` in withApi OR `auth()` call in Server Action
- [ ] Does it query the DB? → RLS enabled on the table?
- [ ] Does it touch files/uploads? → File type + size validation
- [ ] Does it call external APIs? → Rate limit + timeout set
- [ ] Does it return sensitive data? → Strip fields before return

### Arcjet (request-level security)
Used in middleware + high-risk routes:

```typescript
import arcjet, { shield, detectBot, tokenBucket } from "@arcjet/next";

const aj = arcjet({
  key: process.env.ARCJET_KEY!,
  rules: [
    shield({ mode: "LIVE" }),           // block common attacks (SQLi, XSS, CSRF)
    detectBot({ mode: "LIVE",           // block scrapers/bots
      allow: ["CATEGORY:SEARCH_ENGINE"] }),
    tokenBucket({ mode: "LIVE",         // rate limiting
      refillRate: 10, interval: 10, capacity: 100 }),
  ],
});
```

### Supabase RLS (mandatory)
Every table exposed to client MUST have RLS enabled. No exceptions.

```sql
-- Enable RLS
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;

-- User can only see their own rows
CREATE POLICY "user_owns_row" ON your_table
  FOR ALL USING (auth.uid() = user_id);

-- Service role bypasses RLS (use in server-only contexts)
```

Rules:
- Tables without RLS are publicly accessible — treat as a security incident
- Always verify auth inside Server Actions AND Route Handlers (not just middleware)
- Never trust `user_id` from client — always read from `auth.uid()` server-side

### Input Validation (Zod)
```typescript
const schema = z.object({
  email: z.string().email().max(255),
  amount: z.number().positive().max(1_000_000),
  slug: z.string().regex(/^[a-z0-9-]+$/),
});

// Always: strip unknown fields
schema.strip().parse(input);
```

### Secret Management
- NEVER hardcode: API keys, connection strings, tokens
- ALWAYS: `process.env.KEY` with Zod validation in `src/lib/env.ts`
- Env validated at build time via `@t3-oss/env-nextjs`

---

## Database Patterns (Drizzle ORM)

### BaseRepository Pattern
```typescript
class UserRepository extends BaseRepository<typeof users> {
  // findAll(), create(), delete() inherited
  
  async findByEmail(email: string) {
    return db.query.users.findFirst({
      where: eq(users.email, email),
    });
  }
}
```

### Schema Conventions
```typescript
export const users = pgTable("users", {
  id: integer("id").generatedAlwaysAsIdentity().primaryKey(), // identity over serial
  email: varchar("email", { length: 255 }).notNull().unique(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

### Preventing N+1 (relational queries)
```typescript
// WRONG: N+1
const posts = await db.select().from(posts);
for (const post of posts) {
  post.author = await db.select().from(users).where(eq(users.id, post.authorId));
}

// CORRECT: relational query
const posts = await db.query.posts.findMany({
  with: { author: true },
});
```

### Connection Pooling
- Use Supabase pgBouncer (transaction mode)
- Disable prepared statements: `?pgbouncer=true` in connection string
- Never open a new connection per request in serverless

### Migration Safety
1. Never drop columns in production without a deprecation cycle
2. Add columns with defaults before removing old ones
3. Always test migrations on staging with prod-sized data first

---

## Caching Strategy

```
Cache Layer          | Tool                  | When
---------------------|----------------------|---------------------------
Request memoization  | React cache()        | Same request, multiple calls
Page/data cache      | Next.js use cache    | Cross-request, revalidate on mutation
External data        | Upstash Redis        | Rate limiting, sessions, expensive ops
```

### use cache Pattern
```typescript
import { unstable_cache as cache } from 'next/cache';

const getProduct = cache(
  async (id: string) => db.query.products.findFirst({ where: eq(products.id, id) }),
  ['product'],
  { tags: ['products'], revalidate: 3600 }
);

// In Server Action (after mutation):
revalidateTag('products');
```

**Rule**: Every mutation MUST pair with cache invalidation. No exceptions.

### Upstash Redis (rate limiting + session)
```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "10 s"),
});

const { success } = await ratelimit.limit(identifier);
if (!success) return errorResponse("Rate limit exceeded", 429);
```

---

## Authentication Patterns

### Supabase Auth (primary pattern)
```typescript
// Server Component / Server Action
import { createClient } from '@/lib/supabase/server';

const supabase = createClient();
const { data: { user } } = await supabase.auth.getUser();
if (!user) redirect('/login');
```

- Verify in every Server Action + Route Handler — middleware alone is NOT sufficient
- Use `getUser()` not `getSession()` (getSession doesn't validate against server)
- RLS policies enforced automatically via Supabase client

### OAuth (Google)
```typescript
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: `${origin}/auth/callback` },
});
```

---

## Background Jobs (Inngest)

```typescript
import { inngest } from "@/lib/inngest";

// Define function
export const sendWelcomeEmail = inngest.createFunction(
  { id: "send-welcome-email" },
  { event: "user/created" },
  async ({ event, step }) => {
    await step.run("send-email", async () => {
      await resend.emails.send({ ... });
    });
  }
);

// Trigger from Server Action
await inngest.send({ name: "user/created", data: { userId } });
```

Use for: email sends, webhooks processing, file processing, scheduled tasks.
Never: block Server Actions/API routes with slow work — always defer to Inngest.

---

## Email Architecture (Resend)

| Type | Pattern |
|------|---------|
| User-triggered (contact form) | Server Action → Inngest → Resend |
| System-triggered (Stripe webhook) | Route Handler → Inngest → Resend |
| Templates | React Email components (XSS-safe by default) |

```typescript
import { Resend } from 'resend';
import { WelcomeEmail } from '@/emails/welcome';

const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'InfiGroup <noreply@infigroup.co>',
  to: user.email,
  subject: 'Welcome',
  react: WelcomeEmail({ name: user.name }),
});
```

---

## Observability Stack

| Layer | Tool | When to use |
|-------|------|-------------|
| Error tracking | Sentry | All unhandled errors, performance traces |
| App logging | Pino (structured JSON) | Server-side events, audit logs |
| Infrastructure | Vercel Logs | Zero-config deployment logs |

### Sentry Setup
```typescript
// Capture with context
Sentry.withScope((scope) => {
  scope.setUser({ id: user.id });
  scope.setExtra('requestId', requestId);
  Sentry.captureException(error);
});
```

### Pino Logging
```typescript
import pino from 'pino';
const logger = pino({ level: process.env.LOG_LEVEL || 'info' });

logger.info({ userId, action: 'checkout.completed', orderId }, 'Order created');
logger.error({ err, userId, action: 'payment.failed' }, 'Payment error');
```

Log at: request start (info), mutation success (info), errors (error), slow queries >500ms (warn).

---

## Performance Checklist

Before marking backend work done:
- [ ] No N+1 queries (use relational queries or batch)
- [ ] Connection pooling configured (pgBouncer, no per-request connections)
- [ ] Mutations paired with cache invalidation
- [ ] Slow/external operations deferred to Inngest
- [ ] Rate limiting on all public endpoints
- [ ] DB queries have indexes on filtered/sorted columns
- [ ] No synchronous file I/O in request path
- [ ] Zod parsing uses `.strip()` (drops unknown fields)
- [ ] Response doesn't include unnecessary data (select specific columns)

---

## 2026 Production Stack Reference

| Layer | Tool |
|-------|------|
| Framework | Next.js 16 (App Router) |
| ORM | Drizzle ORM |
| Database | PostgreSQL via Supabase |
| Auth | Supabase Auth |
| Internal API | tRPC |
| External API | REST Route Handlers (`withApi`) |
| Background jobs | Inngest |
| Caching | `use cache` + Upstash Redis |
| Rate limiting | Upstash @upstash/ratelimit + Arcjet |
| File uploads | Vercel Blob |
| Email | Resend + React Email |
| Realtime | Supabase Realtime |
| Monitoring | Sentry + Pino |
| Deployment | Vercel + Turborepo |
| Validation | Zod (Standard Schema v1) |
| Security layer | Arcjet (shield + bot detection + rate limiting) |
