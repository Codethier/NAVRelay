# AGENTS.md: NAVRelay

Last updated: 2026-08-25

## 1. Purpose

NAVRelay is an open-source, API-first invoicing platform for Hungarian businesses. It issues invoices through two managed workflows: `NAVRELAY_RENDERED`, where NAVRelay allocates the number and renders the PDF, and `TRUSTED_CLIENT_RENDERED`, where NAVRelay allocates the number, locks the snapshot, and a trusted client returns PDF bytes from a scoped render package. Every issued invoice has one canonical document in S3-compatible storage, is reported directly to NAV, and is available through authenticated review or revocable audit links. It supports self-hosted and hosted multi-tenant use. Stripe supplies payment and tax evidence only; its PDFs and receipts are never canonical documents.

### Product position and go/no-go rule

Avoid building NAVRelay only to save one low-volume company a small Billingo or Számlázz.hu fee. Build it when at least one of these is required:

- open-source and self-hosted deployment;
- a stable developer-first REST API instead of a proprietary invoice UI;
- customer-owned S3 storage and reproducible audit exports;
- direct NAV access without a commercial invoice intermediary;
- multi-organization SaaS with human and machine access;
- reusable Stripe and other payment-provider adapters;
- transparent, versioned tax and NAV mappings with an auditable process.

The initial product covers invoicing, archival, NAV reporting, and audits. It does not try to copy every Billingo feature or become a general accounting, banking, inventory, or ERP system.

## 2. Product principles

These rules are mandatory:

- **API first.** Add each major business operation to the documented public API before the UI.
- **Direct NAV.** Do not depend on Billingo, Számlázz.hu, or another invoice SaaS for NAV reporting.
- **One immutable document.** Every issued invoice has one canonical document whose exact bytes remain unchanged.
- **Managed numbering.** NAVRelay allocates every final invoice number from a NAVRelay-managed series. Clients and providers never supply one.
- **S3 storage.** PostgreSQL keeps metadata and object references. Never keep durable production documents on an app filesystem or Railway volume.
- **Multi-tenant and multi-user.** Users may join several organizations. Organizations may have several users and machine clients.
- **Auditable.** Issued data is append-only. Log privileged actions, make exports reproducible, and allow scoped access by login or share link.
- **No guessed compliance.** Use current official specifications and approved policy for tax, required content, NAV mappings, archival, corrections, and statutory exports.
- **No destructive corrections.** Correct, cancel, or reverse an invoice with a new linked document, never an in-place edit.
- **Final NAV result.** A transaction ID is not success. Continue processing until NAV returns a final result.
- **Idempotency.** Duplicate API calls or Stripe webhooks, worker retries, and process restarts must not duplicate invoices or NAV submissions.

## 3. Instruction priority

Resolve conflicts in this order:

1. Current Hungarian law and official NAV specifications.
2. Decisions in this file.
3. Current repository code and migrations.
4. `defaultstack.md` and the user's default-stack conventions.
5. Framework defaults and third-party examples.

Never weaken immutability, authorization, tenancy, idempotency, or audit rules to simplify code.

## 4. Legal and compliance boundary

This is an engineering specification, not legal advice. Before production, obtain written approval from a Hungarian accountant, tax adviser, or tax lawyer for:

- invoice and tax scenarios, required fields and wording, VAT and exemption, EU B2B reverse charge, and EU B2C or OSS;
- foreign currency and HUF VAT, plus advance, final, corrective, and cancellation invoices;
- electronic-invoice authenticity and integrity, including exact NAV `electronicInvoiceHash` behavior;
- retention, deletion, and the required "adóhatósági ellenőrzési adatszolgáltatás" export;
- trusted-renderer identities and profiles, upload controls, handback deadline, issue-state semantics, and PDF/NAV hash timing;
- Stripe payment and tax profiles and mappings.

Never use a Stripe invoice PDF or receipt as a NAVRelay invoice document. A trusted client may return PDF bytes only for an active, scoped render request created after NAVRelay allocates the number and locks the invoice snapshot.

## 5. Default technology stack

Use repository-pinned versions. Otherwise follow the user's default stack unless this file overrides it.

### Core stack

- TypeScript for frontend, backend, worker, and shared packages.
- Modern Node.js with native ESM.
- pnpm pinned through `packageManager`.
- Nuxt, Vue Composition API, SSR, Nuxt UI, Tailwind CSS, and Nuxt i18n.
- NestJS organized by business domain.
- Swagger/OpenAPI generated from NestJS controllers and DTOs.
- `nuxt-open-fetch` for the generated frontend business API client.
- `class-validator` DTO validation on every write boundary.
- Railway PostgreSQL.
- MikroORM with generated migrations.
- Better Auth cookie sessions plus the official Organization plugin.
- Request-aware backend localization.
- Vitest, Supertest, Testcontainers, and Playwright.
- ZeptoMail behind a backend email abstraction.
- S3-compatible storage as the required primary document store.

### Deployment defaults

Use separate Railway services for the Nuxt app, NestJS API, PostgreSQL, and durable worker. NAV submission and status checks, document processing, exports, and email belong in the worker, never as untracked web-process work.

GitHub Actions is required. Every pull request and protected-default-branch push must run the root `pnpm run verify` unchanged. The stable `verify` check must block merges. If Railway deploys automatically, release the same verified commit SHA.

## 6. Repository layout

Prefer this pnpm workspace:

```text
NAVRelay/
├─ apps/
│  ├─ web/                 # Nuxt user, admin, and audit UI
│  ├─ api/                 # NestJS public and internal API
│  └─ worker/              # NAV, PDF, export, and email jobs
├─ packages/
│  ├─ nav-online-invoice/  # NAV v3 transport, XML, signing, parsing
│  ├─ invoice-domain/      # Shared values and pure calculations
│  ├─ pdf-renderer/        # Deterministic invoice rendering
│  ├─ storage/             # S3 abstraction and integrity helpers
│  ├─ api-client/          # Generated output only, if retained
│  └─ test-fixtures/       # Official-schema and project fixtures
├─ docs/{compliance,architecture,operations}/
├─ .github/workflows/
│  ├─ verify.yml           # required PR and default-branch gate
│  └─ nav-contract.yml     # trusted manual or scheduled NAV contract run
├─ defaultstack.md
├─ AGENTS.md
├─ pnpm-workspace.yaml
└─ package.json
```

