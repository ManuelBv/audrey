# Audrey — Ontological Document Review Engine (ODRE): Draft Spec

Status: DRAFT for review. No code written yet. Everything here comes from the design conversation of 2026-10-25.

## 1. Purpose

Upload or paste a document (PDF, TXT; legal, scientific, books). Audrey extracts claims at subsentence, sentence, paragraph, section and page level and compares each against authoritative sources. Every claim receives a Bayesian Score (BS), never a true/false verdict.

Naming note: "ontological" is used loosely (empirical, factual, real-world). Strictly, the fit is "empirical"/"epistemic". Name expansion is an open decision (see section 12).

## 2. Bayesian Score (BS)

- Integer 1–99. Never 0 or 100 (absolutist claims are not scientific). 1 ≈ almost certainly false, 99 ≈ almost certainly true.
- Computed, not guessed by an LLM: prior (by domain and claim type) updated by per-evidence likelihood ratios, combined in log-odds, clamped to 1–99.
- Evidence items are clustered by independence cluster before combining, so sources deriving from the same underlying data count once.
- Each claim shows two numbers: the score, and evidence coverage / credible interval (a 50 from no evidence differs from a 50 from conflicting strong studies).
- Until calibrated against labelled benchmarks (FEVER, SciFact or similar), the UI labels it an "evidence-weighted score", not a probability.
- Ambiguous claims are split into interpretations, each scored separately (e.g. "the sun revolves around the earth": acceptable as a coordinate-frame description, false as a physical description).
- Claim types: empirical-descriptive, empirical-causal (graded by study design), legal (what the text says, in force, jurisdiction, date; own sub-score), and non-scorable (opinion, prediction, definition).
- Roll-up from claim to paragraph, section, page, document is not a plain average; it accounts for how load-bearing each claim is.

## 3. Domains

- Tree with multiple roots: Science (biology, physics, ...), Engineering (civil, military, ...), Medicine, Law, others.
- Document-level domain by default; override per chapter, section or claim (a book may span several roots).
- At upload the user is asked for the domain. "Don't know" → domain inferred from text and the review is marked **INFERRED**, with top candidates, confidence, and the fixed warning that the review may not apply to the original domain. Declared → **DECLARED**. Low-confidence inference is shown, not silently resolved.
- Domain controls word-sense disambiguation (e.g. "induce" in psychology vs "inductance" in electronics), source routing, priors and source weights.

## 4. Sources and the registry

Tiers:

| Tier | Sources | Weight |
|---|---|---|
| 1 Authoritative | Peer-reviewed literature, systematic reviews, legislation and regulation, government statistics, standards, technical manuals | Highest |
| 2 Computed knowledge | Wolfram\|Alpha (via its API; see Wolfram notes below) | High for quantitative/physical facts |
| 3 Other web | Institutional pages, reference works, encyclopedias (Wikipedia = tertiary: follow citations to the primary source and score that) | Lower, capped |
| 4 News | All outlets classed as opinion | Minimal; if a news item cites a primary document, trace and score the primary |

Weights live in config so they can be tuned.

Wolfram|Alpha notes (researched 2026-10-25 from products.wolframalpha.com; confirm against your API Plan Summary):
- Public info: https://products.wolframalpha.com/api (APIs), /api/pricing, /api/termsofuse, /api/documentation (Full Results API).
- Free access for non-commercial use, advertised as up to 2,000 calls a month. Full Results API is $50 per 1,000 queries with a minimum prepayment, attribution and a contract. LLM API price is not published (contact Wolfram).
- Terms prohibit using the services to train or fine-tune AI models or to build training datasets from service data. Required disclaimer in our own terms: results are not a certified or definitive source of information (exact wording in the terms).
- Wolfram's own citation guidance: cite "Wolfram Alpha LLC", the query and access date; most results are computed from multiple underlying sources, and the "Source information" button gives the background sources. Audrey should store the query, access date and underlying sources with each Wolfram evidence item.
- Design implications: use the API, not scraping (scraping terms not checked). Wolfram evidence counts as computed knowledge, weighted below the underlying primary source when that is identifiable. Whether snapshots of results may be stored and displayed to other users is unchecked, so keep them minimal until confirmed. Consider per-user Wolfram App IDs (BYOK) so usage and licensing sit with the user.

Source roles: **Evidence** (supports/refutes), **Modifier** (changes trust in another source, e.g. Retraction Watch), **Navigation/tertiary**.

Registry entry fields: name, URL, domains (many-to-many), source type (journal, legislation, statistics, standard, technical manual, ...), role, tier, access method (API / bulk / scrape / manual), licence and terms, version/revision (standards and manuals are valid per revision and date), independence cluster, independence flag (manufacturer manuals), provenance (added by, when, justification, origin tag), review date.

Registry governance:
- Stored as a file in the repo (YAML or JSON), loaded by the backend. One global list.
- Changes enter via suggestions with an origin tag: **User** (feedback form with reason) or **AI researcher** (in-app researcher prompt).
- Owner approves or rejects. Approval = a git commit adding the entry. Nothing affects scores before that.
- Each review stores the registry commit hash.
- Open issues: paywalled standards (ISO, IEEE), military manuals (public ones only), manufacturer bias. Licences not yet verified.

## 5. Review screen

- Left: document with highlighted claims.
- Right: sources sidebar. Per source: name, a one-line statement of whether the claim is supported/contradicted or what the alternative claim is, and the source.
- Lower right: chat panel for custom questions about the document (RAG, section 7).
- Per claim, the user clicks **Accept** to commit the claim and its sources.

## 6. Persistence and immutability

