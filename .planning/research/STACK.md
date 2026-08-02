# Technology Stack

**Project:** Plataforma de Adoção de Animais  
**Researched:** 2026-08-02  
**Confidence:** MEDIUM — package versions were checked in the npm registry and deployment/product constraints were checked against current first-party documentation. Free-tier quotas and terms can change, so recheck them at deployment time.

## Recommendation

Build a TypeScript monorepo with a separate **Next.js public web app** and a small **Express REST API**. Deploy the web app to Vercel, the API to Render, use Neon Postgres through Prisma, and store every animal photo in Cloudinary. This is the best MVP trade-off: it meets the explicit REST/JWT requirement without turning the frontend into a backend framework project, keeps the database durable on a genuinely non-expiring free plan, and leaves clean boundaries for later filters, messaging, admin, and image work.

Use **PostgreSQL, not MongoDB**. Ownership, uniqueness, availability status, future filtering, and eventual moderation/reporting are relational concerns. Prisma gives fast TypeScript development now while preserving SQL/Postgres portability later.

## Recommended Stack

### Core Framework

| Technology | Version | Purpose | Why |
|---|---:|---|---|
| Node.js | 24 LTS | Runtime for both apps | One current LTS runtime and TypeScript toolchain; configure the same major version locally and in Render. |
| TypeScript | 7.0.2 | Application language | Shared request/response types and fewer auth/CRUD integration errors. |
| Next.js | 16.2.12 | Public React frontend | App Router, responsive server-rendered public pages, `next/image`, and first-class Vercel deployment. Keep it a frontend consumer of the REST API. |
| React | 19.2.8 | UI runtime | Required by Next.js; use components and native browser APIs before adding state libraries. |
| Express | 5.2.1 | REST backend | Small, mature, explicit route/middleware model; faster to ship than NestJS for this two-resource API and easy to deploy as one Render web service. |

### Database

| Technology | Version | Purpose | Why |
|---|---:|---|---|
| PostgreSQL (Neon) | Neon Free | Durable relational data | Free plan has no time limit and no card requirement; Neon supplies pooled connections and a straightforward paid upgrade path without moving databases. |
| Prisma ORM + `@prisma/client` + `@prisma/adapter-pg` | 7.9.1 | Schema, migrations, typed queries, seed | Model `User` and `Animal`, enforce the owner relation in queries, and run repeatable migrations/seeds in local and hosted environments. Prisma 7 uses a PostgreSQL driver adapter at runtime. |

### Infrastructure

| Technology | Version | Purpose | Why |
|---|---:|---|---|
| Vercel Hobby | Current service | Next.js frontend deployment | Git-connected, zero-configuration Next deployment with previews. Set the public API base URL as a Vercel environment variable. |
| Render Free Web Service | Current service | Express API deployment | Free public Node web service with Git deploys and environment variables. It is appropriate for this demo/MVP API, with a documented cold-start trade-off. |
| Cloudinary Node SDK | 2.10.0 | Hosted image upload, delivery, deletion | Do not persist uploads on Render's ephemeral filesystem. Store Cloudinary `secure_url` and `public_id` in the animal record; use the public URL with `next/image`. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---|---:|---|---|
| `jsonwebtoken` | 9.0.3 | Sign and verify JWT access tokens | At login and protected REST middleware only. Keep the signing secret server-only and set explicit algorithm/expiry. |
| `bcryptjs` | 3.0.3 | Password hashing | On registration and login comparison. Use an async cost factor of 12. |
| `zod` | 4.4.3 | Validate input at API boundary | Validate every auth, animal, query, and upload metadata payload before persistence. |
| `multer` | 2.2.0 | Parse multipart image submissions | Use `memoryStorage`, allow image MIME types only, cap file count and size, then stream directly to Cloudinary. Never write uploads to disk. |
| `cloudinary` | 2.10.0 | Cloudinary API client | Use server-side signed `upload_stream` and `destroy`; the Cloudinary secret must never reach the browser. |
| `cors` | 2.8.6 | Browser-origin control | Allow only the deployed Vercel URL and localhost development URL, with the needed methods/headers. |
| `helmet` | 8.3.0 | HTTP security headers | Install on the API before routes. |
| `express-rate-limit` | 8.6.1 | Login/register throttling | Apply a stricter limiter to `/auth/login` and `/auth/register`, not a blanket limiter that harms public browsing. |
| `pg` + `@prisma/adapter-pg` | 8.22.0 / 7.9.1 | Pooled PostgreSQL runtime connection | Create one `pg.Pool` using Neon’s pooled URL and inject `PrismaPg` into the singleton Prisma Client. Required by the current Prisma 7 setup. |
| `react-hook-form` + `@hookform/resolvers` | 7.84.0 / 5.7.1 | Client-side form UX | Use only for registration, login, and animal editor forms; keep Zod validation authoritative on the API too. |
| `Vitest` + `Supertest` | 4.1.10 / 7.2.2 | API unit/integration tests | Test auth failures and owner-only update/delete against a test database; do not rely on manual browser checks. |

