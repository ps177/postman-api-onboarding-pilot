# Adaptation Analysis: Claims Processing API

This document explains what I would change to onboard a second service, the Claims Processing API, using the same setup I built for the Payment Refund API.

**Note:** I have not run this. It is a plan based on reading the claims spec and on what I learned while building the payments setup. The GitLab example below is a sketch, not tested code.

## Why I picked Claims

Claims is the most different from Payments:

- It runs on a mix of Lambda and ECS (Payments runs only on Lambda).
- Its team uses GitLab, not GitHub, for CI.
- It uses a different login method and has a different set of environments.

If the setup works for Claims, it will work for the easier services too.

---

## 1. Payments vs Claims at a glance

| | Payments (built) | Claims (planned) |
|---|---|---|
| Where it runs | Lambda | Mix of Lambda and ECS |
| CI tool | GitHub Actions | **GitLab** |
| Endpoints | 6 | 6, plus a health check |
| Environments | 4: prod, uat, qa, dev | **3: prod, staging, dev** |
| URL version | `/v2` | `/v3` |
| Login method | OAuth (needs a person to log in through a browser) or JWT | **API key** or OAuth "client credentials" (no person needed) |
| Can a robot (CI, monitor) log in? | Hard | **Easy** |
| Health check needs login? | Yes (probably a mistake) | No (correct) |
| Example responses in the spec | Many | **None** |
| Error responses described | Most of them | Few. No 401 or 403 at all |

---

## 2. What stays exactly the same

- The same workflow file and the same Postman action (`postman-api-onboarding-action@v0`).
- The same steps: create workspace, upload spec, create 3 collections (Baseline, Smoke, Contract), create environments, create mock and monitor, save everything to the repo.
- The same naming style, for example `[CLM] claims-processing-api`.
- The same two settings I added for Payments:
  - passing the base URL for each environment myself, and
  - turning off the automatic CI file (`generate-ci-workflow: 'false'`).
- The same checks after the run, and the same rerun behaviour (a second run updates the same workspace, it does not create a new one).

---

## 3. What changes

### 3a. Only the inputs

When you run the workflow, you fill in a form. For Claims, only these values change:

| Input | Payments | Claims |
|---|---|---|
| Service name | `payment-refund-api` | `claims-processing-api` |
| Domain | `payments` | `insurance` |
| Short code | `PAY` | `CLM` (to confirm with the customer) |
| Spec link | payments spec | claims spec |
| Environments | `["prod","uat","qa","dev"]` | `["prod","staging","dev"]` |
| Base URLs | payments URLs | claims URLs (real ones from the customer) |

**The workflow file itself does not change.** Everything else that is different is about working with the customer's teams, explained below.

### 3b. GitLab instead of GitHub

The Postman actions only run on GitHub. The claims team uses GitLab. So I split the work into two parts.

**Part 1: Creating everything in Postman (one time per service)**

This step only needs two things: a link to the spec, and the Postman API key. It does not have to run inside the claims team's GitLab.

| Option | How it works | Good | Not so good |
|---|---|---|---|
| **A. Run it from a central GitHub repo (my recommendation for now)** | The platform team keeps one GitHub repo. They run the same workflow there, with the spec link pointing to the claims spec in GitLab. | The claims team does nothing. Same workflow as Payments. | GitHub needs permission to read the spec from GitLab (a GitLab token). The exported collections get saved in the GitHub repo, not in the claims team's GitLab repo. |
| **B. Build it in GitLab** | Write a GitLab job that does the same steps using Postman's API. | Everything stays in GitLab. | We would have to rebuild what Postman's actions already do. More work to build and to look after. |

**Part 2: Running the tests regularly (every code change)**

This part is easy in GitLab. The Postman CLI (a command-line tool) can run any Postman collection from any CI tool. A GitLab job would look roughly like this (sketch, not tested):

```yaml
# .gitlab-ci.yml (sketch)
postman_contract_tests:
  stage: test
  image: ubuntu:24.04
  script:
    - apt-get update && apt-get install -y curl
    - curl -fsSL https://dl-cli.pstmn.io/install/unix.sh | sh
    - postman login --with-api-key "$POSTMAN_API_KEY"
    - postman collection run "$CLAIMS_CONTRACT_COLLECTION_ID" -e "$CLAIMS_DEV_ENV_ID"
```

