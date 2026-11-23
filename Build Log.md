# Build Log: Payment Refund API onboarding

Running log of what was done, what broke, why, and how it was fixed.
Format per issue: **what happened → why → what I did**.

---

## Setup

**1. Postman Enterprise trial**
- The upgrade dialog defaulted to "Start Solo Trial". Selecting the Enterprise plan changed the button to "Start Enterprise Trial". The UI differed from the provided walkthrough video.
- Confirmed via the in-app banner "Enterprise Trial ending in 7 days".
- The trial excludes Organizations, Insights and API Catalog.

**2. Credentials**
- Generated a Postman API key (PMAK).
- Installed the Postman CLI, ran `postman login`, and extracted the access token from `~/.postman/postmanrc`. This token is session-scoped and expires.

**3. Repo and specs**
- Public repo `ps177/postman-api-onboarding-pilot`, with the 3 specs under `specs/`.
- Verified that the raw spec URL is fetchable.

**4. GitHub PAT**
- Fine-grained token, scoped to this one repo, expiring in 30 days.
- Read/write on Actions, Contents, Secrets, Variables and Workflows.
- Why: the default `GITHUB_TOKEN` cannot push files under `.github/workflows/`.

**5. Secrets**
- Added `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN` and `GH_FALLBACK_TOKEN` as encrypted repo secrets. None are committed to git.

---

## Spec review (payment-refund-api)

6 endpoints, 4 servers (prod, uat, qa, dev), OAuth2 (authorizationCode) + JWT.

What is good: clear endpoints, documented error codes (400/401/404/409/422/429/500/503 on `POST /refunds`), rich examples (which is why the mock works), a `status` enum and a `refundId` pattern that contract tests can check.

Issues found:
1. Server URLs are `*.example.com` and are not real hosts.
2. **Success** response schemas have no `required` fields, so contract tests check types but not presence. (Error schemas do declare `required`.)
3. `/health` inherits global auth (no `security: []` override).
4. OAuth2 offers only the `authorizationCode` flow, which needs a human in a browser and is unusable by CI or monitors.
5. The `full_refund` example sends `amount: null`, but the schema says integer and is not nullable.
6. `amount` means the refund amount in the request but the original transaction amount in the response.
7. `paymentGatewayRefundId` is top-level in the schema but inside `metadata` in the example.
8. Links point to `/retry` and `/transactions/{id}`, which are not defined.
9. The `500` response contains a `service_unavailable` example that belongs under `503`.
10. Business-rule errors are split inconsistently between `400` and `422`.
11. `AUTHORIZATION_ERROR` is in the enum and scopes exist, but no `403` response is defined.
12. Rate limits appear only in the description; no `Retry-After` or rate-limit headers are defined.

Decision: onboard the spec as-is and document the issues. A CSE does not silently fix a customer's spec.

---

## Starter asset review

Bypassed the starter composite action and called the upstream action directly. Kept the `workflow_dispatch` input pattern.

1. `spec-url` is wired to `inputs.spec-path`, so a file path is sent instead of a URL.
2. The workflow lives in `starter-assets/workflows/`, which GitHub never runs.
3. Only `github.token` is passed and there is no `gh-fallback-token`, so the action cannot write workflow files.
4. No `environments-json`, so it defaults to `["prod"]` while the spec has 4 environments.
5. `postman-access-token` is optional, which risks a silent partial success.

---

## Workflow runs

**Pre-check**
- The run form first showed the starter's inputs, because the wrong content had been committed. Replaced it with our workflow.
- Lesson: verify what is actually committed before running.

**Onboarding Run #1 (20 Nov, 00:09 IST): ✅ green, 1m 18s, 11 annotations**
- Node 20 deprecation warning for `actions/checkout@v4`: harmless.
- Spec lint warnings (`$ref` rule): some look like false positives.
- Requester invite failed: "Only one role is supported for user 60316975". The requester is the API key owner. Production should use a service account.
- Sub-teams check failed: "Team feature is not available". Enterprise trial limitation.

**State persistence**
- No repo variables were created, although the brief says state persists via repository variables.
- In v0, state lives in the committed `.postman/resources.yaml`: workspace ID, 3 collection IDs, 4 environment IDs, 1 spec ID.
- Risk: deleting this file makes the next run create a duplicate workspace.
- `localResources` lists all 3 specs in `specs/`, but one `resources.yaml` maps to one workspace. A second service in this repo would likely collide, so this is a repo topology decision.
- Asset IDs are prefixed with a personal user ID. Production should use a service account.

**Robot commit and PAT behaviour**
- The action committed "chore: sync Postman artifacts and metadata" as "Postman CSE", pushed with the PAT.
- A PAT push triggers other workflows, so the generated `ci.yml` ran automatically. A `GITHUB_TOKEN` push would not have.

**Generated CI Run #1: ❌ "Invalid workflow file" (line 43)**
- Cause: `if: secrets.POSTMAN_SSL_CLIENT_CERT_B64 != ''`. GitHub does not allow `secrets` in step `if:` conditions.
- Bug in the upstream action's CI template. Onboarding was green, but the generated CI could never run.
- Fix (2 lines): expose the secret as job-level `env`, then test `env.X` in the `if:`.

**Generated CI Run #3: file valid, smoke collection executed**
- 24 failures, 2 root causes. When a request fails, its assertions fail with it.
- Root cause 1: `Invalid URI "http:///refunds"`. `baseUrl` was empty because the spec servers were not applied.
- Root cause 2: "00 - Resolve Secrets" fetches auth from AWS Secrets Manager (eu-west-2) via Postman Vault. No AWS keys, so it returns 400.

