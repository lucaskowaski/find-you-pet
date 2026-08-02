# Feature Landscape

**Domain:** Plataforma web de adoção de animais (MVP)
**Researched:** 2026-08-02
**Confidence:** MEDIUM — the product charter is the authority for scope; current MDN, W3C WAI, and OWASP guidance verifies the interaction, accessibility, and upload safeguards.

## Product Boundary

The v1 product has one complete journey: a donor creates an account, publishes an animal with its details and images, and a public visitor finds the animal in a grid, opens its detail page, and calls the donor. A feature belongs in this MVP only when it removes friction from that journey or protects its data; it must not introduce a second communication, payment, administration, organization, or mobile-app workflow.

## Table Stakes

Features users need for the publish → discover → call journey to work.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Account signup, login, and persistent authenticated session | A donor needs an identity before creating or maintaining listings. | Med | JWT authenticates protected API requests; unauthenticated visitors can still browse. |
| Owner-scoped animal CRUD | Donors must be able to correct, retire, and manage their own listings. | Med | Create, edit, and delete are authenticated; API must reject edit/delete attempts for another owner with `403`. |
| Structured animal listing | A visitor needs comparable, decision-relevant information. | Low | Fields: name, species, size, age, sex, city, description, availability status, contact phone, and images. |
| Image upload and display | Photos make a listing identifiable and materially improve adoption decisions. | Med | Require at least one image when publishing; support adding/removing/reordering images while editing. Store provider URLs, not image bytes in the core database. |
| Public grid of available animals | This is the starting point for a visitor who has no account. | Low | Show only `available` listings; each card has an image fallback, name, species, size/age, city, status, and a clear link to its detail page. No filters or advanced search. |
| Public animal detail | Visitors need the full record before contacting the donor. | Low | Show all public listing fields, an accessible image gallery, current availability, and one prominent contact action. |
| Direct phone contact | The defined v1 conversion event is contacting the donor without building messaging. | Low | Render the supplied phone number as readable text plus a descriptive `tel:` link (for example, “Ligar para o doador de Luna”). `tel:` behaviour varies by device, so the visible number remains essential. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/a) |
| Responsive and accessible web interactions | The MVP is web-first but must work on narrow and wide screens and with keyboard/assistive technology. | Med | Every input has a programmatic label and actionable errors; informative animal photos have meaningful alt text; decorative visuals use `alt=""`. [W3C WAI](https://www.w3.org/WAI/tutorials/images/) |
| Local, deployed, and seeded demonstration | It is an acceptance requirement, not a future product feature. | Low | Provide local setup, a public deployment on free-tier services, README instructions, and roughly ten seed listings. |

## Testable MVP Requirements

### Authentication and authorization

1. A new user can register with valid required credentials, sign in, receive/use a JWT-backed session, and sign out or lose access when the token is absent/invalid.
2. A logged-out visitor can load the public grid and public detail URLs without being redirected to login.
3. A logged-in donor can create a listing and later edit or delete that same listing.
4. A donor cannot edit or delete another user's listing through either the UI or a forged REST request; the server returns `403` and leaves the target unchanged.

### Listing publication

1. The create/edit form collects the nine agreed listing fields and image files. Required fields cannot be submitted blank; enum fields accept only defined values; text length limits are documented in the API/UI contract.
2. Phone input uses an appropriate telephone control and server-side normalization/validation. Browser `type="tel"` alone is insufficient because it does not enforce a universal number format. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/tel)
3. Publishing requires at least one successfully uploaded image. The interface shows upload failure without discarding already-entered form data.
4. On successful creation or edit, the server persists the complete record and image URLs, and the public pages reflect the new information. Delete removes the listing from the public grid and makes its public detail unavailable.
5. `available` is the only status shown in the public grid. Marking an animal unavailable removes it from public discovery and its detail route must not expose its phone number.

### Public discovery and contact

1. A public visitor sees an empty-state message when no available animals exist and a stable grid when they do.
2. Selecting a card navigates to the matching public detail record; it never exposes donor authentication data or management controls.
3. The detail page shows name, animal attributes, city, description, availability, gallery, and the donor's supplied phone number.
4. The contact action has descriptive text, can be reached and activated by keyboard, and uses a `tel:` URI containing a normalized number. The rendered number is also selectable/copyable for desktops that cannot call directly. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/a)

### Validation and content safety