Do not duplicate business rules in `apps/api` and `apps/worker`. Share domain services, but keep controllers, worker consumers, and provider adapters as separate entry points.

## 7. High-level architecture

```text
Browser -> Nuxt -> Railway private network -> NestJS
Machine client -> public /v1/* with API key -> NestJS
Stripe -> signed public webhook -> NestJS
NestJS -> PostgreSQL for domain state, jobs, and audit metadata
NestJS -> S3 for canonical PDFs, NAV response artifacts, manifests, and exports
NestJS -> durable queue -> worker -> NAV, S3, ZeptoMail
Public API -> /docs and /openapi.json
```

NestJS owns business rules, authorization, issuance, tenancy, and idempotency; the worker handles retriable external work; PostgreSQL owns state and metadata; S3 owns canonical bytes.

## 8. Tenant and user model

NAVRelay is multi-tenant from the first migration.

### Required entities

Mirror Better Auth's runtime model exactly in MikroORM migrations:

- `User`, `Session`, `Account`, `Verification`;
- `Organization`;
- `OrganizationMembership`, matching Better Auth `Member`;
- `OrganizationInvitation`.

NAVRelay owns at least:

- `ApiClient`, `ApiKey`, `NavConnection`, `InvoiceSeries`, `TaxProfile`;
- `Invoice`, `InvoiceLine`, `InvoiceDocument`, `InvoiceRenderRequest`, `RendererProfile`, `InvoiceRelation`, `PaymentReference`;
- `NavSubmission`, `NavSubmissionMessage`;
- `StripeIntegration`, `StripeEvent`;
- `AuditEvent`, `AuditShare`, `ExportJob`, `StoredObject`;
- `BackgroundJob` or an equivalent durable queue model.

Users may join several organizations. Every tenant-owned row needs an explicit `organizationId`, directly or through an unavoidable parent.

### Better Auth organization ownership

- Enable `organization()` on the server and `organizationClient()` on the frontend.
- Better Auth writes users, sessions, organizations, members, invitations, active-organization state, and member roles at runtime.
- MikroORM owns all DDL and migration history. Mirror every enabled Better Auth and Organization-plugin table, field, index, and unique constraint. Disable Better Auth production auto-migration.
- Define initial roles and permissions as typed static code through Better Auth access control. Do not add parallel `Role` or `Permission` tables.
- Disable dynamic organization roles initially. If added later, use Better Auth's dynamic model and mirror its organization-role table.
- API clients and keys remain NAVRelay entities. They never act as Better Auth users or members.
- Disable or intercept generic Better Auth organization deletion. Organizations with retained invoices enter a closed archival state; never cascade-delete retained records.

### Human roles

- `owner`: ownership, billing, credentials, members, and all data.
- `admin`: settings, users, integrations, and invoice operations.
- `accountant`: invoices, NAV results, exports, corrections, and customer data.
- `operator`: create drafts and issue supported invoices, without credential administration.
- `auditor`: read-only invoices, documents, NAV results, and exports.
- `viewer`: limited read-only operations.

Roles are organization-scoped. A user-level universal role is never the authorization source.

### Machine access

Machine clients are not fake users. API clients and keys need:

- organization and explicit permission scopes;
- hashed secrets at rest and an identifiable prefix;
- creation, rotation, last-used, expiry, and revocation times;
- configurable rate limits and usage accounting;
- full audit logging;
- one-time secret display with no later retrieval.

API keys are the main machine authentication method. Network location and obscurity are not access controls.

## 9. Authorization and tenant isolation

The backend must enforce organization ownership on every query and mutation:

- never trust a caller-supplied `organizationId`;
- derive allowed organizations from human membership or the API-key principal;
- scope every lookup for invoices, series, render requests, documents, customers, exports, and shares by organization;
- prefer repository and service methods that require organization context;
- prevent cross-tenant S3 key access;
- authorize before creating a presigned URL;
- limit each audit share to its immutable scope.

## 9A. API exposure and product endpoints

The API and its documentation are public. Authentication and authorization protect operations.

### Public endpoints

- Serve the versioned machine API at public HTTPS `/v1/*`.
- Serve supported Billingo v3 compatibility on a dedicated public host such as `billingo-api.example.com/v3/*`.
- Publish `/openapi.json` and Swagger UI at `/docs`.
- Require API keys, organization and permission scopes, rate limits, and idempotency for machine writes.
- Treat Nuxt as a first-party client of the same business API for invoice editing, administration, review, audits, and exports.
- Nuxt may use same-origin BFF routes and Railway private networking, but it must still pass through NestJS business rules and authorization.
- Give each provider a narrow public webhook route with provider-specific signature verification and durable event deduplication.
- Never expose worker, queue-admin, migration, or maintenance endpoints publicly.

One NestJS codebase may serve session routes, the machine API, and webhooks. Keep their guards, DTOs, and namespaces separate while sharing domain services and persistence.

### Billingo API v3 compatibility

Enable compatibility by default for each organization, gated by API-key scopes. Native `/v1` remains canonical and preferred. For explicitly supported operations, a Billingo client may need only a base URL, credential, and idempotency-input change. Publish every known difference.

- Pin each release to a Billingo OpenAPI version. Start with `3.0.15`.
- Keep a versioned matrix of supported operations, fields, document types, and known differences.
- Keep Billingo controllers, DTOs, serializers, and error mapping at the HTTP boundary. Translate to the same application services as `/v1`; never persist Billingo DTOs as the invoice aggregate.
- Publish separate native and compatibility OpenAPI documents. Never claim unsupported operations.
- For supported operations, preserve `/v3`, `X-API-KEY`, `snake_case`, integer compatibility IDs, pagination, status codes, error envelopes, content types, and rate-limit headers.
- Use stable organization-scoped mappings between compatibility IDs and internal IDs. Never expose internal UUIDs or resolve compatibility IDs globally.
- Map Billingo document blocks only to configured NAVRelay-managed series and approved renderer modes. Never accept external numbering or arbitrary PDFs through the adapter.
- Support only implemented and approved operations and tax or document cases. Return a documented compatibility error: `501` for an unimplemented operation and `422` for unsupported input or scenarios. Never reinterpret receipts, spending records, waybills, document types, VAT codes, or corrections.
- Preserve all NAVRelay authorization, tenancy, exact arithmetic, immutability, S3, jobs, audit, retention, and final-NAV-result rules.
- Use a unique Billingo `vendor_id` as the document-creation idempotency key when the contract permits it. Otherwise require `Idempotency-Key` for legal or externally visible writes. Reject missing keys; do not deduplicate by body hash alone.
- Keep NAVRelay-only features and richer lifecycle data under `/v1`. Do not add undocumented fields to Billingo responses.
- Test against the pinned contract, generated Billingo clients, project golden fixtures, tenant-isolation cases, and retries. Live comparisons require an official Billingo test account, explicit opt-in, and no production target.
- Write independent, unaffiliated documentation. Do not copy Billingo docs, examples, UI, code, or branding, or imply endorsement or full parity.

