# AGENTS.md — NAVRelay

Last updated: 2026-08-24

## 1. Purpose

NAVRelay is an open-source, API-first invoicing platform for Hungarian businesses.

The product should feel like an API-first Billingo or Számlázz.hu alternative, while remaining auditable, self-hostable, and suitable for a hosted multi-tenant SaaS offering. It must issue or ingest invoices, store the canonical invoice document in S3-compatible object storage, submit the required invoice data directly to the Hungarian NAV Online Számla API, and expose invoices for authorized review through either authenticated user access or revocable audit links.

The system must support both of these document paths:

1. **Generated document:** NAVRelay allocates the invoice number and generates the final PDF.
2. **Provided document:** an external system supplies an already-finalized PDF and its invoice data; NAVRelay stores the exact bytes, reports the invoice to NAV, and provides the same audit and export facilities.

Stripe is an important integration, but NAVRelay must not be designed as a Stripe-only application. Stripe invoices are one possible external document source. Stripe receipts are not accepted as invoice documents.

### Product position and go/no-go rule

NAVRelay is not justified merely as a way for one low-volume company to avoid a small Billingo or Számlázz.hu subscription. It is justified when one or more of these are product requirements:

- open-source and self-hosted deployment;
- a stable developer-first REST API rather than a proprietary invoicing UI;
- customer-owned S3 storage and reproducible audit exports;
- direct NAV connectivity without a commercial invoicing intermediary;
- multi-organization SaaS operation with human and machine access;
- reusable Stripe and other payment-provider adapters;
- transparent, versioned tax/NAV mappings and an auditable processing trail.

Do not attempt to reproduce every Billingo feature. The initial product is a focused invoicing, archival, NAV-reporting, and audit platform. General accounting, banking, inventory, and broad ERP features remain outside the core thesis.

## 2. Product principles

The following principles are mandatory:

- **API first:** every important business operation is available through a documented public API before it is exposed in the web UI.
- **Direct NAV integration:** do not depend on Billingo, Számlázz.hu, or another invoicing SaaS for NAV reporting.
- **Canonical immutable document:** every issued invoice has exactly one canonical document whose exact bytes are preserved.
- **S3 is the primary document store:** the database stores metadata and object references; durable invoice documents do not live on an application filesystem or Railway volume.
- **Multi-tenant and multi-user:** one user may belong to multiple organizations, and one organization may contain multiple users and machine clients.
- **Auditable by design:** issued data is append-only, every privileged action is logged, exports are reproducible, and access can be granted through a login or scoped share link.
- **No silent legal assumptions:** tax treatment, mandatory invoice content, NAV mappings, archival mode, correction rules, and statutory exports are compliance-critical. Implement them from current official specifications and approved product policy, not from guesswork.
- **No destructive correction:** an issued invoice is never edited in place. Corrections, cancellations, and reversals create new linked documents.
- **Reliable asynchronous reporting:** an invoice is not considered successfully reported merely because NAV returned a transaction ID. Processing must continue until the final NAV result is known.
- **Idempotency everywhere:** duplicate API calls, duplicate Stripe webhooks, worker retries, and process restarts must not create duplicate invoices or duplicate NAV submissions.

## 3. Instruction priority

When instructions conflict, apply them in this order:

1. Current Hungarian law and current official NAV specifications.
2. Explicit project decisions in this file.
3. The repository's current implementation and migrations.
4. `defaultstack.md` and the user's default-stack conventions.
5. Framework defaults and third-party examples.

Never weaken an invoice immutability, authorization, tenancy, idempotency, or audit requirement merely to simplify an implementation.

## 4. Legal and compliance boundary

This file is an engineering specification, not a legal opinion.

Before a production launch, the owner must obtain written validation from a Hungarian accountant, tax adviser, or tax lawyer for at least:

- supported invoice types and tax scenarios;
- mandatory invoice fields and wording;
- VAT and exemption mappings;
- EU B2B reverse-charge behavior;
- EU B2C and OSS handling;
- foreign-currency invoices and HUF VAT presentation;
- advance, final, corrective, and cancellation invoices;
- the selected electronic-invoice authenticity and integrity method;
- the exact NAV `electronicInvoiceHash` behavior used by the implementation;
- statutory retention duration and deletion policy;
- the required “adóhatósági ellenőrzési adatszolgáltatás” export format;
- whether a configured Stripe-generated PDF satisfies every mandatory requirement for each enabled scenario.

Coding agents must not claim that any arbitrary Stripe PDF is automatically a valid Hungarian invoice. The system may accept it only through an explicitly supported and tested external-document profile.

## 5. Default technology stack

Follow the repository-pinned versions, but use the user's default stack unless this file overrides it.

### Core stack

- TypeScript across frontend, backend, worker, and shared packages.
- Modern Node.js with native ESM.
- pnpm, pinned through `packageManager`.
- Nuxt with Vue Composition API and SSR.
- Nuxt UI and Tailwind CSS.
- NestJS backend organized by business domain.
- Swagger/OpenAPI generated from NestJS controllers and DTOs.
- `nuxt-open-fetch` generated client for frontend business APIs.
- `class-validator` DTO validation on every write boundary.
- Railway PostgreSQL.
- MikroORM with generated migrations.
- Better Auth with cookie-based human sessions and the official Organization plugin for organizations, memberships, invitations, active-organization state, and organization-scoped access control.
- Nuxt i18n and request-aware backend localization.
- Vitest, Supertest, Testcontainers, and Playwright.
- ZeptoMail through a backend email abstraction.
- S3-compatible object storage as the mandatory primary document store.

### Deployment defaults

Use separate Railway services for:

1. Nuxt web application.
2. NestJS API.
3. PostgreSQL.
4. Durable background worker.

The worker is separate because NAV submission, status checks, document processing, exports, and email delivery must not run as untracked background work inside the web process.

GitHub Actions CI is mandatory for this project. Every pull request and every push to the protected default branch must run the repository root `pnpm run verify` unchanged. The stable `verify` check is a required merge check. Production deployment should be gated on that same commit SHA when Railway automatic deployment is enabled.

## 6. Repository layout

Prefer a pnpm workspace with the following conceptual layout:

```text
NAVRelay/
├─ apps/
│  ├─ web/                 # Nuxt user, admin, and audit UI
│  ├─ api/                 # NestJS public and internal API
│  └─ worker/              # Durable jobs: NAV, PDF, exports, email
├─ packages/
│  ├─ nav-online-invoice/  # NAV v3 transport, XML, signing, parsing
│  ├─ invoice-domain/      # Shared domain values and pure calculations
│  ├─ pdf-renderer/        # Deterministic invoice rendering
│  ├─ storage/             # S3 abstraction and integrity helpers
│  ├─ api-client/          # Generated output only, if kept as a package
│  └─ test-fixtures/       # Official-schema and project fixtures
├─ docs/
│  ├─ compliance/
│  ├─ architecture/
│  └─ operations/
├─ .github/
│  └─ workflows/
│     ├─ verify.yml         # required PR and default-branch gate
│     └─ nav-contract.yml   # trusted/manual or scheduled NAV test contract run
├─ defaultstack.md
├─ AGENTS.md
├─ pnpm-workspace.yaml
└─ package.json
```