1. Client validation gives immediate field-level feedback, but the REST API repeats all validation and authorization checks; no client check is treated as a security boundary.
2. The image service accepts a small allowlist of image types, verifies actual file type rather than trusting the request `Content-Type`, applies a configured size/count limit, generates storage names, and permits uploads only for authenticated users. These are the current OWASP baseline controls for uploads. [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
3. Listing text is handled as plain text and safely encoded on output; it must not execute as HTML/JavaScript. [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
4. Each form control has a visible/programmatic label and validation errors identify both the invalid field and what must change. [W3C WAI](https://www.w3.org/WAI/tips/developing/)

## Differentiators

These modest details make the narrow MVP trustworthy without adding a second product flow.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Explicit availability lifecycle | Prevents a visitor from calling about an animal that is no longer available. | Low | A single owner-controlled `available`/`unavailable` state is enough; do not add application workflow, reservations, or approval queues. |
| Contact-first detail page | Lets an interested visitor act in one step after reviewing the animal. | Low | One visible, descriptive call CTA and a copyable number; no chat, contact form, or lead tracking. |
| Clear photo + facts hierarchy | Lets visitors recognize the animal and scan the essential attributes quickly. | Low | This is a content/layout standard, not a design-system or recommendation engine project. |

## Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Advanced search, filters, sorting, maps, or recommendations | Adds taxonomy, query, relevance, and empty-state complexity before the core journey is validated. | Render a simple public grid of available animals. |
| Internal chat, messaging, inquiry forms, or lead inbox | Creates message persistence, moderation, delivery, and privacy obligations. | Show the donor's phone number and `tel:` call action. |
| Payments, fees, donations, contracts, or adoption applications | Changes a listings product into a transactional/regulatory workflow. | Keep adoption coordination outside the platform in v1. |
| Notifications, reminders, or email/SMS campaigns | Requires delivery infrastructure, preferences, retries, and consent management. | Let donors manually update availability; visitors contact them directly. |
| Admin dashboard, moderation queue, reports, or analytics product | Requires roles, operational policy, and staff workflow that v1 has not defined. | Enforce owner-only mutation at the API; defer centralized administration. |
| Shelter/org profiles, teams, verification, or separate account classes | The charter intentionally has one account type for shelters and individuals. | Use the same donor account and listing model for both. |
| Native mobile app | Doubles delivery surfaces while responsive web already serves the acceptance journey. | Test the responsive web layout on mobile viewports. |
| Social login, password recovery, profile customization, favorites, or saved searches | Each creates a new account-management or retention flow with no v1 dependency. | Ship basic signup/login and direct discovery only. |

## Feature Dependencies

```text
User account + JWT
  → authenticated owner identity
  → create / edit / delete own animal
  → availability status + uploaded image URLs
  → public available-animal grid
  → public animal detail
  → visible phone number + tel: contact action

Image validation + storage integration → successful image upload → listing publication
Server-side validation + owner authorization → safe CRUD → trustworthy public content
```

## MVP Recommendation

Prioritize:

1. **Identity and owner authorization:** signup/login/JWT plus server-enforced ownership, because all authenticated authoring depends on it.
2. **Reliable listing publication:** structured fields, image upload, API validation, and availability state, because this creates the inventory visitors need.
3. **Public discovery and direct contact:** grid, detail, and accessible phone action, because it completes the sole v1 success path.

Defer every anti-feature until users demonstrate that a phone-first, public-listing workflow is insufficient. In particular, do not turn “unavailable” into a multi-stage adoption process or add filters just because the seed set grows.

## Risks and Validation Focus

| Risk | Why It Matters | MVP Mitigation / Acceptance Check |
|------|----------------|-----------------------------------|
| Owner check only in the UI | A user could mutate another record through the REST API. | Test cross-user `PATCH`/`DELETE` calls and require `403` from the server. |
| Invalid or malicious uploads | Can cause storage cost, broken cards, or content-safety exposure. | Enforce type, size, count, generated filename, authenticated upload, and graceful failure; test a spoofed MIME type and oversize file. |
| Stale availability exposes phone data | Visitors may contact donors about adopted animals. | Public queries filter to available records; change status and confirm removal from grid/detail. |
| Phone entry is unusable or inaccessible | The only conversion action fails on mobile, keyboard, or desktop. | Test a valid normalized phone link on mobile and desktop, readable text, keyboard activation, and validation feedback for malformed input. |
| Image-only information excludes users | A visitor using assistive technology cannot identify an animal. | Require meaningful alt text derived from animal name/species or author-supplied description; verify gallery controls have labels. |
| Scope creep | Chat, filters, and roles delay the one journey needed to validate demand. | Treat the anti-feature list as release gating criteria; record requests for later phases rather than adding them in v1. |

## Sources

- Project scope and acceptance constraints: [PROJECT.md](../PROJECT.md) — **HIGH** confidence (authoritative project source).
- [MDN: anchor element and telephone links](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/a) — **MEDIUM** confidence (current primary documentation, verified through web search; published two months ago).
- [MDN: `input type="tel"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/tel) — **MEDIUM** confidence (current primary documentation; published last month).
- [W3C WAI: Images Tutorial](https://www.w3.org/WAI/tutorials/images/) and [accessible-development tips](https://www.w3.org/WAI/tips/developing/) — **MEDIUM** confidence (current W3C guidance, verified through web search).
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html) and [Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html) — **MEDIUM** confidence (current OWASP guidance, verified through web search).