- Accepted claims and their sources are immutable in the database, per user.
- Reopening a past review reads from the database; no live AI call.
- Store with each accepted source: excerpt, URL, retrieval date, content hash, registry commit hash, score inputs (priors, likelihood ratios, clusters). Excerpts and hashes only, not full text (copyright position unverified).
- A re-run creates a new review version; old ones are preserved.

## 7. RAG and chat

- Anthropic's Contextual Retrieval design: at ingest, each chunk gets short document-specific context prepended before both embedding and BM25 indexing; retrieval combines both.
- Per-document pre-work at ingest produces claim assertions plus the document's internal logic.

## 8. Second analysis layer (phase 2)

Document logic and implications: e.g. "page 4 discusses covenants on your leasehold in England; here is what they mean and why they may matter to you". This approaches legal advice: flag and explain only, with a clear disclaimer. Built after claim verification.

## 9. BYOK and security

- User enters an LLM API key in the account area. It is stored encrypted in the backend (AES-256-GCM; encryption key outside the database, in a secrets manager or environment secret). Encrypted, not hashed, because the backend must send it to the provider.
- Write-only in the UI: after saving, show only the last 4 characters. Never logged.
- Passwords: argon2id (or bcrypt) with per-user salt.
- Standards: OWASP Top 10, ASVS, and OWASP Top 10 for LLM Applications. Uploaded documents and fetched web pages are untrusted input (prompt injection); the AI can only produce proposals and never changes scores or the registry directly.
- Uploads: type and size validation, malware scanning.
- Cost: show an estimated token cost before large runs. Prompts and pipeline are provider-neutral.

## 10. Scope: general from the start (decided 2026-10-25)

No domain-limited first slice. The user pastes text or uploads TXT/PDF, specifies the domain (or "don't know" → INFERRED), and the AI investigates according to that domain using the source registry. Medicine is the first test domain, not a scope limit. Test cases: vaccines and autism (expect a score near 1), the sun/earth reading split, and the "induce" domain disambiguation. Consequence: the registry needs seed entries for every domain Audrey is tested on.

## 11. Comparable products (researched 2026-10-25, vendor claims unverified)

Originality.ai, Winston AI, Manus, Detecting-ai.com, GetSolved.ai, scite; open source Loki, ClaimCheck, Claim_Verification_Pipeline, SciFact-style repos, aiFactCheck. RootCause.ai (also listed as Perceptura) is enterprise causal AI over structured data, not document review; its ontology idea (entities, time, location) is relevant to claim resolution.

## 12. Stack and hosting (proposed; researched 2026-10-25, third-party sources, prices unconfirmed)

- Frontend: React, Vite, TanStack Router and TanStack Query as a client-side SPA (TanStack Start not needed: FastAPI is the server). Typed client generated from FastAPI's OpenAPI schema.
- Backend: Python, FastAPI, plus a worker for ingest and review jobs (book-length reviews run for minutes).
- Database: PostgreSQL with pgvector for vectors and relational data (users, reviews, immutable accepted claims, registry feedback). Hybrid retrieval: vector plus BM25; Postgres full-text search is not true BM25, so use a BM25 extension if the host supports it, otherwise compute BM25 in the Python service. Qdrant is a later swap if scale demands; benchmark on real data in the first slice.
- Size estimate: ~100,000-word book ≈ 270 chunks ≈ 3–5 MB with index (estimate), far below pgvector's cited 1M–5M vector ceilings.
- Hosting: single provider with static site, web service, worker and Postgres. Leading candidate Render (`render.yaml` blueprint, roughly $20–33 a month at smallest always-on tiers); alternative Railway (Hobby $5 a month plus usage). Owner accepts a paid plan to keep everything in one place. Unconfirmed: pgvector availability on either host's managed Postgres (fallback: Neon for the database).
- Auth: handled in the app itself (no OAuth for now).
- Repository: https://github.com/ManuelBv/audrey (public, open source).

## 13. Architecture decisions so far (discussion log, 2026-10-25)

- Review screen shows the **original file** (PDF, TXT, MD) with highlights over claim locations; not an extracted-text view.
- Accepted file formats v1: TXT, MD, and PDFs **with a text layer**. Scanned/image PDFs are out of scope; the user is told clearly when a PDF has no text.
- Claim anchors: TXT = character-offset range; PDF = page number plus word bounding boxes in PDF coordinates (rendered by pdf.js as overlays); every anchor also stores the exact quote plus a few words of prefix and suffix (W3C text-quote selector idea) so claims can be re-found if extraction shifts. MD is rendered with quote-anchored highlights, with a raw-source toggle (default, unconfirmed).
- The original file is stored and its SHA-256 recorded on every review.
- Review runs are **progressive**: the document opens immediately and claims are highlighted and scored as the worker finishes them, streamed via server-sent events. Job and per-claim progress live in Postgres, so a closed tab does not stop the job; on return the page shows full progress from the database. Streams must be resumable after reconnect. The worker needs the user's decrypted API key at run time; failures keep partial results and are retryable.
- Review **depth is chosen before the run** (tiered, to control the user's cost): Quick = major claims per section; Standard = every checkable sentence; Deep = every checkable subsentence claim plus paragraph/section/page roll-ups. Each tier is a superset of the one below, so upgrading a finished review only processes the new claims. A cost estimate is shown before each run.
- Unchecked: PyMuPDF word-box output (from prior knowledge), file storage option (Postgres large objects vs persistent disk vs S3-compatible object storage) and its cost on Render.

## 14. Open decisions

Decided 2026-10-25: ODRE name expansion stays ("Ontological Document Review Engine"); no limited first slice (section 10); licence AGPL-3.0 (see LICENSE).

1. pgvector availability on the chosen host (section 12).
2. Wolfram|Alpha: storage/display of result snapshots, and per-user App IDs vs one owner key (commercial terms may apply).
3. Seed source lists per domain (registry content).
4. Architecture plan and data model.