## Implementation Shape

Use npm workspaces:

```text
pet-site/
  apps/
    web/                 # Next.js: public catalogue, detail, auth, owner area
    api/                 # Express: /api/v1/auth and /api/v1/animals
  packages/
    contracts/           # shared TypeScript DTOs/constants only; no Prisma imports
  prisma/
    schema.prisma
    seed.ts
```

The MVP schema remains deliberately small:

- `User`: `id`, unique `email`, `passwordHash`, `name`, timestamps, and (if refresh tokens are added) one hashed refresh-token value.
- `Animal`: `id`, `ownerId` foreign key, standardized animal fields, `status`, `images` JSON (`{ publicId, secureUrl, width, height }[]`), timestamps.

Do not create a generic media service or image table in phase 1. JSON image metadata is sufficient for a small fixed photo list. A later phase can normalize it if sorting, moderation, image audit, or per-image captions are needed.

For Prisma 7, use `prisma.config.ts` with `DIRECT_URL` for CLI migrations and seed execution. At API runtime, initialize one `pg.Pool` from the pooled `DATABASE_URL`, pass it through `PrismaPg` to one application-wide `PrismaClient`, and close the pool only during process shutdown. Do not create a Prisma Client per request.

### REST Contract

Prefix all routes with `/api/v1` and keep the public/mobile-ready contract independent of Next.js:

```text
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/animals                 # public available listing
GET    /api/v1/animals/:id             # public detail + donor phone
POST   /api/v1/animals                 # authenticated owner
PATCH  /api/v1/animals/:id             # authenticated owner only
DELETE /api/v1/animals/:id             # authenticated owner only
POST   /api/v1/animals/:id/images      # authenticated owner only
DELETE /api/v1/animals/:id/images/:publicId # authenticated owner only
```

Authorization must be database-backed, not UI-backed: each modifying handler queries with both `id` and `ownerId: req.auth.userId`; a missing record returns `404`. Never accept an `ownerId` from request input. On deletion, delete Cloudinary assets after identifying their stored `publicId` values; on a partial failure, retain the database state and return an actionable error/log entry rather than hiding an orphaned asset.

### JWT Session Design

Use a short-lived access JWT (15 minutes) and refresh token (7 days). Return both only as `HttpOnly`, `Secure` cookies; never put bearer tokens in `localStorage`. Since the initial Vercel and Render domains are cross-site, configure CORS with the exact frontend origin and `credentials: true`, set cookies to `SameSite=None`, and protect state-changing cookie requests with a double-submit CSRF token (`X-CSRF-Token` compared to a readable CSRF cookie). Hash the stored refresh token, rotate it on `/refresh`, and clear it on logout. If the team later uses `app.example.com` and `api.example.com`, the same design can move to `SameSite=Lax` with the shared site domain.

