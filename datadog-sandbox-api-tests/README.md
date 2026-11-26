# datadog-sandbox-api-tests

A Claude skill that runs API call tests against a **Datadog sandbox org** using the Postman CLI, then reports real pass/fail results.

> Status: drafted but not yet run end to end against Postman or Datadog. Expect small fixes on the first run.

## What it does

When you ask Claude to test, smoke-check or debug Datadog API calls (for example "run the Datadog tests in Postman" or "check my sandbox"), the skill:

1. Confirms the Datadog site and that the target org is a sandbox.
2. Reads your keys from environment variables (never from files or chat).
3. Runs the bundled Postman collection with `postman collection run`.
4. Reports pass/fail counts, the exit code, and for each failure the request, status code and Datadog's error message.
5. Checks that anything created by write tests was cleaned up.

## Contents

| File | Purpose |
| --- | --- |
| `SKILL.md` | Instructions Claude follows: workflow, safety rules, failure table, how to extend |
| `assets/datadog-sandbox.postman_collection.json` | The tests (Postman v2.1 format) |
| `assets/datadog-sandbox.postman_environment.json` | Environment template with blank secrets |

## The tests

**Folder 1: Auth and read-only** (safe to run any time)
- Validate API key (`GET /api/v1/validate`)
- List monitors, dashboards, users and Synthetics tests
- Metric query over the last hour
- Negative test: an invalid key must be rejected (403)

**Folder 2: Write and cleanup (sandbox only)**
- Create a monitor named `postman-test-monitor`, fetch it, delete it, then confirm it returns 404

Each request has `pm.test` assertions on the status code and on response shape, not on volatile values such as counts or timestamps.

## Requirements

- Postman CLI installed and logged in (`postman --version`)
- A Datadog **sandbox** org with an API key and an application key. The app key needs scopes for the endpoints you test (for example `monitors_write` for folder 2)
- Keys exported in your shell:

```bash
export DD_API_KEY=...   # never commit these
export DD_APP_KEY=...
```

## Install

Copy this folder into your skills directory, for example `~/.claude/skills/datadog-sandbox-api-tests/`, keeping `SKILL.md` at the top and the JSON files under `assets/`.

## Run it manually

Read-only checks:

```bash
postman collection run assets/datadog-sandbox.postman_collection.json \
  -e assets/datadog-sandbox.postman_environment.json \
  --env-var "api_key=$DD_API_KEY" --env-var "app_key=$DD_APP_KEY" \
  --env-var "site=datadoghq.com" \
  -i "01 - Auth and read-only"
```

Write tests (sandbox only): use `-i "02 - Write and cleanup (sandbox only)"` instead.

Set `site` to your region: `datadoghq.com` (US1), `datadoghq.eu`, `us3.datadoghq.com`, `us5.datadoghq.com`, `ap1.datadoghq.com` or `ddog-gov.com`. Flags can differ between CLI versions, so check `postman collection run --help` if one is rejected.

## Safety design

- **Keys** are passed at run time with `--env-var` and never written to the environment file or printed.
- **Writes** run only after you confirm the org is a sandbox, and everything they create is tagged `postman-test` and deleted in the same run.
- **Rate limits** (429) are reported, not retried in a loop.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| 403 on `/validate` | Wrong site for the key, or key revoked |
| 403 on other endpoints | App key missing or lacks the needed scope |
| 401 | Missing or incorrect API key |
| 400 on metric query | `from`/`to` must be Unix seconds, or the query is malformed |
| Nonzero exit with HTTP 200 | A `pm.test` assertion failed |
| Proxy or connection 403 | The network blocks `api.<site>`; allow the host |

## Extending

Add requests to the collection with assertions saved inside each request. Log intake uses a different host (`http-intake.logs.<site>`), and metric submission is `POST /api/v2/series`. See https://docs.datadoghq.com/api/latest/ for request bodies.