### Core input workflows

All inputs use one managed invoice engine:

1. **NAVRelay-rendered issuance.** Accept structured data, validate the tax profile, allocate from a NAVRelay-managed series, lock the snapshot, render and store the PDF, then report to NAV.
2. **Trusted-client-rendered issuance.** Validate and allocate exactly as above, then return the server-assigned number, immutable snapshot, renderer metadata, and a scoped render-attempt token. The trusted client returns PDF bytes for validation, hashing, canonical storage, and NAV reporting.
3. **Provider-triggered issuance.** Translate authoritative provider state into one of the two managed renderer modes. Providers never supply the invoice number or canonical PDF and must not implement another invoice engine.

No API accepts an already issued invoice, externally assigned number, standalone final PDF, or unrestricted S3 key.

The editor is optional UI, not another issuance mode. Headless deployments may use only the documented API and webhooks.

## 10. Invoice series and blocks

Invoice series are the numbering authority. All are NAVRelay-managed; drafts have no number. Native issuance accepts only an enabled issuance profile; a compatibility adapter may pass a block identifier mapped to a preconfigured organization series. No caller submits a raw series identifier or configuration, numbering mode, sequence, or final number.

### Series fields

Store organization, internal name, public prefix or number format, current or next sequence, optional year reset, document-type policy, allowed renderer modes, optional currency limits, active state, first and last issue times, immutable configuration history or snapshots, and audit metadata.

### Allocation

Allocate only in the transaction that creates the locked `ISSUING` record and immutable snapshot. Use a row lock, atomic update, or another PostgreSQL-safe serialization method. Concurrent requests must never get the same number.

```text
begin
  lock series and validate that it is active and compatible
  allocate next sequence
  create ISSUING invoice with a unique final number
  store immutable invoice and series snapshots
  create durable render request or render job
  increment counter
  append audit event
commit
```

Do not make external calls, render PDFs, upload to S3, call NAV, or send email inside this transaction.

### Series immutability

After the first invoice, never change the prefix or formatting rules, reduce or reuse a sequence, or delete the series. Deactivate it and keep enough historical configuration to reconstruct every number.

## 11. Invoice aggregate

The immutable invoice snapshot is locked when the number is allocated and includes:

- organization and supplier identity and tax details;
- required customer identity and tax details;
- number, issue date and time, performance date, and any due date;
- currency and any exchange-rate data;
- lines, quantities, units, discounts, and adjustments;
- net, tax, and gross amounts;
- tax categories and legal reason codes;
- payment method and reference;
- language, notes, and required wording;
- original, correction, and cancellation relations;
- source system and identifiers.

Keep document metadata and mutable business, document, NAV, and delivery state outside the snapshot; append their history and lock document metadata at canonicalization. Version the snapshot schema and store a SHA-256 hash of its deterministic serialization; render requests bind to that version and hash.

Use exact decimals or integer minor units. Never calculate money or tax with JavaScript floating point.

### Tax boundary

NAVRelay is not an autonomous tax adviser.

- Every invoice selects an explicit, versioned, organization-approved `TaxProfile`.
- Callers provide the profile ID and facts only. Profiles define seller and customer VAT status, jurisdiction, rate or exemption, evidence, wording, rounding, currency and exchange rates, and NAV mapping.
- Start narrow; fail closed or require review for unsupported or ambiguous cases.
- Provider calculators may supply evidence. NAVRelay validates it and owns the snapshot and NAV mapping; provider output is not legal approval.
- Store profile and mapping versions, external calculation ID, facts, and result.

## 12. Invoice lifecycle

Keep separate state fields:

- Business: `DRAFT`, `ISSUING`, `ISSUED`, `CORRECTED`, `CANCELLED`.
- Document: `MISSING`, `GENERATING`, `AWAITING_CLIENT_PDF`, `STAGED`, `CANONICAL`, `FAILED`.
- NAV: `NOT_REQUIRED`, `PENDING`, `SUBMITTING`, `PROCESSING`, `ACCEPTED`, `ACCEPTED_WITH_WARNINGS`, `REJECTED`, `RETRY_REQUIRED`, `MANUAL_REVIEW`.
- Delivery: `NOT_REQUESTED`, `QUEUED`, `SENT`, `FAILED`.

Allocation sets `ISSUING` and locks invoice and series snapshots; it is neither editable nor reversible. Rendering failure cannot release or reuse the number, and retries use the same snapshot. Canonicalization sets `ISSUED`. NAV failure leaves the issued invoice for retry or review and never rewinds it. Issue timing follows approved policy.

## 13. Document modes

Each allocated invoice may produce only one canonical `InvoiceDocument`, and every `ISSUED` invoice must have one. Rendering location never changes numbering authority: NAVRelay always owns the series, number, and immutable invoice snapshot.

Both modes validate the draft, profiles, and server-resolved series; allocate and lock the invoice and series snapshots in one PostgreSQL transaction; render or receive the PDF; validate and hash the exact bytes; store them unchanged as the canonical S3 object; and set `CANONICAL` and `ISSUED`. NAV becomes runnable only after canonical PDF bytes and required hashes exist; enqueue delivery only after canonicalization.

### 13.1 NAVRelay-rendered PDF

For `NAVRELAY_RENDERED`, the worker renders only from the snapshots. Record renderer identity and version, template version, locale, fonts, and assets. Rendering must be reproducible for investigation and never replace a canonical object.

### 13.2 Trusted-client-rendered PDF

