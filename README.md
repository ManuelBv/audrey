# Audrey

**Audrey** is a planned open-source *Ontological Document Review Engine* (ODRE): upload a document, and Audrey checks what it claims against authoritative sources, then shows how well the evidence supports each claim.

> **Status: planning only. There is no code yet.** The design is written down in [docs/general/plan.md](docs/general/plan.md) and the build order in [docs/general/progress.md](docs/general/progress.md). Nothing in this README is implemented today.

## What it will do

- You paste text or upload a document: plain text, Markdown, or a PDF with a text layer (legal texts, scientific papers, books).
- You say which field it belongs to (medicine, law, engineering, finance, and so on), or let Audrey infer it. Reviews always show whether the field was declared or inferred, because the same word can mean different things in different fields.
- Audrey extracts the claims in the document, from whole sentences down to parts of sentences.
- Each claim is compared with a curated list of authoritative sources (peer-reviewed literature, legislation, standards, official statistics). Only free, public sources are used at first.
- The document is shown with the claims highlighted. Next to it you see the evidence found, a one-line verdict, and the alternative claim where the evidence points elsewhere.
- You accept claims one by one. Accepted claims and their evidence are saved and never change, so reopening an old review does not call an AI again.

"Ontological" here means empirical and factual: what holds up in the real world.

## The Bayesian Score

Audrey never says true or false. Each claim gets a **Bayesian Score from 1 to 99**: close to 1 means the evidence points strongly against it, close to 99 strongly in its favour. It is never 0 or 100, because absolute certainty is not scientific.

The score is calculated from a prior and the strength of each piece of evidence, not guessed by a language model, and it is shown next to how much evidence there was, since a 50 from no evidence is not a 50 from conflicting strong studies. Until it has been checked against labelled benchmarks it is presented as an evidence-weighted score, not a probability. Opinions, predictions and definitions are marked "not scorable" instead of being given a number.

## Design principles

- **Authoritative sources are a reviewed list in this repository.** Users and an AI researcher can suggest additions, but the owner approves each one by commit. Journals and legislation rank above reference works, and news is treated as opinion.
- **Bring your own keys.** Users add their own language-model and embedding-provider keys. Keys are stored encrypted on the server and are never shown again. Providers sit behind interfaces so more can be added.
- **Accepted results are immutable** and record which models did the research.
- **Public reviews are optional and cautious.** A review can be published with a delay and an undo, shows only short excerpts (never the original file), redacts anything that looks like personal data, and appears under a random three-word pseudonym. An admin can take any public review down.
- **No account without approval.** Registration is gated by an admin, and sensitive actions are rate limited.
- **Tests come first.** No feature ships without unit, integration and end-to-end tests, and database tests never touch the production database.

## Planned stack

React, Vite and TanStack Router on the front end; Python and FastAPI on the back end; PostgreSQL with pgvector for data and search (BM25 plus embeddings); Cloudflare R2 for stored files; Docker locally; Render for hosting. These are plans, several still to be verified against official documentation (see the "Research owed by step" table in [progress.md](docs/general/progress.md)).

## Roadmap

The work is split into 14 numbered steps, starting with the project scaffold. See [docs/general/progress.md](docs/general/progress.md) for the order, what each step depends on, and what is deferred.

## Documentation

| File | Contents |
|---|---|
| [docs/general/plan.md](docs/general/plan.md) | Product behaviour, shared vocabulary, stack, cross-cutting rules, open decisions |
| [docs/general/progress.md](docs/general/progress.md) | Step tracker, deferred items, research still owed |
| [docs/research/](docs/research/) | Candidate authoritative sources, comparable products, notes on Wolfram\|Alpha |

## Contributing

The project is at the planning stage and is not yet open for code contributions. Comments on the plan are welcome as issues once the repository is accepting them.

## Licence

[GNU Affero General Public License v3.0](LICENSE). If you run a modified version of Audrey as a network service, you must offer its source to the users of that service.
