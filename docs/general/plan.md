# Audrey — Plan

Product behaviour, architecture decisions, stack and ubiquitous language in one file. Written collaboratively in the first working session (2026-10-25). Revisit only when a decision actually changes.

Related files: live step tracker [progress.md](progress.md); research that informed decisions in [../research/](../research/) (seed Authority candidates, comparable products, Wolfram|Alpha notes).

ODRE = Ontological Document Review Engine ("ontological" is used loosely for empirical, factual, real-world; strictly the fit is "empirical"/"epistemic"; the owner kept the name). Repository: https://github.com/ManuelBv/audrey (public, AGPL-3.0).

## Product behaviour

### Purpose and scope

The user pastes text or uploads a Document (TXT, MD, or PDF with a text layer: legal, scientific, books), states its Domain (or "don't know"), and picks a Depth. Audrey extracts Claims at subsentence, sentence, paragraph, section and page level and compares each against Authorities. Every Claim receives a Bayesian Score, never a true/false verdict. The product is general from the start: no domain-limited first slice. Medicine is the first test domain, not a scope limit. Test cases: vaccines and autism (expect a score near 1), the sun/earth reading split, and the "induce" domain disambiguation. The Registry needs seed entries for every domain tested.

### Bayesian Score

- Integer 1–99, never 0 or 100 (absolutist claims are not scientific). 1 ≈ almost certainly false, 99 ≈ almost certainly true.
- Computed, not guessed by an LLM: a prior (by Domain and Claim Type) updated by per-Evidence likelihood ratios, combined in log-odds, clamped to 1–99. The scoring engine is pure logic with no UI or network dependencies.
- Evidence is grouped by Independence Cluster before combining, so sources deriving from the same underlying data count once.
- Each Claim shows two numbers: the score and Evidence Coverage / credible interval (a 50 from no Evidence differs from a 50 from conflicting strong studies).
- Until calibrated against labelled benchmarks (FEVER, SciFact or similar), the UI labels it an "evidence-weighted score", not a probability.
- Ambiguous Claims are split into Interpretations, each scored separately (e.g. "the sun revolves around the earth": acceptable as a coordinate-frame description, false as a physical description).
- Claim Types: empirical-descriptive; empirical-causal (graded by study design); legal (what the text says, in force, jurisdiction, date; own sub-score); non-scorable (opinion, prediction, definition: marked "not scorable", no number).
- Roll-up from Claim to paragraph, section, page and Document is not a plain average; it accounts for how load-bearing each Claim is.

### Claim extraction (decided 2026-10-25)

- A Claim is checkable if an Authority in its Domain could in principle support or contradict it: empirical, legal (what a text says, whether it is in force) and quantitative statements. Opinions, predictions, definitions, questions and instructions are marked "not scorable" and get no number.
- Pipeline (same shape as the open-source Loki tool): (1) split the Document into sentences deterministically, keeping exact character positions for Anchors; (2) the LLM, given the surrounding Chunk context, resolves pronouns, rewrites each sentence into standalone atomic Claims and labels each Claim Type and checkability; (3) Depth decides which Claims are checked: Quick = Claims the LLM ranks as central to their section, Standard = every checkable sentence, Deep = every atomic subsentence Claim plus roll-ups.
- The checkable/not-scorable label is an LLM judgment; the user can see it and override it per Claim. Details are refined in the step 8 spec.

### Domains