Do not duplicate business rules between `apps/api` and `apps/worker`. Domain services may be shared, but HTTP controllers, worker consumers, and provider adapters remain separate entry points.

## 7. High-level architecture

```text
Public browser                    Machine/API client                    Stripe
     |                                  | API key + scopes                 | signed webhook
     v                                  v                                  v
Nuxt web application          api.example.com/v1/*           api.example.com/webhooks/stripe
     |                                  |                                  |
     | Railway private network          +---------------+------------------+
     v                                                  v
NestJS control routes                            NestJS API
                                                       |
                         api.example.com/docs ----------+  Swagger UI
                    api.example.com/openapi.json -------+  OpenAPI contract
                                                       |
                                                       +---- PostgreSQL: domain state, jobs, audit metadata
                                                       |
                                                       +---- S3: canonical PDFs, XML, receipts, manifests, exports
                                                       |
                                                       v
                                                Durable job queue
                                                       |
                                                       v
                                                  Worker service
                                                  |     |      |
                                                  |     |      +---- ZeptoMail
                                                  |     +----------- S3
                                                  +----------------- NAV Online Számla v3
```

The NestJS API is the authority for business rules, authorization, invoice issuance, tenancy, and API idempotency. The worker performs retriable external work. S3 is the authority for canonical document bytes, while PostgreSQL is the authority for document metadata, state, relationships, and audit history.

## 8. Tenant and user model

NAVRelay is multi-tenant from the first migration.

### Required entities

Authentication and human organization management use Better Auth's runtime model, mirrored exactly in MikroORM migrations:

- `User`
- `Session`
- `Account`
- `Verification`
- `Organization`
- `OrganizationMembership` / Better Auth `Member`
- `OrganizationInvitation`

NAVRelay-owned domain entities include at least:

- `ApiClient`
- `ApiKey`
- `NavConnection`
- `InvoiceSeries`
- `TaxProfile`
- `Invoice`
- `InvoiceLine`
- `InvoiceDocument`
- `InvoiceRelation`
- `PaymentReference`
- `NavSubmission`
- `NavSubmissionMessage`
- `StripeIntegration`
- `StripeEvent`
- `AuditEvent`
- `AuditShare`
- `ExportJob`
- `StoredObject`
- `BackgroundJob`, or an equivalent durable queue model

A user may belong to more than one organization. Every tenant-owned row must contain an explicit `organizationId`, either directly or through an unavoidable parent relation.

### Better Auth organization ownership

- Enable Better Auth's `organization()` server plugin and `organizationClient()` frontend plugin.
- Better Auth owns runtime writes for users, sessions, organizations, members, invitations, active-organization state, and member-role assignment.
- MikroORM owns all DDL and migration history. Mirror every enabled Better Auth and Organization-plugin table, column, index, unique constraint, and plugin field exactly; Better Auth must not auto-migrate production.
- Define the initial NAVRelay organization roles and permissions statically in typed code through Better Auth's access-control API. Do not create parallel `Role` and `Permission` tables for the initial release.
- Dynamic per-organization roles are disabled initially. If enabled later, adopt Better Auth's dynamic access-control model and mirror its organization-role table instead of inventing a competing authorization store.
- Machine clients and API keys remain NAVRelay domain entities and never masquerade as Better Auth users or organization members.
- Organization deletion through a generic Better Auth endpoint must be intercepted or disabled. An organization with retained invoices must enter an archival/closed state rather than cascading deletion of legally retained domain records.

### Human roles

Start with these roles:

- `owner`: organization ownership, billing, credentials, members, all data.
- `admin`: organization settings, users, integrations, invoice operations.
- `accountant`: invoices, NAV results, exports, corrections, customer data.
- `operator`: create drafts and issue supported invoices, without credential administration.
- `auditor`: read-only access to invoices, documents, NAV results, and exports.
- `viewer`: limited read-only operational access.

Roles are organization-scoped. Do not store a universal role directly on the user as the authorization source.

### Machine access

Machine clients are not fake users. Use separate API clients and API keys with:

- organization scope;
- explicit permission scopes;
- hashed tokens at rest;
- identifiable token prefix;
- creation, rotation, last-used, expiry, and revocation timestamps;
- configurable rate limits and usage accounting;
- complete audit logging;
- no ability to retrieve the original secret after creation.

API keys are the primary machine-authentication mechanism. Do not rely on network location or obscurity for access control.

## 9. Authorization and tenant isolation

Every query and mutation must enforce organization ownership on the backend.

Mandatory rules:

- Never trust `organizationId` merely because the caller supplied it.
- Resolve allowed organizations from the authenticated human membership or API-key principal.
- Never query an invoice, series, document, customer, export, or audit share by global identifier without also enforcing organization scope.
- Prefer repository/service methods that require an organization context.
- Prevent cross-tenant object-key access in S3.
- Presigned URLs must be generated only after backend authorization.
- An auditor share may expose only the resources captured by its immutable scope.
- Include explicit cross-tenant negative tests for every important resource.

## 9A. API exposure and product surfaces

NAVRelay is public and API-first. Authentication and authorization protect operations; the public existence of the API and its documentation is not considered a security weakness.

### Public surfaces

- Expose the versioned machine API under `/v1/*` on a public HTTPS endpoint.
- Publish the generated OpenAPI document at `/openapi.json` and an interactive Swagger UI at `/docs`.
- Require API-key authentication, organization scope, permission scopes, rate limiting, and idempotency for machine writes.
- The Nuxt application is a first-party client of the same business API. It provides human workflows, including an invoice editor, organization administration, invoice review, audit access, and exports.
- Browser calls for the owned UI may use same-origin Nuxt server routes/BFF endpoints and Railway private networking, but they must not bypass NestJS business rules or authorization.
- Stripe and other provider webhooks use narrowly scoped public webhook routes with provider-specific signature verification and durable event deduplication.
- Internal worker, queue-administration, migration, and maintenance endpoints must not be publicly exposed.

One NestJS codebase may serve the session-based control plane, documented machine API, and webhook routes. Keep their guards, DTOs, and route namespaces explicit, while sharing the same domain services and persistence model.

### Core input workflows

The product exposes three entry workflows over one invoicing engine:

1. **Managed issuance through API or invoice editor**
   - The caller or Nuxt editor supplies structured invoice data.
   - NAVRelay validates the selected tax profile, allocates a number from a managed series, renders the canonical PDF, stores it in S3, and submits the normalized data to NAV.

2. **Finalized document import through API**
   - The caller supplies structured invoice data plus an already-final PDF and an externally allocated invoice number.
   - NAVRelay validates the enabled external-document profile, stores the exact PDF bytes in S3 without re-rendering, and submits the same normalized invoice data to NAV.

