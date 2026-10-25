# Audrey — Progress

Live step tracker. Read at the start of every session. Updated once per feature, after the owner has done a manual check and confirmed the Reviewer's PASS.

Each step becomes a folder `docs/features/YYYY-MM-DD-NNN-feature-name/` created by the Speccer, which also chooses the Full or Short route.

## Status

Planning complete for the project level (plan.md). No code yet. No feature specced.

## Steps (scaffold first confirmed; remaining order open to discussion by number)

| # | Step | Status |
|---|---|---|
| 1 | Scaffold: repo layout, frontend and backend skeletons, tests, CI, `render.yaml` | Not started |
| 2 | Registration (incl. three-word pseudonym, email verification), login, password reset, admin approval panel and daily digest | Not started |
| 3 | Provider settings and encrypted Credential store | Not started |
| 4 | Document upload, storage, text extraction with Anchors | Not started |
| 5 | Document viewer with Claim highlights (PDF, TXT, MD) | Not started |
| 6 | Registry loader and Registry Suggestion queue | Not started |
| 7 | Retrieval: BM25, then embeddings, Contextual Retrieval | Not started |
| 8 | Domain selection and inference, Claim extraction | Not started |
| 9 | Bayesian Score engine (pure logic) | Not started |
| 10 | Evidence retrieval per Domain and progressive Review Jobs | Not started |
| 11 | Accept flow and immutability | Not started |
| 12 | Document chat | Not started |
| 13 | Later: document logic and implications layer | Not started |
| 14 | Public reviews: publish and unpublish an accepted Review, public listing, read-only view (depends on 4, 5, 11) | Not started |

## Next

Order confirmed by the owner on 2026-10-25: scaffold (step 1) first; later steps are discussed and referred to by number. US law domain skipped for now. Next: run the Speccer on step 1.

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
