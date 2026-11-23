# Adaptation Analysis: Claims Processing API

How the onboarding pattern proven on the Payment Refund API would be applied to the Claims Processing API (`specs/claims-processing-api-openapi.yaml`).

**Status:** analysis only. Nothing in this document has been run. Snippets are design sketches, not a tested implementation.

## Why Claims

Of the two remaining specs, Claims is the most different from Payments: different infrastructure (mixed Lambda/ECS), a different CI system (GitLab CI), a different auth model and a different environment set. If the pattern holds here, it holds for the easier cases too. Loan Origination (ECS/ALB, GitHub Actions) would change only inputs and network details.

---

## 1. The two services side by side

| | Payment Refund (built) | Claims Processing (analysed) |
|---|---|---|
| Infrastructure | Lambda + API Gateway | Mixed Lambda/ECS |
| CI system | GitHub Actions | **GitLab CI** (stated in the spec description) |
| Endpoints | 6 | 6 + health: file claim, list, get, adjudicate, create settlement, get settlement |
| Servers | 4: prod, uat, qa, dev (`/v2`) | **3: prod, staging, dev** (`/v3`) |
| Auth | OAuth2 `authorizationCode` + JWT bearer | **API key (`X-API-Key` header)** + OAuth2 **`clientCredentials`** |
| Auth usable by automation? | No for OAuth (needs a browser); JWT needs an internal issuer | **Yes**: both API key and client credentials are machine-friendly |
| `/health` auth | Inherits global auth (likely wrong) | `security: []` (correct) |
| Response examples | Rich | **None** (one request example only) |
| Error responses | Detailed, with schemas | Sparse: no 401/403 anywhere; 404/409 have no schema |
| Request `required` fields | Present | Present (`FileClaimRequest`, `AdjudicationRequest`, `SettlementRequest`) |
| Success response `required` fields | None | None |

---

## 2. What stays the same

- The onboarding workflow file and `postman-api-onboarding-action@v0`.
- The bootstrap → repo-sync chain: workspace, Spec Hub upload, three collections, environments, mock, monitor.
- Collection types and naming (`[CODE] service-name`, Baseline / Smoke / Contract).
- `generate-ci-workflow` handling and passing base URLs explicitly with `env-runtime-urls-json`.
- The validation checklist (§5 of the README) and rerun/idempotency behaviour.
- The split between what the spec can test and what needs service knowledge.

## 3. What changes

### 3a. Inputs only (no workflow change)

| Input | Payments | Claims |
|---|---|---|
| `project_name` | `payment-refund-api` | `claims-processing-api` |
| `domain` / `domain_code` | `payments` / `PAY` | `insurance` / `CLM` (to agree with the customer) |
| `spec_url` / `spec_path` | payment spec | claims spec URL and path |
| `environments` | `["prod","uat","qa","dev"]` | `["prod","staging","dev"]` |
| `env_urls` | `api*.payments.example.com/v2` | `claims-api*.example.com/v3`, from the spec servers, to be replaced with real URLs |

That is the whole difference for onboarding itself. The changes that matter are around it.

### 3b. CI system: GitLab instead of GitHub Actions

The Postman actions are GitHub Actions and will not run in GitLab. Split the problem in two:

**Onboarding (create the workspace, collections, environments, mock, monitor).** This only needs the spec at a fetchable URL and the Postman API. Two options:

| Option | How | Trade-off |
|---|---|---|
| **A. Central onboarding repo on GitHub (recommended for the pilot)** | The platform team's GitHub repo runs the same workflow, with `spec_url` pointing at the claims spec in GitLab | Zero change for the insurance team. Needs a GitLab read token (or a mirrored spec) so the runner can fetch a private raw file. Repo-sync would commit exported collections to the central GitHub repo, not to the claims GitLab repo |
| **B. Native GitLab job** | A `.gitlab-ci.yml` job calls the Postman API / CLI directly | Keeps everything in GitLab, but re-implements what bootstrap and repo-sync do. More to build and maintain |

**Ongoing tests in the claims team's own pipeline.** This is simple, because the generated collections can be run by the Postman CLI from any CI system. Design sketch (not run):

