# FinalSay Security and Scalability Audit

**Assessment date:** 12 November 2026. This is a repository/code-and-local-data review, not a penetration test or cloud configuration audit.

## Security findings

### Positive controls

- Passwords are hashed with bcrypt via Passlib; no plaintext password is persisted.
- Public registration cannot set privileged roles; Admin/reviewer/issuer accounts are operator-provisioned.
- JWT bearer authentication and role dependencies protect operational routes; notice reads are owner-scoped where appropriate.
- SQLAlchemy ORM and parameterized expressions are used for normal database queries; no string-built SQL query from user notice text was identified in reviewed app paths.
- Production settings reject the built-in/short JWT secret, SQLite, wildcard/non-HTTPS CORS origins, mock comparison mode, and disabled scheduled polling.
- Source fetching uses HTTPS, exact adapter host allowlists, public-address resolution, robots checks, constrained redirects, bounded response bytes/time, retries and a host request interval.
- Uploaded source files are processed in memory; redacted text and extraction artifacts are persisted, not original upload bytes.
- Error handling for registration avoids returning raw database driver errors.

### Risks and recommendations

| Priority | Finding | Evidence and recommendation |
|---|---|---|
| High | GitHub Pages workflow injects a plain HTTP backend URL | `.github/workflows/deploy-pages.yml` builds with `VITE_API_URL=http://3.230.123.9:8000/api`. An HTTPS Pages origin will generally block mixed-content API calls, and HTTP exposes requests in transit. Replace with a stable HTTPS API origin behind TLS before publishing/using that workflow. Actual deployment state is not established. |
| High | No explicit upload byte-size limit in student submit route | `api/ingestion.py` calls `await file.read()` before parsing and validates by MIME/extension, so large requests can consume memory. Enforce request/file size limits at reverse proxy and API layer, inspect file signatures, and bound pages/pixels before OCR. |
| Medium | No rate-limit/abuse control found on registration, login, or submissions | Add edge/API throttling, account/device abuse controls and monitoring before public exposure. Avoid lockout designs that enable denial of service. |
| Medium | Browser JWT is in localStorage | Bearer token is not automatically sent as a cookie, lowering CSRF exposure, but successful XSS could read the token. Maintain output escaping, deploy CSP/security headers, minimize third-party scripts, and consider a hardened HttpOnly-cookie architecture if product requirements justify it. |
| Medium | Development defaults are unsafe for public hosting | Default JWT secret and wildcard CORS are development-only. Production validators reject them, but deployments must actually set `FINALSAY_ENVIRONMENT=production` and supply exact HTTPS origins/secrets. |
| Medium | DNS validation and connection resolution are separate operations | A hostname is validated as public before urllib opens the request; DNS is not explicitly pinned to the validated IP. Exact allowlists sharply reduce scope, but DNS rebinding/TOCTOU is a residual SSRF-hardening consideration. Pin resolved public addresses or revalidate at connection time if expanding source management. |
| Medium | No deployment evidence or observed cloud controls | Compose and ECS configuration require external secret/database/TLS infrastructure. Confirm security groups, ALB TLS, IAM least privilege, secret rotation, backups, log retention, and database network isolation in the actual account. |
| Low | Regex PII redaction is heuristic | Email/phone/roll/name patterns do not guarantee removal of all personal data; validate against Indian-language and institutional samples and avoid representing the output as anonymization. |
| Low | Error audit data may contain source URL/context | Restrict access to Admin run logs and apply retention/redaction rules for source errors. Do not put credentials into source URLs. |

### Authentication, role boundaries, and CSRF/XSS

JWTs use HS256 with configured expiration (default 24 hours). Login accepts OAuth2 form fields; registration returns a student account. Admin permissions control source configuration, issuer grants, benchmark curation and root management. Reviewer routes are distinct; issuer publication requires an active grant for the target institution. There is no refresh token or account recovery/email verification flow. CORS is permissive by default in development and production validation requires explicit HTTPS origins.