- A tree with multiple roots: Science (biology, physics, ...), Engineering (civil, military, ...), Medicine, Law, others. Initial tree (a machine-readable file in the repo, changes approved by the owner via commit): Science (Biology, Physics, Chemistry, Astronomy, Earth science, Mathematics); Medicine (Clinical medicine, Public health, Pharmacology); Engineering (Civil, Military, Electrical, Mechanical, Software); Law (UK law; US law skipped for now); Finance and Business (Accounting and auditing, Corporate finance, Economics, Markets and investing, Taxation, Business operations and management); General (fallback). Domains with no seed Authorities stay selectable but show a low-coverage warning. When inference is unsure between sibling domains it picks their parent and the Review says so.
- The Document-level Domain is the default; it can be overridden per chapter, section or Claim (a book may span several roots).
- Declared Domain → `DECLARED`. "Don't know" → inferred from the text and the Review is marked `INFERRED`, showing top candidates, confidence, and the fixed warning that the Review may not apply to the original Domain. Low-confidence inference is shown, not silently resolved.
- Domain controls word-sense disambiguation (e.g. "induce" in psychology vs "inductance" in electronics), Authority routing, priors and weights.

### Authorities, tiers and the Registry

| Tier | Sources | Weight |
|---|---|---|
| 1 Authoritative | Peer-reviewed literature, systematic reviews, legislation and regulation, government statistics, standards, technical manuals | Highest |
| 2 Computed knowledge | Wolfram\|Alpha (deferred, see research notes) | High for quantitative/physical facts |
| 3 Other web | Institutional pages, reference works, encyclopedias (Wikipedia = tertiary: follow its citations to the primary source and score that) | Lower, capped |
| 4 News | All outlets classed as opinion | Minimal; if a news item cites a primary document, trace and score the primary |

Weights live in config so they can be tuned. Roles of an Authority: Evidence (supports/refutes), Modifier (changes trust in another Authority, e.g. Retraction Watch), Navigation/tertiary.

Registry entry fields: name, URL, Domains (many-to-many), source type (journal, legislation, statistics, standard, technical manual, ...), role, tier, access class (open, free key, licence needed, paywalled), access method (API / bulk / scrape / manual), licence and terms, version/revision (standards and manuals are valid per revision and date), Independence Cluster, independence flag (manufacturer manuals), provenance (added by, when, justification, origin tag), review date.

Registry governance:
- The Registry is a machine-readable file in the repo (source of truth), synced into the database so the LLM can use it. One global list.
- v1 includes only Authorities that can be queried free of charge (paid APIs excluded for now). Free-key Authorities use an owner-held service key, not a per-user Credential. The starter seed set (medicine, biology, physics and astronomy, UK law, engineering) is approved; entries are created later after verifying licence, API and URL.
- Changes enter as Registry Suggestions with an origin tag: `USER` (feedback form with a reason) or `AI_RESEARCHER` (in-app researcher prompt). The owner approves or rejects; approval is a git commit. Nothing affects scores before that.
- Known issues: paywalled standards (ISO, IEEE), military manuals (public ones only), manufacturer bias.

### Review screen

- Left: the original Document with highlighted Claims. Right: sidebar listing, per Authority consulted, the Evidence and a one-line statement of whether the Claim is supported or contradicted, or what the alternative claim is. Lower right: chat panel for custom questions about the Document.
- Per Claim, the user clicks Accept to commit the Claim and its Evidence.

### Persistence and immutability

- Accepted Claims and their Evidence are immutable in the database, per user. Reopening a Review reads from the database; no live AI call. A re-run creates a new Review; old ones are preserved.
- Each Snapshot stores: excerpt, URL, retrieval date, content hash, Registry commit hash, score inputs (priors, likelihood ratios, clusters). Excerpts and hashes only, not full text (copyright position unverified).
- Each Claim records which model did the research: for every LLM step (Claim extraction, domain inference, Evidence assessment, explanation) the Provider, the exact model identifier as returned by the API, the prompt version and the sampling parameters (e.g. temperature); embeddings record their Provider, model and dimension. This is stored per step, not just per Review, because a user can change models between runs or mid-job. The private and public Review screens show the models used.

### Retrieval and chat

Anthropic's Contextual Retrieval design: at ingest each Chunk gets short Document-specific context prepended before both embedding and BM25 indexing; retrieval combines both. Per-Document pre-work at ingest produces Claim assertions plus the Document's internal logic. Chat answers use the same retrieval.