This is slightly more setup than storing a JWT in local storage, but it avoids making an XSS bug an account-token theft bug and provides the required persistent authenticated session.

### Image Upload Design

The frontend submits multipart data to the protected API. The API validates file count, MIME, byte length, and animal ownership, then streams each permitted image to Cloudinary using a signed server-side upload. Store only Cloudinary metadata in Postgres. Restrict Cloudinary uploads to an `animals/<animalId>` folder, allow JPEG/PNG/WebP, cap the MVP at five photos and 5 MB each, and request a bounded display transformation. Cloudinary documents that its API secret must not be exposed in client code; a signed backend flow honors that requirement.

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|---|---|---|---|
| API framework | Express 5 | NestJS | Excellent once modules, roles, queues, and teams grow, but decorators/modules add setup with no phase-1 value. |
| API placement | Separate Express API | Next.js Route Handlers only | Would be deployable, but blurs the explicitly required REST backend boundary and makes later mobile/admin clients more coupled to the web app. |
| Database | Neon Postgres | Render Free Postgres | Render's free database expires 30 days after creation, which is unacceptable for a persistent demo/portfolio MVP. |
| Database model | PostgreSQL + Prisma | MongoDB + Mongoose | A document store does not simplify the central ownership/relation or future filtering/reporting needs; Prisma/Postgres gives safer migrations and constraints. |
| Image hosting | Cloudinary | Render local filesystem | Render free service storage is ephemeral and is lost on restart/redeploy/spin-down. |
| Upload approach | Signed API-to-Cloudinary stream | Unsigned browser preset | The latter exposes a reusable upload capability and makes per-owner authorization/control less direct. |
| Authentication | JWT in HttpOnly cookies + CSRF | JWT in localStorage | Local storage makes token exfiltration easier when an XSS bug occurs. |
| UI styling | CSS Modules + a small global token file | Heavy component/design system | The brief calls for a clean, functional responsive MVP; avoid the cost and visual opinion of a large UI system. |
| Client data layer | Native `fetch` + small typed API client | React Query at launch | There are few pages and mutations. Add TanStack Query only when caching, optimistic updates, or filter state make it pay for itself. |

## What Not to Use in Phase 1

- Do not use Render's free Postgres as the primary database: it has a hard 30-day lifetime.
- Do not use SQLite in production: the free API instance has no persistent disk.
- Do not add Redis, queues, WebSockets, search engines, full-text search SaaS, or a CMS: none supports the single adoption flow yet.
- Do not expose a Cloudinary API secret or use an unrestricted unsigned preset.
- Do not add OAuth, password reset email, role systems, chat, payments, or admin tooling. They are explicitly outside the validated MVP path.
- Do not introduce GraphQL. REST is an explicit project constraint and is clearer for this resource model.

## Installation

```bash
# root workspace
npm install -D typescript@7.0.2 tsx@4.23.5 vitest@4.1.10

# frontend (apps/web)
npm install next@16.2.12 react@19.2.8 react-dom@19.2.8 react-hook-form@7.84.0 zod@4.4.3 @hookform/resolvers@5.7.1

# REST API (apps/api)
npm install express@5.2.1 @prisma/client@7.9.1 @prisma/adapter-pg@7.9.1 pg@8.22.0 dotenv@17.4.2 jsonwebtoken@9.0.3 bcryptjs@3.0.3 zod@4.4.3 multer@2.2.0 cloudinary@2.10.0 cors@2.8.6 helmet@8.3.0 express-rate-limit@8.6.1
npm install -D prisma@7.9.1 supertest@7.2.2 @types/node@26.1.2 @types/pg@8.20.3 @types/express @types/jsonwebtoken @types/multer @types/supertest
```