For `TRUSTED_CLIENT_RENDERED`, validate the renderer profile, identity, and client authorization, then create an `InvoiceRenderRequest` and durable outbox row during allocation. Return the invoice and attempt IDs, server number, immutable series snapshot/version, invoice snapshot/hash, approved renderer/template metadata, expiry, and a short-lived token or staging target. Show the token once, store only its hash, and never log it. Bind the request to organization, invoice, attempt, snapshot, renderer, content type, size, and any staging key.

The client renders only from that package and returns PDF bytes. It cannot supply or change the series, number, dates, tax, totals, wording, or NAV codes. Validate its identity, request scope, PDF type, size, and content checks. Store returned bytes without rewriting, optimizing, linearizing, stamping, metadata changes, or re-rendering. This is renderer output for a NAVRelay-numbered, locked invoice, not an imported invoice.

The handback flow is:

```text
POST /v1/invoices
  -> ISSUING invoice, final number, immutable render package, attempt token

PUT /v1/invoice-render-requests/{renderRequestId}/document
  -> validate exact PDF bytes
  -> finalize directly, or stage for explicit finalization

POST /v1/invoice-render-requests/{renderRequestId}/finalize
  -> verify a presigned staging upload and promote unchanged bytes
  -> canonical S3 object
  -> ISSUED
  -> NAV and delivery jobs

POST /v1/invoices/{invoiceId}/render-requests
  -> new attempt for the same locked number and snapshot
```

Each `InvoiceRenderRequest` is one attempt with state `ACTIVE`, `FINALIZING`, `SUPERSEDED`, `EXPIRED`, `FAILED`, or `CANONICAL`. An active attempt accepts repeated identical bytes with the same token until expiry. After canonicalization, an identical retry with a valid token returns the existing result and cannot write again; conflicting bytes fail. A new attempt locks the invoice, keeps the number, snapshot, and renderer-profile version, issues a new token, and atomically supersedes the old active attempt. Never supersede a finalizing attempt. A database guard lets only the current attempt finalize and canonicalize. Reject late callbacks from expired, failed, or superseded attempts.

An authorized status lookup and create-attempt operation recover a lost response or expired token without allocating another number. The renderer is NAVRelay's internal service or an approved, scoped `ApiClient` listed by the snapshotted `RendererProfile`. Store its identity and profile version on the attempt and document.

Every client-rendered attempt has a policy-set bounded expiry. Timeout, expiry, invalid PDF, or renderer failure leaves the invoice `ISSUING`, records an audit event, sets the document `FAILED`, and requires operator retry or review. It never becomes an editable draft. Keep its number, snapshot, attempts, incident, and audit history in invoice lists, sequence reports, operational exports, and retention controls. Legal resolution after the allowed window follows approved policy. Never roll back or reuse the number.

Client-renderer trust is explicit. Its authenticated handback attests that the PDF represents the bound snapshot, but generic PDF parsing cannot prove full semantic equality. Each `RendererProfile` names approved identities, template versions, deadlines, disclosure scope, attestation rules, and validation. If policy requires stronger machine-verifiable matching, use `NAVRELAY_RENDERED` or reject issuance.

If the approved electronic-invoice method requires the PDF hash in the NAV request, the renderer handback and NAV submission must satisfy the approved timing rule. Disable `TRUSTED_CLIENT_RENDERED` for any profile that cannot do so. Allocation may create a `NavSubmission` intent in `PENDING`, but not a runnable job. The canonicalization transaction inserts the NAV outbox row.

Never accept a standalone final PDF, caller-supplied invoice number, remote PDF URL, or unrestricted S3 key.

## 14. PDF validation

For trusted-client-rendered PDFs:

- Check PDF magic, actual content type, and size; reject encryption or passwords unless approved.
- Malware-scan public uploads behind the storage abstraction.
- Require an active organization-, invoice-, snapshot-, renderer-, and token-scoped attempt.
- Optional text extraction may reject obvious number, supplier, or total mismatches. The immutable snapshot remains authoritative; OCR never is.
- Store validation results, renderer identity, and profile version. Never modify canonical bytes; keep previews separate.

## 15. Hashing and integrity

Store SHA-256, algorithm, byte length, MIME type, S3 key, S3 version ID when available, upload and finalization times, and any required NAV electronic-invoice hash and algorithm. Internal SHA-256 differs from `electronicInvoiceHash`; the versioned NAV adapter derives the latter from the current schema, `completenessIndicator`, archival method, official docs, and test vectors. Periodically verify object bytes and report integrity.

## 16. S3 object storage

### Storage rules

- Keep buckets private. Separate environments by bucket or strict prefix and credentials; use server-side encryption and avoid personal data in keys.
- Never store durable invoices on local disk or Railway volumes.
- Enable versioning where available and prefer Object Lock or equivalent retention. If a provider cannot meet immutability needs, change it or approve a compensating control.
- Keep canonical metadata in PostgreSQL; S3 metadata is not the business record. Never overwrite a canonical key, and give each new legal document a new object and row.
- Use lifecycle rules only for temporary uploads or staging, previews, and expired exports; never for retained canonical invoices.
- Back up or replicate documents under the approved disaster-recovery policy.

Suggested layout:

```text
organizations/{organizationId}/
  invoices/{issueYear}/{seriesId}/{invoiceId}/
    canonical/invoice.pdf
    nav/submission-{attemptId}-request.xml
    nav/submission-{attemptId}-response.xml
    nav/submission-{attemptId}-result.xml
    metadata/invoice-snapshot.json
  exports/{exportId}/export.zip
  audit-manifests/{manifestId}.json
```

Use opaque internal IDs in keys. Show invoice numbers through metadata and the app, not as the only object identifier.

### Presigned URLs

- Authorize before issuing a short-lived URL. Bind uploads to one organization, invoice, snapshot version/hash, render request, renderer, content type, size, attempt token, and staging key; finalization must verify the staged object before unchanged promotion to a new canonical key.
- Bind downloads to one canonical object. Never expose permanent public URLs or store presigned URLs.

## 17. Retention and deletion

Store `retentionUntil` and policy version on invoices and objects. The initial Hungarian target is at least eight years; final duration requires legal approval and may be longer.

- UI and API calls cannot shorten retention.
- Organization deletion cannot remove retained documents; closure moves them to restricted archival state.
- Short lifecycles apply only to staging, failed staging, previews, and expired exports.
- Legal holds override deletion. Audit every deletion and fail closed when retention is uncertain.

