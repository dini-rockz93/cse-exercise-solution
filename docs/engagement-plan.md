# Engagement Plan: Value, Rollout, Handoff

Presentation content for sections 1, 3 and 4 of the deck. Every number is either
**measured** (from this repo's real runs) or an **assumption** (marked *A*) to validate
with the VP of Platform Engineering in week 1.

---

## 1. Value

### The pain today (from the customer brief)

- Finding and successfully calling an internal API **takes hours**: jumping between
  systems, hitting dead ends, relying on tribal knowledge.
- Writing tests against an internal endpoint starts from nothing: no spec, no collection,
  no environment config.
- Hundreds of workspaces with no consistent structure. Specs are missing or have drifted.
  There's no catalog and no contract testing.

### How Postman fixes it

Each problem maps to a Postman capability. Rows marked *Built* were created for
payment-refund by one onboarding run, from the spec alone. The platform team runs the
workflow; service teams don't set anything up by hand in Postman.

| Problem today | Postman capability | What it gives engineers | Status |
|---|---|---|---|
| No catalog | A workspace per service, owned by a service account, with its spec in Spec Hub | One place to find an API, its docs and its environments | Built: `01-workspace-overview.png`, `02-spec-hub.png` |
| Tests start from zero | Baseline, smoke and contract collections generated from the spec | Requests and schema checks for every operation, with no hand-written tests | Built: `postman/exports/payment-refund-api/` |
| Specs drift | Postman CLI running the contract collection in CI on every push | Drift flagged in the pipeline, not by a consumer in staging | Built against the mock; runtime drift needs a reachable dev URL and credential |
| No ongoing signal | Postman monitors on non-production environments | Scheduled checks with alerts | Monitor exists, unscheduled, pointed at the mock (`07-monitor.png`); scheduling and alert routing are planned |
| Services with no spec | Postman Insights, or a code-first spec the owner writes | A spec to onboard where none exists | Planned (lane D), not demonstrated |

### What one onboarding run replaces

| Manual step today | After | Evidence |
|---|---|---|
| Find the spec (or the person who knows the API) | Spec in Spec Hub, in a workspace named after the service | Measured: `02-spec-hub.png` |
| Hand-build a collection, request by request | Baseline, smoke and contract collections generated | Measured: 3 collections covering all 6 operations |
| Work out base URLs and env vars per stage | dev / qa / uat / prod environments with the right `baseUrl` | Measured: `04-environment-prod.png` |
| Wait for a backend before integrating | Mock server answering realistic examples | Measured: `06-mock-response.png` |
| Write tests from scratch | 207 spec-derived contract assertions plus smoke checks, in CI | Measured: latest contract run (CI run 44470584474) |
| Wire CI | CI workflow committed to the repo | Measured: `ci.yml` |
| Discover the spec is wrong when a consumer breaks | Defects flagged by CI on day one | Measured: 2 spec defects found, fixed, re-verified |

**Time per service:**

| | Manual (*A*) | Automated (measured) |
|---|---|---|
| First onboarding | ~1–2 engineer-days (*A*) | **94 s** workflow elapsed time (measured) + ~15 min of inputs (*A*) |
| Re-sync after a spec change | ~2–4 hours (*A*) | **40–47s**, assets updated in place, no duplicates |
| Time to first successful call | 2–4 hours (*A*, brief says "hours") | **< 5 minutes**: open workspace → pick env → send (mock works before a backend exists) |

**At pilot scale (50 services):**
- Onboarding: 50 × ~1.5 days ≈ **75 engineer-days avoided** (*A*), against roughly
  12 hours of input-filling with the automation.
- Discovery: if pilot engineers hit "find and call an internal API" **10 times a week**
  at ~2 hours each (*A*), that's **~20 engineer-hours a week back**, or about half an
  engineer, recurring.

### The "incidents" this prevents (not production outages)

In a large enterprise most incidents are friction, not outages. This automation targets:

| Incident type | How it happens today | What changes |
|---|---|---|
| **Blocked integration** | A team waits days for docs, URLs or a working example | Catalog + environments + mock: unblocked the same day |
| **Contract drift breaks a consumer** | The spec says one thing, the service does another, and a consumer finds out in staging | Contract tests flag drift on each push once CI reaches a real runtime; today they found a published example that breaks its own schema (`amount: null` vs `integer`) |
| **Wrong docs copied into code** | A developer copies a docs example that fails validation | The spec's own examples are validated against its schema |
| **Red pipelines everyone ignores** | Noisy, untrusted checks | Advisory mode until real environments exist, then blocking. No alert fatigue from a monitor that's always red |
| **Audit question: "which APIs exist, who owns them?"** | Spreadsheet archaeology | Catalog with an owner per workspace (service account + team) |

### Metrics for the CIO readout (Day 60)

| Metric | Baseline (week 1) | Day-60 target (*A*) |
|---|---|---|
| Pilot services in the catalog with spec + collections | ~0 | 35–40 of 50 |
| Services with contract tests in CI | 0 | 35–40 |
| Median time to first successful call | measure with 3–5 engineers | < 5 min |
| Spec defects caught before a consumer | not tracked | tracked per wave |
| Platform-team onboardings without CSE | 0 | ≥ 10 |

**Slide headline:** *"One run turns a spec into a catalogued, mockable, tested API in under
2 minutes, and on day one it found two defects in the spec itself."*

---

## 2. Rollout: ~50 services in 60 days, handoff by Day 90

### Step 0: sort the services, don't just repeat the same steps 50 times

Week 1 inventory (with the senior DevOps engineer) puts every service in one lane. Each
lane has a different critical path:

| Lane | Profile | Path | Est. share (*A*) |
|---|---|---|---|
| **A: fast lane** | GitHub Actions + a usable spec | Onboarding workflow as-is (5 inputs) | ~50% |
| **B: no or drifted spec, on API Gateway** | Most Lambda services | Export the spec from API Gateway (`postman-aws-spec-discovery-action`) → onboard → drift report to the service team | ~25% |
| **C: GitLab CI** | The insurance group | One-time port to the CLI (`docs/adaptation-claims.md`), then lane A inputs | ~15% |
| **D: ECS/ALB/K8s with no spec** | No gateway export available | Postman Insights or code-first generation, plus service-team spec ownership | ~10% |

### Critical path

Access and credentials are the critical path, not the automation:

```
Service account + workspace policy ─┐
GitHub App (repo writes) ───────────┼─► Pilot (payments) ─► Wave 1 ─► Wave 2 ─► Day 60 readout
Inventory + lane sort ──────────────┘          │
CI identity per domain (JWT / client creds) ───┴─► switch CI from mock to real dev env
GitLab runner + token ─► GitLab port (claims) ─────────────────────► joins Wave 2
AWS read role (OIDC) ─► API Gateway spec export (lane B) ──────────► Wave 2
```

### Timeline

| Weeks | Track 1: rollout | Track 2: parallel enablers | Who drives |
|---|---|---|---|
| **1** | Inventory + lane sort. Baseline metrics. Agree success criteria with the VP | Request: service account, GitHub App, AWS read role, GitLab token, CI identities | CSE + DevOps engineer |
| **2–3** | **Pilot: 3–5 payments services** (willing EM). Turn the workflow into a reusable `workflow_call` template in the platform repo | GitLab port for claims starts | CSE does, DevOps watches |
| **4–5** | **Wave 1: rest of payments + lending lane A.** Lending's QA lead (prior Postman user) champions the tests. mTLS certificates via the `ssl-client-*` inputs | Lane B: API Gateway export for Lambda services without specs | **DevOps does, CSE watches** |
| **6–8** | **Wave 2: lane B services + claims (GitLab).** Drift reports go to service teams | First services switch CI from mock to real dev envs as credentials land | Platform team runs it, CSE reviews |
| **Day 60** | **CIO readout:** catalog coverage, CI coverage, time to first call, defects caught | | VP presents, CSE supports |
| **9–13** | Lane D (no-spec services). Governance: spec linting + breaking-change checks in CI. Monitors scheduled with alert routing | Org-wide expansion proposal (hundreds of APIs) | Platform team owns; CSE exits by Day 90 |

**Being honest with the VP:** lanes A–C (about 40 services) can realistically be in the
catalog with CI tests by Day 60. Lane D depends on service teams writing or accepting
specs, so it may run into the Day 60–90 window.

### What can run in parallel

- Credential requests run **in parallel** with the pilot (the pilot uses the mock until
  CI identities land, as this repo does).
- The GitLab port is a **separate track**. It doesn't block GitHub waves.
- Within a wave, onboarding is **per-repo and independent**, so the platform team can run
  10 dispatches at once (the concurrency guard is per service).

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| CI credentials take weeks (security review) | Pilot on the mock in advisory mode, as proven here. Report "catalog + mock" value first |
| Drifted specs generate noisy failures | Advisory mode per service until the spec owner triages. Drift reports, not red builds |
| Bot commits on every run (noise) | Filter bot commits in CI. Raise with Postman as product feedback |
| Hand-edits in Postman drift from the pipeline (e.g. monitor environment) | Everything managed through workflow inputs. Hand-edits go in a decision log |
| Trial and seat limits during rollout | Confirm licensing before Wave 1 |

---

## 3. Handoff and ownership

### What we leave behind

| Artifact | Purpose |
|---|---|
| **Reusable onboarding workflow** (`workflow_call`, pinned action versions) | One line per service repo to onboard |
| **GitLab CI template** | Same pattern for the insurance group |
| **Check-secrets preflight** | First thing to run when a new repo onboards |
| **Runbook** | Onboard a service; rotate keys; switch `CI_TARGET`; schedule a monitor; troubleshooting table (from this repo's "Issues I hit") |
| **Service inventory + lane sheet** | What's onboarded, the blocker per service, the owner |
| **Ownership matrix** | Who fixes what (below) |
| **Decision log** | Why mock-mode CI, why monitors stay paused, why the starter was bypassed |
| **Metrics view** | The Day-60 numbers, kept current |

### Who owns what after Day 90

| Area | Platform team (4 people) | Service teams | CSE / TAM |
|---|---|---|---|
| Onboarding template + action version upgrades | **Owns** | – | Advises on major versions |
| Credentials (service account key, GitHub App, rotation) | **Owns** | – | – |
| Onboarding a new service (5 inputs) | Self-serve | **Can self-serve** | – |
| Spec quality, fixing contract failures | Routes | **Owns** | – |
| Service-specific tests (business rules, auth flows, test data) | – | **Owns** | Optional PS engagement |
| `CI_TARGET` and monitor schedules / alert routing | **Owns** | Consulted | – |
| New CI platform, new auth pattern, product gaps | Requests | – | **CSE/TAM** |

**Self-serve by Day 90:** onboarding, reruns, adding environments, switching mock → real,
rotating credentials, and the common failures in the runbook.
**Still needs CSE/TAM:** a new CI system, a new auth scheme, major action upgrades, and
anything that turns out to be a product gap (filed as feedback).

### The working session (90 minutes, with the senior DevOps engineer, week 2)

| Time | Activity | Outcome |
|---|---|---|
| 10 min | Walk through one real run: inputs → assets → state file → CI | Shared mental model |
| 30 min | **They drive:** onboard a payments service live; CSE only answers questions | First onboarding done by the customer |
| 30 min | **Break/fix drills** from real issues: read-only GitHub token (403), local-only env value (monitor hits `http://refunds/`), drifted spec failing contract tests, expired credential | They've already fixed each failure once |
| 20 min | Agree lane assignments, owners and the Wave 1 list | A plan they own |

Follow-up sessions follow "I do → we do → you do": CSE drives the pilot, pairs on
Wave 1, and only reviews Wave 2.

### Day-90 exit criteria (CSE exits when all are true)

1. The platform team has onboarded **≥ 10 services without CSE help**.
2. A new service goes from request to catalog in **< 1 hour**, inputs included.
3. The platform team resolved **at least one real failure using only the runbook**.
4. Pilot services run CI against **real dev environments**, with blocking tests.
5. Monitors are **scheduled with alert routing** for services with reachable URLs.
6. Product feedback is filed (bot noise commits, monitor environment reset on rerun),
   and the org-wide expansion proposal goes to the VP.

**After Day 90:** TAM for ongoing support. PS if they want service-specific test suites
or the org-wide rollout delivered for them.
