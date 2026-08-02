# Architecture Patterns: Plataforma de Adoção de Animais

**Domain:** Animal-adoption marketplace MVP
**Researched:** 2026-08-02
**Confidence:** MEDIUM — the JWT, Cloudinary, and local deployment mechanics are backed by current primary documentation; application-level resource and schema decisions are deliberately MVP-oriented recommendations.

## Recommended Architecture

Build a responsive single-page frontend and a separately deployed REST API. Keep the API as one modular monolith: it owns authentication, authorization, validation, persistence, and the server-side Cloudinary integration. This is the smallest architecture that can complete the core journey while keeping a clean extension boundary for filtering, messaging, administration, and image delivery later.

```text
Public visitor / authenticated donor
              |
              v
Frontend (independent static deployment)
  public catalogue + detail | auth + donor management
              |
              | HTTPS JSON / multipart, Bearer JWT only on protected calls
              v
REST API (independent service deployment)
  route/controller -> auth middleware -> use-case/service -> repository
                                           |                 |
                                           v                 v
                              Media adapter (Cloudinary)   Relational DB
                                                        users, animals,
                                                        animal_images
```

The frontend must never contain Cloudinary's API secret. The API alone performs signed, server-authenticated media operations. Cloudinary's Upload API explicitly supports backend authentication and warns not to expose the API secret in public client code. [Cloudinary Upload API](https://cloudinary.com/documentation/image_upload_api_reference)

### Component Boundaries

| Component | Responsibility | Communicates With |
|---|---|---|
| Frontend public area | Fetch and render the available-animal list and public detail; render the donor's phone only on detail. | Public API endpoints |
| Frontend donor area | Sign-up/sign-in, token/session handling, create/edit/delete own listings, select image files, and surface API errors. | Auth and protected API endpoints |
| API transport layer | HTTP routes, request parsing, response DTOs, input-size/type limits, and error mapping. It has no authorization business logic. | Auth middleware and use-case services |
| Auth module | Register/login, password hashing, JWT creation/verification, and `currentUser` extraction. | Users repository; protected route middleware |
| Animal module | Listing rules, detail lookup, create/update/delete, status transitions, and owner authorization. | Animal/image repositories; media adapter |
| Media adapter | Upload/destroy external assets and map Cloudinary responses to app-owned image metadata. | Cloudinary only; called by Animal module |
| Repositories / database | Transactions and queries; no HTTP or Cloudinary calls. | Users, animals, animal_images tables |
| Seed command | Idempotently creates development/demo users and roughly ten public listings, with stable fixture images or image metadata. | API persistence/media boundary, invoked explicitly |

Do not create separate auth, media, search, or messaging services in Phase 1. The modules are code boundaries inside one API, not separately deployed services. That keeps the MVP operationally small while avoiding a future rewrite of route handlers into domain services.

## Domain Model and API Contract

Use a relational schema with an explicit `animal_images` child table even if the public requirements name only Users and Animals. Images are still part of an animal's persisted state; a normalized child relation avoids JSON-array update races, preserves ordering, and makes later image metadata/optimization additive.

```text
users
  id (UUID), name, email (unique), password_hash, created_at, updated_at

animals
  id (UUID), owner_id -> users.id, name, species, size, age,
  sex, city, description, status, contact_phone,
  created_at, updated_at, published_at

animal_images
  id (UUID), animal_id -> animals.id, provider_public_id (unique),
  secure_url, width, height, format, bytes, position, created_at
```

`status` starts as a constrained enum (`AVAILABLE`, `ADOPTED`, optionally `DRAFT` only if the UI truly needs it). Public queries return only `AVAILABLE`; a donor's management query returns that donor's records of every allowed status. Persist the owner relationship as `animals.owner_id`, never as an email or a client-supplied owner field.

Recommended minimal endpoint surface:

| Access | Endpoint | Purpose |
|---|---|---|
| Public | `GET /v1/animals?limit=&cursor=` | Stable default ordering (`published_at DESC, id DESC`) of available animals. Include cursor pagination now; do not add filter parameters yet. |
| Public | `GET /v1/animals/:id` | One available listing plus ordered image DTOs and contact phone. Return 404 when missing or not public. |
| Public | `POST /v1/auth/register` / `POST /v1/auth/login` | Create account or exchange credentials for a short-lived access token. |
| Protected | `GET /v1/me/animals?limit=&cursor=` | Current donor's listings, including non-public status when used. |
| Protected | `POST /v1/animals` | Create a listing owned by `currentUser.id`; ignore/reject any client `ownerId`. |
| Protected | `PATCH /v1/animals/:id` | Modify a listing only after owner policy passes. |
| Protected | `POST /v1/animals/:id/images` | Upload one validated image through the API and append it at a server-assigned `position`. |
| Protected | `DELETE /v1/animals/:id/images/:imageId` | Owner-only removal of one image and provider cleanup. |
| Protected | `DELETE /v1/animals/:id` | Owner-only deletion of listing and all associated images. |
| Public | `GET /health` | Dependency-light liveness/readiness signal; do not expose secrets or internal configuration. |

Return presentation DTOs, not database rows. The public animal DTO omits `owner_id`, password data, and provider internals such as `provider_public_id`; the management DTO may additionally expose `status` and edit fields. This avoids coupling UI forms and future admin policy to persistence columns.

## Authentication and Owner Authorization

JWT is an authentication credential, not the authorization decision. Create a signed access token whose `sub` is the immutable user ID and whose `exp` bounds its lifetime; verify signature and expiration on every protected request. Configure and verify `iss`/`aud` when the deployment context uses them. RFC 7519 defines `sub`, `exp`, `iss`, and `aud` as registered claims but leaves applications to define which claims are required. [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)

Authentication middleware should only produce a trustworthy `currentUser`:

```ts
type CurrentUser = { id: string };

async function requireAnimalOwner(animalId: string, currentUser: CurrentUser) {
  const animal = await animals.findById(animalId);
  if (!animal || animal.ownerId !== currentUser.id) throw notFound();
  return animal;
}
```

Every protected mutation calls the policy after loading the target resource, including image deletion. This is the key guard against insecure direct object references: possession of a valid token must not permit edits to another donor's animal. OWASP's REST guidance separates authentication from authorization and calls for access controls on protected resources/actions. [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)

For the MVP, use an in-memory access token in the frontend and require sign-in again after a page refresh, or use an HTTP-only, `Secure`, `SameSite` cookie if persistent login is a validated UX need. Do not place a long-lived bearer token in local storage just to make persistence convenient. Record the session/refresh-token decision explicitly before implementation, because it affects CSRF controls and logout semantics.

## Image Upload and Deletion Lifecycle

Prefer API-proxied multipart uploads in Phase 1. It gives the API one place to validate MIME type, byte size, ownership, maximum count, and listing existence, and it keeps provider credentials off the browser. A later direct-to-Cloudinary flow can be added behind the same media adapter using short-lived server-generated signatures; the API remains responsible for confirming and persisting the final asset.

```text
Authenticated owner -> POST image multipart -> API validates + owner policy
  -> Media adapter uploads to Cloudinary under animals/{animalId}/{generatedId}
  -> DB transaction inserts animal_images metadata
  -> API returns image DTO

DELETE image -> owner policy -> DB deletes image row / marks cleanup pending
  -> Media adapter destroys saved provider_public_id with invalidation
  -> retry/log cleanup failure; never accept a client-supplied provider ID
```

Store the Cloudinary public ID as the deletion key, along with `secure_url`, dimensions, format, byte size, and display order. Cloudinary identifies assets with a public ID and supports deletion by that value; its delete endpoint can invalidate CDN copies, but cache invalidation propagates over seconds to minutes. [Cloudinary asset identifiers](https://cloudinary.com/documentation/upload_parameters#identification) and [destroy API](https://cloudinary.com/documentation/image_upload_api_reference#destroy_method)

There is no distributed transaction across the database and Cloudinary. Treat cleanup as a small, recoverable saga: do not delete an existing database image record until authorization succeeds; after the DB change, invoke provider deletion, and log/retry failures from a durable cleanup marker/job. On listing deletion, enqueue one cleanup task per stored public ID. This prevents an API request retry or provider timeout from leaving either unsafe authorization behavior or silent, permanent orphaned media. It is intentionally lightweight—an admin cleanup screen is deferred.

## Data Flow

### Public discovery

1. The frontend requests `GET /v1/animals` without a token.
2. The animal query applies `status = AVAILABLE`, cursor ordering, and a bounded default page size.
3. The API joins/loads only the primary ordered image required by the card and returns public DTOs.
4. Detail fetch returns the full ordered image set and contact telephone; the frontend makes a `tel:` action.

### Donor creates a listing

1. Donor registers/logs in. API validates credentials, hashes passwords, and returns a bounded JWT access credential.
2. Frontend sends an authenticated listing create request; API derives `owner_id` from JWT `sub`, validates standardized fields, and persists the new animal.
3. Frontend uploads selected images one at a time to the listing's protected image endpoint.
4. API enforces the same owner check, uploads via media adapter, persists returned metadata, and returns the new ordered image DTO.
5. Once status becomes `AVAILABLE`, the listing is visible through the public query.

### Donor changes or deletes a listing

1. The bearer token is verified; API loads the animal by route ID.
2. API compares `animal.owner_id` with `currentUser.id`. A mismatch receives the same non-disclosing 404 response as a missing resource.
3. Only then does it patch fields, remove an image, or soft/hard-delete the listing according to the chosen data-retention policy.
4. Media cleanup uses server-stored public IDs and has retriable failure handling.

## Patterns to Follow

### Pattern 1: Modular monolith with dependency direction

**What:** Route handlers depend on use cases; use cases depend on repository/media interfaces; concrete database and Cloudinary clients implement those interfaces at the outer edge.

**When:** Throughout Phase 1, even though all code ships in one API deployment.

**Why:** The core animal rules stay testable and can later serve filtering, messaging, or admin endpoints without duplicating owner checks.

```ts
interface MediaStore {
  upload(input: { animalId: string; file: UploadedFile }): Promise<StoredImage>;
  destroy(publicId: string): Promise<void>;
}

async function addAnimalImage(input: AddImageInput, actor: CurrentUser) {
  const animal = await requireAnimalOwner(input.animalId, actor);
  validateImage(input.file);
  const stored = await mediaStore.upload({ animalId: animal.id, file: input.file });
  return images.insert({ animalId: animal.id, ...stored, position: await images.nextPosition(animal.id) });
}
```

### Pattern 2: Cursor-ready, query-object public catalogue

**What:** Centralize public listing inputs in a `ListAnimalsQuery` even when the MVP only uses `cursor` and `limit`.

**When:** Build the list endpoint before a design asks for filters.

**Why:** Later `species`, `size`, `city`, and availability filters extend a validated query object and database indexes instead of creating divergent routes or client-side filtering of all records.

### Pattern 3: Explicit policy functions

**What:** One reusable `requireAnimalOwner` policy guards every animal and image mutation.

**When:** On `PATCH`, `DELETE`, and every nested image endpoint.

**Why:** This makes authorization reviewable and reusable for a future admin policy such as `canManageAnimal(actor, animal)` without spreading role checks through controllers.

## Anti-Patterns to Avoid

### Client-owned Cloudinary credentials or unrestricted unsigned preset

**What:** Browser uploads directly with a secret or a broad reusable preset.

**Why bad:** It exposes control of account storage or bypasses the application ownership/count/type rules.

**Instead:** Proxy MVP uploads through the API. If direct upload becomes necessary, issue a narrow, short-lived server signature and have the API verify/persist the returned provider asset.

### Authorizing from client data or token claims alone

**What:** Accepting `ownerId` from a request body or trusting a role/UI condition without loading the animal.

**Why bad:** Any authenticated user can forge an ID or target another listing.

**Instead:** Fetch the record and compare its persisted `owner_id` to the verified JWT subject in one policy function.

### Coupling a public image URL to an opaque JSON field

**What:** Storing a JSON URL array on `animals` and deleting provider assets by a URL sent by the client.

**Why bad:** Ordered gallery updates race, provider identifiers are lost, and cleanup cannot be authorized/retried safely.

**Instead:** Store a dedicated image record with private provider identifier and public presentation metadata.

### Seeding on application startup

**What:** API container inserts demo data each time it starts.

**Why bad:** Production deploys mutate customer data and duplicate fixtures.

**Instead:** Provide an explicit idempotent `seed` command for local/demo environments, and a separate migration command for schema changes.

## Build Order: End-to-End MVP Slices

1. **Foundation and public read slice** — define migrations, animal/image DTOs, seeded available records, API health endpoint, and a frontend public list/detail using a fixture image URL. This proves independent deployments and the visitor half of the core value without waiting for auth.
2. **Identity and owner policy slice** — add users, password hashing, registration/login, JWT verification, protected `/me/animals`, and `requireAnimalOwner` tests. The policy arrives before the first mutation so it is never retrofitted around insecure CRUD.
3. **Donor listing slice** — implement protected create/edit/status plus frontend management form. Validate all standardized animal fields server-side and publish one listing end to end.
4. **Real image lifecycle slice** — introduce Cloudinary adapter, multipart validation, image records, ordered gallery, owner-only removal, deletion cleanup/retry, and associated tests. Start with upload after a persisted listing to avoid unowned/orphaned uploads.
5. **Operational delivery slice** — idempotent seed command for about ten listings, environment validation, CORS allow-list for the deployed frontend origin, API health check, README, and independent frontend/API deployment. Use Docker Compose for local reproducibility only; Docker documents health checks and `service_healthy` dependency ordering for local services. [Docker Compose services reference](https://docs.docker.com/reference/compose-file/services/)

This ordering delivers a usable public browse/detail flow first, then safely turns it into the full donor-to-adopter journey. It intentionally postpones filters until public data and ordering are real, and postpones direct uploads/image transformations until the provider lifecycle is observable.

## Scalability and Evolution

| Concern | MVP / ~100 users | ~10K users | ~1M users |
|---|---|---|---|
| Public listing | Bounded cursor query, primary-image join, DB index on `(status, published_at, id)`. | Add index-backed filters and CDN cacheable public responses. | Read replicas/search index and CDN/API caching with invalidation strategy. |
| Images | API-proxied upload, store original metadata, Cloudinary delivery URL. | Use signed direct upload and named delivery transformations behind `MediaStore`. | Async processing/queue, responsive derived formats, observability and lifecycle retention policy. |
| Authorization | Single owner policy. | Add role/policy adapter for moderators; audit mutations. | Central policy/audit service only if product complexity justifies it. |
| Messaging | Not built; telephone is public detail data. | Add `conversations`/`messages` independent of `animals` ownership policy. | Real-time delivery, moderation, rate limits, notifications. |
| Deployment | Static frontend + one API + managed DB; explicit environment config. | Separate worker for cleanup/async jobs. | Horizontally scale stateless API and isolate worker/search workloads. |

### Decisions that Preserve the Future Path Without Adding Scope

- Keep filter inputs out of the UI, but use cursor pagination, stable ordering, a `ListAnimalsQuery`, and indexes compatible with `status/species/size/city` later.
- Keep contact phone as a public, listing-level field now. A future messaging feature adds new `conversations` and `messages` tables/modules; it does not alter ownership of `animals`.
- Keep the actor-policy signature generic (`canManageAnimal(actor, animal)`) so an admin capability can be added by changing policy implementation, not public route structure.
- Keep Cloudinary behind `MediaStore` and retain provider metadata. Named transformations, responsive formats, direct signed upload, and asynchronous cleanup become adapter changes rather than a rewrite of animal CRUD.
- Keep deployments independent: frontend receives only `API_BASE_URL`; API receives DB, JWT, Cloudinary, and CORS configuration. Never place production secrets in the frontend build or committed `.env` files. Docker also advises against using ordinary environment variables for passwords in container environments. [Docker environment-variable guidance](https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/)

## Sources

| Source | Used for | Confidence |
|---|---|---|
| [Cloudinary Upload API](https://cloudinary.com/documentation/image_upload_api_reference) | Server authentication, deletion by public ID, invalidation behavior | MEDIUM (primary source reached via web search) |
| [Cloudinary upload parameters](https://cloudinary.com/documentation/upload_parameters#identification) | Asset/public identifiers and media metadata implications | MEDIUM (primary source reached via web search) |
| [RFC 7519 — JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519) | JWT claims and expiration design | MEDIUM (primary specification reached via web search) |
| [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) | Per-resource authorization boundary | MEDIUM (authoritative security guidance reached via web search) |
| [Docker Compose services reference](https://docs.docker.com/reference/compose-file/services/) | Health checks and local dependency ordering | MEDIUM (primary source reached via web search) |