3. **Provider-driven workflow through signed webhooks**
   - A payment or billing provider event invokes either managed issuance or finalized-document import.
   - Provider adapters must never implement a second invoicing engine; they translate authoritative provider state into one of the two core workflows.

The invoice editor is optional product UI, not a separate issuance mode. Headless deployments may use only the Swagger-documented API and webhooks.

## 10. Invoice-series and invoice-block model

“Invoice block” and “invoice series” refer to the numbering authority and sequence from which invoice numbers are allocated.

### Series modes

Support two explicit modes:

1. `MANAGED`
   - NAVRelay owns number allocation.
   - Used primarily with NAVRelay-generated PDFs.
   - The next number is allocated atomically in PostgreSQL.

2. `EXTERNAL`
   - Another system already assigned the final invoice number.
   - Used for supplied PDFs, including supported Stripe Invoice profiles.
   - NAVRelay must not allocate a second number.
   - NAVRelay enforces uniqueness and records observed sequence information, but the external issuer remains responsible for its numbering behavior.

Do not mix managed and external numbering in one series.

### Series fields

At minimum, model:

- organization;
- internal name;
- public prefix or number format;
- numbering mode;
- current or next sequence value for managed series;
- optional year-reset policy;
- document type policy;
- currency restrictions, if configured;
- active/inactive state;
- first and last issue timestamps;
- immutable configuration snapshot or version history;
- audit metadata.

### Managed allocation

Allocate a managed invoice number only inside the same database transaction that creates the issued invoice record.

Use a database row lock, atomic update, or another PostgreSQL-safe serialization mechanism. Concurrency must never produce duplicate numbers. Do not preallocate a number in a long-lived draft.

The conceptual transaction is:

```text
begin
  lock invoice series
  validate that it is active and compatible
  allocate next sequence
  create issued invoice with unique final number
  increment series counter
  append audit event
commit
```

External calls, PDF rendering, S3 upload, NAV requests, and email must not run while this database transaction is open.

### Series immutability

After the first invoice has been issued from a managed series:

- do not change its numbering mode;
- do not silently change its prefix or formatting rules;
- do not decrement or reuse sequence values;
- do not delete it;
- deactivate it instead;
- preserve all historical settings required to reconstruct issued numbers.

## 11. Invoice aggregate

An invoice is a domain aggregate, not merely a PDF or a NAV XML document.

At minimum, preserve an immutable issued snapshot of:

- organization/supplier identity and tax details;
- customer identity and tax details required for the scenario;
- invoice number;
- issue date and time;
- performance date;
- payment due date, where applicable;
- currency;
- exchange-rate information, where applicable;
- line items;
- quantities and units;
- net, tax, and gross amounts;
- discounts and adjustments;
- tax category and legal reason codes;
- payment method/reference;
- language;
- notes and mandatory wording;
- original/correction/cancellation relationships;
- source system and source identifiers;
- canonical document metadata;
- NAV reporting state.

Use exact decimal arithmetic or integer minor units as appropriate. Never use JavaScript floating-point arithmetic for monetary totals or tax calculations.

### Tax-determination boundary

The initial release is not a general-purpose autonomous tax adviser.

- Each issued invoice must use an explicit, versioned `TaxProfile` approved for the organization and transaction type.
- Public callers select from enabled profile identifiers and provide the required facts; they do not submit arbitrary NAV XML codes or free-form tax conclusions.
- A tax profile defines supported seller/customer VAT status, jurisdiction, VAT rate or exemption, required evidence, mandatory wording, rounding, currency/exchange-rate behavior, and NAV mapping policy.
- Begin with a deliberately narrow capability matrix. Unsupported or ambiguous combinations fail closed or enter pre-issuance manual review.
- Stripe Tax or another calculator may provide a calculation result and evidence, but NAVRelay validates that result against an enabled profile and remains responsible for the invoice snapshot and NAV mapping. Do not treat a provider's result as automatic Hungarian legal approval.
- Persist the tax-profile version, external calculation identifier, relevant input facts, result, and mapping-policy version for every issued invoice.

## 12. Invoice lifecycle

Use explicit states. Do not overload one status field for document, business, NAV, and delivery state.

### Business state

A useful baseline is:

- `DRAFT`
- `ISSUING`
- `ISSUED`
- `CORRECTED`
- `CANCELLED`

### Document state

- `MISSING`
- `STAGED`
- `GENERATING`
- `CANONICAL`
- `FAILED`

### NAV state

- `NOT_REQUIRED`
- `PENDING`
- `SUBMITTING`
- `PROCESSING`
- `ACCEPTED`
- `ACCEPTED_WITH_WARNINGS`
- `REJECTED`
- `RETRY_REQUIRED`
- `MANUAL_REVIEW`

### Delivery state

- `NOT_REQUESTED`
- `QUEUED`
- `SENT`
- `FAILED`

Once `ISSUED`, business data and the canonical document are immutable. A failure to report to NAV does not make the invoice disappear; it creates an operational incident that must be retried or reviewed.

## 13. Document modes

Every issued invoice has exactly one canonical `InvoiceDocument`.

### 13.1 Generated PDF

For `GENERATED` mode:

1. Validate the invoice draft and selected managed series.
2. Allocate the final number and create the immutable issued snapshot.
3. Render the PDF from that snapshot.
4. Compute hashes from the exact rendered bytes.
5. store the exact bytes in S3;
6. mark the stored object canonical;
7. enqueue NAV reporting;
8. enqueue delivery only after the document is canonical.

The renderer must be deterministic enough that the same immutable input and renderer version can be investigated later, but the system must never regenerate a canonical document and silently replace it.

Store the renderer name, renderer version, template version, locale, and any relevant font/template asset version as metadata.

### 13.2 Provided PDF

For `PROVIDED` mode:

- the caller supplies a final PDF and all normalized invoice data;
- the caller supplies the final external invoice number;
- the invoice uses an `EXTERNAL` series;
- NAVRelay calculates hashes from the exact received bytes;
- NAVRelay stores those exact bytes without rewriting, optimizing, linearizing, stamping, or re-rendering them;
- NAVRelay validates file type, size, upload ownership, and configured source profile;
- NAVRelay never treats an S3 ETag as a cryptographic document hash;
- the canonical object is immutable after issuance.

For large or direct uploads, use a staged presigned-upload flow:

```text
POST /v1/document-uploads
  -> returns scoped upload URL and upload token

client uploads PDF to staging key

POST /v1/invoices
  -> references upload token and external invoice number
  -> worker/API verifies bytes, hashes, and promotes to canonical storage
```

A caller must never be allowed to supply an unrestricted S3 key.

### 13.3 Stripe document profile

Stripe support is an adapter over the provided-document flow.

