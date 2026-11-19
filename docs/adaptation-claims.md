# Adaptation Analysis: Claims Processing Service

> **Status:** design analysis. Nothing in this document has been executed. The working
> implementation is the payment-refund service (see the [README](../README.md)).
> The GitLab pipeline below is a proposed shape built from Postman's published
> non-GitHub CLI docs, not a tested pipeline.

## Why claims as the second service

Of the two remaining specs, claims is the one that breaks the most assumptions in the
payment-refund workflow:

| | Payment refund (built) | Claims (this analysis) |
|---|---|---|
| CI/CD | GitHub Actions | **GitLab CI**, and the team won't switch |
| Runtime | Lambda / API Gateway | Mixed Lambda / ECS |
| Auth (from spec) | OAuth2 **authorization code** or JWT bearer | **API key** (`X-API-Key`) or OAuth2 **client credentials** |
| Environments | dev, qa, uat, prod | dev, staging, prod |
| Base path | `/v2` | `/v3` |
| `/health` auth | Inherits global auth (spec bug) | `security: []` (correct) |
| Write flow | Create, then cancel | Create, then **adjudicate, then settle** (state machine) |

Loan origination was the alternative. Its main difference is mTLS, which OpenAPI 3.0
can't express. The action already accepts `ssl-client-cert` and `ssl-client-key`
inputs, so loan is mostly a credentials problem, not a pipeline problem. Claims tests
more of the pattern.

## What stays the same

This is the main point. The onboarding **logic** doesn't change at all:

1. Mint a short-lived Postman access token from a service-account API key.
2. Bootstrap: create or reuse the workspace, upload the spec to Spec Hub, and generate
   the baseline, smoke and contract collections.
3. Repo sync: export the collections to `postman/`, create environments, the mock and
   the monitor, and commit `.postman/resources.yaml`.
4. Reruns read `.postman/resources.yaml` and update assets in place. On payment refund,
   the second run reused the same workspace, collection, mock and monitor IDs and
   created no duplicates.
5. Run smoke and contract tests with the Postman CLI against a chosen environment.

The Postman-side assets, naming, state file, and the "what the generated tests prove
vs. what they don't" boundary are identical. Postman publishes the same two steps as
npm CLIs (`postman-bootstrap`, `postman-repo-sync`) for exactly this case, so the
GitHub Action is just a wrapper we're not using here.

## What changes

### 1. The CI runner: GitLab CI instead of GitHub Actions

> **Proposal, not run.** Nothing in this section has been executed. Before presenting
> any of it as working, verify: the CLI flag names (taken from the documented
> kebab-case action inputs), `--dotenv-path` output keys, whether a Postman CLI
> container image exists (otherwise install it in the job), and whether repo-sync's
> commit-back and workspace-to-repo linking support GitLab remotes. If linking isn't
> supported, the catalog still works and only the Git link in Postman is lost.

The same two steps as the GitHub action, as Postman's npm CLIs, plus the pieces a
GitLab pipeline has to supply itself: a concurrency guard, a handoff of the generated
IDs to the test jobs, a runner that can reach private endpoints, and separate write and
read-only test jobs.