### Second analysis layer (later, step 13)

Document logic and implications, e.g. "page 4 discusses covenants on your leasehold in England; here is what they mean and why they may matter to you". This approaches legal advice: flag and explain only, with a clear disclaimer. Built after claim verification.

### Public reviews (step 14)

- A user can publish an accepted Review. The public page uses the same layout as the private screen but is read-only and never includes the original file: it shows short excerpts written `[...] several words [...]`, followed by Evidence, scores and explanation. Excerpt caps (accepted by the owner 2026-10-25; a product rule, not a legal safe harbour): at most 25 words per excerpt, at most 5% of the Document's words in total and never more than 400 words in total, whichever limit is reached first. The Document's title, author and page are acknowledged.
- Friction: publishing moves a Review from `private` to `scheduled` with a visible countdown (10 minutes, confirmed by the owner 2026-10-25) and an Undo button, then to `public`. The public listing only shows Reviews whose scheduled time has passed.
- Redaction: at publish time, text that looks like personal data or secrets is redacted from the public copy (candidate tool: Microsoft Presidio, which gives no completeness guarantee and does not detect secrets, so a separate secrets scanner is needed). The user sees a preview and confirms; the interface states that detection can miss items.
- The public copy is a separate stored snapshot, so unpublishing deletes it. Private data is untouched. The user can unpublish at any time and the public URL then stops working. Copies others already saved or search engines indexed cannot be recalled; public pages carry a no-index tag by default.
- Anonymity: only the user's three-word pseudonym is shown publicly; the real name stays in the private profile.
- Moderation: an admin can take down any public Review with a reason and an audit log entry; viewers get a report button. A terms page states that the admin may delete any public Review at their discretion, for any reason, without liability (owner's wording, 2026-10-25). Enforceability of that clause and any statutory notice-and-takedown duties are unverified; have a lawyer review the terms before a wider public launch.
- Public pages carry the standing disclaimer that scores are AI-generated, evidence-weighted outputs and not a certified or definitive source.
- Public endpoints are separate and read-only, returning only published fields (no user ids, chat, keys or unpublished Claims).
- Copyright (US fair use, UK fair dealing for criticism/review/quotation) and personal-data points come from web searches, not legal advice; get legal advice before a wider public launch.

## Ubiquitous language

These terms appear verbatim in code, types and tests. No informal synonyms.

| Term | Meaning |
|---|---|
| Document | A file the user uploaded (TXT, MD, or PDF with a text layer). Never called "source". |
| Authority | An entry in the Registry: an authoritative source such as PubMed or legislation.gov.uk. |
| Evidence | An excerpt from an Authority bearing on a Claim, stored with its Snapshot. |
| Snapshot | Excerpt, URL, retrieval date, content hash and Registry commit hash stored with Evidence. |
| Registry | The global, owner-approved list of Authorities, a file in the repo. |
| Registry Suggestion | A proposed Authority with origin `USER` or `AI_RESEARCHER`. Not used for scoring until the owner approves it by commit. |
| Claim | An assertion found in a Document, at subsentence, sentence, paragraph, section or page level. |
| Claim Type | empirical-descriptive, empirical-causal, legal, or non-scorable. |
| Interpretation | One reading of an ambiguous Claim, scored separately. |
| Anchor | Where a Claim sits in a Document: character range (TXT), or page plus word bounding boxes (PDF); always with exact quote, prefix and suffix. |
| Review | One run of Audrey over a Document at a chosen Depth. A re-run is a new Review. |
| Depth | Quick (major Claims per section), Standard (every checkable sentence), Deep (every checkable subsentence Claim plus roll-ups). |
| Domain | A node in the domain tree. Provenance is `DECLARED` or `INFERRED`. |
| Retrieval Mode | `HYBRID` (embeddings plus BM25) or `BM25_ONLY`. Stored on every Review. |
| Bayesian Score (BS) | Integer 1–99, computed from a prior and Evidence likelihood ratios. Never 0 or 100. |
| Evidence Coverage | How much Evidence supports a BS; shown next to the score. |
| Independence Cluster | Group of Authorities sharing an underlying data origin; counted once when combining Evidence. |
| Accept | The user commits a Claim and its Evidence; it becomes immutable. |
| Chunk | A contextualized piece of a Document indexed for retrieval (BM25 and embedding). |
| Review Job | The background work behind a Review, with persisted progress. |
| Provider | An LLM or embedding vendor reached through an adapter. |
| Credential | A user's encrypted API key for a Provider. |

## Stack

- Frontend: React, Vite, TanStack Router, TanStack Query; client-side SPA (TanStack Start not needed because FastAPI is the server). Typed client generated from FastAPI's OpenAPI schema. PDFs rendered with pdf.js.
- Backend: Python, FastAPI. v1 runs background jobs inside the single backend service (database-backed queue); a separate worker comes later if needed.
- Database: PostgreSQL with pgvector (supported on Render Postgres per its docs; enable with `CREATE EXTENSION vector;`). Relational data, vectors and job state in one database. BM25: Postgres full-text search is not true BM25, so use a BM25 extension if available on the host, otherwise compute BM25 in the Python service. Qdrant is a later swap only if measured need appears.
- Hosting: Render, defined by `render.yaml` at the repo root with `rootDir` and `buildFilter` per service. Owner will pay for the always-on tier: managed Postgres and a static frontend alongside the backend web service. Estimated monthly cost (third-party figures, unverified against render.com): web Starter ~$7 + Postgres Basic-256mb ~$6 + 5 GB storage at $0.30/GB = $1.50, so ~$14.50. Render storage is billed separately from instance type, in multiples of 5 GB, and cannot be reduced. Alternative: Railway. Fallback database: Neon free tier (too small for sentence-level vectors).
- Docker: the backend has one Dockerfile used both locally (docker compose with a pgvector Postgres container; image name unverified) and on Render (Render supports Docker-based services and prebuilt images per third-party summaries; confirm on render.com). The frontend runs with Vite locally and deploys as a static site.
- Scheduled work (admin digest): in-process scheduled job on the single backend instance for v1; Render cron job (type cron, UTC, minimum $1 a month) if multiple instances are ever needed.
- Email: Resend (owner signed up 2026-10-25; free plan 3,000 a month capped at 100 a day per its pricing page). Sending is in v1 scope, via an HTTPS API call (Render blocks SMTP on free instances and port 25 reportedly everywhere); Resend sits behind an email sender interface.
- File storage: original Documents live in Cloudflare R2 (free tier 10 GB per third-party sources, no egress fees) behind a storage interface so the provider can change. Expected volume: 10–20 books plus dozens to hundreds of PDFs (~1 GB, estimate). The database holds only metadata, text chunks and vectors.
- Auth: in-app registration and login, admin approval of new users, no OAuth for now.
- Testing: three layers, all required. Unit tests (backend pytest and frontend Vitest suggested) cover pure logic and components. Integration tests cover API endpoints, the database (test database only) and adapters against fakes. End-to-end tests (Playwright suggested) drive a real browser against the local Docker stack and cover each feature's user-visible behaviour. Rule: no functionality without its corresponding tests. A feature is not done, and the Reviewer fails it, unless it has unit tests, integration tests and an end-to-end scenario for its user-visible behaviour; logic with no user-visible behaviour of its own (for example the Bayesian Score engine) still needs unit and integration tests and is exercised end-to-end through the feature that uses it. Tests are written first (TDD). Database tests run only against a dedicated test database filled with example (fixture) data, never the production database; see rule 13 for the guard that enforces this. Tool choices are confirmed by the step 1 Researcher.

## Cross-cutting rules (apply to every feature)

1. Every table carries `user_id`. The app starts with one user (the owner) but has full registration, login and an admin panel where the admin approves users. The code is AGPL and may be self-hosted by others.
2. Accepted Claims and their Evidence are immutable. Reopening a Review reads from the database and makes no live AI call. A re-run creates a new Review.
3. Each Review records: file SHA-256, Registry commit hash, Domain provenance (`DECLARED`/`INFERRED` plus candidates and confidence), Retrieval Mode, Depth, embedding model and chat model (per-step model details are recorded on each Claim, see Persistence and immutability).
4. Provider abstraction: chat and embeddings each sit behind an interface with one adapter per Provider. v1 chat Provider is Anthropic. Embedding Provider v1: Voyage AI (confirmed by the owner 2026-10-25), with OpenAI text-embedding-3-small as the second adapter. Each user chooses providers and models on a settings page and enters a Credential for each.
5. Credentials: encrypted (AES-256-GCM) with the key held outside the database, never hashed because the backend must use them; write-only in the UI (last 4 characters shown); validated with a test call on save; never logged. Passwords: argon2id with per-user salt. Standards: OWASP Top 10, ASVS, OWASP Top 10 for LLM Applications.
6. Uploaded Documents and fetched web content are untrusted input (prompt injection). The AI only produces proposals; it never changes scores or the Registry directly. Uploads: type and size validation, malware scanning.
7. Embeddings record model name and dimension. Changing the embedding model means re-embedding, shown with a cost estimate and confirmed first. Stale embeddings block chat and Review until confirmed. With no embedding key, run `BM25_ONLY` with a visible banner.
8. A cost estimate is shown before each Review run and before any re-embed.
9. Registration is approval-gated: a new user starts `pending` and cannot log in or use the app until an admin approves. The users table has `status` (pending, approved, rejected, suspended), `approved_by`, `approved_at` and a reason; every admin action (approve, reject, suspend, takedown) is written to an audit table (who, what, when, why). The first admin is created by a one-time bootstrap (setup command or environment variables, never stored in the repo) because nobody exists to approve it; the owner will also register a normal user account and approve it as admin. Registration is rate-limited.
10. Email sending is in scope for v1 (owner confirmed 2026-10-25 that it is not deferred and signed up to Resend). Features: email verification via a single-use, expiring, hashed token before a user enters the approval queue (`email_verified_at` column); self-service password reset by emailed single-use token; and a daily admin digest containing only a count and a link (the admin panel list remains the main view). Sending goes through an email sender interface (Resend adapter first; SMTP or others later). Resend facts (checked 2026-10-25 against resend.com docs): before a domain is verified, the shared sender onboarding@resend.dev can deliver only to the email address of the Resend account (other recipients get a 403); a verified domain is required to mail anyone else. Verification adds DNS records (DKIM at resend._domainkey, SPF and return-path on a send subdomain, TXT plus MX or, for newer domains, CNAME records; DMARC is advised after verification), usually verifies within about 15 minutes and can take up to 72 hours; a proxied Cloudflare CNAME blocks verification. Consequence: the admin account and the owner's normal user account can only receive real email at the Resend account address until the domain is verified, Owner decision 2026-10-25: the owner's domain manuelcheta.com (registered with an ISP in Iceland, DNS panel capabilities unchecked) will not be verified in Resend for now; email is for the owner only. Consequences: other people who register will not receive the verification email, so they cannot reach the approval queue (a welcome side effect, but not the security gate; admin approval is). A failed send (for example the 403) must never be shown to the registrant and is only logged, and the response must look the same as a successful registration. The bootstrapped admin account is created already verified; the owner's normal user account receives its verification email at the Resend account address. When the domain is verified later, the DNS host must allow DKIM, SPF and return-path records (moving DNS hosting is an option if the ISP does not). In tests, the email sender is always a fake.
11. Rate limiting applies to registration, login (failed attempts, with temporary lockout), uploads, starting a Review, and the public pages and report button; exact limits are set in the relevant feature specs. Counters live in memory or Postgres while there is one backend instance. The real client IP must be read from the proxy header only when it comes from the trusted proxy.
12. Secrets live only in git-ignored `.env` files locally and in Render environment variables in production. A committed `.env.example` lists every variable with a placeholder and description (no real values). The backend fails fast at startup if a required variable is missing. No secret may use the `VITE_` prefix (exposed to the browser). Planned variables: DATABASE_URL, ENCRYPTION_KEY (back up separately: losing it makes stored Credentials unrecoverable), SESSION_SECRET, ADMIN_BOOTSTRAP_EMAIL/PASSWORD, R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET, EMAIL_PROVIDER/EMAIL_API_KEY/EMAIL_FROM, NCBI_API_KEY (optional). Users' own LLM and embedding keys are stored encrypted per user in the database, not in `.env`.
13. Quality gates: a pre-commit hook scans staged files for secrets; a pre-push hook runs ALL tests including end-to-end; CI repeats both on every push because local hooks can be bypassed or missing in a fresh clone. Owner decision 2026-10-25: commit often, push only when the owner says so. Test layers and the no-functionality-without-tests rule are in Stack (Testing). Database tests use a separate test database in its own container; the production database address exists only in Render's environment, and test setup aborts unless the database name contains `test` and the host is local. Tests never call real Anthropic, Voyage, R2 or Resend; they use fakes behind the Provider and storage interfaces. TDD applies throughout.
14. At registration each user receives a random three-word pseudonym (unique, independent of user id or email), used for all public attribution (see Public reviews).

## Architecture decisions

- Review screen shows the original file with highlights over Claim locations. Accepted formats v1: TXT, MD, PDFs with a text layer. Scanned PDFs are out of scope; the user is told clearly when a PDF has no text.
- Anchors: TXT = character range; PDF = page plus word bounding boxes (pdf.js overlays); all with exact quote plus prefix and suffix (W3C text-quote selector idea) so Claims can be re-found if extraction shifts. MD is rendered by default with quote-anchored highlights and a raw-source toggle (confirmed by the owner 2026-10-25). PyMuPDF word boxes are from prior knowledge, not yet checked.
- Reviews are progressive: the Document opens immediately; Claims are highlighted and scored as the job finishes them, streamed via server-sent events. Review Job and per-Claim progress live in Postgres, so a closed tab does not stop the job; the page shows full progress on return; streams are resumable; failures keep partial results and are retryable. The job uses the user's decrypted Credential at run time.
- Depth is chosen before the run. Each Depth is a superset of the one below, so upgrading only processes new Claims.
- Embeddings are required and are sentence-level (owner decision 2026-10-25). Estimated database size at 10–20 books plus hundreds of PDFs: ~2 GB with vectors only, ~2.4 GB with an HNSW index (estimates). Queries are scoped to one Document, so v1 uses exact (non-indexed) vector search filtered by document id and builds no global HNSW index, which avoids the index's memory needs (HNSW degrades sharply when it does not fit in RAM). Cross-document search would need an index later. halfvec (16-bit) could halve vector size; availability on Render's pgvector version unchecked. Embedding cost is negligible (~$0.003 per 100,000-word book at $0.02 per million tokens).
- Wolfram|Alpha is deferred: development proceeds without it.

## Open decisions

1. Seed Authorities for Finance and Business (candidates not yet researched).
2. Public reviews: redaction and secrets-scanning tools (step 14 research); lawyer review of the terms page before a wider public launch.
3. Email: domain verification in Resend is postponed by the owner; revisit before letting anyone but the owner register.
4. Wolfram|Alpha: snapshot storage terms and key model (deferred).
5. National Archives case-law licence application, before building that Authority.
6. Unverified items to check in the scaffold step: R2 with boto3, PyMuPDF word boxes, Render prices and Docker settings, BM25 extension availability, the pgvector Docker image name, halfvec support.