**Other findings in the generated CI (documented, not changed)**
- CI targets the PROD environment on every push and every 6 hours.
- Smoke and Contract contain `POST /refunds` and `POST /cancel`. Against real prod this would create and cancel real refunds. Smoke must be read-only.
- `--env-var "Production"` lacks the `key=value` format; looks suspicious.
- The 6-hour cron overlaps with Postman monitors; one should own scheduled checks.

**Environment file (prod) after run #1**
- `baseUrl` empty.
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are empty secrets.
- `AWS_REGION` hardcoded to `eu-west-2`; `AWS_SECRET_NAME` is `api-credentials-<env>`.

**Onboarding Run #2 (20 Nov, 00:31 IST): ✅ green, 3m 39s. Three things tested in one run.**
- `env-runtime-urls-json` accepted by v0; `baseUrl` populated in all 4 environments.
- `generate-ci-workflow: 'false'` honoured; the `ci.yml` fix was preserved.
- **Idempotent:** same workspace ID `7376447b…` reused; only one `[PAY] payment-refund-api` workspace exists.

**Full log analysis (downloaded log archive)**
- GitHub showed about 10 annotations per step; the full logs had 52 lint warnings. Always read the full logs.
- Mock (on Baseline) and monitor (on Smoke) were created in run #1; run #2 "discovered existing" ones and reused them.
- Idempotency mechanism: `collection-sync-mode: refresh` regenerates into a temporary collection, refreshes the existing one in place, then deletes the temporary one. IDs stay stable.
- 🔴 Contract tests report PASS for headers, body and schema even when the request errored and no response came back (the scripts return early when no response contract is selected). These passes check nothing.
- Built-in tests failed 6 of 6, but the job stayed green: tests run with `set +e` and results go only to an artifact.
- `CONTRACT_SECURITY_NOT_VALIDATED`: OAuth2 and JWT are not proven at runtime.
- `monitor-type: cloud`: a cloud monitor cannot reach internal VPC APIs.
- Credential preflight OK: the API key and access token resolve to the same team.
- Spec lint took about 2.5 minutes of the 3m 39s run.

---

## Verification on 23 Nov

**Mock server**
- `GET <mock>/health` returned `mockRequestNotFoundError` (mock alive, path not matched). `GET <mock>/v2/health` returned the spec's `healthy` example. The mock keeps the spec's `/v2` base path.
- The MOCKS panel in the Items view is empty, but the mock appears under **Services → Mock Servers** as `payment-refund-api Mock` (active). Not a bug: Postman v12 lists mocks in the Services view.

**Monitor**
- Found under **Services → Monitors** as `payment-refund-api - Smoke Monitor`, status **Paused**.
- 2 runs, one per onboarding run, both on the US (East) cloud runner, 7 requests and 18 failed tests each:
  - Run after onboarding #1: `REQUEST` error, Invalid URI `http:///refunds/...` (empty `baseUrl`).
  - Run after onboarding #2: `NETWORK` error, getaddrinfo ENOTFOUND `api.payments.example.com`.
- The error category moved from REQUEST to NETWORK: our configuration was fixed, and the remaining failure is the customer dependency (real host).
- The monitor never ran on a schedule; it ran only when onboarding triggered it.

**Environments**
- All four (prod, uat, qa, dev) present. UAT briefly looked missing because the sidebar panel was scrolled; confirmed after refresh. Also present in `.postman/resources.yaml` and `postman/environments/uat.postman_environment.json`.

**Where the tests live**
- Tests are in each request's post-response script, not at collection level (the collection Tests tab is empty by design).
- Baseline: 0 tests. Smoke: 6 requests × 3 = 18 (2xx, response time < `RESPONSE_TIME_THRESHOLD` default 2000 ms, non-empty body). Contract: 6 requests × 6 = 36 (operation mapping, status defined, headers, body presence, Content-Type, JSON schema).

**Contract run against the mock: 36/36 passed (supplementary)**
- Created a manual environment `payment-refund-api - mock` with `baseUrl = <mock>/v2`, deselected "00 - Resolve Secrets", ran `[Contract]` in the Collection Runner.
- 36/36 passed, 0 errors. This proves the generated scripts run correctly. It does not prove the real API conforms: the mock returns the spec's own examples.
- Known spec contradictions did not fail the run: `paymentGatewayRefundId` inside `metadata` passes because success schemas have no `required` and allow extra fields; `amount: null` is in a request body, and contract tests only check responses.
- "Get refund status" took 1701 ms on the mock, close to the Smoke threshold of 2000 ms.

**"00 - Resolve Secrets" ordering bug**
- It is listed last in Smoke and Contract (runner order, v3 YAML export and v2.1 JSON export all agree). It should run first so other requests get a token. In a real run, no request would be authenticated.

**JSON exports**
- `postman/collections/` holds the action's v3 YAML format. Exported all three collections from Postman as Collection v2.1 JSON into `postman/exports/`. Postman's export scanned for credentials; I also checked the files for PMAK keys, JWTs, AWS keys and token values: none present.

---

## What the customer's platform team must provide

1. A real base URL per environment.
2. A scoped IAM identity that can read Secrets Manager.
3. Secrets named `api-credentials-<env>`, or an override of that convention, in the correct AWS region.
4. A service-account JWT or client-credentials OAuth for automation.
5. A Postman service account instead of personal keys, and a rotation process for the access token.
6. A network path, or private runner, so monitors/CI can reach internal APIs.
7. Test data, such as a captured `transactionId`.
8. A decision on which environment CI targets (dev or qa, not prod), and who owns scheduled checks (CI or monitor).

---

## AI usage

- **AI-generated (Claude):** workflow YAML, the `ci.yml` fix, the spec issue list, starter asset review, log analysis summaries, and drafts of this log.
- **Done or validated by me:** account and trial setup, credentials, repo, running every workflow, reading outputs, and verifying the artifacts in Postman and GitHub.