```yaml
# PROPOSED — not executed. Based on postman-api-onboarding-action/docs/non-github-ci.md
stages: [onboard, test]

variables:
  SERVICE: claims-processing-api

onboard:claims:
  stage: onboard
  image: node:20
  tags: [vpc-runner]                    # self-hosted runner inside the network (see below)
  resource_group: onboard-claims        # one onboarding run at a time = no duplicate assets
  rules:
    - if: $CI_PIPELINE_SOURCE == "web"  # manual "Run pipeline", like workflow_dispatch
    - changes: [specs/claims-processing-api-openapi.yaml]
  script:
    - npm install -g @postman-cs/onboarding-bootstrap @postman-cs/onboarding-repo-sync @postman-cs/onboarding-resolve-service-token
    - apt-get update -qq && apt-get install -y -qq jq
    - postman-resolve-service-token --postman-api-key "$POSTMAN_API_KEY" --postman-region us > token.json
    - export POSTMAN_ACCESS_TOKEN="$(jq -r .token token.json)" POSTMAN_TEAM_ID="$(jq -r '.["team-id"]' token.json)"
    - >
      postman-bootstrap --project-name "$SERVICE"
      --spec-path specs/claims-processing-api-openapi.yaml
      --postman-api-key "$POSTMAN_API_KEY" --postman-access-token "$POSTMAN_ACCESS_TOKEN"
      --postman-team-id "$POSTMAN_TEAM_ID" --result-json bootstrap.json
    - >
      postman-repo-sync --project-name "$SERVICE"
      --workspace-id "$(jq -r '.["workspace-id"]' bootstrap.json)"
      --baseline-collection-id "$(jq -r '.["baseline-collection-id"]' bootstrap.json)"
      --smoke-collection-id "$(jq -r '.["smoke-collection-id"]' bootstrap.json)"
      --contract-collection-id "$(jq -r '.["contract-collection-id"]' bootstrap.json)"
      --environments-json '["dev","staging","prod"]'
      --env-runtime-urls-json "$CLAIMS_ENV_URLS"
      --postman-api-key "$POSTMAN_API_KEY" --postman-access-token "$POSTMAN_ACCESS_TOKEN"
      --postman-team-id "$POSTMAN_TEAM_ID" --result-json sync.json --dotenv-path sync.env
    # ID handoff: expose the IDs the test jobs need as job variables
    - |
      {
        echo "SMOKE_UID=$(jq -r '.["smoke-collection-id"]' bootstrap.json)"
        echo "CONTRACT_UID=$(jq -r '.["contract-collection-id"]' bootstrap.json)"
        echo "DEV_ENV_UID=$(jq -r '.["environment-uids-json"] | fromjson | .dev' sync.json)"
        echo "STAGING_ENV_UID=$(jq -r '.["environment-uids-json"] | fromjson | .staging' sync.json)"
      } > ids.env
  artifacts:
    reports:
      dotenv: ids.env                   # test jobs receive SMOKE_UID, CONTRACT_UID, *_ENV_UID

# Write flow (file -> adjudicate -> settle): dev only, seeded data, cleanup afterwards.
test:claims:dev:
  stage: test
  tags: [vpc-runner]
  needs: [onboard:claims]
  resource_group: claims-dev-data       # runs can't race on the same seeded policy
  script:
    - curl -fsSL https://dl-cli.pstmn.io/install/unix.sh | sh
    - postman login --with-api-key "$POSTMAN_API_KEY"
    - postman collection run "$SMOKE_UID" -e "$DEV_ENV_UID" --env-var "apiKey=$CLAIMS_CI_API_KEY_DEV" --env-var "policyId=pol_ci_seed_001"
    - postman collection run "$CONTRACT_UID" -e "$DEV_ENV_UID" --env-var "apiKey=$CLAIMS_CI_API_KEY_DEV"
  after_script:
    - ./scripts/cleanup-ci-claims.sh pol_ci_seed_001   # owned by the insurance team

# Read-only probes against staging (and prod, if approved): GET operations only.
test:claims:readonly:
  stage: test
  tags: [vpc-runner]
  needs: [onboard:claims]
  script:
    - curl -fsSL https://dl-cli.pstmn.io/install/unix.sh | sh
    - postman login --with-api-key "$POSTMAN_API_KEY"
    - postman collection run "$READONLY_SMOKE_UID" -e "$STAGING_ENV_UID" --env-var "apiKey=$CLAIMS_CI_API_KEY_RO"
```

`READONLY_SMOKE_UID` is a GET-only smoke collection, produced by the action's smoke-flow
step with `flow-derive-scope: read-only`, or curated by hand. The generated smoke
collection files claims, so it never runs outside dev.

**Who owns what this pipeline needs:**

| Need | Owner | Notes |
|---|---|---|
| Self-hosted runner in the VPC (`vpc-runner`) | Platform team | The claims endpoints are private; GitLab's shared runners can't reach them. Needs egress to `api.getpostman.com`, `registry.npmjs.org`, `dl-cli.pstmn.io` |
| `POSTMAN_API_KEY` (service account), project access token (`write_repository`) | Platform team | Masked, protected CI/CD variables |
| `CLAIMS_CI_API_KEY_DEV` (read/write, dev only), `CLAIMS_CI_API_KEY_RO` (read-only) | Insurance team + security | Two keys, so a leaked read-only key can't file claims |
| Seeded policy `pol_ci_seed_001` and `cleanup-ci-claims.sh` | Insurance team | Without these, "one claim per incident per policy" returns 409 on every run |
| `CLAIMS_ENV_URLS` (dev, staging, prod entry points) | Platform team | API Gateway stage or ALB per environment |

What changes in the plumbing:

| Concern | GitHub (payment refund, built) | GitLab (claims, proposed) |
|---|---|---|
| Trigger | `workflow_dispatch` | Manual pipeline (`CI_PIPELINE_SOURCE == "web"`) or `rules: changes:` on the spec |
| Concurrency guard | `concurrency: onboard-<service>` | `resource_group: onboard-claims` (and one per test-data set) |
| ID handoff to tests | `ci.yml` reads `.postman/resources.yaml` | `artifacts: reports: dotenv` from the onboarding job |
| Reaching private endpoints | Not needed in the pilot (mock) | Self-hosted runner inside the network |
| Secrets | Repo secrets `POSTMAN_API_KEY`, `GH_PAT` | Masked, protected CI/CD variables |
| Writing back to the repo | Fine-grained PAT (Contents, Variables, Workflows) | Project access token with `write_repository` (unverified with repo-sync) |
| Write tests | Mock or dev only, never prod | Dev only, seeded data, cleanup in `after_script` |
| Read-only probes | Not built | GET-only collection against staging/prod |

