# APIM to Postman POC

A working proof that an API whose source of truth lives in **Azure API Management** flows into Postman as a spec, generates contract and smoke tests plus governance linting, and runs those gates in CI, with **Git as the source of truth**. The spec in this repo was produced by a real `az apim api export`, not hand-written.

## The two stages, then CI

### Stage 1: extract the spec from APIM (free, programmatic)
APIs in APIM already carry an OpenAPI definition. You export it, you do not write it.

The spec in `openapi/p2p-payments.yaml` was produced with:
```bash
az apim api export \
  --resource-group rg-banking-demo \
  --service-name banking-apim-demo \
  --api-id p2p-payments \
  --export-format OpenApiYamlFile \
  --file-path ./openapi
```
To pull every API in an instance, loop it:
```bash
az apim api list -g <rg> --service-name <apim> --query "[].name" -o tsv |
while read api; do
  az apim api export -g <rg> --service-name <apim> --api-id "$api" \
    --export-format OpenApiYamlFile --file-path ./openapi
done
```
At scale, use **APIOps** (`Azure/apiops-cli`): `apiops extract` writes every API's spec plus policies into a Git repo and scaffolds the Azure DevOps or GitHub pipeline. It keeps GitHub as the source of truth and filters by team.

### Stage 2: spec into Postman, then into CI
From the exported spec, Postman produces the rest:
- **Spec Hub** holds the versioned spec (synced from Git).
- A runnable **collection** is generated from the spec.
- `postman/collections/contract-tests.json` validates documented status codes and the API's `Error` envelope schema across all 8 endpoints.
- `postman/collections/smoke-tests.json` checks each endpoint is up (2xx, fast).
- `postman spec lint` runs OWASP and AI-readiness governance rules.
- `.github/workflows/ci.yml` runs lint, smoke, and contract on every push and PR, the same sequence you run in Azure DevOps.

## What the API looks like
8 operations under the `Peer-to-Peer` tag: list/get P2P transfers, send money, list payment requests, request money, pay and decline requests, and get transfer limits. It uses APIM's `Ocp-Apim-Subscription-Key` header auth and a shared `Error` envelope (`{ error: { code, message, request_id } }`).

## Repo layout
```
openapi/p2p-payments.yaml                         # real az apim api export (source of truth)
postman/collections/contract-tests.json           # contract gate (8 endpoints)
postman/collections/smoke-tests.json              # smoke gate (8 endpoints)
postman/environments/dev-mock.postman_environment.json  # baseUrl -> a Postman mock of the API
.github/workflows/ci.yml                           # lint -> smoke -> contract
```

## Run it locally
```bash
postman spec lint ./openapi/p2p-payments.yaml --no-report-events
postman collection run ./postman/collections/smoke-tests.json    -e ./postman/environments/dev-mock.postman_environment.json
postman collection run ./postman/collections/contract-tests.json -e ./postman/environments/dev-mock.postman_environment.json
```

## Governance in CI
Governance linting needs a logged-in session. Add a `POSTMAN_API_KEY` repo secret to lint against your team's rulesets (OWASP, AI-readiness) and publish results to the API Catalog. Without it, CI runs a syntax-only lint so the build still passes.

## Notes from building it
- The dev env points `baseUrl` at a Postman **mock** of this API, including the server base path (`/p2p-payments`). Cloud mocks match path variables against the example's saved value, so the sample requests use `string` for `:request_id` / `:transfer_id`. Against a real staging backend you use real IDs and point `baseUrl` there.
- Requests force `x-mock-response-code` so the mock returns the documented success example (201 on create, 202 on decline). A real backend ignores that header.
- Wiring the Postman workspace to Git for live two-way sync is a one-time setup in the Postman app (Workspace settings, Integrations, connect to Git). Everything else here is scriptable and runs in CI.
