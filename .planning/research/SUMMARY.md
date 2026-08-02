# Project Research Summary

**Project:** Plataforma de Adoção de Animais  
**Domain:** Responsive animal-adoption listing MVP  
**Researched:** 2026-08-02  
**Confidence:** MEDIUM

## Executive Summary

This is a narrow public-listings product, not an adoption workflow platform: a donor signs up, publishes an available animal with photos and a public contact number, and a visitor browses, opens the detail, and calls. Build that single journey as a responsive Next.js web app consuming a separately deployed Express REST API, with PostgreSQL as the durable source of truth and Cloudinary for media.

The MVP must remain small but its safeguards are non-negotiable. Ownership must be enforced by the API against the persisted record, public queries must expose only available listings, and a donor must explicitly consent to public phone disclosure. Use authenticated, bounded server-side image uploads and deploy with allowlisted CORS, server-only secrets, migration/seed separation, and an end-to-end smoke test. Do not add filters, chat, payments, roles, administration, notifications, or a mobile app.

## Key Findings

### Recommended Stack

Use an npm-workspace TypeScript monorepo: **Next.js 16.2.12 / React 19.2.8** for the responsive public and donor web UI, and **Express 5.2.1** for the independent `/api/v1` REST API. The separation honors the explicit REST requirement and keeps mobile/admin extensions decoupled from the web frontend.

**Core technologies:**

- **Node.js 24 LTS + TypeScript 7.0.2:** one current runtime and shared contracts across apps.
- **PostgreSQL on Neon Free + Prisma 7.9.1:** durable relational storage, constraints, migrations, and typed ownership queries; use Neon rather than expiring Render Free Postgres.
- **Vercel Hobby + Render Free Web Service:** simplest independent web/API deployment; accommodate Render cold starts in the UI.
- **Cloudinary 2.10.0:** persistent image delivery without relying on Render's ephemeral filesystem.
- **Zod, bcryptjs, jsonwebtoken, multer, helmet, cors, express-rate-limit:** API-boundary validation, password hashing, JWT session handling, bounded multipart uploads, headers, origin control, and targeted abuse protection.

Keep only `User` and `Animal` as phase-1 domain entities. Resolve the research disagreement on image persistence in favor of the stated simple two-entity scope: store a small, ordered `Animal.images` metadata value containing Cloudinary `publicId`, secure URL, dimensions, and position. It is not a generic media subsystem; retain `publicId` so deletion remains authorized and recoverable.

### Expected Features

**Must have (table stakes):**

- Signup, login, logout, and a JWT-backed authenticated donor session.
- Server-enforced owner-only create, edit, availability change, and deletion of listings.
- Structured listings: name, species, size, age, sex, city, description, availability, public phone, and images.
- At least one successfully uploaded image to publish; bounded add/remove image support during editing.
- Public grid of available animals, public detail/gallery, readable phone number, and accessible `tel:` call action.
- Responsive, keyboard-accessible forms/pages, roughly ten fictional seed listings, README, local run path, and free-tier deployment.

**Trustworthy MVP details:** a simple available/unavailable lifecycle, clear photo-and-facts hierarchy, explicit phone-publication consent, plain-text rendering of listing content, and accessible image alt text.

**Defer (v2+):** advanced search/filtering/sorting/maps, chat or inquiry forms, payments/applications, notifications, admin/moderation/reporting, organization roles, native mobile, social login, password reset, profiles, favorites, and saved searches.

### Architecture Approach

Deploy a frontend and one modular-monolith API. Within the API, route/transport code delegates to auth and animal use cases; those use cases call repository and Cloudinary-adapter boundaries. Return public and management DTOs rather than database rows: public list cards omit phone, owner IDs, and provider identifiers; public detail returns the phone only for an available listing.

**Major components:**

1. **Public frontend:** available catalogue, detail/gallery, visible/copyable phone, and `tel:` action.
2. **Donor frontend:** account forms and own-listing management, with API errors preserved alongside form state.
3. **Auth and animal API modules:** validation, login/JWT verification, reusable ownership policy, status rules, and DTO mapping.
4. **Persistence and media adapter:** Prisma/Postgres ownership queries plus server-side Cloudinary upload/deletion using stored provider IDs.
5. **Explicit operational commands:** migrations and idempotent seed run separately; health endpoint, environment validation, and deployment configuration stay outside app-start side effects.

### Critical Pitfalls

1. **Broken object ownership:** derive the actor only from a verified JWT and query/mutate by both animal ID and persisted `ownerId`; test two-account cross-owner update, delete, and image-delete failures.
2. **Unconsented or stale phone disclosure:** require affirmative public-phone consent, exclude phones from list DTOs, and immediately hide detail phone data when unavailable or deleted.
3. **Unsafe/unbounded uploads:** API-proxy signed Cloudinary uploads; allow only JPEG/PNG/WebP, verify actual type, cap five photos and 5 MB each, generate names, and never expose Cloudinary secrets.
4. **Weak JWT/password handling:** bcrypt hash passwords, short-lived verified tokens, generic login failures, focused login/register rate limits, and no long-lived token in local storage.
5. **Free-tier/configuration failures:** never store files locally, use exact CORS origins with credentials, redact secrets/phones from logs, separate migrations from seeds, document cold starts and provider limits, and smoke-test the deployed journey.

## Implications for Roadmap

### Phase 1: Foundation and public discovery

**Rationale:** Public browse/detail establishes the visitor half of the only value path and locks the public DTO/privacy boundary before authenticated writes exist.

**Delivers:** workspace structure, Prisma schema/migration, health endpoint, idempotent seed fixtures, bounded available-only list/detail API, responsive grid/detail with fixture image URLs, empty state, and accessible phone presentation on detail only.

