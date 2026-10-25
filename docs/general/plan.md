# Audrey — Plan

Architecture decisions, stack and ubiquitous language. Written collaboratively in the first working session (2026-10-25). Revisit only when an architectural decision actually changes.

Product behaviour (what Audrey does, BS scoring, source tiers, review screen) lives in [../spec.md](../spec.md). The live step tracker is [progress.md](progress.md). Candidate authoritative sources per domain: [../research/seed-source-candidates.md](../research/seed-source-candidates.md).

ODRE = Ontological Document Review Engine. Repository: https://github.com/ManuelBv/audrey (public, AGPL-3.0).

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
| Depth | Quick (major claims per section), Standard (every checkable sentence), Deep (every checkable subsentence claim plus roll-ups). |
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
- Backend: Python, FastAPI, plus a worker process for ingest and Review Jobs.
- Database: PostgreSQL with pgvector (supported on Render Postgres per its docs; enable with `CREATE EXTENSION vector;`). Relational data, vectors and job state in one database. BM25: Postgres full-text search is not true BM25, so use a BM25 extension if available on the host, otherwise compute BM25 in the Python service. Qdrant is a later swap only if measured need appears.
- Hosting: Render, defined by `render.yaml` at the repo root with `rootDir` and `buildFilter` per service. Owner will pay for the always-on tier. v1 runs one backend web service with background jobs in-process (database-backed queue) plus managed Postgres and a static frontend; a separate worker comes later if needed. Estimated monthly cost (third-party figures, unverified against render.com): web Starter ~$7 + Postgres Basic-256mb ~$6 + 5 GB storage at $0.30/GB = $1.50, so ~$14.50. Render storage is billed separately from instance type, in multiples of 5 GB, and cannot be reduced. Alternative: Railway. Fallback database: Neon free tier (too small for sentence-level vectors).
- Auth: in-app registration and login, admin approval of new users, no OAuth for now.

## Cross-cutting rules (apply to every feature)

1. Every table carries `user_id`. The app starts with one user (the owner) but has full registration, login and an admin panel where the admin approves users. The code is AGPL and may be self-hosted by others.
2. Accepted Claims and their Evidence are immutable. Reopening a Review reads from the database and makes no live AI call. A re-run creates a new Review.
3. Each Review records: file SHA-256, Registry commit hash, Domain provenance (`DECLARED`/`INFERRED` plus candidates and confidence), Retrieval Mode, Depth, embedding model and chat model.
4. Provider abstraction: chat and embeddings each sit behind an interface with one adapter per Provider. v1 chat Provider is Anthropic. Embedding Provider v1: Voyage AI (confirmed by the owner 2026-10-25), with OpenAI text-embedding-3-small as the second adapter.
5. Credentials: encrypted (AES-256-GCM) with the key held outside the database, never hashed because the backend must use them; write-only in the UI (last 4 characters shown); validated with a test call on save; never logged. Passwords: argon2id with per-user salt. Standards: OWASP Top 10, ASVS, OWASP Top 10 for LLM Applications.
6. Uploaded Documents and fetched web content are untrusted input (prompt injection). The AI only produces proposals; it never changes scores or the Registry directly. Uploads: type and size validation, malware scanning.
7. Embeddings record model name and dimension. Changing the embedding model means re-embedding, shown with a cost estimate and confirmed first. Stale embeddings block chat and Review until confirmed. With no embedding key, run `BM25_ONLY` with a visible banner.
8. A cost estimate is shown before each Review run and before any re-embed.
9. At registration each user receives a random three-word pseudonym (unique, independent of user id or email), used for all public attribution. Public reviews use a separate redacted snapshot (see spec section 12).

## Architecture decisions

- Review screen shows the original file with highlights over Claim locations. Accepted formats v1: TXT, MD, PDFs with a text layer. Scanned PDFs are out of scope; the user is told clearly when a PDF has no text.
- Anchors: TXT = character range; PDF = page plus word bounding boxes (pdf.js overlays); all with exact quote plus prefix and suffix (W3C text-quote selector idea) so Claims can be re-found if extraction shifts. MD is rendered by default with quote-anchored highlights and a raw-source toggle (owner informed 2026-10-25, no objection). PyMuPDF word boxes are from prior knowledge, not yet checked.
- Reviews are progressive: the Document opens immediately; Claims are highlighted and scored as the worker finishes, streamed via server-sent events. Review Job and per-Claim progress live in Postgres, so a closed tab does not stop the job; the page shows full progress on return; streams are resumable; failures keep partial results and are retryable. The worker uses the user's decrypted Credential at run time.
- Depth is chosen before the run. Each Depth is a superset of the one below, so upgrading only processes new Claims.
- Retrieval: Anthropic's Contextual Retrieval design (chunk context prepended before both embedding and BM25), hybrid retrieval. Embeddings are required and are **sentence-level** (owner decision 2026-10-25, revising an earlier chunk-level lean). Estimated database size at 10–20 books plus hundreds of PDFs: ~2 GB with vectors only, ~2.4 GB with an HNSW index (estimates). Queries are scoped to one Document, so v1 uses exact (non-indexed) vector search filtered by document id and builds no global HNSW index, which avoids the index's memory needs (HNSW degrades sharply when it does not fit in RAM). Cross-document search would need an index later. halfvec (16-bit) could halve vector size; availability on Render's pgvector version unchecked. Embedding cost is negligible (~$0.003 per 100,000-word book at $0.02 per million tokens).
- Settings page: per user, choose chat Provider and model and embedding Provider and model, and enter a Credential for each. The same mechanism will hold other service keys later.
- Registry governance: a file in the repo, one global list. Suggestions (origin `USER` via feedback form with a reason, or `AI_RESEARCHER`) are reviewed by the owner; approval is a git commit. Roles of an Authority: Evidence, Modifier (e.g. Retraction Watch), Navigation/tertiary (e.g. Wikipedia, followed to primary sources).
- Authorities are defined in the repo (machine-readable Registry file, source of truth) and synced into the database so the LLM can use them. v1 includes only Authorities that can be queried free of charge (paid APIs excluded for now). Each Authority records an access class (open, free key, licence needed, paywalled); free-key Authorities use an owner-held service key, not a per-user Credential.
- Wolfram|Alpha is deferred: development proceeds without it. Notes on its terms are in the spec.
- Original Documents are stored in Cloudflare R2 (owner decision 2026-10-25; free tier 10 GB per third-party sources, no egress fees) behind a storage interface so the provider can change. Expected volume: 10–20 books plus dozens to hundreds of PDFs (~1 GB, estimate). The database holds only metadata, text chunks and vectors. R2 compatibility with boto3 is untested and must be checked in the scaffold step.

## Open decisions

1. Wolfram|Alpha: snapshot storage terms and key model (deferred).
2. (decided) MD shows rendered by default with a raw toggle.
