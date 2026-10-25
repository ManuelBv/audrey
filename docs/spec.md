# Audrey — Ontological Document Review Engine (ODRE): Draft Spec

Status: DRAFT for review. No code written yet. Everything here comes from the design conversation of 2026-10-25.

## 1. Purpose

Upload or paste a document (PDF, TXT; legal, scientific, books). Audrey extracts claims at subsentence, sentence, paragraph, section and page level and compares each against authoritative sources. Every claim receives a Bayesian Score (BS), never a true/false verdict.

Naming note: "ontological" is used loosely (empirical, factual, real-world). Strictly, the fit is "empirical"/"epistemic". The name expansion "Ontological Document Review Engine" was kept by the owner on 2026-10-25.

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

## 9. Scope: general from the start (decided 2026-10-25)

No domain-limited first slice. The user pastes text or uploads TXT/PDF, specifies the domain (or "don't know" → INFERRED), and the AI investigates according to that domain using the source registry. Medicine is the first test domain, not a scope limit. Test cases: vaccines and autism (expect a score near 1), the sun/earth reading split, and the "induce" domain disambiguation. Consequence: the registry needs seed entries for every domain Audrey is tested on.

## 10. Comparable products (researched 2026-10-25, vendor claims unverified)

Originality.ai, Winston AI, Manus, Detecting-ai.com, GetSolved.ai, scite; open source Loki, ClaimCheck, Claim_Verification_Pipeline, SciFact-style repos, aiFactCheck. RootCause.ai (also listed as Perceptura) is enterprise causal AI over structured data, not document review; its ontology idea (entities, time, location) is relevant to claim resolution.


## 11. Architecture and decisions

Moved to [general/plan.md](general/plan.md).

## 12. Public reviews (decided 2026-10-25; step 14)

- A user can publish an accepted Review. The public page uses the same layout as the private screen but is read-only and never includes the original file: it shows short excerpts written `[...] several words [...]`, followed by the Evidence, scores and explanation. Excerpt length and total share of the Document are capped (numbers to be set in the step-14 spec; product rule, not a legal safe harbour); title, author and page of the Document are acknowledged.
- **Friction:** publishing moves a Review from `private` to `scheduled` with a visible countdown (default 10 minutes, unconfirmed) and an Undo button, then to `public`. Public listing only shows reviews whose scheduled time has passed.
- **Redaction:** at publish time, text that looks like personal data or secrets is redacted from the public copy (candidate tool: Microsoft Presidio, which gives no completeness guarantee and does not detect secrets; a separate secrets scanner is needed). The user sees a preview of the redactions and confirms. The interface states that detection can miss items.
- The public copy is a separate stored snapshot, so unpublishing deletes it. Private data is untouched. The user can unpublish at any time; the public URL then stops working. Copies already saved by others or indexed by search engines cannot be recalled; public pages carry a no-index tag by default.
- **Anonymity:** each user gets a random three-word pseudonym at registration (curated word list, generated independent of user id or email, unique-indexed). Only the pseudonym is shown publicly; the real name stays in the private profile.
- **Moderation:** an admin can take down any public review with a reason and audit log entry; viewers get a report button; the terms need a takedown process.
- Public pages carry the standing disclaimer that scores are AI-generated, evidence-weighted outputs and not a certified or definitive source.
- Public endpoints are separate read-only endpoints returning only published fields (no user ids, chat, keys or unpublished claims).
- Copyright (US fair use, UK fair dealing for criticism/review/quotation) and personal-data questions are summarised from web searches; get legal advice before a wider public launch.
