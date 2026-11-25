---
name: datadog-sandbox-api-tests
description: Run and maintain Postman-based API call tests against a Datadog sandbox/test org. Use whenever the user wants to test, smoke-check, validate or regression-test Datadog API calls (monitors, dashboards, metrics, logs, events, synthetics, users, API/app key validity), run or extend a Postman collection against Datadog, or debug a failing Datadog API request, even if they only say "check my sandbox" or "run the Datadog tests in Postman".
---

# Datadog sandbox API tests (via Postman)

Runs Datadog API calls against the user's **sandbox** org with the Postman CLI, asserts on the responses, and reports real pass/fail counts. Starter assets live next to this file:

- `assets/datadog-sandbox.postman_collection.json` - smoke + CRUD tests (Postman v2.1 format)
- `assets/datadog-sandbox.postman_environment.json` - environment template (no secrets)

## Why the guardrails exist

The API keys grant real access to a Datadog org. Tests that write (monitors, events, metrics) leave artifacts, and a wrong site or wrong org means writing into production. So: confirm the target, keep secrets out of files and output, and clean up what you create.

## Workflow

1. **Confirm the target.** Ask (once) for: Datadog site (`datadoghq.com`, `datadoghq.eu`, `us3.datadoghq.com`, `us5.datadoghq.com`, `ap1.datadoghq.com`, `ddog-gov.com`) and confirm it is the sandbox org. Never run the write folder against an org the user hasn't said is a sandbox. Starting with read-only is always fine.
2. **Credentials.** Expect `DD_API_KEY` and `DD_APP_KEY` in the environment. Most v1/v2 endpoints need both; `/api/v1/validate` needs only the API key. If missing, ask the user to export them. Do not ask them to paste keys into chat, do not write them to the environment file, and do not echo them. Pass them at run time with `--env-var`.
3. **Resolve Postman.** Check `postman --version`; if missing or not logged in, follow the `postman:bootstrap` skill. Local files need no workspace, but cloud collections do. If the Postman MCP server is unavailable (e.g. proxy/network refusal), continue with the CLI and local files and tell the user.
4. **Run read-only first**, then writes if approved:
   ```bash
   postman collection run assets/datadog-sandbox.postman_collection.json \
     -e assets/datadog-sandbox.postman_environment.json \
     --env-var "api_key=$DD_API_KEY" --env-var "app_key=$DD_APP_KEY" \
     --env-var "site=datadoghq.com" \
     -i "01 - Auth and read-only" -r junit
   ```
   Run `-i "02 - Write and cleanup (sandbox only)"` only after the user confirms the sandbox. Check `postman collection run --help` if a flag differs in the installed version.
5. **Report actual results**: pass/fail counts, exit code, and for each failure the request, status code and Datadog's `errors` message. Note that a nonzero exit can mean a failed `pm.test` even on HTTP 200.
6. **Verify cleanup** after write runs: the collection deletes what it creates, but if a run aborted midway, search for leftovers (monitor name prefix `postman-test-`) and delete them, mentioning it to the user.

## Reading failures

| Status | Usual cause |
| --- | --- |
| 403 `Forbidden` on `/validate` | Wrong site for the key, or key revoked |
| 403 on other endpoints | App key missing or lacks scope (e.g. `monitors_write`) |
| 401 | Missing/incorrect `DD-API-KEY` |
| 400 on `/query` | `from`/`to` must be Unix seconds; bad query syntax |
| 429 | Rate limited; check `X-RateLimit-Reset` and retry later, do not hammer |
| Connection/proxy 403 | Network policy blocks `api.<site>`; user must allow the host |

## Extending the tests

Add new requests to the collection with `pm.test` assertions saved inside the request, so they outlive the session. Pattern: assert status, assert the response shape (key fields exist), never assert on volatile values (counts, timestamps). Log intake uses a different host (`http-intake.logs.{{site}}`), and metric submission is `POST /api/v2/series`. Prefer the Datadog docs (`https://docs.datadoghq.com/api/latest/`) for request bodies rather than guessing. For new write tests, tag created resources with `postman-test` and delete them in a final request.

## Out of scope

Don't use this against production orgs, load-test the API (rate limits are per org), or store keys in collections. For load tests use `postman:performance-testing` with explicit user confirmation.
