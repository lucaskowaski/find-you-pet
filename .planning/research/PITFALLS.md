# Domain Pitfalls

**Domain:** MVP web platform for animal-adoption listings  
**Researched:** 2026-08-02  
**Overall confidence:** MEDIUM — the findings are cross-checked against current OWASP, Cloudinary, and Brazilian government sources; the eventual deployment provider is not yet selected.

The MVP deliberately exposes a donor's telephone number so an interested adopter can make direct contact. That makes privacy, abuse resistance, and record ownership part of the core implementation—not future “nice to have” work. The mitigations below stay within scope: they use validation, consent, authorization, and provider configuration rather than chat, an admin console, complex moderation, or payments.

## Critical Pitfalls

### Pitfall 1: Client-controlled ownership (IDOR/BOLA)

**Confidence:** MEDIUM  
**What goes wrong:** An authenticated donor changes `/animals/:id`, an image ID, or a request body `ownerId` to edit or delete another donor's listing or media. Hiding edit controls in the UI does not prevent direct REST requests.

**Why it happens:** The API trusts a route parameter, body field, or JWT/UI role alone instead of checking the selected database record. OWASP identifies missing object-level authorization on endpoints receiving an object ID as a common API failure.

**Consequences:** Unauthorized deletion, alteration, or disclosure of listings and donor data; a user can also remove someone else's Cloudinary asset if nested image routes are unchecked.

**Low-scope prevention:**

- Derive the current user only from a verified JWT; never accept `ownerId`/`userId` from the client on create or update.
- On every update, delete, and image-delete query, constrain by both `animal.id` and `animal.owner_id = authenticatedUser.id`; return `404`/`403` otherwise.
- Keep public read endpoints explicitly separate from authenticated mutation routes and use an allowlist DTO so `ownerId`, `status`, and image-provider IDs cannot be mass-assigned accidentally.
- Add integration tests with two accounts proving cross-owner update, delete, and image deletion fail.

**Detection / warning signs:** A request succeeds after changing only an animal or image UUID; the API has a generic `findById(id)` then mutates it; ownership exists only in React/Vue route guards; or an update payload accepts `ownerId`.

**Address in:** **Authentication and owner-only CRUD phase**, before exposing any create/edit/delete UI.

### Pitfall 2: Public phone disclosure without informed, reversible consent

**Confidence:** MEDIUM  
**What goes wrong:** A donor assumes their phone is private, later receives spam or harassment, or cannot promptly remove it. A phone number is personal data under LGPD, even when its display is intentional.

**Why it happens:** Treating phone display as just another animal field, burying disclosure in a generic privacy link, or retaining the number after a listing is deleted or made unavailable.

**Consequences:** Harm to donors, loss of trust, avoidable LGPD transparency/consent risk, and a public data trail in previews, logs, or abandoned listings.

**Low-scope prevention:**

- Make `phonePublicConsent: true` an explicit, required checkbox beside a plain-language statement: the number is shown publicly on the listing solely for adoption contact and may be copied by visitors.
- Show the exact number in the donor's preview before publication; make edit, unpublish, and delete remove it from public API responses immediately.
- Collect no unnecessary donor data in the listing. Keep contact phone out of list-card responses; return it only on the public detail endpoint required by the product.
- Publish a short privacy notice naming the controller/contact channel, purpose, providers that process data (hosting, database, media), and the practical route to request correction/deletion. This is transparency work, not a new product feature.

**Detection / warning signs:** The form can submit without an affirmative disclosure choice; a deleted/unavailable animal still returns a phone through API/cache; seed data contains real numbers; logs or error responses serialize full donor records.

**Address in:** **Public listing/detail and data-model phase**; recheck in the **deployment QA phase**.

### Pitfall 3: Unsafe or unbounded image uploads

**Confidence:** MEDIUM  
**What goes wrong:** Attackers or accidental users upload non-images, huge images, files with misleading MIME types, or excessive images. A public unsigned Cloudinary preset can also be reused to consume the account's quota.

**Why it happens:** Browser MIME type and filename extension are trusted; uploads have no size/count restriction; the app exposes a permissive unsigned preset; Cloudinary credentials are placed in client code; or provider public IDs are accepted from the request without verification.

**Consequences:** Free-tier exhaustion, service disruption, unexpected media delivery/storage charges, harmful media being delivered, overwritten assets, and orphaned provider files after failed edits/deletes.

**Low-scope prevention:**

- Prefer a backend-generated **signed** upload flow after JWT authentication; keep the Cloudinary API secret server-only. If a client-side unsigned preset is unavoidable, restrict it to `jpg`, `jpeg`, `png`, and `webp`, set a small `max_file_size`, normalize dimensions, set `disallow_public_id`, and use a dedicated folder/preset.
- Apply server-side limits before signing/recording media: an image-only allowlist, byte limit, maximum images per animal, and fixed description/title lengths. Check the uploaded provider response before saving its public ID.
- Store Cloudinary `public_id` (and URL/metadata) with the animal image record. Delete the provider asset only after proving the authenticated user owns its parent animal; queue/retry failed cleanup rather than silently losing track of it.
- Serve only the public delivery URL required for the public listing; never expose provider API secrets or arbitrary transformation parameters to the client.