```yaml
# .gitlab-ci.yml (sketch)
postman_contract_tests:
  stage: test
  image: ubuntu:24.04
  script:
    - apt-get update && apt-get install -y curl
    - curl -fsSL https://dl-cli.pstmn.io/install/unix.sh | sh
    - postman login --with-api-key "$POSTMAN_API_KEY"
    - postman collection run "$CLAIMS_CONTRACT_COLLECTION_UID" -e "$CLAIMS_DEV_ENV_UID"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

The pattern is the same; only the execution layer changes. The collection and environment UIDs come from the `.postman/resources.yaml` written during onboarding.

### 3c. Repo topology

One `.postman/resources.yaml` holds one workspace ID. Onboarding Claims into the payments pilot repo would collide with the Payments workspace. Claims gets its own state: either its own repo, or (with option A) its own folder/state file in the central repo. This needs to be decided before the second service, not after.

### 3d. Auth and secrets

- Payments needed a JWT or OAuth with a browser step. Claims can use an **API key** or **OAuth client credentials**, both suitable for CI and monitors. This is an improvement.
- The generated "00 - Resolve Secrets" request pulls credentials from AWS Secrets Manager (`api-credentials-<env>`). The shape of the stored secret differs (an API key or client ID/secret instead of a JWT), so the insurance team must confirm what is stored and where. To be validated on the first run.
- The "Resolve Secrets" ordering bug found on Payments (it runs last) would apply here too.

### 3e. Infrastructure: mixed Lambda/ECS

- The spec has one server per environment. If some endpoints are served by Lambda (behind API Gateway) and others by ECS (behind an ALB), the team must confirm whether a single gateway fronts both. If not, one `baseUrl` per environment is not enough.
- Network reachability must be checked for **both** paths: a private API Gateway and an internal ALB are reached differently. The Postman cloud monitor (US East) will reach neither if they are private.

### 3f. What the generated assets will and won't give us

- **Mock server will be weak.** The spec has no response examples, so the mock has no realistic responses to return. Ask the insurance team to add examples before consumers rely on the mock.
- **Contract coverage is thinner than Payments.** No 401/403 responses are defined, so any auth failure fails "status code is defined by OpenAPI". 404/409 and the settlement 400 have no schema, so their bodies cannot be checked. Success schemas have no `required` fields.
- **Health is unauthenticated (good) but only `200` is defined.** If the service reports `unhealthy` with a 503, the contract test fails on "status code not defined", even though 503 is the correct behaviour.

---

## 4. Claims spec issues to raise with the insurance team

1. No response examples, so the mock is of little use.
2. No `401` or `403` responses, despite two auth schemes and OAuth scopes.
3. `404`, `409` and the settlement `400` have no response schema.
4. `Claim.lineOfBusiness` has no enum, although the request and list filter define one.
5. `AdjudicationResult.decision` has no enum, although the request defines `approved / denied / partial_approval`.
6. `approvedAmount` is required in `AdjudicationRequest` even when the decision is `denied`.
7. Nothing links the settlement `amount` to the adjudicated `approvedAmount`; that rule lives only in people's heads.
8. A file-upload rate limit is described, but there is no upload endpoint.
9. `/health` defines only `200`; there is no `503` for degraded/unhealthy.
10. Business rules (active policy, filed within 90 days, one claim per incident) exist only in description text.

None of these block onboarding. Like Payments, the spec would be onboarded as-is and the issues returned to the owning team.

---

## 5. What we need from the platform and insurance teams

| Need | Why |
|---|---|
| Real base URLs for prod, staging, dev, and confirmation of whether Lambda and ECS share one host | Environments and `env_urls` input |
| An API key or OAuth client credentials for automation, and where they are stored | Smoke/contract runs and monitors |
| Decision: central GitHub onboarding repo (A) or native GitLab job (B) | Onboarding path |
| A GitLab read token or mirrored spec, if option A | The GitHub runner must fetch the spec |
| A GitLab runner and CI variables (`POSTMAN_API_KEY`, collection and environment UIDs) | Ongoing contract tests in their pipeline |
| Network path to private API Gateway and internal ALB, or a private runner | Monitors and CI reaching the API |
| Test data: an active policy ID and an incident date within 90 days | Business-rule tests |
| Response examples and 401/403 responses added to the spec | Useful mock and stronger contract tests |
| Which environment CI may target (`settlement` creates payments) | Avoid side effects in prod |

---

## 6. Conclusion

For a service on different infrastructure, with a different CI system and a different auth model, the onboarding workflow changes only in its inputs. Everything that actually differs is coordination: which CI runs the tests, how the spec is fetched, which credentials exist, and whether the network allows the tests to reach the API. That is the repeatable template: a fixed workflow plus a per-service input set, plus a per-service checklist of platform-team dependencies.