The collection and environment IDs come from the `.postman/resources.yaml` file that onboarding creates.

**In short:** same pattern, only the place where it runs is different.

### 3c. Where the Claims setup lives

The action saves its memory in one file: `.postman/resources.yaml`. This file holds only **one** workspace ID. If I onboard Claims into the same repo as Payments, the two would clash.

So Claims needs its own place: either its own repo, or its own folder in the central repo. This must be decided before onboarding the second service.

### 3d. Login and secrets

- **Good news:** Claims supports an API key and OAuth client credentials. Both work for robots (CI and monitors) without a person logging in. Payments did not have this.
- The collections include a request called "00 - Resolve Secrets". It fetches login details from AWS Secrets Manager (a secure store in AWS), using a secret named `api-credentials-<env>`. For Claims, the stored secret will be an API key or client ID/secret instead of a JWT. The claims team must confirm what is stored and where. This needs testing on the first run.
- In Payments, I found that "00 - Resolve Secrets" runs **last** instead of first, so no request gets logged in. The same problem would happen for Claims.

### 3e. Lambda and ECS mix

- The spec gives one URL per environment. But some claims endpoints may run on Lambda and others on ECS. I need to ask the team: is there one front door (one URL) for both, or two different URLs? If two, one base URL per environment is not enough.
- If the APIs are private (inside the bank's network), the Postman cloud monitor cannot reach them. This has to be checked for both the Lambda side and the ECS side.

### 3f. What the generated tests and mock will be like

- **The mock will not be very useful.** A mock replies using the examples in the spec. The claims spec has no response examples, so the mock has nothing realistic to send back. The claims team should add examples.
- **The contract tests will be weaker than for Payments:**
  - There is no 401 or 403 in the spec. So if a login fails, the test fails with "status code not defined", instead of a clear message.
  - Some error responses (404, 409, settlement 400) have no body format, so their body cannot be checked.
  - Success responses do not mark any field as required, so a missing field would not be caught.
- **Health check:** only `200` is described. If the service is unhealthy and correctly returns `503`, the test fails, because `503` is not in the spec.

---

## 4. Problems I found in the Claims spec

These do not stop onboarding. Like with Payments, I would onboard the spec as it is and send this list to the claims team.

1. No example responses, so the mock is weak.
2. No 401 or 403 responses, even though the spec has two login methods.
3. 404, 409 and the settlement 400 have no response body format.
4. `lineOfBusiness` has a fixed list of values in the request, but not in the response.
5. `decision` has a fixed list (`approved`, `denied`, `partial_approval`) in the request, but not in the response.
6. `approvedAmount` is required even when the claim is denied.
7. Nothing says the settlement amount must match the approved amount. That rule is not written anywhere in the spec.
8. The description mentions a rate limit for file uploads, but there is no upload endpoint.
9. The health check has no `503` response for "unhealthy".
10. Business rules (policy must be active, claim within 90 days, one claim per incident) are only written as text, so no test can be generated for them.

---

## 5. What I need from the customer's teams

| What I need | Why |
|---|---|
| Real URLs for prod, staging and dev, and whether Lambda and ECS share one URL | To set up the environments |
| An API key or client credentials for automated runs, and where they are stored | So tests and monitors can log in |
| A decision: run onboarding from a central GitHub repo (Option A) or build it in GitLab (Option B) | To know where onboarding runs |
| A GitLab token, if Option A | So GitHub can read the spec from GitLab |
| A GitLab runner and CI variables (Postman key, collection and environment IDs) | To run tests in their own pipeline |
| Network access to the private APIs, or a private runner | So monitors and CI can actually reach the API |
| Test data, like an active policy ID and a recent incident date | To test the business rules |
| Example responses and 401/403 added to the spec | For a useful mock and better tests |
| Which environment the tests may use (creating a settlement starts a payment) | To avoid side effects in prod |

---

## 6. Summary

For Claims, the workflow stays the same. **Only the inputs change.**

The real differences are about coordination with the customer:

- which CI tool runs the tests,
- how the spec is fetched,
- which login details exist, and
- whether the tests can reach the API over the network.

So the repeatable template is:

1. **One fixed workflow**, the same for every service.
2. **A small set of inputs** per service.
3. **A checklist** of what to get from the platform team for that service.