Pin exact versions for the first deploy; use Renovate/Dependabot only after the MVP is stable. Re-run `npm view <package> version` as part of dependency updates rather than silently assuming these versions remain current.

## Seed and Deployment Guidance

1. Keep `prisma/seed.ts` idempotent. Create one known development donor with a password read from a non-production seed variable, then create approximately ten animals linked to it. Use stable Cloudinary sample/public URLs or checked-in public image URLs for seed data; do not require production Cloudinary credentials merely to seed locally.
2. Provision Neon first. Set `DIRECT_URL` to its direct connection string for `prisma.config.ts` CLI migrations, and `DATABASE_URL` to its **pooled** connection string for the running Render API. Run `prisma migrate deploy`, then the explicit Prisma 7 command `prisma db seed` once.
3. Create Render's web service from `apps/api`, set `NODE_ENV=production`, `DIRECT_URL`, `DATABASE_URL`, JWT/refresh/CSRF secrets, Cloudinary credentials, and the exact `WEB_ORIGIN` URL. Build with `prisma generate && npm run build`; start with `npm start`. Add `/healthz` for deployment diagnostics.
4. Create the Vercel project from `apps/web`, set `NEXT_PUBLIC_API_ORIGIN` to the Render HTTPS URL, and add that Vercel URL to `WEB_ORIGIN` on Render. Configure `next.config` remote image patterns for `res.cloudinary.com` only.
5. Expect the first Render request after 15 minutes idle to take roughly a minute. Make this visible in the UX with a loading/retry state; do not try to defeat the free-tier sleep policy with a cron ping.
6. Put a concise README at the root: prerequisites, variables without secret values, local `npm run dev`, migration/seed commands, test commands, Vercel/Render deployment setup, and known free-tier cold-start behavior.

## Sources

All sources below are first-party. Findings and free-tier details carry **MEDIUM** confidence because they were independently cross-checked through official current pages, while terms/quotas are subject to provider changes.

- [Next.js on Vercel documentation](https://vercel.com/docs/frameworks/full-stack/nextjs) — zero-configuration deployment, Git previews, and image behavior (MEDIUM).
- [Next package registry](https://www.npmjs.com/package/next), [React package registry](https://www.npmjs.com/package/react), and [Express package registry](https://www.npmjs.com/package/express) — version checks (MEDIUM).
- [Prisma getting started](https://docs.prisma.io/docs/getting-started), [Prisma database connections](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections), [Prisma seeding](https://www.prisma.io/docs/orm/prisma-migrate/workflows/seeding), and [Prisma package registry](https://www.npmjs.com/package/prisma) — Prisma/PostgreSQL support, current driver-adapter design, explicit v7 seed behavior, and version check (MEDIUM).
- [Neon pricing](https://neon.com/pricing) and [Neon free-to-production FAQ](https://neon.com/faqs/postgres-services-free-to-production) — free plan limits and upgrade path (MEDIUM).
- [Render deploy-for-free documentation](https://render.com/docs/free) — free web cold starts, ephemeral filesystem, and 30-day free Postgres expiration (MEDIUM).
- [Cloudinary Node.js integration](https://cloudinary.com/documentation/node_integration), [Cloudinary upload API](https://cloudinary.com/documentation/image_upload_api_reference), and [Cloudinary plans](https://cloudinary.com/documentation/billing_and_plans) — signed upload flow, secret handling, and free-plan credits (MEDIUM).
- npm registry version checks: [`jsonwebtoken`](https://www.npmjs.com/package/jsonwebtoken), [`bcryptjs`](https://www.npmjs.com/package/bcryptjs), [`zod`](https://www.npmjs.com/package/zod), [`multer`](https://www.npmjs.com/package/multer), [`cloudinary`](https://www.npmjs.com/package/cloudinary), [`helmet`](https://www.npmjs.com/package/helmet), and [`express-rate-limit`](https://www.npmjs.com/package/express-rate-limit) (MEDIUM).