The frontend stores bearer tokens in localStorage, not cookies. Therefore classic ambient-cookie CSRF is not the primary risk; XSS exposure is more relevant. React escapes ordinary rendered strings by default, but this does not replace CSP, dependency review, safe URL handling, and a production security-header policy. No explicit CSP or backend rate limiter was found in the inspected config.

## Reliability and source safety

The poller records source-level statuses and retries transient failures. A scheduler retry/backoff loop and shared poll lock are implemented. The current DB shows many errors and partial runs, including recent DNS and bad-file-descriptor failures. Observed ability to log an error is not the same as reliable operation; production needs alerting, dashboards, runbooks, and actionable per-source SLOs. The database integrity check passes, but no backup/restore drill was performed for this report.

Source hosts are constrained, but site terms, institution authorization, and privacy rules still need operational review. A source trust policy records where a notice came from; it does not cryptographically certify that a source page is authoritative or endorsed by the institution. No trust ranking or digital signature verification is present.

## Performance and scalability

No reproducible API latency, OCR throughput, scraper throughput, or model inference benchmark was found or run. Do not infer production capacity from the 2,284 stored rows or passing tests.

- Scraping: a single poll lock prevents overlapping cycles; per-host requests are sequential with at least 0.5 seconds between client requests, bounded retries, source page limits and capped PDFs. This favors politeness and bounded work over high concurrency. Current backfills are not proven complete for all portals.
- Retrieval: semantic comparison scopes to current official notices in one institution, then ranks the available rows. The current active institutions contain 415, 720 and 990 current notices. Embeddings are cached in process by text, so cache memory grows with observed texts; multi-worker processes do not share it. The implementation is suitable for prototype-scale corpora but requires measurement/indexing strategy for very large corpora.
- NLI: detailed claim matching can call a cross-encoder on selected claim pairs and runs on CPU by default; worst-case latency can rise with notice length and claims. Text is truncated/bounded in model paths; no GPU service or async inference queue is present.
- OCR: PDF embedded extraction processes up to 30 pages/30,000 characters; OCR fallback is limited to the first two pages with bounded per-page timeouts. Image OCR depends on system Tesseract and installed language packs. It can miss later-page details or misread scans.
- Database: SQLite is appropriate for local prototype use; PostgreSQL is supported and contains poll coordination support. High write concurrency, horizontal API scale, queueing and retention policies were not measured.
- API/upload: uploaded bytes are read into memory and no explicit max upload size was identified. This is the most direct resource-exhaustion issue before public use.

## Hosting and deployment

Current code defaults to localhost development with SQLite. Docker Compose describes a FastAPI container and PostgreSQL; the Dockerfile packages backend and local ML dependencies. GitHub Actions include frontend Pages and backend ECS/Fargate workflows. ECS templates require secrets, task/execution roles, database URL, explicit HTTPS origins, cluster/service and AWS configuration. None of those files proves a currently running AWS service. A prior report explicitly stated no AWS deployment; deployment status should be rechecked in the AWS account before a release claim.

## Prioritized next steps

1. Replace the HTTP GitHub Pages API target with a verified HTTPS endpoint and perform an authenticated browser smoke test.
2. Add hard body/file size and decompression/page/pixel limits before upload parsing/OCR.
3. Enable rate limiting and security headers/CSP at the edge; document token lifetime/revocation and incident handling.
4. Restore portal reachability; alert on repeated failure; validate source-by-source extraction and backfill completeness with source owners.
5. Run backup/restore, dependency/container scanning, secret rotation and TLS/load-balancer verification in the actual cloud account.
6. Measure end-to-end submission latency and OCR/model throughput on a representative labeled corpus; report model error rates by category/institution/language.
7. Obtain real student and two independent reviewer activity before claiming operational user validation or human agreement.
8. Treat Polygon deployment as separate: configure secrets securely, deploy only with approval, and verify a real receipt/read-back before claiming chain anchoring.