- Accept a Stripe `invoice_pdf` only from an authenticated Stripe webhook/import flow or a server-side allowlisted fetch.
- Verify the Stripe webhook signature.
- Deduplicate by Stripe event ID and Stripe invoice ID.
- Persist Stripe object IDs and the Stripe API version used.
- Do not accept Stripe receipts as invoices.
- Do not assume the Stripe document is compliant merely because Stripe calls it an invoice.
- Require a configured Stripe invoice profile that maps and validates supplier details, customer details, invoice number, dates, lines, tax treatment, mandatory wording, currency behavior, and totals.
- If a profile cannot guarantee all mandatory fields for a scenario, reject that scenario or use NAVRelay-generated PDFs instead.
- Download the final PDF once, hash those exact bytes, and preserve them in S3. Do not depend on the long-term availability or byte stability of an external URL.

## 14. PDF validation

For a provided PDF:

- verify the PDF magic header and actual content type;
- enforce a configurable size limit;
- reject encrypted/password-protected files unless an approved use case explicitly supports them;
- run malware scanning behind a storage/upload abstraction when operating as a public SaaS;
- do not use OCR as a compliance authority;
- optionally extract text to detect obvious invoice-number or supplier mismatches, but treat this only as a validation aid;
- require structured invoice data separately from the PDF;
- store validation results and the selected document profile version.

Do not modify the canonical bytes to add watermarks, page numbers, stamps, or metadata. Any derived preview is a separate non-canonical object.

## 15. Hashing and integrity

Store at least:

- a general internal content hash, initially SHA-256;
- byte length;
- MIME type;
- S3 object key;
- S3 version ID when available;
- upload/finalization timestamp;
- hash algorithm identifier;
- any NAV-specific electronic-invoice hash and its algorithm, when the selected NAV mode requires one.

Do not conflate the internal SHA-256 checksum with NAV's `electronicInvoiceHash`. The NAV hash input and allowed algorithm depend on the current NAV schema, `completenessIndicator`, and selected electronic-invoice/archival method. Implement this inside the versioned NAV adapter from current official documentation and test vectors.

The system must be capable of periodically verifying stored object bytes against the recorded internal hash and producing an integrity report.

## 16. S3 object-storage requirements

S3-compatible object storage is mandatory for production documents.

### Storage rules

- Buckets are private.
- Use separate buckets or strict prefixes and credentials per environment.
- Do not store durable invoice documents on local disk or Railway volumes.
- Enable versioning where the provider supports it.
- Prefer a provider supporting Object Lock or equivalent retention controls.
- If the selected S3-compatible provider lacks required immutability capabilities, do not pretend otherwise; use a suitable provider or document an approved compensating control.
- Use server-side encryption.
- Keep object names free of unnecessary personal data.
- Store canonical object metadata in PostgreSQL; S3 metadata is not the business source of truth.
- Never overwrite a canonical key. New legal documents use new keys and new database records.
- Use lifecycle rules only for temporary uploads, previews, and expired exports, never for retained canonical invoices.
- Back up or replicate documents according to the approved disaster-recovery policy.

### Suggested object layout

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

Use opaque internal identifiers in keys. Present human invoice numbers through metadata and the application, not as the sole object identifier.

### Presigned URLs

- Generate short-lived URLs only after authorization.
- Bind upload URLs to one organization, expected content type, size range, and staging key.
- Bind download URLs to one canonical object and a short expiry.
- Do not expose permanent public URLs.
- Do not store presigned URLs in the database as if they were durable references.

## 17. Retention and deletion

Model retention explicitly on stored objects and invoices.

- Store `retentionUntil` and the policy/version that produced it.
- The initial Hungarian product policy should target at least eight years of retention, but the final production rule requires legal approval and must remain configurable for longer periods.
- Never allow a user to shorten an already-applied retention period through an ordinary UI or API operation.
- Organization deletion must not delete documents that remain under mandatory retention.
- Account closure should transition retained data to a restricted archival state.
- Temporary uploads, failed staged documents, previews, and expired exports may have short lifecycle policies.
- Legal holds override normal deletion.

A deletion operation must produce an audit event and must fail closed when retention status is uncertain.

## 18. NAV Online Számla integration

Implement NAV Online Számla as a versioned provider adapter. Version 3 is the initial target.

Official environments must remain configurable. The current official endpoints are conceptually:

```text
Test frontend: https://onlineszamla-test.nav.gov.hu/
Test API:      https://api-test.onlineszamla.nav.gov.hu/invoiceService/v3
Production:    https://api.onlineszamla.nav.gov.hu/invoiceService/v3
```

Never hard-code an environment selected from a user-controlled request. Each organization has an explicit NAV connection environment, and production credentials cannot be used against test or vice versa.

### Required operations

The adapter must cover at least:

- token exchange;
- invoice submission through `manageInvoice`;
- transaction-status lookup;
- supported correction/cancellation or technical-annulment flows;
- taxpayer lookup where the product enables it;
- parsing technical and business validation messages;
- schema/version capability reporting.

### Credentials

Each organization uses its own NAV technical-user credentials.

- Encrypt credentials at rest with an application-level key-management abstraction.
- Restrict decryption to the worker/integration service that needs it.
- Never log passwords, signing keys, exchange keys, raw authorization material, or request signatures.
- Support credential rotation without rewriting historical submissions.
- Separate test and production credentials.
- Provide an explicit connection test that does not issue an invoice.

### Submission process

```text
invoice becomes ISSUED and canonical document exists
  -> create durable NAV submission job
  -> build and XSD-validate exact request
  -> obtain exchange token
  -> call manageInvoice with a stable operation identity
  -> persist request, response, transaction ID, and timestamps
  -> poll/query transaction status through durable jobs
  -> parse technical and business messages
  -> mark ACCEPTED, ACCEPTED_WITH_WARNINGS, REJECTED, or MANUAL_REVIEW
```

A transaction ID is not the final success state.

### Retry rules

- Retry network timeouts, connection resets, temporary NAV failures, and safe status queries with bounded backoff.
- Do not blindly retry permanent validation errors.
- Ensure a retry does not create a second logical submission.
- Persist attempt count, next-attempt time, last error code, and normalized error category.
- Escalate after a configurable deadline or attempt threshold.
- Provide an operator action to retry after a corrected configuration, without mutating the issued invoice.

### Stored NAV artifacts

Retain, subject to approved privacy and retention policy:

- schema version;
- normalized invoice snapshot;
- exact outbound XML;
- exact inbound response XML;
- transaction ID;
- final processing result;
- technical/business validation messages;
- request IDs and correlation IDs;
- timestamps and attempt records;
- adapter version.

Never expose credential-derived fields or secrets through audit shares.

## 19. NAV schema and mapping discipline

Treat the official XSD and interface documentation as executable contracts.

- Vendor the exact supported schema version or fetch it through a controlled update process.
- Pin a schema checksum/version in the repository.
- Validate outbound XML before submission.
- Use official examples and project-owned golden fixtures.
- Keep tax mappings in typed, versioned policy modules.
- Do not scatter NAV XML field names throughout controllers and entities.
- Separate normalized invoice data from NAV-specific transport DTOs.
- Record which mapping-policy version produced each submission.
- A schema upgrade requires migration analysis, compatibility tests, and release notes.