**Addresses:** structured listings, public browsing, details, direct phone contact, responsiveness/accessibility, seed/demo.

**Avoids:** availability drift, public phone leakage in cards, unbounded catalogue responses, and startup seeding in production.

### Phase 2: Identity and owner authorization

**Rationale:** Owner policy must precede all listing mutations so insecure CRUD is never retrofitted.

**Delivers:** user persistence, registration/login/logout, short-lived JWT-backed session, password hashing, rate limits, auth middleware, reusable `requireAnimalOwner`, donor listing view, and cross-user authorization tests.

**Addresses:** donor identity and persistent authenticated session.

**Avoids:** IDOR/BOLA, weak password/token handling, client-controlled ownership, and account enumeration.

### Phase 3: Donor listings and publication safeguards

**Rationale:** With trusted identity available, build the actual authoring flow and its privacy/quality constraints.

**Delivers:** create/edit/delete/status endpoints and responsive forms, strict Zod validation, normalized phone field, required public-phone consent and preview, publish/unpublish behavior, and public-list refresh semantics.

**Addresses:** owner-scoped CRUD, all standardized listing fields, explicit availability lifecycle, and public contact consent.

**Avoids:** malformed data, stale available listings, phone disclosure without consent, and output injection.

### Phase 4: Cloudinary image lifecycle

**Rationale:** Add real upload only after a persisted, owned animal exists; this prevents unowned uploads and makes cleanup identity-aware.

**Delivers:** protected multipart upload/delete, ordered `Animal.images` metadata, gallery/editor UI, limits and type verification, Cloudinary adapter, and logged/retriable cleanup on deletion failure.

**Addresses:** required publishing image and editable gallery.

**Avoids:** secret exposure, permissive presets, quota abuse, client-supplied provider IDs, and orphaned media.

### Phase 5: Release readiness and deployment

**Rationale:** The acceptance criteria require a working free-tier deployment, not merely local functionality.

**Delivers:** Neon/Render/Vercel environment configuration, exact CORS and cookie/CSRF configuration if persistent cookie sessions are used, README and `.env.example`, migrations/seeds, loading/retry UX for cold starts, and production smoke test from registration through public detail.

**Addresses:** local and deployed demonstration, documentation, and operational reliability.

**Avoids:** ephemeral persistence, missing configuration, secret/log leakage, wildcard CORS, and a broken first production request.

### Phase Ordering Rationale

- The public read model comes first to prove the core discover-and-call outcome; authenticated publication then fills the catalogue safely.
- Authentication and reusable server-side ownership policy are dependencies of every write and image operation.
- Media follows persisted listings, so authorization, limits, provider IDs, and cleanup have one clear lifecycle.
- Deployment is finalized after the complete path exists, while its environment constraints inform every prior phase.

### Research Flags

Phases likely needing deeper research during planning:

- **Phase 2:** select and specify one session mechanism. The stack recommends HttpOnly-cookie access/refresh tokens plus CSRF for cross-site Vercel/Render deployment, while architecture notes an in-memory token alternative; choose cookie sessions because the project requires a maintained session, then validate browser cookie/CORS behavior in the actual deployment domains.
- **Phase 4:** Cloudinary signed-upload API, actual file-type verification, and cleanup retry handling require provider-specific implementation research.
- **Phase 5:** recheck current Vercel, Render, Neon, and Cloudinary free-tier terms, cold-start behavior, and quotas before deployment.

Phases with standard patterns (skip research-phase):

- **Phase 1:** Prisma schema/migration, stable cursor-ready public listing, DTOs, and responsive browse/detail are well-established.
- **Phase 3:** standard server-side validation and owner-scoped REST CRUD patterns are well documented once the session contract is chosen.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Official package/provider sources support the choice, but free-tier terms and exact current versions change. |
| Features | HIGH | Project scope is authoritative; accessibility and upload requirements are backed by W3C/MDN/OWASP. |
| Architecture | MEDIUM | Strong standard patterns; the session and image-storage representation required scope reconciliation. |
| Pitfalls | MEDIUM | OWASP, Cloudinary, LGPD, and ANPD support safeguards; operational provider behavior remains variable. |

**Overall confidence:** MEDIUM

### Gaps to Address

- **Session implementation:** decide and document cookie/refresh/CSRF details before Phase 2; do not use local-storage persistence as a shortcut.
- **Provider validation:** confirm quotas, cold starts, domains, environment limits, and pricing at Phase 5 rather than treating free plans as stable guarantees.
- **Data retention/privacy:** define the minimal retention/deletion behavior for listing phone data and failed Cloudinary cleanup; the MVP needs a short privacy notice but not an admin workflow.
- **Image model evolution:** the bounded `Animal.images` value is sufficient for MVP. Normalize to an image table only when image-specific moderation, captions, audit, or complex ordering becomes a validated need.

## Sources

- [STACK.md](STACK.md) — recommended versions, deployment topology, Neon/Cloudinary/Prisma guidance.
- [FEATURES.md](FEATURES.md) — authoritative MVP journey, table stakes, accessibility, and exclusions.
- [ARCHITECTURE.md](ARCHITECTURE.md) — modular-monolith boundaries, DTOs, build order, and media lifecycle patterns.
- [PITFALLS.md](PITFALLS.md) — OWASP/LGPD-informed ownership, privacy, upload, and deployment safeguards.
- [PROJECT.md](../PROJECT.md) — authoritative product scope, acceptance constraints, and out-of-scope features.

---
*Research completed: 2026-08-02*  
*Ready for roadmap: yes*