## 18. NAV Online Számla

Implement NAV Online Számla as a versioned adapter, initially v3. Keep official environments configurable:

```text
Test frontend: https://onlineszamla-test.nav.gov.hu/
Test API:      https://api-test.onlineszamla.nav.gov.hu/invoiceService/v3
Production:    https://api.onlineszamla.nav.gov.hu/invoiceService/v3
```

Do not select the environment from a user-controlled request. Each organization has an explicit NAV environment. Never mix test and production credentials.

### Required operations

Support token exchange, `manageInvoice`, transaction-status lookup, approved correction, cancellation, or technical-annulment flows, optional taxpayer lookup, technical and business message parsing, and schema or capability reporting.

### Credentials

Each organization has its own NAV technical user.

- Encrypt credentials at rest through an application key-management abstraction.
- Allow decryption only in the worker or integration service that needs it.
- Never log passwords, signing or exchange keys, authorization material, or signatures.
- Rotate credentials without changing historical submissions.
- Keep test and production credentials separate.
- Offer a connection test that does not issue an invoice.

### Submission

```text
ISSUED invoice plus canonical document
  -> durable NAV job
  -> build and XSD-validate exact request
  -> get exchange token
  -> manageInvoice with stable operation identity
  -> store request, response, transaction ID, and times
  -> durable status polling
  -> parse technical and business messages
  -> ACCEPTED, ACCEPTED_WITH_WARNINGS, REJECTED, or MANUAL_REVIEW
```

A transaction ID is not final success.

### Retries

- Retry timeouts, resets, temporary NAV failures, and safe status queries with bounded backoff.
- Do not blindly retry permanent validation errors.
- Preserve one logical submission across retries.
- Store attempt count, next-attempt time, last error code, and normalized error category.
- Escalate after configurable time or attempt limits.
- Let operators retry after configuration fixes without changing the issued invoice.

### Stored NAV artifacts

Subject to privacy and retention policy, keep schema and adapter versions, normalized snapshot, exact outbound and response XML, transaction ID, final result, technical and business messages, request and correlation IDs, times, and attempt records. Never expose secrets or credential-derived fields through shares.

## 19. NAV schema, mapping, and compliance scope

Treat official XSD and interface docs as executable contracts. Vendor or update schemas through a controlled process; pin the version or checksum; XSD-validate outbound XML; use official examples and project golden fixtures; keep typed, versioned tax mappings; record the mapping-policy version per submission; keep NAV XML names out of controllers and entities; and separate normalized invoice data from transport DTOs. Schema upgrades require migration analysis, compatibility tests, and release notes.

`packages/nav-online-invoice` owns namespaces, schemas, XSD validation, fixtures, password hashing, request signatures and IDs, token exchange, `manageInvoice`, retry identity, status polling, response parsing, normalized messages, endpoints, and credential separation. Controllers, Stripe adapters, and domain services must not duplicate it.

Compliance covers VAT status and jurisdiction; taxable, exempt, out-of-scope, reverse-charge, and OSS cases; currency and exchange rates; advance and final invoices; correction, cancellation, replacement, and annulment chains; required wording and PDF content; rounding and summaries; electronic hashes; archival; statutory exports; and NAV or legal changes.

Maintain an explicit supported-scenario matrix. Add one case at a time, only with official fixtures, domain tests, NAV test-environment checks, and compliance approval.

## 20. Corrections, cancellations, and relations

Issued invoices are immutable. Model original, corrective or modifying, cancellation or storno, advance-to-final, and approved replacement or revision relations. Each new legal document gets a NAVRelay-managed number, canonical PDF, NAV submission, and explicit link to the original. Preserve the chain in the UI, API, and exports; never edit an issued row.

## 21. Public API design

Use versioned REST under `/v1` and generate OpenAPI from NestJS. Billingo `/v3` is a boundary adapter over the same application services, never a second engine or source of domain rules.

Baseline resources:

```text
/v1/organizations
/v1/members
/v1/api-clients
/v1/api-keys
/v1/nav-connections
/v1/invoice-series
/v1/invoice-drafts
/v1/invoices
/v1/invoices/{invoiceId}/document
/v1/invoices/{invoiceId}/nav-submissions
/v1/invoices/{invoiceId}/relations
/v1/invoices/{invoiceId}/render-requests
/v1/invoice-render-requests/{renderRequestId}
/v1/invoice-render-requests/{renderRequestId}/document
/v1/invoice-render-requests/{renderRequestId}/finalize
/v1/audit-shares
/v1/exports
/webhooks/stripe
```

Routes may evolve, but resource boundaries must stay clear.

### API requirements

- JSON except explicit uploads and downloads.
- Generated examples, error schemas, machine-readable codes, and localized messages.
- Cursor pagination and stable filters for date, series, number, customer, source, business state, and NAV state.
- Request and correlation IDs, versioning, and deprecation policy.
- No `any` request bodies.
- Native issuance DTOs contain no caller-controlled series or configuration, final number, sequence, or numbering mode. Compatibility block IDs may only resolve a preconfigured organization mapping.
- Derive tenant scope from authentication, never request content.

## 22. Idempotency

Require `Idempotency-Key` for caller writes that may issue a document, create a payment-linked invoice, start an export, or create a share. Store organization and principal, method and normalized route, key, request hash, status and result, safe response code and body, creation time, and expiry. The same hash returns the original result; a different hash conflicts.

Never persist plaintext render tokens or replay them. If an issuance response loses its token, create a new render attempt for the same locked invoice and number.

Enforce unique `(organizationId, stripeEventId)`, `(organizationId, stripeInvoiceId)`, `(organizationId, sourceSystem, sourceTransactionId)`, `(organizationId, invoiceNumber)`, and `(navConnectionId, logicalSubmissionId)`. Provider IDs are source and idempotency references only, never invoice numbers.

## 23. Stripe

Stripe is an optional adapter for payment state, source data, and tax evidence. It never owns a NAVRelay series, final number, or canonical PDF. Never fetch or import Stripe `invoice_pdf` or receipts as invoice documents.

### Issuer workflow

The initial Stripe workflow is `PAYMENT_TRIGGERED_NAVRELAY`:

1. Durably persist and deduplicate the event.
2. In the worker, fetch and verify authoritative payment and transaction data.
3. Map the facts through an approved tax profile.
4. Call the managed issuance service used by `/v1`.
5. NAVRelay resolves the server-side series, allocates the number, and uses `NAVRELAY_RENDERED` or `TRUSTED_CLIENT_RENDERED`.
6. Store Stripe IDs only as source, payment, reconciliation, and idempotency references.

Choose the tax source, series policy, and renderer mode explicitly per integration. Never infer them from event arrival order.

### Tax sources

1. `NAVRELAY_TAX_PROFILE`: enabled, versioned, and the default for the initial narrow Hungarian cases.
2. `STRIPE_TAX`: store the full calculation and evidence and map it only through an approved profile. It is not legal approval, tax registration or filing, or permission to infer unrestricted EU, OSS, exemption, or reverse-charge treatment.

Allow `STRIPE_TAX` only with managed issuance and tested profiles.

### Webhooks and event policy

Verify the raw body and signature before parsing, durably deduplicate before acknowledgement, and process fetched authoritative objects in the worker. Store applicable account, customer, Checkout Session, PaymentIntent, charge, subscription, invoice, credit note, tax calculation, and API-version IDs. Ignore Stripe invoice numbers and PDFs. Handle duplicates, retries, delayed methods, reordering, and bounded reconciliation without issuing twice.

Require verified payment success. For Checkout, retrieve the Session, verify payment status, support successful asynchronous methods, and uniquely link the Session and PaymentIntent.

For approved Stripe Billing flows:

- `invoice.paid` may trigger exactly one NAVRelay-managed invoice;
- `invoice.finalized` updates provider state or reconciliation only; it never imports or creates a canonical document;
- `invoice.payment_failed` updates payment state; it never issues or changes a canonical invoice;
- `invoice.voided`, `credit_note.created`, refunds, and disputes enter explicit correction or reconciliation flows and are not automatically Hungarian storno or modifying invoices.

Checkout and Billing events for one transaction must converge through source uniqueness and idempotency.

### Initial recommendation

Keep the Nuxt editor and trusted renderer as first-party API clients. Start with payment-triggered one-off issuance. Add subscription triggers or Stripe Tax only after source-data, tax, correction, and NAV tests plus professional compliance approval. Provider pricing stays outside the invoice domain, and configuration changes never rewrite history.

## 24. Audit log

Keep an append-only security and compliance audit log. Each event stores organization, actor type and ID, action, resource type and ID, time, request or correlation ID, applicable user agent, redacted configuration before and after data, outcome, and reason or source.

Log successful and failed login; membership and role changes; API-key creation, rotation, and revocation; NAV credentials and connection tests; invoice lifecycle and forbidden edits; render requests, handbacks, canonicalization, integrity checks, and downloads; NAV attempts and results; share creation, access, expiry, and revocation; export creation and download; retention; and deletion. Application logs do not replace this log.

## 25. Audit access by login

Authenticated Better Auth organization members with auditor access can filter by date, number, series, source, customer, business state, and NAV state; view or download canonical PDFs; view normalized data and safe NAV results; follow relations; and access permitted exports, hashes, integrity status, and, where authorized, access history.

Auditors cannot modify invoices, series counters, NAV credentials, or canonical documents.

## 26. Audit access by share link

Support revocable accountless links. `AuditShare` stores organization, creator, immutable scope, optional date, number, or series limits, allowed fields and actions, expiry, optional password hash and download limit, revocation, access times, and access log.

Use high-entropy opaque tokens stored only as hashes and shown once; support immediate revocation and short expiry; set `noindex` and a restrictive referrer policy; rate-limit token and password attempts; never expose S3 credentials, secret NAV requests, or unrestricted organization data; and authorize short-lived object URLs after scope checks. Shares do not replace statutory exports.

## 27. Exports

Exports are asynchronous immutable snapshots. Support the required Hungarian statutory format, selected PDFs with safe metadata, and explicitly implemented accounting CSV or XML formats.

Example:

```text
export.zip
├─ manifest.json
├─ invoices/{invoiceId}.pdf
├─ metadata/invoices.jsonl
├─ metadata/relations.jsonl
├─ nav/results.jsonl
└─ nav/safe-results/
```

The manifest records export ID, organization, filter snapshot, creation time, creator or share identity, invoice IDs and numbers, hashes and sizes, app version, and format version. Only temporary export objects may expire, on a configurable schedule; source invoices and canonical documents may not.

## 28. Email

Use ZeptoMail behind a backend abstraction. Follow organization link or attachment policy, localize templates, use a download endpoint or short-lived signed URL, record attempts and provider IDs, and retry without regenerating or replacing PDFs. Email is delivery, not archival storage.

## 29. Privacy and security

Invoice data is sensitive. Validate inputs; encrypt secrets and use TLS; keep S3 private with least-privilege credentials; hash API keys, share tokens, and render tokens; redact authorization headers and cookies, passwords, keys, signatures, raw invoice payloads, and customer PII from logs; rate-limit public and machine endpoints and protect login and forms; verify webhooks; allowlist provider hosts, prevent SSRF, and enforce fetch size and content-type limits; reject remote PDFs and accept renderer bytes only through scoped requests; malware-scan public uploads; reauthorize downloads; distrust browser totals, tax, roles, and ownership; document hosted-SaaS processing roles; and support data-subject requests without breaking retention.

## 30. Background jobs

Use a durable queue with safe multi-worker claims, retries, scheduling, and dead-letter or manual-review states. PostgreSQL is the default unless Redis is needed.

Required jobs include NAVRelay PDF rendering; trusted-client render dispatch, timeout, retry, and returned-PDF verification and canonicalization; NAV submission and status lookup; email; exports; integrity checks; staging and export expiry; stuck-workflow reconciliation; and Stripe synchronization.

The API may enqueue but must not run an untracked memory queue. Worker loops live in `apps/worker` and may keep that service awake.

## 31. Transactional outbox

When a database change requires later work, commit the domain change and job or outbox row in one PostgreSQL transaction. This includes number and snapshot allocation plus a render job or request, a received Stripe event plus invoice-processing work, and a final NAV result plus delivery or notification.

