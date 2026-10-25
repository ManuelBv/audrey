# Audrey — Progress

Live step tracker. Read at the start of every session. Updated once per feature, after the owner has done a manual check and confirmed the Reviewer's PASS.

Each step becomes a folder `docs/features/YYYY-MM-DD-NNN-feature-name/` created by the Speccer, which also chooses the Full or Short route. Decisions and rules live in [plan.md](plan.md); every step follows its cross-cutting rules and the "no functionality without its corresponding tests" rule (unit, integration and end-to-end).

## Status

Project-level planning is complete (plan.md). No code yet and no feature specced. Working rule: commit often, push only when the owner says so.

## Steps

Scaffold first (confirmed by the owner 2026-10-25); the remaining order is open to discussion by number. "Depends on" lists steps whose output the step needs.

| # | Step | Depends on | Status |
|---|---|---|---|
| 1 | Scaffold: repo layout (frontend, backend), Dockerfile and docker compose with a test database, `render.yaml`, `.env.example` with fail-fast config, provider and storage interface skeletons with fakes, pre-commit secret scan, pre-push full test run, CI, unit/integration/end-to-end test harnesses, test-database guard | none | Not started |
| 2 | Users: registration with three-word pseudonym and email verification, login, password reset, user statuses (pending, approved, rejected, suspended), admin bootstrap, admin approval panel with audit table, daily admin digest, email sender interface (Resend adapter), rate limiting on registration and login | 1 | Not started |
| 3 | Provider settings and encrypted Credential store: chat and embedding Provider and model per user, Anthropic and Voyage adapters (OpenAI embedding adapter second), AES-256-GCM storage, write-only keys, test call on save | 1, 2 | Not started |
| 4 | Documents: upload (TXT, MD, text-layer PDF), validation and malware scan, R2 storage with SHA-256, text extraction with Anchors, clear message for PDFs without text, upload rate limiting | 1, 2 | Not started |
| 5 | Document viewer with Claim highlights: PDF via pdf.js overlays, TXT, MD rendered by default with raw toggle | 4 | Not started |
| 6 | Registry and domain tree: machine-readable files synced to the database, Authority access classes, seed Authorities for the approved domains (free-only), Registry Suggestion queue (`USER` feedback form, `AI_RESEARCHER`), owner approval by commit | 1, 2 | Not started |
| 7 | Retrieval: Chunks with contextual retrieval, BM25 plus sentence-level embeddings (exact search scoped to one Document), `BM25_ONLY` fallback with banner, stale-embedding block and re-embed with cost estimate | 3, 4 | Not started |
| 8 | Domain selection (`DECLARED`/`INFERRED`, parent fallback, low-coverage warning) and Claim extraction pipeline (checkable definition, atomic Claims, Claim Types, user override) | 3, 6, 7 | Not started |
| 9 | Bayesian Score engine (pure logic): priors, likelihood ratios, log-odds, Independence Clusters, Evidence Coverage, Interpretations, roll-ups, clamp 1–99 | 1 | Not started |
| 10 | Evidence retrieval per Domain and progressive Review Jobs: Depth choice with cost estimate, server-sent events, resumable progress in Postgres, per-step model records | 3, 7, 8, 9 | Not started |
| 11 | Accept flow and immutability: immutable Claims and Snapshots, reopening reads from the database, re-run creates a new Review | 10 | Not started |
| 12 | Document chat over the Document using the same retrieval | 7 | Not started |
| 13 | Later: document logic and implications layer (flag and explain, with disclaimer) | 11 | Not started |
| 14 | Public reviews: 10-minute scheduled publish with undo, redacted excerpt-only snapshot, caps (25 words, 5%, 400 words), three-word pseudonym attribution, unpublish, admin takedown with audit log, report button, no-index, disclaimer, read-only public endpoints | 4, 5, 11 | Not started |

## Parked or deferred

- Seed Authorities for Finance and Business (candidates not yet researched).
- Wolfram|Alpha (snapshot storage terms, key model) and the National Archives case-law licence application (both deferred by the owner).
- Email domain verification in Resend (postponed; revisit before anyone but the owner registers).
- Redaction and secrets-scanning tools, and a lawyer's review of the terms page (step 14).
- Out of scope for now: US law domain, scanned PDFs, cross-document search (needs a global vector index), Qdrant.

## Next

Run the Speccer on step 1.

## Research owed by step

Facts recorded in plan.md from memory or third-party sources that the named step's Researcher must verify against official documentation before relying on them.

| Step | To verify |
|---|---|
| 1 | Render prices (web Starter, Postgres Basic-256mb, 5 GB storage) and Docker service settings in `render.yaml`; pgvector Docker image name and `CREATE EXTENSION vector` on Render; test and hook tooling (pytest, Vitest, Playwright, pre-commit, secret scanner); CI on GitHub Actions for a public repo |
| 2 | Password hashing library settings (argon2id); session handling behind Render's proxy |
| 4 | R2 with the Python S3 client (boto3), including presigned URLs; PyMuPDF word-level bounding boxes; malware scanning tool for uploads |
| 6 | Registry and domain-tree file format |
| 7 | BM25 extension availability on Render Postgres, otherwise a Python BM25 library; halfvec support; Voyage model names, dimensions and current prices |
| 14 | Presidio accuracy on English legal text; a secrets scanner (detect-secrets or similar) |
