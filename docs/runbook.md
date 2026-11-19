# Runbook: Operating the Onboarding Pipeline

For the platform team. Covers recovery after a partial run, the manual changes a rerun
can undo, switching CI targets, and the failures seen while building this pilot.

> Scope: one service per repo. `vars.SERVICE_NAME` (default `payment-refund-api`)
> names the service that CI tests and exports.

---

## 1. Recovery: bootstrap succeeded, repo sync failed

**Symptoms**
- The Onboard Service summary shows `bootstrap=success, repo-sync=failure` and a
  "Partial success" note.
- No new `chore: sync Postman artifacts and metadata` commit on `main`.
- `.postman/resources.yaml` doesn't have the new IDs, or doesn't exist yet (first run).

**Why it matters:** the Postman assets now exist, but the repo has no record of them.
A plain rerun relies on that state file to find them, so it might create a second set.
On reruns of this pilot, bootstrap also adopted the workspace through its repo link
(*"Repository … is already linked to workspace …; adopting it"*), so the workspace is
less at risk than the collections, mock and monitor.

**Steps: don't retry until 1–3 are done.**

1. **Get the IDs bootstrap created** from the failed run's summary (workspace URL,
   spec ID, collections JSON), or its log:
   ```bash
   gh run view <run-id> --log | grep -E 'workspace-id|spec-id|collection|Using existing|Created'
   ```
2. **Check what the repo already has:**
   ```bash
   git pull && git log -1 --format='%h %s %cr' -- .postman/resources.yaml && cat .postman/resources.yaml
   ```
3. **Check what Postman has, and look for duplicates.** Run in a terminal; the key
   isn't echoed:
   ```bash
   read -s "PMAK?PMAK: " && echo && curl -s "https://api.getpostman.com/workspaces/<workspace-id>" -H "X-Api-Key: $PMAK" \
     | jq '.workspace | {collections: [.collections[]?.name], environments: [.environments[]?.name], mocks: [.mocks[]?.name], monitors: [.monitors[]?.name]}'; unset PMAK
   ```
4. **Decide:**

   | What you found | Action |
   |---|---|
   | State file already has the IDs | Plain rerun |
   | Assets exist, state file doesn't have them | Rerun with `existing_ids_json` (below) |
   | Duplicates exist (e.g. two `[Smoke]` collections) | Delete the extra copies in Postman (note which), then rerun with the IDs to keep |
   | Nothing was created | Fix the cause, then a plain rerun |

5. **Rerun with the IDs to keep**, so nothing is created twice:
   ```bash
   gh workflow run onboard-service.yml -f project_name=payment-refund-api \
     -f existing_ids_json='{"workspace_id":"<ws>","spec_id":"<spec>","baseline_collection_id":"<uid>","smoke_collection_id":"<uid>","contract_collection_id":"<uid>"}'
   ```
   Allowed keys: `workspace_id`, `spec_id`, `baseline_collection_id`,
   `smoke_collection_id`, `contract_collection_id`, `monitor_id`, `mock_url`. The preflight
   rejects anything else.
6. **Confirm:** a new sync commit exists, `.postman/resources.yaml` lists the same IDs
   you passed, and the next CI run's summary shows the expected target.

**Common cause of repo-sync failing:** the GitHub token can't write. A `403` on
variables or contents means its fine-grained permissions are read-only (see §4).

---

## 2. Manual changes a rerun can undo

| Change made by hand | Risk on rerun | What to do |
|---|---|---|
| Monitor re-pointed from `prod` to the `payment-refund-api - mock` environment | **Observed 19 Nov:** the next run no longer matched it and **created a second monitor on prod**, which ran once. The state file doesn't record monitors | Pin the monitor to keep: `gh variable set POSTMAN_MONITOR_ID --body <id>` (onboarding passes it as `monitor-id`). Delete a duplicate, or keep it disabled and note it in the decision log (the pilot keeps `1f1e41a4…` as evidence). After every run, check the monitor count and environment. Keep monitors **unscheduled** until they target a reachable, non-production environment |
| Environment value edited in the app | Local-only values are invisible to monitors and CI | Use **☁ Share** (shared value) for anything a monitor or CI needs; never share secrets this way |

Record every hand-edit in a decision log. If something has to survive reruns, it should be
a workflow input, not a click.

---

## 3. Switching the CI target

CI never targets production: the smoke collection creates refunds.

| `vars.CI_TARGET` | Behaviour |
|---|---|
| `mock` | Dev environment with `baseUrl` = `vars.POSTMAN_MOCK_URL`. Smoke/contract **advisory**; the `POST /refunds → 201` happy path is **blocking** |
| `dev` | Dev environment as-is. Everything **blocking**. Needs a reachable dev URL and a CI credential |
| unset / anything else | CI fails with an explanation |

```bash
gh variable set CI_TARGET --body dev
```

Read-only production probes, if wanted, belong in a separate GET-only collection and job.

---

## 4. Failures seen while building this pilot

| Symptom | Cause | Fix |
|---|---|---|
| `200` on reads, `403` on every write (GitHub API) | Fine-grained token had no repository permissions; reads passed only because the repo is public | Actions, Contents, Variables, Workflows = read and write; verify with a write probe that returns `422` |
| Monitor requests go to `http://refunds/...`, 6 connection errors | `baseUrl` saved as a **local** value; monitors run in the cloud | ☁ Share the value |
| `AuthenticationError`, key "length 0" | Testing from a shell where the variable wasn't set | Test in a terminal with `read -s` |
| Every request "No response" | Environment points at a host that doesn't resolve (`*.example.com`) | Point at a reachable environment, or the mock |
| `POST /refunds` → `503` on the mock | 9 examples share one request; the mock picks one | CI checks the 201 happy path with `x-mock-response-code: 201`; the generic smoke run stays advisory |
| Advisory step shows a green check but failed | `continue-on-error` | Read the job summary counts, not the check marks |
| Preview GC fails with `E404 @postman/fs-utils` | Upstream package not published; no previews in this pilot | Workflow is manual-only; product feedback filed |

---

## 5. Credentials

| Credential | Where | Rotation |
|---|---|---|
| Postman service-account API key | Secret `POSTMAN_API_KEY` | Rotate in Postman → `gh secret set POSTMAN_API_KEY`; run **Check secrets** |
| GitHub token for commit-back | Secret `GH_PAT` (pilot); a GitHub App in production | Before expiry; run **Check secrets** |
| Postman access token | Minted per run from the API key | Nothing to rotate |
