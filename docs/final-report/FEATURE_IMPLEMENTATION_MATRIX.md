# FinalSay Feature Implementation Matrix

**Assessment date:** 12 November 2026. “Fully implemented and tested” means relevant code and automated tests exist and passed in this audit; it does not imply a production deployment or real user activity.

| Feature | Status | Evidence and current limits |
|---|---|---|
| Student self-registration | **Implemented and tested** | `/api/auth/register`, bcrypt hashing, forced `student` role; auth tests cover login and `/me`. Current database has zero users; no live user registration is evidenced. |
| Login/JWT session | **Implemented and tested** | OAuth2 form login returns HS256 bearer JWT; client stores token in localStorage and loads `/auth/me`. No refresh-token flow. |
| Admin/reviewer/issuer provisioning | **Implemented, operator-driven** | Provisioner scripts and Admin issuer-membership API. Public registration cannot self-assign privileged roles. Current DB has zero users. |
| Role-based pages/API permissions | **Implemented and tested** | React protected routes and `require_role`; tests cover protected access/privilege assignment. Admin UI is source/provenance control, not a complete staff dashboard. |
| Student text submission | **Implemented and tested** | Multipart form `/api/ingest/submit`, extraction and comparison pipeline. Current DB has zero submissions. |
| PDF/image submission | **Implemented, environment-dependent** | PDF text extraction with bounded OCR fallback; image OCR through Tesseract. OCR absence returns 422 asking for text. No explicit upload byte limit was found in the submit route; bytes are read in memory. |
| Text extraction, redaction, fields | **Implemented and tested** | `services/extraction.py`: dates/issuer/deadline/audience/action and redaction patterns. Regex extraction/redaction is imperfect and needs multilingual/document-specific validation. |
| Semantic candidate retrieval | **Implemented and tested** | MiniLM embeddings plus lexical/title ranking; same-institution current official scope; top-K. Earlier isolated DB evidence demonstrates a controlled cross-URL revised-title candidate. Current DB contains no submissions/edges. |
| NLI comparison / categories | **Implemented and tested** | DeBERTa NLI, evidence/rationale, relation taxonomy and confidence gate. Scores are not calibrated probabilities; accuracy on representative live notices is not established. |
| Reviewer queue and resolution | **Implemented and tested in isolated tests** | Queue/case APIs and reviewer UI; unresolved outcomes can route to review. Current relation edge and review case tables are empty. |
| Blind reviewer benchmark | **Implemented, not operationally populated** | Account-scoped annotations and completion fingerprints; Kappa requires two reviewers with overlap. DB: 45 pairs, zero annotations, zero users; no actual Kappa. |
| Source-specific scraping | **Implemented; currently degraded** | Six adapters/types and source safety controls. Three profiles are active; latest selected-source runs have errors. Only VTU has `ok` runs in this snapshot. |
| Scheduled and manual scraping | **Implemented** | Startup and two-hour interval scheduler; Admin manual poll; lock and audit rows. Current run table has 269 events (49 ok, 145 partial, 75 error). Scheduled uptime/future timer behavior is not a deployment guarantee. |
| Source backfill and duplicate handling | **Implemented and tested** | Bounded cursors, source keys, partial unique current-key index, unchanged/revised upsert. Current selected source rows show no duplicate current keys. VTU/Delhi/Mumbai completion state is per DB cursor and may remain incomplete. |
| Same-URL version history | **Implemented and tested** | Prior row is retained and new row linked through `previous_version_id`, `version_no`, and `is_current`. A changed raw file hash supports new version even when OCR text is the same. |
| Cross-URL revision linkage | **Partially implemented** | Candidate retrieval/comparison can find and classify a new-URL notice. Scraper stores distinct source key as distinct notice; no universal durable cross-URL relationship is inferred during ingestion. |
| Issuer notice publishing | **Implemented; real use unverified** | `/api/issuer/publish` checks issuer role and active institution grant, persists official notice, and builds Merkle root. No issuer account or membership exists in current DB. |
| Admin source management | **Implemented** | Source list/configuration, enable/disable, manual polling, run history. Source creation only permits fixed adapter types/hosts. |
| Provenance/local anchors | **Implemented and persisted** | SHA-256, Merkle roots/proofs, local hash chain; current DB has 105 local roots and 54,516 proof rows. Integrity proof does not guarantee source authenticity. |
| Solidity/Polygon anchoring | **Implemented locally, public deployment absent** | Solidity contract and Web3 adapter; Foundry test suite passes 4 tests. Current DB has zero Polygon roots; no deployment/transaction evidence. |
| Student/Admin provenance display | **Implemented** | Routes and UI components exist. Current roots are local-only. |
| Search/filters/analytics | **Partially implemented** | Notice listing, candidate ranking, queue, benchmark status/Kappa UI exist. No broad institution analytics or established performance metrics. |
| PWA | **Implemented** | Vite PWA plugin, manifest/service-worker build; no native mobile app. |
| Production Docker/AWS config | **Prepared, not verified/deployed** | Docker image, Compose, ECS task/workflows and production setting checks exist. No cloud deployment is established; local Docker/AWS execution is outside this test. |
| Complete product workflows on real users | **Not verified** | Current database has no users/submissions/edges/annotations. Earlier temporary DB tests are useful integration evidence but not a current production-like journey. |

## Audit results

- Backend: 202 passed; 4 warnings from installed framework/dependency deprecations.
- Frontend: TypeScript/Vite/PWA build passed.
- Solidity: Foundry 1.8.7, four contract tests passed.
- Current local SQLite database: integrity `ok`; all stored notice kinds are `official`; 2,164 current and 120 historical.
- Latest source state: VTU/Delhi/Mumbai enabled; recent run records include DNS errors and `OSError: [Errno 9] Bad file descriptor`.

## Important distinction

The codebase's automated tests frequently use isolated temporary databases, deterministic mock comparison, mocked HTTP and synthetic fixtures. Such tests are valid evidence of logic and API behavior, but they do not create real users, prove source freshness, or measure model accuracy on future production traffic. This report uses separate status language for code/tests, current database observations, and hosted operation.
