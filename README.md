# APIM to Postman POC

A reference proof that an API whose source of truth lives in **Azure API Management** can flow into Postman as a spec, generate contract and smoke tests plus governance linting, and run those gates in CI, with **Git as the source of truth**.

## The two stages, then CI

### Stage 1: get the spec out of APIM (free, programmatic)
APIs in APIM already have an OpenAPI definition. You extract it, you do not hand-write it.

- **APIOps** (`Azure/apiops-cli`): extracts every API's OpenAPI definition and policies from APIM into a Git repo. Keeps GitHub as the source of truth, filters by team, scaffolds the Azure DevOps or GitHub pipeline.
- **Scripted `az`** (one-time pull): `az apim api list` to enumerate, then `az apim api export --export-format OpenApiJsonFile` per API.

`openapi/p2p-payments.yaml` stands in for that extracted spec.

### Stage 2: spec into Postman, then into CI
- The spec is pushed to Postman Spec Hub (via Git-native sync, or the Postman API).
- From the spec, Postman generates the runnable collection.
- `postman/collections/contract-tests.json` validates every response against the spec's documented status codes and schema shapes.
- `postman/collections/smoke-tests.json` checks each endpoint is up: success status, fast, non-empty body.
- `postman spec lint` runs OWASP and AI-readiness governance rules.
- `.github/workflows/ci.yml` runs all three on every push and PR, the same sequence you would run in Azure DevOps.

## Repo layout
```
openapi/p2p-payments.yaml                         # the spec (source of truth, = APIM export)
postman/collections/contract-tests.json           # contract gate
postman/collections/smoke-tests.json              # smoke gate
postman/environments/dev-mock.postman_environment.json  # baseUrl -> a Postman mock
.github/workflows/ci.yml                           # lint -> smoke -> contract
```

## Run it locally
```bash
postman spec lint ./openapi/p2p-payments.yaml --no-report-events
postman collection run ./postman/collections/smoke-tests.json   -e ./postman/environments/dev-mock.postman_environment.json
postman collection run ./postman/collections/contract-tests.json -e ./postman/environments/dev-mock.postman_environment.json
```

## Governance in CI
Governance linting needs a logged-in session. Add a `POSTMAN_API_KEY` repo secret to lint against your team's rulesets (OWASP, AI-readiness) and to publish results to the API Catalog. Without it, CI runs a syntax-only lint so the build still passes.

## Wiring Postman's Git-native sync (one-time, in the Postman app)
1. Postman desktop, open the target workspace.
2. Workspace settings, Integrations, connect to Git, authorize GitHub, pick this repo and branch.
3. Choose what syncs (spec, collections, environments). Commit and push from Postman or the terminal, either direction.

The Postman API does not expose this workspace-to-Git link today, so it is a UI step. Everything else here is scriptable and runs in CI.