## 19A. NAV implementation complexity and scope control

Treat NAV integration as two different complexity layers.

### Transport layer: moderate and containable

Keep these concerns inside `packages/nav-online-invoice`:

- XML namespaces and schema-version handling;
- XSD validation and golden fixtures;
- password hashing, request signatures, token exchange, and request IDs;
- `manageInvoice` submission and safe retry identity;
- asynchronous transaction-status polling;
- response parsing and normalized technical/business messages;
- test-versus-production endpoint and credential separation.

No controller, Stripe adapter, or invoice-domain service should reproduce this protocol logic.

### Invoicing/compliance layer: high complexity

The larger risk is not making an HTTP request to NAV. It is correctly modeling and maintaining:

- seller/customer VAT statuses and jurisdiction;
- taxable, exempt, out-of-scope, reverse-charge, and OSS scenarios;
- foreign currency and exchange-rate rules;
- advance/final invoices;
- modifying, cancellation, replacement, and technical-annulment chains;
- mandatory invoice wording and PDF content;
- rounding and summary consistency;
- electronic-invoice hashes, archival policy, and statutory exports;
- behavior changes when NAV schemas or Hungarian rules change.

Therefore, every release has an explicit supported-scenario matrix. A narrow, fully tested matrix is acceptable; an undocumented claim to support arbitrary invoices is not. Expand one scenario at a time with official fixtures, domain tests, NAV test-environment verification, and compliance sign-off.

## 20. Corrections, cancellations, and relationships

Issued invoices are immutable.

Represent legal relationships explicitly:

- original invoice;
- modifying/corrective invoice;
- cancellation/storno document;
- advance and final invoice relationship, when supported;
- replacement or revision relationship, when supported by approved policy.

Each correcting or cancelling document:

- receives its own final number from the applicable series or external authority;
- has its own canonical PDF;
- has its own NAV submission;
- references the original invoice explicitly;
- preserves a complete chain visible in the UI, API, and exports.

Never implement “edit issued invoice” as a shortcut.

## 21. Public API design

Use versioned REST endpoints under `/v1` and generate OpenAPI from NestJS.

A baseline resource design is:

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
/v1/document-uploads
/v1/audit-shares
/v1/exports
/webhooks/stripe
```

The exact routes may evolve, but the resource boundaries must remain clear.

### API requirements

- JSON request/response bodies except for explicit upload/download endpoints.
- OpenAPI-generated examples and error schemas.
- Machine-readable error codes plus localized human messages.
- Cursor pagination for large collections.
- Stable filters for date range, series, number, customer, source, business state, and NAV state.
- Request/correlation IDs.
- Explicit API versioning and deprecation policy.
- No `any` request bodies.
- No implicit tenant selected solely from request content.

## 22. Idempotency

Require `Idempotency-Key` for externally initiated write operations that may issue a document, create a payment-linked invoice, start an export, or create a share.

Persist at least:

- organization/principal;
- HTTP method and normalized route;
- key;
- request hash;
- status;
- resulting resource ID;
- response code and safe response body;
- creation and expiry timestamps.

If the same key is reused with a different request hash, return a conflict. If it is reused with the same request, return the original result.

Also enforce unique source constraints such as:

- `(organizationId, stripeEventId)`;
- `(organizationId, stripeInvoiceId)`;
- `(organizationId, externalSource, externalInvoiceId)`;
- `(organizationId, invoiceNumber)`;
- `(navConnectionId, logicalSubmissionId)`.

## 23. Stripe integration

Stripe is optional and isolated behind a provider module. Webhook subscription is the primary event-delivery mechanism, backed by reconciliation so missed or delayed events can be repaired.

### Stripe workflow selection

Each Stripe integration selects an explicit issuer workflow and tax source. These are separate decisions and must never be inferred from whichever webhook happens to arrive first.

#### Issuer workflows

1. `PAYMENT_TRIGGERED_NAVRELAY`
   - Stripe is the payment source for a one-off purchase.
   - After authoritative payment success, NAVRelay performs the normal managed-issuance workflow: it validates the configured tax profile, allocates the invoice number, renders the PDF, stores it, and reports it to NAV.
   - This is the default Stripe mode for the initial product because NAVRelay retains full control over Hungarian invoice content and numbering.

2. `STRIPE_FINALIZED_INVOICE_IMPORT`
   - Stripe owns the invoice lifecycle, final invoice number, line and tax snapshot, and rendered PDF.
   - NAVRelay subscribes to Stripe invoice events, fetches the authoritative finalized Invoice object and PDF, validates an approved Stripe document profile, stores the exact bytes, and reports normalized data to NAV.
   - This is most useful for Stripe Billing subscriptions, prorations, and other cases where Stripe already owns the billing lifecycle.
   - It is opt-in per organization and invoice profile, not enabled globally.

Do not mix Stripe-owned and NAVRelay-owned invoice numbering within one invoice series. A Stripe-finalized invoice is view-only after import; corrections use explicit linked correction workflows rather than the NAVRelay invoice editor.

#### Tax sources

1. `NAVRELAY_TAX_PROFILE`
   - NAVRelay applies a deliberately enabled, versioned tax profile.
   - This is the default for managed issuance and the initial narrow set of supported Hungarian scenarios.

2. `STRIPE_TAX`
   - Stripe Tax calculates the transaction tax and supplies customer-location and tax-result evidence.
   - NAVRelay persists the complete calculation result and maps it only through an approved profile before NAV reporting.
   - Stripe Tax output is a calculation input, not legal approval, not a substitute for tax registration or filing, and not permission to infer unrestricted EU, OSS, exemption, or reverse-charge behavior.

For the initial release, allow `STRIPE_TAX` with `STRIPE_FINALIZED_INVOICE_IMPORT` only for specifically tested profiles. Do not build the product around the assumption that every Stripe Tax result can be translated automatically into a valid Hungarian invoice and NAV payload.

### Webhook rules

- Verify the raw request body and Stripe signature before parsing or persisting business data.
- Store and deduplicate the Stripe event ID before acknowledging it.
- Return success quickly after durable persistence; perform Stripe fetches, PDF downloads, validation, and invoice processing in the worker.
- Fetch authoritative objects from Stripe rather than trusting expandable webhook fragments.
- Preserve Stripe account, customer, checkout session, payment intent, charge, subscription, invoice, credit-note, tax-calculation, and API-version identifiers as applicable.
- Handle duplicate delivery, retries, delayed payment methods, and event reordering.
- Do not issue an invoice twice when payment and invoice events both exist for one economic transaction.
- Run a bounded reconciliation job that identifies relevant Stripe objects or events not yet represented in NAVRelay.

### Event policy

For `PAYMENT_TRIGGERED_NAVRELAY`:

- Use an authoritative successful-payment state, not webhook delivery alone, as the issuance precondition.
- For Checkout, verify the retrieved Session and its payment status; support asynchronous payment methods through their successful-payment event path.
- Persist a unique relation to the Stripe Checkout Session and PaymentIntent so retries cannot issue another invoice.

For `STRIPE_FINALIZED_INVOICE_IMPORT`:

- `invoice.finalized` is the canonical document-import trigger because the invoice has reached its finalized lifecycle state.
- `invoice.paid` updates payment state; it is not a second issuance trigger.
- `invoice.payment_failed` updates operational/payment state without changing the canonical invoice.
- `invoice.voided`, `credit_note.created`, refunds, and disputes enter explicit correction/reconciliation workflows. Never assume a Stripe void, credit note, or refund is automatically equivalent to a Hungarian storno or modifying invoice.
- For one-time Checkout invoicing, enable post-payment invoice creation only when the organization deliberately accepts Stripe's invoice pricing and the selected document profile has passed compliance validation.

### Initial product recommendation

- Keep the generic Swagger API and NAVRelay-generated invoices as the core product.
- Treat the Nuxt invoice editor as an optional first-party API client.
- Support provided-PDF import independently of Stripe.
- Make payment-triggered NAVRelay issuance the default one-off Stripe workflow.
- Add Stripe-owned Invoice plus Stripe Tax import as a narrow, opt-in subscription-oriented adapter after sample PDFs, tax outputs, corrections, and NAV mappings pass automated tests and professional compliance review.
- Keep provider pricing outside the invoice domain model; changing provider mode must never rewrite historical invoices.

## 24. Audit log

Maintain an append-only audit log for security- and compliance-relevant actions.

Each event should include:

- organization;
- actor type and actor ID;
- action code;
- resource type and ID;
- timestamp;
- request/correlation ID;
- user agent where appropriate;
- before/after metadata for mutable configuration, with secrets redacted;
- success/failure outcome;
- reason or originating integration.

Log at least:

- login and failed login;
- membership and role changes;
- API-key creation, rotation, and revocation;
- NAV credential changes and connection tests;
- invoice creation, issue, correction, cancellation, and attempted forbidden edits;
- document upload, canonicalization, integrity verification, and download;
- NAV submission attempts and final outcomes;
- audit-share creation, access, expiry, and revocation;
- export creation and download;
- retention or deletion actions.

Application logs are not a substitute for the domain audit log.

## 25. Audit access by login

Authenticated auditors use Better Auth and organization-scoped membership.

The audit UI must support:

- date and invoice-number filters;
- invoice series filters;
- source and customer filters;
- business and NAV-status filters;
- canonical PDF viewing/downloading;
- normalized invoice-data viewing;
- safe NAV result viewing;
- relation-chain viewing;
- export creation where permitted;
- integrity hash and verification status;
- access-history visibility for privileged roles.

Auditors never receive permission to modify invoice data, series counters, NAV credentials, or canonical documents.

## 26. Audit access by share link

Support revocable, scoped audit-share links without creating a full user account.

An `AuditShare` must define:

- organization;
- creator;
- immutable resource scope;
- optional date range;
- optional invoice-number/series range;
- allowed fields and actions;
- expiry;
- optional password hash;
- optional download limit;
- revoked timestamp;
- first and last access;
- access log.

Security requirements:

- use a high-entropy opaque token;
- store only a secure token hash;
- show the plaintext token once;
- support immediate revocation;
- default to short expiry;
- set `noindex` and a restrictive referrer policy;
- rate-limit token and password attempts;
- never expose raw S3 credentials;
- issue short-lived presigned object URLs only after share-scope validation;
- never expose secret-bearing NAV requests or unrestricted organization data.

A share link is a convenience and audit-delivery mechanism, not the only statutory export capability.

## 27. Exports

Exports are asynchronous, immutable snapshots.

Support at least:

1. **Statutory audit export** implemented against the current required Hungarian format.
2. **Convenience archive export** containing selected PDFs and safe metadata.
3. **Accounting export** in configured CSV/XML formats where explicitly implemented.

A convenience ZIP may look like:

```text
export.zip
├─ manifest.json
├─ invoices/
│  ├─ {invoiceId}.pdf
│  └─ ...
├─ metadata/
│  ├─ invoices.jsonl
│  └─ relations.jsonl
└─ nav/
   ├─ results.jsonl
   └─ safe-receipts/