### 2. Auth: easier for CI than payment refund

Payment refund's OAuth uses the **authorization code** flow, which needs a browser login
and can't run in CI, so it would have to use the JWT path. Claims offers two
machine-friendly options:

- **API key (`X-API-Key`)**: simplest. The platform team issues a CI-only key with
  read and write access to **non-production claims data**, stored as a masked GitLab
  variable and injected at run time.
- **OAuth2 client credentials**: if the insurance team prefers it. Postman can fetch the
  token itself with client ID and secret variables. Scope the token to `claims:read`
  for smoke runs, not `settlements:process`.

Either way, credentials are injected by the test jobs at run time (as above), never stored
in the collections. The payment-refund build already skips the action's built-in tests
(`skip-built-in-tests`) for the same reason.

### 3. Environments and base URLs

Three environments, `["dev","staging","prod"]`, with `/v3` base URLs. Only the inputs
change.

### 4. Test data and the state machine

Claims is a **workflow API**: a claim is filed, then adjudicated (409 if it's in the wrong
state), then settled. Unlike reading a refund status, the generated smoke run will
**create real claims** in whatever environment it targets. The decisions this needs:

- Smoke tests run **read-only** against staging and prod (the action's
  `flow-derive-scope: read-only`). Write flows only run against dev, or against the mock.
- A seeded test policy (`pol_ci_*`) so filing doesn't hit "one claim per incident per
  policy" duplicate rejections on every run (409).
- Cleanup or reset ownership: who purges CI-created claims in dev?

### 5. Mixed Lambda / ECS

This doesn't change the Postman onboarding at all, because Postman only sees URLs.
It matters in two places:

- **Base URL per environment** may point at an API Gateway stage (Lambda) or an ALB
  (ECS) depending on the route. We need the **public entry point** per environment,
  not per compute type.
- **Spec source of truth**: if routes are split across Lambda and ECS, the spec may be
  maintained in two places. Who owns the single spec file?

## Spec observations (found by reading it, before onboarding)

Generating tests from this spec would surface these, the same way payment refund's
contract run found `amount: null` and the unauthenticated `/health`:

1. **The description promises a feature the spec doesn't have.** Rate limits mention
   "File upload: 20 requests/minute", but there's no upload endpoint. Either the spec
   is missing operations or the description is out of date.
2. **No response examples.** Only `POST /claims` has a request example. The mock will
   have nothing realistic to return, so CI against the mock would be much weaker than
   it was for payment refund. We'd ask the team for examples, or generate them from
   the schemas and have them review the result.
3. **`lineOfBusiness` is an enum in the request but a free string in the `Claim` response.**
   Contract tests can't catch an invalid value coming back.
4. **Money as float.** `estimatedDamage` is `number/double`, while payments uses integer
   cents. That's a cross-domain inconsistency to flag for governance rules, not to fix
   per service.
5. **The auth model is consistent and `/health` is correctly public.** This is the
   cleanest of the three specs on security.

## What we need from the customer's platform and insurance teams

| # | Ask | Owner | Blocks |
|---|---|---|---|
| 1 | GitLab project access, plus a maintainer to approve the `.gitlab-ci.yml` change | Insurance eng lead | Everything |
| 2 | Masked CI/CD variables: `POSTMAN_API_KEY` (service account), a project access token with `write_repository` | Platform team | Onboarding job |
| 3 | GitLab runner with outbound HTTPS to `api.getpostman.com`, `registry.npmjs.org`, `dl-cli.pstmn.io` | Platform team | Onboarding job |
| 4 | Reachable base URLs for dev, staging and prod (API Gateway or ALB entry points) | Platform team | Real test runs |
| 5 | A CI-only API key (or OAuth client) scoped to non-prod, read-mostly | Insurance team + IdP owner | Authenticated tests |
| 6 | A seeded test policy and a cleanup owner for dev data | Insurance team | Write-path smoke tests |
| 7 | Response examples, and a decision on the "file upload" drift | Insurance team | Useful mocks |

## Effort estimate

| | Payment refund (first service) | Claims (second service) |
|---|---|---|
| Onboarding logic | Built | **Reused unchanged** |
| CI wiring | ~1 hour (incl. debugging the token permissions) | ~1–2 hours to write and validate the GitLab job *once*. Then it's a template for every GitLab service |
| Per-service inputs | name, spec path, environments, URLs | name, spec path, environments, URLs, auth variable name |
| Customer dependencies | URLs, JWT for CI | Items 1–7 above. **This is the critical path, not the automation** |

**Conclusion:** for a GitHub service, onboarding the next service means changing about
five inputs. For the GitLab group it's a one-time port of the same two CLI steps, then
the same five inputs. What takes calendar time is access, credentials and test data
from the customer.