No transaction spans PostgreSQL, S3, and NAV. Upload, validate, hash, and promote unchanged bytes to a new canonical S3 key first. Then, in one PostgreSQL transaction, lock the invoice and active render attempt, record canonical object metadata, move the invoice to `ISSUED`, and insert the NAV outbox row. NAV calls happen later. Reconciliation must recover an orphaned S3 object or an interrupted database finalization without replacing bytes or submitting early.

A crash after commit must not lose work. A retry must not duplicate the domain action.

## 32. Frontend

Use Nuxt, Nuxt UI, Tailwind, and Nuxt i18n. Initial areas:

- organization switcher, dashboard, and alerts;
- invoices, details, drafts, and `NAVRELAY_RENDERED` or `TRUSTED_CLIENT_RENDERED` flows;
- series, blocks, customers, members, roles, API clients, keys, renderer profiles, and trusted identities;
- NAV connection and status, Stripe, audit logs and shares, exports, storage and integrity, and organization and retention settings.

Translate all user text, starting with English and Hungarian and using English fallback unless the repository changes this. Use generated business clients. Client code must not duplicate authorization or legal rules.

## 33. OpenAPI and SDKs

NestJS controllers and DTOs define OpenAPI. Commit the generated document per repository convention; generate the Nuxt client with `nuxt-open-fetch`; keep generated code separate and unedited; publish a TypeScript SDK only after API stabilization; review compatibility and release notes for public changes; and make `pnpm run verify` detect contract drift.

## 34. Database

Use MikroORM's Data Mapper, Identity Map, and Unit of Work with PostgreSQL integration tests, generated migrations for every schema change, and production schema synchronization disabled. Use explicit transactions for issuance, allocation, outbox creation, and other atomic work; prefer `EntityManager` and repositories. Limit raw SQL or QueryBuilder to narrow, documented concurrency needs such as `FOR UPDATE SKIP LOCKED`. Map entities to public DTOs; never return raw entities.

## 35. Testing

Prefer behavior and integration tests over isolated mocks.

### Mandatory critical tests

- Better Auth organization creation, invitation acceptance, multi-organization switching, scoped roles, and retained-data deletion blocks.
- Valid scoped API keys work; invalid, expired, revoked, or under-scoped keys fail. Machine writes enforce rate limits, idempotency, and tenant isolation.
- Stripe signature failures, duplicate or reordered events, and source-transaction deduplication converge to one managed invoice; `invoice.finalized` alone never issues, and invalid source or tax data fails closed.
- Managed allocation is concurrency-safe; native requests cannot supply series or configuration, numbering mode, sequence, or final number; compatibility block IDs resolve only preconfigured organization series; every number belongs to an authorized managed series; identical retries return the same invoice.
- Render packages contain the server number and immutable invoice and series snapshots. Attempt tokens are shown once, hashed, unlogged, retry-safe, and scoped to organization, invoice, snapshot, renderer, content, size, and expiry.
- Renderer callbacks are idempotent: identical bytes return the original result, conflicting bytes fail, and lost responses or expired tokens recover without another number. A replacement attempt atomically supersedes the old active attempt. Superseded, expired, failed, cross-tenant, and late callbacks fail; only one attempt finalizes.
- Invalid, encrypted, or snapshot-mismatched PDFs fail closed. Renderer identity, profile version, trust boundary, and required validation are enforced. Timeout or failure is audited, stays visible for sequence review, never releases the number, and retries only the same snapshot.
- Cross-tenant invoice or document access fails. Issued data and canonical objects cannot be edited or replaced. Hashes match exact generated or renderer-returned bytes, and trusted-client PDFs stay byte-for-byte unchanged.
- Presigned staging cannot canonicalize before finalization, which verifies the exact staged bytes. S3 failure leaves recoverable state, not false issuance or delivery.
- Canonical metadata, `ISSUED`, and the NAV outbox commit atomically after S3 canonicalization; NAV cannot start before canonical bytes and required hashes exist, and no NAV call runs in that transaction.
- NAV temporary failures retry durably; permanent errors enter manual review without duplicate issuance; transaction IDs are not final acceptance.
- Audit-share scope, expiry, password, and revocation; export manifests; correction chains; retention blocks deletion; and log redaction work.
- OpenAPI, generated clients, and docs expose no external-number or finalized-document import path.

### Test tools

Use Vitest; Supertest and `@nestjs/testing`; Testcontainers PostgreSQL; MinIO or another real S3-compatible service; NAV, Stripe, email, and S3 adapter mocks; official NAV XML/XSD fixtures and project golden files; opt-in NAV test-environment contract tests; and Playwright on Chromium, Firefox, and WebKit with isolated mutating-test lanes.

## 36. Verification

The root must expose `pnpm run verify`, covering lint and format, TypeScript, backend and frontend tests, PostgreSQL and S3 integration, concurrency and idempotency, OpenAPI drift, locale completeness, production builds, and Playwright E2E. Claim full verification only when it passes; report skipped or blocked suites exactly.

### GitHub Actions gate

Create `.github/workflows/verify.yml` with stable job and check name `verify`. Run it on `pull_request`, protected-default-branch pushes, and `merge_group` when used. Use read-only permissions, pinned Node and pnpm, `pnpm install --frozen-lockfile`, required Playwright dependencies, and unchanged `pnpm run verify` on GitHub-hosted Linux with Docker and Testcontainers. Never expose organization or production secrets to PR code. Mock NAV, Stripe, email, and S3 at adapter boundaries in PRs; run live NAV checks separately through trusted `workflow_dispatch` or scheduled protected-branch workflows with test credentials, never untrusted forks. Cancel superseded runs, require `verify` on the default branch, and gate Railway auto-deploys to the verified commit SHA.

## 37. Railway

Use private networking between Nuxt, NestJS, worker, and PostgreSQL; run MikroORM migrations pre-deploy, not every start; use bounded database wake-up retries, small pools, finite idle timeouts, no keepalive for sleeping services, durable worker polling and schedules, readiness endpoints without per-probe database queries, and server-side Railway variables or an approved secret manager. Never upload a local working tree as the production deployment path.

## 38. Observability