**Detection / warning signs:** The same preset accepts a PDF/video/executable-looking file, an upload request has no byte limit, a user can specify another `public_id`, provider usage climbs while listing count does not, or deletion removes the database row but leaves the asset hosted.

**Address in:** **Animal CRUD with media phase**, with the preset and secrets configured before the UI is enabled.

### Pitfall 4: Weak authentication and JWT handling

**Confidence:** MEDIUM  
**What goes wrong:** Passwords are stored or compared unsafely, login errors disclose whether an email exists, tokens never expire, tokens are accepted without signature/issuer/expiry validation, or ownership is inferred from a client-provided claim.

**Why it happens:** “JWT authentication” is treated as issuing a token rather than a complete server-side authentication boundary. A rushed MVP may use a default secret, put secrets in the frontend bundle, or omit login throttling.

**Consequences:** Account takeover, donor listing takeover/deletion, user enumeration, and amplification of the public-contact privacy risk.

**Low-scope prevention:**

- Use the framework's maintained password-hash primitive (bcrypt/Argon2 as selected by the stack); never persist plaintext or reversible passwords.
- Put a high-entropy JWT secret in deployment environment variables, use a short expiry, and verify signature, algorithm, expiry, issuer, and audience server-side in one auth middleware.
- Return one generic invalid-credentials response and apply a small per-IP/per-account throttle to login and registration; use HTTPS-only production URLs.
- Keep the JWT payload minimal (`sub` plus non-sensitive claims). Authorization still queries `owner_id` from the database for every mutation.

**Detection / warning signs:** A JWT secret appears in a committed `.env`, browser bundle, README, or client code; login says “email not found”; an expired/algorithm-swapped token is accepted; a single IP can make unlimited login attempts; or user records contain password-like plaintext.

**Address in:** **Authentication foundation phase**, before owner CRUD or public deployment.

## Moderate Pitfalls

### Pitfall 5: Abuse and resource consumption overwhelm a free tier

**Confidence:** MEDIUM  
**What goes wrong:** Bots create accounts/listings, hammer login, enumerate the public catalogue, or repeatedly upload media. Unbounded `pageSize`, request bodies, and provider operations consume bandwidth, database connections, and free quotas.

**Why it happens:** The happy-path MVP has no hard bounds because the data set begins with roughly ten seeded animals. OWASP lists request frequency, pagination, payload size, and third-party service spend as resource-consumption controls.

**Consequences:** Slow or unavailable public browsing, exhausted provider quotas, failed deployments, and surprise charges if a provider permits overage.

**Low-scope prevention:** Cap public pagination (for example, a conservative fixed maximum), request-body size, image count/size, and description length; apply basic rate limits to register/login and all authenticated write/upload-signature routes; configure provider spend/quota alerts where available. This is sufficient for the MVP—do not build a moderation back office or CAPTCHA workflow unless abuse actually appears.

**Detection / warning signs:** Sudden spikes in `429`/`5xx`, repeated identical POSTs from one IP, provider dashboards showing usage disconnected from active listings, or API responses with unbounded arrays.

**Address in:** **API foundation phase** for limits, then **deployment/observability phase** for alerts and production tuning.

### Pitfall 6: Listing quality and availability drift

**Confidence:** MEDIUM (domain-driven recommendation; no dedicated adoption-workflow source was used)  
**What goes wrong:** Listings have invalid/ambiguous standardized fields, unusable telephone formats, a photo unrelated to the animal, or remain “available” after adoption. Visitors lose trust and call donors about animals no longer available.

**Why it happens:** The database accepts arbitrary strings for species, size, sex, city, status, age, and phone; seed data is treated as production-quality data; unavailable listings are not consistently excluded from public browse responses.

**Consequences:** Broken browse/detail flow, unreliable seed/demo, data that is costly to clean later, and unnecessary donor contact.

**Low-scope prevention:** Use enum/allowlist validation for species, size, sex, and availability; a bounded, normalized phone field; required name/city/description; and a single server-side status transition. Public list queries must select only `status = AVAILABLE`; existing donors can still edit/unpublish their own listing. Seed fictional, clearly non-personal phone numbers and representative values.

**Detection / warning signs:** Multiple spellings of the same species/size, null or impossible ages, public cards where `status` is unavailable, number formatting that cannot be dialed, or real-looking personal details in seed fixtures.

**Address in:** **Data model and public catalogue phase**.

### Pitfall 7: Leaking secrets or personal data through deployment configuration, CORS, and logs

**Confidence:** MEDIUM  
**What goes wrong:** Production uses development CORS (`*` with credentials), API/database/media secrets are committed or exposed to the client, errors return stack traces, and logs retain JWTs, passwords, or donors' phone numbers.

**Why it happens:** Local development and free-tier deployment use different URLs and environment-variable mechanisms, but configuration is not explicitly reviewed before release.

**Consequences:** Token/secret compromise, unwanted cross-origin calls, privacy leakage, inability to rotate credentials safely, and difficult production debugging.

