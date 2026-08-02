<!-- GSD:project-start source:PROJECT.md -->

## Project

**Plataforma de Adoção de Animais**

Uma plataforma web onde abrigos e pessoas físicas podem cadastrar animais disponíveis para adoção. Na Fase 1, pessoas interessadas navegam por uma listagem pública, consultam o detalhe do animal e entram em contato diretamente pelo telefone informado pelo doador.

**Core Value:** Permitir que um animal disponível para adoção seja publicado e encontrado facilmente, com um canal direto para o interessado falar com seu doador.

### Constraints

- **Scope**: Fase 1 deve permanecer um MVP funcional com um único fluxo principal — orçamento e cronograma são deliberadamente enxutos.
- **Backend**: API REST com cadastro/login, JWT e CRUD de animais — requisito explícito da entrega.
- **Data**: Duas entidades centrais, usuários e animais — manter o modelo inicial simples.
- **Images**: Upload em Cloudinary no plano gratuito ou serviço equivalente — evitar custo de armazenamento no MVP.
- **UX**: Web responsiva, limpa e funcional, sem elaboração visual excessiva — foco no fluxo de adoção.
- **Delivery**: Aplicação publicada, repositório GitHub do cliente, README de instalação e seed com cerca de 10 registros — critérios de aceite.

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommendation

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

- `User`: `id`, unique `email`, `passwordHash`, `name`, timestamps, and (if refresh tokens are added) one hashed refresh-token value.
- `Animal`: `id`, `ownerId` foreign key, standardized animal fields, `status`, `images` JSON (`{ publicId, secureUrl, width, height }[]`), timestamps.

### REST Contract

### JWT Session Design

### Image Upload Design

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

# root workspace

# frontend (apps/web)

# REST API (apps/api)

## Seed and Deployment Guidance

## Sources

- [Next.js on Vercel documentation](https://vercel.com/docs/frameworks/full-stack/nextjs) — zero-configuration deployment, Git previews, and image behavior (MEDIUM).
- [Next package registry](https://www.npmjs.com/package/next), [React package registry](https://www.npmjs.com/package/react), and [Express package registry](https://www.npmjs.com/package/express) — version checks (MEDIUM).
- [Prisma getting started](https://docs.prisma.io/docs/getting-started), [Prisma database connections](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections), [Prisma seeding](https://www.prisma.io/docs/orm/prisma-migrate/workflows/seeding), and [Prisma package registry](https://www.npmjs.com/package/prisma) — Prisma/PostgreSQL support, current driver-adapter design, explicit v7 seed behavior, and version check (MEDIUM).
- [Neon pricing](https://neon.com/pricing) and [Neon free-to-production FAQ](https://neon.com/faqs/postgres-services-free-to-production) — free plan limits and upgrade path (MEDIUM).
- [Render deploy-for-free documentation](https://render.com/docs/free) — free web cold starts, ephemeral filesystem, and 30-day free Postgres expiration (MEDIUM).
- [Cloudinary Node.js integration](https://cloudinary.com/documentation/node_integration), [Cloudinary upload API](https://cloudinary.com/documentation/image_upload_api_reference), and [Cloudinary plans](https://cloudinary.com/documentation/billing_and_plans) — signed upload flow, secret handling, and free-plan credits (MEDIUM).
- npm registry version checks: [`jsonwebtoken`](https://www.npmjs.com/package/jsonwebtoken), [`bcryptjs`](https://www.npmjs.com/package/bcryptjs), [`zod`](https://www.npmjs.com/package/zod), [`multer`](https://www.npmjs.com/package/multer), [`cloudinary`](https://www.npmjs.com/package/cloudinary), [`helmet`](https://www.npmjs.com/package/helmet), and [`express-rate-limit`](https://www.npmjs.com/package/express-rate-limit) (MEDIUM).

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `$gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `$gsd-debug` for investigation and bug fixing
- `$gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `$gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