Use structured JSON logs in production and readable development logs. Include service, environment, request or correlation ID, safe organization and resource IDs, job ID, safe NAV transaction ID, outcome, duration, attempt, and next retry. Never log credentials, secrets, keys, tokens, authorization headers or cookies, full invoice or customer bodies, PDF bytes, or unredacted authenticated NAV requests. Alert on stuck `ISSUING` invoices; stuck, failed, or expired render requests; missing canonical documents; stuck or rejected NAV jobs; expiring or invalid credentials; failed email; S3 integrity failures; dead letters; sequence conflicts; and abnormal duplicate or idempotency conflicts.

## 39. Self-hosted and hosted editions

Use the same tested core for self-hosted single-organization, self-hosted multi-organization, and hosted SaaS. Entitlements may differ; issuance, hashing, storage, NAV mapping, tenancy, and audit behavior may not. Self-hosting accepts a PostgreSQL URL; S3 endpoint, region, bucket, and credentials; public web and API URLs; Better Auth secret and trusted origins; encryption key or KMS; NAV environment; email provider; and optional Stripe settings. Never fall back to local durable storage when S3 fails.

## 40. Commercial and open-source constraints

Keep core features provider-neutral, open-source, and viable for hosting. Avoid licenses that block distribution or hosting and record third-party licenses; do not choose the project license until the owner decides; keep AGPL, open-core, and dual licensing possible; and do not depend on undocumented private hosted-only features.

## 41. Initial non-goals

Do not add these without an explicit scope change:

- general-ledger accounting, payroll, inventory, or warehouse management;
- banking reconciliation beyond payment references;
- tax-adviser marketplaces, unsupported country regimes, or inferred tax logic replacing an accountant;
- externally numbered or finalized third-party invoice imports, caller-assigned numbers or series, or Stripe PDFs or receipts as canonical documents;
- arbitrary PDF editing or OCR as legal truth;
- browser access to user S3 credentials.

## 42. Initial product slice

The first production-capable slice includes:

1. Better Auth organizations, invitations, memberships, static roles, and separate machine keys.
2. NAVRelay-managed series with atomic server allocation, `NAVRELAY_RENDERED`, and `TRUSTED_CLIENT_RENDERED` handback through scoped renderer profiles, requests, and tokens.
3. Private S3 canonical storage, hashes, and integrity checks.
4. Per-organization NAV test and production connections, durable submission, and final status.
5. Invoice lists, details, downloads, and NAV diagnostics.
6. Login-based auditor access, scoped expiring shares, and manifest-backed date or number exports.
7. One approved Stripe-triggered managed-issuance flow with durable events and reconciliation.
8. Public Swagger/OpenAPI, scoped API keys, rate limits, English and Hungarian UI/API messages, and full critical-path tests with mandatory GitHub Actions `verify`.

It is not production-ready until legal and compliance sign-off and live test invoices are validated in NAV's test environment.

## 43. Invoice issuance definition of done

An issuance feature is done only when:

- authorization, organization scope, typed DTO validation, and exact totals work;
- idempotency and NAVRelay-managed allocation work; native callers cannot supply series identifiers or configuration, sequences, modes, or final numbers, and compatibility blocks only use preconfigured mappings;
- the immutable snapshot, canonical S3 PDF, exact hash, and storage metadata exist and normal APIs cannot replace them;
- trusted-client rendering, when used, is identity- and token-scoped, idempotent, and retryable only against the locked snapshot and number;
- a durable NAV job or approved no-report policy exists; NAV state is visible and diagnosable, audit events exist, and applicable delivery is queued;
- OpenAPI and client output are current, and targeted tests plus full `pnpm run verify` pass.

## 44. Prohibited shortcuts

Never:

- hide Billingo or Számlázz.hu behind NAVRelay;
- create an external series, accept a caller- or provider-assigned number, or import an issued invoice or standalone finalized PDF;
- store canonical PDFs in Stripe, PostgreSQL blobs, local disk, or Railway volumes; replace or edit them; reuse numbers; or allocate outside a safe transaction;
- mark NAV success at transaction ID or poll NAV in the request process;
- accept arbitrary remote PDF URLs, Stripe receipts or invoice PDFs as canonical documents, or bytes outside an active scoped `TRUSTED_CLIENT_RENDERED` request;
- expose public S3 buckets or permanent URLs, or treat S3 ETags as legal or integrity hashes;
- log raw secrets or invoice bodies, rely on frontend authorization, or guess tax;
- silently change NAV mappings or schema versions, or delete retained invoices;
- weaken or make `verify` optional, or expose Better Auth or control routes on the machine-API host;
- treat Stripe Tax as legal approval, or map Stripe voids, credit notes, refunds, or disputes directly to Hungarian corrections or storno.

## 45. Agent workflow

Before changing code:

1. Read this file and `defaultstack.md`.
2. Map affected invariants, tenants, documents, NAV mappings, and retention rules.
3. Check current official NAV or Stripe documentation.
4. Add or update generated MikroORM migrations for schema changes.
5. Test success, duplicates, retries, authorization, and failure recovery.
6. Regenerate OpenAPI and client artifacts after API changes.
7. Run targeted tests, then unchanged `pnpm run verify`; keep the PR's `verify` check green.
8. Report legal assumptions, missing specifications, unverified provider behavior, and blocked tests.

When uncertain, fail closed: do not issue, expose, overwrite, or delete a legal document on a guess.

## 46. Official implementation references

Check current versions and these official references before implementation:

- NAV Online Számla repository: https://github.com/nav-gov-hu/Online-Invoice
- NAV production portal: https://onlineszamla.nav.gov.hu/
- NAV test portal: https://onlineszamla-test.nav.gov.hu/
- Hungarian invoice-program regulation: https://njt.hu/jogszabaly/2014-23-20-2X
- NAV electronic-invoice guidance: https://nav.gov.hu/ado/afa/Az_elektronikus_szaml20200416
- Better Auth Organization plugin: https://better-auth.com/docs/plugins/organization
- Railway public-networking headers: https://docs.railway.com/networking/public-networking/specs-and-limits
- Stripe Invoice API: https://docs.stripe.com/api/invoices
- Stripe invoice lifecycle: https://docs.stripe.com/invoicing/integration/workflow-transitions
- Stripe webhooks: https://docs.stripe.com/webhooks
- Stripe Checkout invoice creation: https://docs.stripe.com/api/checkout/sessions/create
- Stripe Tax: https://docs.stripe.com/tax

Each production release must pin and document its NAV schema, Stripe API version, renderer version, and mapping-policy version.
