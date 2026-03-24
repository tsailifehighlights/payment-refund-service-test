# Postman Customer Success Engineer Implementation Exercise

**Candidate:** Andrew Tsai · **Role:** Customer Success Engineer, Postman
> **Note:** The same README.md file has been added to both repositories for your convenience

| Repository | Spec | Workflow |
|---|---|---|
| [payment-refund-service](https://github.com/tsailifehighlights/payment-refund-service) | [`specs/payment-refund-api-openapi.yaml`](https://github.com/tsailifehighlights/payment-refund-service/blob/main/specs/payment-refund-api-openapi.yaml) — Lambda/API Gateway, OAuth 2.0 + JWT | [`.github/workflows/postman-onboard-payment-refund.yml`](https://github.com/tsailifehighlights/payment-refund-service/blob/main/.github/workflows/postman-onboard-payment-refund.yml) |
| [loan-origination-service](https://github.com/tsailifehighlights/loan-origination-service) | [`specs/loan-origination-api-openapi.yaml`](https://github.com/tsailifehighlights/loan-origination-service/blob/main/specs/loan-origination-api-openapi.yaml) — ECS/ALB, mTLS + JWT | [`.github/workflows/postman-onboard-loan-origination.yml`](https://github.com/tsailifehighlights/loan-origination-service/blob/main/.github/workflows/postman-onboard-loan-origination.yml) |

---

## Workflow Overview

Both workflows use [`postman-cs/postman-api-onboarding-action@v0`](https://github.com/postman-cs/postman-api-onboarding-action) — a composite action that chains two lower-level actions in sequence:

1. **`postman-bootstrap-action`** — Creates or reuses a Postman workspace, uploads the spec to Spec Hub, lints it via the Postman CLI, generates baseline/smoke/contract collections, and persists all created asset IDs as GitHub repository variables.
2. **`postman-repo-sync-action`** — Consumes bootstrap outputs to create/update environments, link the workspace to the GitHub repo (Bifrost), create a mock server and smoke monitor, and commit exported collections + environments back to the repo under `postman/`.

The composite orchestrator handles input/output wiring between these two actions so callers only need a single `uses:` step. **Idempotency** is enforced by feeding stored repo variable IDs (`POSTMAN_WORKSPACE_ID`, `POSTMAN_SPEC_UID`, etc.) back into subsequent runs — preventing duplicate assets on retrigger.

**Full step flow:**

- Triggered by a push to `main` (when the spec or workflow file changes) or manually via `workflow_dispatch` with an optional `force_recreate` flag
- Checks out the repository (`actions/checkout@v4`)
- Invokes `postman-api-onboarding-action@v0` which internally:
  - Reads the OpenAPI spec from a public raw GitHub URL (`spec-url`)
  - Creates or reuses a Postman workspace (idempotent via `workspace-id` variable)
  - Assigns the workspace to the configured governance group (`governance-mapping-json`)
  - Uploads/updates the spec in Postman Spec Hub
  - Lints the spec via Postman CLI
  - Generates or reuses baseline, smoke, and contract collections
  - Creates or updates Postman environments for each slug in `environments-json` with URLs from `env-runtime-urls-json`
  - Links the workspace to the GitHub repo (requires `postman-access-token`)
  - Creates a mock server and smoke monitor
  - Exports collections and environments to `postman/` in the repo and commits them back
  - Writes all created asset UIDs back to GitHub repository variables for future runs

---

## Universal vs. Per-Service Configuration

> **Note:** For a small amount of services, per-repo GitHub Action Workflow files are used. As the team scales to additional services, we'll want to explore ways to reduce maintenance overhead. These are explored in the Adaptation plan and could include creating a reusable workflow stored in a central repo, a boilerplate template with Cookiecutter, or a monorepo for all specs

| Aspect | Universal (unchanged) | Per-Service (must update) |
|---|---|---|
| Action reference | `postman-cs/postman-api-onboarding-action@v0` | — |
| Secret names | `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GITHUB_TOKEN`, `GH_FALLBACK_TOKEN` | Secret **values** must be set per-repo in GitHub |
| Repo variable names | `POSTMAN_WORKSPACE_ID`, `POSTMAN_SPEC_UID`, `POSTMAN_BASELINE_COLLECTION_UID`, `POSTMAN_SMOKE_COLLECTION_UID`, `POSTMAN_CONTRACT_COLLECTION_UID`, `POSTMAN_ENV_UIDS_JSON` | Values written automatically after first run |
| `force_recreate` input | Boolean flag pattern | Optional to create new assets |
| `permissions` block | `actions: write`, `contents: write` | — |
| Trigger paths pattern | Spec file + workflow file | **Path values** change per service (e.g. `payment-refund-service`, `loan-origination-service`, `claims-processing-service`) |
| `requester-email` | `"andrew.sd.tsai@gmail.com"` | Update to desired Postman account owner |
| `workspace-admin-user-ids` | `"53049599"` | Update to desired Postman user ID |
| `project-name` | — | Update per service (`payment-refund-service`, `loan-origination-service`, etc.) |
| `domain` | — | Update per service (`payments`, `lending`) |
| `domain-code` | — | Update per service (`PAY`, `LOAN`) |
| `spec-url` | Raw GitHub URL pattern | Points to respective repo + filename |
| `environments-json` | — | Update per service (ex. `["prod", "staging","uat","qa","dev"]`) |
| `env-runtime-urls-json` | — | Update per service (ex. `api.payments.example.com` vs. `lending-api.example.com`) |
| `system-env-map-json` | Initially empty string (no system envs configured) | — |
| `governance-mapping-json` | — | Update per service (ex. `"Payments Platform"`, `"Lending Platform"`) |
| Auth scheme (informational) | JWT default (`{{bearerToken}}` placeholder) | Update per service (ex. Payment adds OAuth2; Loan uses mTLS (manual cert config — see below)) |

---

## Additional Configuration Required

The following cannot be configured without customer access and must be provided by the platform/ops team:

**Customer must supply:**

- **Real runtime URLs** — Replace `example.com` placeholders in `env-runtime-urls-json` with actual gateway endpoints (e.g., API Gateway invoke URLs, ALB DNS names). These live in the workflow YAML and should be sourced from the team's AWS account or internal service registry.
- **System environment IDs** — If the Postman org uses system environments, populate `system-env-map-json` with the corresponding Postman system environment UUIDs (found in Postman Admin Console). Currently empty; Bifrost environment association is silently skipped without them.
- **Governance groups** — Group names (`"Payments Platform"`, `"Lending Platform"`, etc.) should exist in the Postman Admin Console before the workflow runs. Missing groups produce a 404 warning but do not fail the run.
- **Auth credentials** — Real bearer tokens, OAuth client credentials, and mTLS certificates should be injected into Postman environments via the Postman Vault or environment variable overrides — not hardcoded in workflow files. For the loan origination service, mTLS certificate configuration is a **manual step** in the Postman client (cannot be set via environment variables).
> **Note:** mTLS cannot be defined in the components/securitySchemes object in OpenAPI 3.0.x but can be defined in OpenAPI 3.1.0+
- **GitLab CI adaptation** — Migration of the platform team's GitLab services require replicating this workflow as a `.gitlab-ci.yml` pipeline. The underlying action logic (Postman API calls) is CI-system-agnostic, but the YAML syntax, secret injection (`$CI_*` variables), and checkout steps differ. A working session is needed to adapt and test this.
- **Services without OpenAPI specs** — Several services in the customer org lack specs or have drifted specs. The bootstrap action requires a valid OpenAPI 3.x spec at `spec-url`. These services need spec generation (via API gateway export, code annotation, or reverse-engineering from traffic) before onboarding. This is a scoping and prioritization conversation with the platform team.
- **Postman Enterprise Access** — Several features require Postman Enterprise Access (paid or trial) such as Spec Hub, governance groups, and Bifrost features. Activate via Postman Admin Console or reach out to your Postman Customer Success Manager for details.

---

## Playbook: Run Instructions

### Prerequisites

```bash
# 1. Install Postman CLI
curl -o- "https://dl-cli.pstmn.io/install/linux64.sh" | sh   # Linux
brew install postman                                            # macOS

# 2. Generate POSTMAN_API_KEY (PMAK) in Postman → Account Settings → API Keys
# 3. Generate POSTMAN_ACCESS_TOKEN via interactive CLI login
postman login   # completes browser PKCE flow
cat ~/.postman/postmanrc | jq -r '.login._profiles[].accessToken'

# 4. Set secrets on each service repo in GithHub -> Repo Settings -> Secrets and variables -> Actions -> Secrets or programmatically
gh secret set POSTMAN_API_KEY --repo <owner>/<repo>
gh secret set POSTMAN_ACCESS_TOKEN --repo <owner>/<repo>
gh secret set GH_FALLBACK_TOKEN --repo <owner>/<repo>   # GitHub Personal Access Token: repo + workflow scopes, PAT generated in GitHub -> Settings -> Developer Settings -> Personal access tokens
```

### Repository Structure (identical per service)

```
<service-repo>/
├── specs/
│   └── <service>-api-openapi.yaml        # OpenAPI spec — source of truth
├── .github/
│   └── workflows/
│       └── postman-onboard-<service>.yml # Onboarding workflow
└── postman/                              # Auto-generated after first run
    ├── collections/
    └── environments/
```

### Initial Run

1. Push the spec or workflow file to `main`, **or** trigger manually in each repo: `Actions → Postman Onboarding → Run workflow`
2. On first run, leave all `workspace-id`, `spec-id`, collection ID inputs empty — the action creates assets and writes UIDs back to repo variables automatically.
3. Verify the run completes with a green checkmark. Inspect step logs for any governance or token warnings.

### Subsequent Runs / Automation

Re-runs are fully idempotent: the stored repo variables are fed back in, preventing duplicate workspace/collection creation. Use `force_recreate: true` in `workflow_dispatch` only if you need to fully reprovision from scratch.

> **Questions or issues?** Contact your Postman CSE for support.

---

## Validation Notes

After a successful run, verify the following:

**GitHub (Actions tab)**
- Workflow run shows green; no red steps. Yellow warnings for governance/token are non-fatal.
- Repo variables set: `POSTMAN_WORKSPACE_ID`, `POSTMAN_SPEC_UID`, `*_COLLECTION_UID` (3x), `POSTMAN_ENV_UIDS_JSON`
- Commit by `Postman FDE` adding `postman/collections/` and `postman/environments/` JSON exports

**Postman**
- Workspace `<DOMAIN-CODE>-<project-name>` created and visible in your org (ex. `[PAY] payment-refund-service`)
- Spec Hub entry generated for the service
- Three collections: `baseline`, `smoke`, `contract` — each with generated requests
- Environments per slug (e.g., `prod`, `staging`) with `baseUrl` and `bearerToken` variables
- Mock server URL active (test with a GET to a health endpoint)
- Smoke monitor scheduled and visible under Monitors

---

## Trade-offs and Limitations

- **Missing specs** — The action requires a valid, accessible OpenAPI 3.x spec at `spec-url`. Services without specs need spec generation first; this is out of scope for the current workflows and requires a separate discovery sprint with the platform team.
- **GitLab CI** — These workflows are GitHub Actions–native. Adapting for GitLab CI requires rewriting the YAML in `.gitlab-ci.yml` format using GitLab's `script:` blocks and CI/CD variable injection. The underlying Postman API calls are identical; only the CI wrapper changes.
- **mTLS (Loan Origination)** — Service-to-service mTLS cannot be configured via Postman environment variables. Certificate configuration is a manual step in the Postman client or Postman Vault. The workflow scaffolds `{{bearerToken}}` for the JWT path only.
- **Mock server / monitor deduplication** — The `postman-repo-sync-action` does not currently accept existing mock server or monitor UIDs as inputs. Each run with `force_recreate: true` (or on first run) will create new mock servers and monitors. Old instances must be manually deleted from the Postman UI.
> **🔐 Secrets, tokens, and key rotation:**
**`POSTMAN_ACCESS_TOKEN` is session-scoped and will expire** — when it does, governance assignment, workspace linking, and system environment associations silently degrade. Periodically re-run `postman login`, extract the new token, and update the GitHub secret. `POSTMAN_API_KEY` (PMAK) is long-lived but should be rotated per your org's key rotation policy. `GH_FALLBACK_TOKEN` should be regenerated if the GitHub PAT expires or permissions change.

**Out of scope for these workflows:**
- AWS account provisioning, API Gateway configuration, or Lambda/ECS deployment
- GitLab CI pipeline setup and testing
- Postman Enterprise license management or SSO configuration
- Spec generation for services without OpenAPI definitions
- mTLS certificate management and Postman Vault configuration
- Org-wide governance rule enforcement and style guide authoring

*(These are addressed in the customer adaptation plan and consulting design deliverables.)*

---

## Issues Encountered During Exercise

| Issue | Solution |
|---|---|
| Unable to activate Enterprise Trial on existing Postman account | Signed up and activated Enterprise Trial with a new email address |
| Postman user ID not visible in dashboard UI | Discovered the `/settings/me` endpoint at `go.postman.co` and made a direct API call to retrieve the numeric user ID |
| Missing configurations such as CLI install + token extraction | Followed the README's `postman login` flow; cross-referenced the Postman CLI learning center docs to fill gaps |
| Initially created one monorepo (`cse-exercise`) instead of per-service repos | Re-structured into separate `payment-refund-service` and `loan-origination-service` repos to match the action's one-repo-per-service assumption |
| Workflow YAML syntax errors (indentation, missing `with:` keys, unsupported variable references) | Used Cursor and Claude to iterate on YAML; troubleshot and validated with repeat runs and reviewing GitHub Actions runner logs |
| Duplicate workspaces and mock servers created on reruns | Enabled repo variable persistence and confirmed `workspace-id` / collection ID inputs are correctly fed back on rerun |
| Broken or missing links in Postman learning center docs | Located correct doc URLs via web search, referenced `https://learning.postman.com` directly, supplemented with AI learning |
| `GITHUB_TOKEN` lacking `workflow` scope for generated CI commits | Added `GH_FALLBACK_TOKEN` (PAT with `repo` + `workflow`) as fallback credential per README guidance |
| Mock server test calls returning errors | Updated `baseUrl` in the Postman environment to match the generated mock server URL from action output |
| Unfamiliarity with GitLab CI, Amazon API Gateway, ECS/ALB topology | Researched via docs, signed up for free accounts, and created test configurations for learning and approach validation |
| Missing mTLS support for OpenAPI 3.0.x | Reviewing latest OpenAPI Initiative specification standards |

---

## AI Usage

| Tool | Usage |
|---|---|
| **ChatGPT, Gemini, YouTube** | General education on GitLab CI, Amazon API Gateway, ECS/ALB, Microservices Architecture; writing refinement and plain-language explanations |
| **Cursor, Claude** | Drafting, iterating, and debugging GitHub Actions YAML; troubleshooting permission and variable scoping issues |
| **Postman Agent Mode** | Exploring the Postman API directly; testing generated collections against the mock server; debugging |
| **Claude, Gamma** | Implementation brainstorming; presentation structure and overall narrative |

**Human-validated (not AI-generated):** Final selection of `project-name`, `domain`, `domain-code`, and environment slugs from the spec; exact `spec-url` format and branch reference; secret names and required GitHub permissions; run and validation checklist steps; all fixes identified from actual workflow runs (token issues, permission errors, duplicate asset behavior). Reviewed all output generated by AI for quality and clarity.

---

## Feedback & Issues for Future CSE Exercise Improvement

I noticed some gaps that could improve the assignment for future candidates and have included them here.

- **No input for existing mock server / monitor UIDs** — `postman-repo-sync-action` lacks `mock-server-id` and `monitor-id` inputs, so new instances are created on every forced rerun. Adding these as optional passthrough inputs (persisted as repo variables alongside collection IDs) would complete the idempotency story.
- **Loan Origination spec missing `500` / `503` responses** — Standard error codes absent from all paths result in GitHub Actions warnings and incomplete contract collection test coverage. Adding these to the spec would improve generated contract test quality.
- **Pre-defining Governance groups** — First-time runners hit a 404 warning for `"Payments Platform"` / `"Lending Platform"` because these groups don't exist in a trial org. The exercise README could note that governance groups need to be created in Postman Admin Console as a setup step to pre-empt this issue.
- **Requester / admin role collision** — On a small Enterprise Trial team, the `requester-email` and `workspace-admin-user-ids` often resolve to the same user, producing a workflow warning about duplicate role assignment. The exercise could note this is expected behavior in trial environments with limited seats.
