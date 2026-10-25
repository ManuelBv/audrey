# Seed Authority Candidates (researched 2026-10-25)

Candidates for the Registry. Approved by the owner on 2026-10-25 as the starter seed set; entries are created per Authority later, after verifying licence, API and URL. Gaps below still need follow-up searches.

Tags: **[S]** supported by the web search done that day; **[K]** from general knowledge only, not verified. The search results were mostly library guides and course notes, not the sources' own sites, so licence, API and exact URL must be verified when an entry is created. Domains were chosen by Claude as a starter set, not specified by the owner.

| Domain | Candidates (up to 10) |
|---|---|
| Medicine | Cochrane Library [S]; PubMed/MEDLINE [S]; AHRQ and USPSTF [S]; TRIP Database [S]; WHO [K]; NICE [K]; CDC [K]; FDA [K]; EMA [K]; ClinicalTrials.gov [K]. Modifier: Retraction Watch. Paid, excluded for now: BMJ Clinical Evidence, DynaMed [S]. |
| Biology | UniProt [S]; GenBank/INSDC [S]; Ensembl [S]; wwPDB [S]; PubMed Central [K]; NCBI Taxonomy [K]; Gene Ontology [K]; Reactome [K]; KEGG [K, recalled restrictive licensing]; bioRxiv [K, preprints, lower weight]. |
| Physics and astronomy | NIST CODATA constants [S]; Particle Data Group [S]; JPL Horizons [S]; NASA ADS [S, weak]; BIPM SI Brochure [K]; IAU [K]; NASA Exoplanet Archive [K]; CERN Open Data [K]; arXiv [K, preprints]; NIST Physical Measurement Lab data [K]. |
| UK law | legislation.gov.uk [S]; National Archives Find Case Law [S]; BAILII [S]; Hansard [S]; Law Commission [S]; UK Parliament APIs [K]; UK Supreme Court judgments [K]; judiciary.uk [K]; HM Land Registry [K, relevant to leasehold]; gov.uk guidance [K]. |
| Engineering | ISO [S]; BSI/British Standards Online [S]; Eurocodes via CEN [S]; ASTM [S]; ASCE [S]; AASHTO [S, weak]; IEEE standards [K]; NIST [K]; public MIL-STD documents via ASSIST [K]; Institution of Civil Engineers [K]. |

## Gaps and conflicts

- Searches returned nothing for WHO, CDC, NICE, FDA, EMA, IEEE, NIST, MIL-STD/ASSIST, UK Parliament APIs, HM Land Registry, Gene Ontology, KEGG, bioRxiv, arXiv or the IAU.
- Sources disagree on whether BAILII records have moved to the National Archives site; check the live sites.
- Most engineering standards are paywalled, so Audrey may be able to cite them but not read them.

## Access findings for free-only checks (researched 2026-10-25)

Owner rule: v1 uses only Authorities that can be queried free of charge. Third-party sources unless noted; confirm on each official page when creating the entry.

| Authority | Free access finding |
|---|---|
| PubMed E-utilities | No key needed; 3 requests/second, 10 with a free NCBI key. A scraper vendor claims a 9,999-record cap per query (unverified). |
| Crossref | Free API with rate limits advertised in response headers (`X-Rate-Limit-Limit`, `X-Rate-Limit-Interval`); source is an older ropensci page. |
| Semantic Scholar | Most endpoints work unauthenticated with a shared rate limit; limits unclear and may change. |
| Europe PMC, OpenAlex | Nothing usable found; check official docs. |
| legislation.gov.uk | Content under Open Government Licence v3.0 (site's own pages); some material has other terms (EUR-Lex-derived, Westlaw-contributed). API long described as free to re-use; check the Developer section. |
| National Archives Find Case Law | Judgments reusable under the Open Justice Licence, including commercially, but bulk programmatic searching ("computational analysis") needs a free licence application assessed by the National Archives. Whether per-claim lookups count is unclear; contact caselawlicence@nationalarchives.gov.uk. |
| Congress.gov API (US Library of Congress) | Free API key from api.congress.gov/sign-up (via api.data.gov), passed as `api_key`; JSON must be requested explicitly (XML is default). Sources are third-party plus a 2022 LoC blog post. |
| loc.gov general API | Terms not found. |