```

The manifest must include:

- export ID;
- organization;
- filter snapshot;
- creation time;
- creator or share identity;
- included invoice IDs and numbers;
- file hashes and sizes;
- application version;
- export-format version.

Temporary export objects may expire after a configurable period. Their source invoices and canonical documents must not expire with them.

## 28. Email delivery

Use ZeptoMail through a backend abstraction.

- Send invoice links or attachments according to organization policy.
- Use localized templates.
- Do not expose permanent S3 URLs.
- Use a customer-facing download endpoint or short-lived signed URL.
- Record delivery attempts and provider message IDs.
- Retrying email must not regenerate or replace the invoice PDF.
- Email is delivery, not archival storage.

## 29. Data privacy and security

Invoice data can contain personal, financial, and tax information.

Mandatory controls:

- validate all external input;
- encrypt secrets at rest;
- use TLS for all external communication;
- keep S3 buckets private;
- use least-privilege S3 credentials;
- hash API keys and audit-share tokens;
- redact authorization headers, cookies, passwords, keys, signatures, raw invoice payloads, and customer PII from application logs;
- rate-limit public and machine endpoints;
- protect login and exposed forms against abuse;
- implement secure webhook verification;
- prevent SSRF in external-PDF fetching;
- allowlist supported external hosts and enforce response-size/content-type limits;
- use malware scanning for public uploads;
- validate authorization again at download time;
- never trust browser-calculated totals, tax, roles, or ownership;
- document data-processing roles for hosted SaaS operation;
- support data-subject workflows without violating mandatory invoice retention.

## 30. Background jobs

Use a durable queue. The default may be PostgreSQL-backed to avoid an unnecessary Redis dependency, but it must support safe multi-worker claiming, retries, scheduling, and dead-letter/manual-review states.

Required job types include:

- render generated PDF;
- verify/promote provided PDF;
- submit invoice to NAV;
- query NAV transaction status;
- send invoice email;
- build export;
- verify stored-document integrity;
- expire staged uploads and exports;
- reconcile stuck workflows;
- Stripe object synchronization when needed.

The API process may enqueue jobs but must not run an untracked in-memory queue. Worker loops belong in `apps/worker` and may keep that Railway service awake.

## 31. Transactional outbox and workflow reliability

When a database state transition requires asynchronous external work, write the domain change and outbox/job record in the same PostgreSQL transaction.

Examples:

- issuing an invoice and enqueueing PDF generation;
- canonicalizing a PDF and enqueueing NAV submission;
- receiving a Stripe event and enqueueing invoice processing;
- accepting a final NAV result and enqueueing delivery/notification.

A process crash after commit must not lose the required external action. A retry must not duplicate the domain action.

## 32. Frontend requirements

Use Nuxt, Nuxt UI, Tailwind, and Nuxt i18n.

Initial application areas:

- organization switcher;
- dashboard and operational alerts;
- invoices and invoice details;
- draft/generated/provided invoice flows;
- series/invoice-block management;
- customers;
- organization members and roles;
- API clients and keys;
- NAV connection setup and status;
- Stripe integration setup;
- audit logs;
- audit shares;
- exports;
- storage/integrity status;
- organization and retention settings.

All user-facing text must be translated. English and Hungarian are the initial locales, with English as fallback unless the repository explicitly changes that decision.

The UI consumes generated business API clients. Do not duplicate authorization or legal rules in client-only code.

## 33. OpenAPI and SDKs

NestJS controllers and DTOs are the source of truth for OpenAPI.

- Generate and commit the OpenAPI document if that is the repository convention.
- Generate the Nuxt client through `nuxt-open-fetch`.
- Keep generated code separate and never hand-edit it.
- Consider generated TypeScript SDK publication only after the public API stabilizes.
- Public API changes require compatibility review and release notes.
- `pnpm run verify` must detect stale generated contracts.

## 34. Database conventions

Use MikroORM's Data Mapper, Identity Map, and Unit of Work correctly.

- PostgreSQL is the real database in development integration tests.
- Use generated MikroORM migrations for every schema change.
- Keep production schema synchronization disabled.
- Use explicit transactions for issuance, sequence allocation, outbox creation, and other atomic workflows.
- Use high-level `EntityManager`/repository APIs whenever practical.
- Narrow raw SQL or QueryBuilder usage is acceptable for PostgreSQL concurrency primitives such as `FOR UPDATE SKIP LOCKED`, but isolate it and document why it is necessary.
- Do not return raw entities as public API responses.
- Use DTOs and explicit mappings.

## 35. Testing requirements

Prefer behavioral and integration coverage over isolated mocks.

### Mandatory critical tests

- Better Auth organization creation, invitation acceptance, multi-organization switching, and organization-scoped role changes behave as configured;
- retained domain data prevents destructive organization deletion;
- a machine API request with a valid scoped API key succeeds and an invalid, expired, revoked, or insufficiently scoped key is denied;
- machine writes enforce rate limits and idempotency without weakening organization isolation;
- a Stripe webhook fails on an invalid signature and a duplicate valid event remains idempotent;
- two concurrent requests cannot allocate the same managed invoice number;
- an idempotent retry returns the same issued invoice;
- a duplicate Stripe webhook does not create another invoice;
- out-of-order `invoice.finalized` and `invoice.paid` events converge to one invoice and correct payment state;
- a Stripe Invoice that fails its configured document/tax profile is not issued or reported silently;
- the same external invoice ID cannot be imported twice;
- cross-tenant invoice and document access is denied;
- issued invoice data cannot be edited;
- canonical S3 objects cannot be replaced through application APIs;
- the stored hash matches exact uploaded/generated bytes;
- a supplied PDF remains byte-for-byte unchanged;
- S3 failure leaves a recoverable workflow, not a false issued-and-delivered state;
- NAV timeout and temporary failure produce a durable retry;
- NAV permanent validation errors enter manual review without duplicate issuance;
- a NAV transaction ID is not treated as final acceptance;
- audit-link scope, expiry, password, and revocation are enforced;
- export manifests match included documents and hashes;
- correction/cancellation chains remain intact;
- API keys cannot cross organization boundaries;
- retention prevents deletion;
- secrets and PII are redacted from logs.

### Test tools

- Vitest across frontend and backend.
- Supertest and `@nestjs/testing` for HTTP/API E2E tests.
- Testcontainers PostgreSQL.
- A real S3-compatible test service such as MinIO in integration tests where byte behavior matters.
- Provider mocks at the NAV, Stripe, email, and production-S3 boundaries.
- Official NAV XML/XSD fixtures and project-owned golden files.
- Opt-in live contract tests against the NAV test environment; never run these as an uncontrolled default test suite.
- Playwright across Chromium, Firefox, and WebKit with isolated application/database lanes for mutating tests.

## 36. Verification command

The repository root must expose:

```bash
pnpm run verify
```

It must run every applicable release-quality check, including:

- linting and formatting checks;
- TypeScript checks;
- backend tests;
- frontend tests;
- PostgreSQL integration tests;
- storage integration tests;
- invoice concurrency/idempotency tests;
- OpenAPI generation and drift checks;
- locale completeness checks;
- production builds;
- Playwright E2E matrix.

Do not claim a change is fully verified unless this command succeeds. Report exact skipped or blocked suites.

### GitHub Actions pull-request gate

Create `.github/workflows/verify.yml` with a stable job/check name of `verify`.

- Trigger it for `pull_request`, pushes to the protected default branch, and `merge_group` when merge queue is enabled.
- Use read-only repository permissions by default.
- Pin the repository Node.js and pnpm versions, install with `pnpm install --frozen-lockfile`, install required Playwright/browser dependencies, and run `pnpm run verify` unchanged.
- GitHub-hosted Linux runners are the initial default because the suite requires Docker/Testcontainers. Do not expose organization or production secrets to pull-request code.
- Mock NAV, Stripe, email, and production S3 at their adapter boundaries in the mandatory PR workflow.
- Keep live NAV test-environment contract tests in a separate trusted workflow, such as manual `workflow_dispatch` and/or a scheduled protected-branch run. It may use only NAV test credentials and must never be required for untrusted fork PRs.
- Configure concurrency so a superseded run for the same PR is cancelled.
- Protect the default branch so `verify` must pass before merge.
- When Railway deploys from GitHub automatically, enable its CI/deployment gate so the verified commit SHA, not an earlier or later revision, is released.

## 37. Railway behavior

Follow the default-stack Railway conventions:

- use private networking between Nuxt, NestJS, worker, and PostgreSQL;
- run MikroORM migrations in the pre-deploy command, not on every start;
- use bounded database wake-up retries;
- keep connection pools small with finite idle timeouts;
- avoid keepalive traffic in services intended to sleep;
- keep durable polling and continuously scheduled jobs in the separate worker service;
- expose cheap readiness endpoints that do not query the database on every probe;
- store secrets only in server-side Railway variables or an approved secret manager;
- do not use local working-tree uploads as the production deployment path.

## 38. Observability

Use structured JSON logs in production and readable logs in development.

Include:

- service name;
- environment;
- request/correlation ID;
- organization ID where safe;
- resource IDs;
- job ID;
- NAV transaction ID where safe;
- outcome and duration;
- retry attempt and next retry time.

Never log:

- NAV credentials;
- Stripe webhook secrets or API keys;
- API-key plaintext;
- audit-share tokens;
- authorization headers or cookies;
- full invoice/customer bodies;
- raw canonical PDF bytes;
- unredacted NAV requests if they contain sensitive authentication material.

Create operational views or alerts for:

- invoices missing canonical documents;
- NAV submissions stuck in processing;
- rejected submissions;
- expiring/invalid NAV credentials;
- failed email deliveries;
- S3 integrity failures;
- dead-letter jobs;
- sequence-allocation conflicts;
- unexpectedly high duplicate/idempotency conflicts.

## 39. Self-hosted and hosted editions

The same core application should support:

- self-hosted single-organization use;
- self-hosted multi-organization use;
- hosted multi-tenant SaaS.

Do not fork compliance logic between editions. Feature entitlements may differ, but issuance, hashing, storage, NAV mapping, tenant isolation, and audit behavior must use the same tested core.

Self-hosting must support externally supplied:

- PostgreSQL URL;
- S3 endpoint, region, bucket, and credentials;
- public web/API URLs;
- Better Auth secret and trusted origins;
- encryption master key or KMS integration;
- NAV environment configuration;
- email provider configuration;
- optional Stripe configuration.

Do not silently fall back to local durable document storage when S3 is unavailable.

## 40. Commercial and open-source constraints

NAVRelay is intended to remain open-source while also being commercially operable as a hosted service.

- Keep core functionality provider-neutral.
- Avoid dependencies whose licenses prevent the intended distribution or hosted model.
- Record third-party licenses.
- Do not add a project license automatically unless the owner explicitly selects one.
- Preserve the possibility of an AGPL/open-core or dual-license strategy, but treat the final license as an unresolved product decision.
- Do not make the hosted service dependent on undocumented private behavior that the self-hosted edition cannot reproduce.

## 41. Non-goals for the initial product

Unless explicitly added later, do not expand the initial scope into:

- full general-ledger accounting;
- payroll;
- inventory/warehouse management;
- banking reconciliation beyond payment references;
- a marketplace of tax advisers;
- unsupported country-specific invoicing regimes;
- arbitrary PDF editing;
- OCR-based invoice ingestion as a source of legal truth;
- direct access to users' S3 credentials from the browser;
- replacing an accountant or tax adviser with inferred tax logic.

## 42. Initial supported product slice

The first production-capable slice should include:

1. Better Auth organizations, invitations, memberships, static organization roles, and separate machine API keys.
2. Managed and external invoice series.
3. Generated PDF issuance.
4. Provided PDF ingestion.
5. Private S3 canonical storage with hashes.
6. Direct NAV test and production connections per organization.
7. Durable NAV submission and final-status handling.
8. Invoice listing, details, PDF download, and NAV diagnostics.
9. Login-based auditor access.
10. Scoped, expiring audit-share links.
11. Date/number-range exports with manifests.
12. Stripe webhook integration for one approved flow, with durable event storage and reconciliation.
13. Public Swagger/OpenAPI documentation with scoped API-key authentication and rate limits.
14. English and Hungarian UI/API messages.
15. Full critical-path test coverage, mandatory GitHub Actions `verify`, and a required pull-request status check.

Do not call the first slice production-ready until the legal/compliance checklist has been signed off and live test invoices have been validated in the NAV test environment.

## 43. Definition of done for invoice issuance

An invoice-issuance feature is done only when all of the following hold:

- authorization and organization scope are enforced;
- input is validated through typed DTOs;
- totals use exact arithmetic;
- idempotency is enforced;
- the series/numbering rule is enforced;
- an immutable issued snapshot exists;
- a canonical PDF exists in S3;
- its exact-byte hash and storage metadata are recorded;
- the object cannot be replaced through normal APIs;
- a durable NAV job exists or the approved policy says reporting is not required;
- NAV state is visible and diagnosable;
- audit events exist;
- applicable delivery is queued;
- generated OpenAPI/client output is current;
- targeted tests and full `pnpm run verify` pass.

## 44. Prohibited shortcuts

Do not:

- use Billingo or Számlázz.hu as a hidden backend;
- store canonical PDFs only in Stripe;
- store canonical PDFs only in PostgreSQL blobs;
- use local disk as production archival storage;
- replace canonical PDFs after issuance;
- edit issued invoice rows in place;
- reuse an invoice number;
- allocate managed numbers outside a safe transaction;
- mark NAV success after only receiving a transaction ID;
- run NAV polling in the request process;
- accept arbitrary remote PDF URLs;
- accept a Stripe receipt as an invoice;
- accept a Stripe Invoice PDF without a validated profile;
- expose public S3 buckets or permanent object URLs;
- use S3 ETag as the legal/integrity hash;
- log raw secrets or invoice bodies;
- rely on frontend authorization;
- silently guess tax treatment;
- silently change a NAV mapping or schema version;
- delete retained invoices when an organization account is closed;
- bypass, weaken, or mark optional the required pull-request `verify` check;
- expose Better Auth/control-plane routes on the machine-API host;
- treat Stripe Tax output as legal approval or as an unrestricted NAV mapping;
- map a Stripe void, credit note, refund, or dispute directly to a Hungarian correction/storno without an approved workflow.

## 45. Agent workflow

Before implementing a change:

1. Read this file and `defaultstack.md`.
2. Identify affected domain invariants, tenants, documents, NAV mappings, and retention rules.
3. Check current official NAV/Stripe documentation when the change touches either integration.
4. Update or add a generated MikroORM migration for schema changes.
5. Add tests for success, duplicate/retry behavior, authorization, and failure recovery.
6. Regenerate OpenAPI and frontend clients when the API changes.
7. Run targeted tests during development.
8. Run `pnpm run verify` before declaring completion.
9. Ensure the GitHub Actions `verify` workflow uses the same command and remains green for the pull request.
10. Report any legal assumption, missing official specification, unverified provider behavior, or blocked test explicitly.

When uncertain, fail closed rather than issuing, exposing, overwriting, or deleting a legal document incorrectly.

## 46. Official implementation references

Verify current versions before implementation:

- NAV Online Számla public repository: https://github.com/nav-gov-hu/Online-Invoice
- NAV Online Számla production portal: https://onlineszamla.nav.gov.hu/
- NAV Online Számla test portal: https://onlineszamla-test.nav.gov.hu/
- Hungarian invoice-program regulation: https://njt.hu/jogszabaly/2014-23-20-2X
- NAV electronic-invoice guidance: https://nav.gov.hu/ado/afa/Az_elektronikus_szaml20200416
- Better Auth Organization plugin: https://better-auth.com/docs/plugins/organization
- Railway public-networking headers: https://docs.railway.com/networking/public-networking/specs-and-limits
- Stripe Invoice API: https://docs.stripe.com/api/invoices
- Stripe invoice lifecycle and finalization: https://docs.stripe.com/invoicing/integration/workflow-transitions
- Stripe webhooks: https://docs.stripe.com/webhooks
- Stripe Checkout invoice creation: https://docs.stripe.com/api/checkout/sessions/create
- Stripe Tax: https://docs.stripe.com/tax

The repository must pin and document the exact NAV schema, Stripe API version, renderer version, and mapping-policy version used by each production release.