**Low-scope prevention:** Commit an `.env.example` containing names only, use platform-managed environment variables for each deployment, restrict CORS to the known web origin, return generic production errors, and redact `Authorization`, password, and phone fields from application logs. Add a startup check that fails clearly when required variables are absent; document the production variable list in the README.

**Detection / warning signs:** Real `.env` files or provider credentials in Git history, a frontend environment variable contains a secret, CORS is wildcarded in production, browser errors display stack traces, or request logging includes headers/full bodies.

**Address in:** **Deployment and release-readiness phase**.

## Minor Pitfalls

### Pitfall 8: Media lifecycle inconsistencies

**Confidence:** MEDIUM  
**What goes wrong:** An image upload succeeds but listing creation fails, a replacement leaves unused assets, or a deleted listing retains a visible media URL.

**Why it happens:** Database writes and Cloudinary operations are separate systems; treating the URL as the only media reference makes cleanup and retry impossible.

**Consequences:** Leaked or confusing images, quota waste, and broken cards.

**Low-scope prevention:** Persist a media record with provider public ID; create the animal/image association only after upload verification; on replace/delete, attempt provider cleanup and record failures for a retry on the next owner action or simple maintenance script. A transactional outbox/job system is not required for this MVP.

**Detection / warning signs:** Cloudinary asset count grows faster than image records, image URLs 404 in public cards, or deleted listings still have accessible image IDs in the database.

**Address in:** **Animal CRUD with media phase**.

### Pitfall 9: Treating the free plan as a permanent, identical production environment

**Confidence:** LOW (provider-specific quotas, sleep behaviour, and terms depend on the stack finally chosen)  
**What goes wrong:** A deployment or database becomes unavailable after quota/suspension, a cold start makes the first request appear broken, or a redeploy loses data because the persistence path was local/ephemeral.

**Why it happens:** Free plans and their limits change; the app is tested only locally; seed/reset steps are run against production without a migration boundary.

**Consequences:** Demo failure, lost listings, inaccessible API, or unexpected upgrade pressure.

**Low-scope prevention:** Select managed persistent database and media services (never host uploads/database files on an app instance), record the selected providers' current quota/cold-start limits in the README, run migrations separately from seed data, and perform a post-deploy smoke test covering register → create listing → upload image → public detail. Pin deployment configuration to explicit environment variables rather than machine defaults.

**Detection / warning signs:** Files are written to the app filesystem, production uses the same command as destructive seeding, first request frequently times out, a provider dashboard reports approaching a cap, or deploy logs show missing environment variables.

**Address in:** **Deployment and release-readiness phase**.

## Phase-Specific Warnings

| Phase topic | Likely pitfall | Low-scope mitigation |
|-------------|----------------|----------------------|
| Data model and public browse/detail | Phone is returned more broadly than intended; unavailable data remains public. | Separate public response DTOs; list only available animals; make public-phone consent explicit and removable. |
| Registration/login/JWT | Token lifecycle or password handling is improvised. | Maintained password hashing; server-only JWT secret; expiry and strict verification middleware; generic errors plus throttle. |
| Owner-only animal CRUD | Ownership is enforced by the frontend or a body field. | Query/mutate by `id + owner_id` from verified session; cross-user authorization tests. |
| Image upload/delete | Permissive preset, leaked secret, uncontrolled uploads, or orphaned provider assets. | Signed backend flow; image/size/count limits; dedicated preset/folder; persist public IDs and verify parent ownership before deletion. |
| Public deployment | Local assumptions leak into free-tier production. | Managed persistence; allowlisted CORS; environment-variable checklist; quota alerts; end-to-end smoke test. |

## Sources

- [OWASP API1:2023 — Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/) — MEDIUM confidence (current primary security guidance, cross-checked).
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — MEDIUM confidence (current primary security guidance, cross-checked).
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) and [OWASP JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html) — MEDIUM confidence (current primary security guidance, cross-checked).
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html) and [OWASP API4:2023 — Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/) — MEDIUM confidence (current primary security guidance, cross-checked).
- [Cloudinary upload presets](https://cloudinary.com/documentation/upload_presets) and [Cloudinary unsigned-upload security considerations](https://support.cloudinary.com/hc/en-us/articles/360018796451-What-are-the-security-considerations-for-unsigned-uploads) — MEDIUM confidence (current vendor documentation, cross-checked).
- [LGPD, Lei nº 13.709/2018](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709compilado.htm) and [ANPD: Titular de Dados](https://www.gov.br/anpd/pt-br/assuntos/titular-de-dados-1) — MEDIUM confidence (primary Brazilian legal/regulatory sources, cross-checked).

## Research Gaps

- The concrete hosting, database, and frontend providers are still undecided. Before implementation, validate their current free-tier quotas, sleep/persistence behavior, environment-variable limits, and any pricing-overage policy against the chosen providers' official documentation.
- This MVP deliberately omits chat, verification, moderation/admin workflows, and adoption-status automation. If growth brings impersonation, animal-welfare disputes, or sustained harassment, research a proportional abuse-reporting/response process before adding those features.
