# Postman API Onboarding — Loan Origination API

**Author:** Akshay Singh
**Service:** Lending Platform API - Loan Origination Service
**Part of:** Postman Customer Success Engineer Technical Assessment

---

## Overview

This repo contains the end-to-end Postman onboarding implementation for the Loan Origination API — the second service onboarded as part of the CSE assessment. It was chosen specifically because it has meaningful differences from the Payment Refund API (Service 1), making the delta between services easy to articulate.

**Key differences from Service 1 (Payments):**

| Characteristic | Payment Refund API | Loan Origination API |
|---|---|---|
| Auth | OAuth 2.0 + JWT | JWT + mTLS |
| Environments | prod, uat, qa, dev (4) | prod, staging, dev (3) |
| Notable endpoints | Partial refunds, HATEOAS links | Multipart file uploads, async underwriting |
| Lint findings | Schema `$ref` issues | Missing 5xx responses + schema issues |

**Repo:**
- GitHub: `singhaksshay/postman-loan-origination-api`
- Postman workspace: Created and linked automatically by the workflow

---

## How I Built This

### Step 1 — Reading the Action READMEs

Before writing any code, I read all three action READMEs to understand how the composite action chains bootstrap and repo-sync together:

- `postman-cs/postman-api-onboarding-action` — the composite orchestrator
- `postman-cs/postman-bootstrap-action` — handles Postman-side setup
- `postman-cs/postman-repo-sync-action` — handles repo-side sync

### Step 2 — Setting Up Authentication

Three secrets required per repo:

- **`POSTMAN_API_KEY`** (PMAK): Long-lived key for all standard Postman API operations
- **`POSTMAN_ACCESS_TOKEN`**: Session-scoped token for internal operations (Bifrost workspace linking, governance). Extracted via `postman login` + `jq`
- **`GH_FALLBACK_TOKEN`**: GitHub PAT with `repo` + `workflow` scopes — needed because the default `GITHUB_TOKEN` can't write workflow files or repo variables

### Step 3 — Spec URL

The action's `spec-url` input requires a publicly accessible URL. The spec is committed to this repo and served via raw GitHub URL:
```
https://raw.githubusercontent.com/singhaksshay/postman-loan-origination-api/main/specs/loan-origination-api-openapi.yaml
```

### Step 4 — Workflow Design Decisions

- Used the composite action (`postman-api-onboarding-action`) rather than calling bootstrap and repo-sync directly — right abstraction for standard onboarding
- Set `generate-ci-workflow: false` to prevent the action from generating a conflicting workflow file
- Environment names (`prod`, `staging`, `dev`) pulled directly from the spec's `servers` block — not invented
- One repo per service — mirrors how this tooling works in real engagements

---

## Workflow Architecture
```
GitHub Push / workflow_dispatch
        │
        ▼
postman-api-onboarding-action@v0 (composite)
        │
        ├── postman-bootstrap-action@v0
        │     ├── Creates/reuses Postman workspace
        │     ├── Uploads spec to Spec Hub + lints it
        │     ├── Generates [Baseline], [Smoke], [Contract] collections
        │     └── Persists IDs as GitHub repo variables for safe reruns
        │
        └── postman-repo-sync-action@v0
              ├── Creates environments from runtime URLs
              ├── Creates mock server + smoke monitor
              ├── Links workspace to GitHub repo (Bifrost)
              └── Commits all artifacts back to repo
```

---

## What's Universal vs. What Changes Per Service

### Universal (identical across all services)

- The composite action reference: `postman-cs/postman-api-onboarding-action@v0`
- Required secrets: `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GH_FALLBACK_TOKEN`
- Permissions block: `actions: write`, `contents: write`
- `generate-ci-workflow: false`
- Output structure: `postman/collections/`, `postman/environments/`, `.postman/`
- Three collection types: Baseline, Smoke, Contract
- Rerun safety via stored repo variables

### Changes Per Service

| Input | Value for this service |
|---|---|
| `project-name` | `loan-origination-api` |
| `domain` | `lending` |
| `domain-code` | `LND` |
| `spec-url` | Raw GitHub URL for this repo's spec |
| `environments-json` | `["prod","staging","dev"]` |
| `env-runtime-urls-json` | 3 URLs from the spec's `servers` block |

---

## Spec Characteristics

- **Auth:** JWT (internal gateway) + mTLS (service-to-service)
- **Notable endpoints:** Multipart file upload (`/documents`), async underwriting trigger, paginated application listing
- **Business rules:** Applicant must be 18+, loan amount $1K–$500K, one active application per product type
- **Lint findings:** Missing 5xx responses on several operations, schema `$ref` recommendations

### mTLS Note

The composite action does not handle client certificate configuration. For mTLS services in a real engagement, client certs and keys would be configured manually in the Postman environment post-onboarding — approximately 1-2 hours per service.

---

## Run Instructions

### Prerequisites

- Postman Enterprise account with API key (`PMAK-...`)
- Postman CLI installed and authenticated (`postman login`)
- GitHub CLI installed and authenticated (`gh auth login`)
- GitHub PAT with `repo` and `workflow` scopes

### Triggering the Workflow

**Manual:**
```bash
gh workflow run onboard-api.yml --repo singhaksshay/postman-loan-origination-api
```

**Watch the run:**
```bash
gh run watch --repo singhaksshay/postman-loan-origination-api
```

**Automatic:** Push a change to the spec file or workflow file.

### Validating the Output
```bash
git pull
ls postman/collections/   # [Baseline], [Smoke], [Contract] folders
ls postman/environments/  # prod, staging, dev JSON files
ls postman/exports/       # JSON collection exports
cat .postman/config.json  # Workspace ID, spec ID, collection IDs
```

### Rerun Behavior

On rerun, the action reads stored GitHub repository variables and reuses existing Postman assets — no duplicates created.

---

## Warnings Encountered (Not Failures)

| Warning | Explanation |
|---|---|
| Requester invite failed (400) | Already the workspace owner — Postman rejected duplicate role. Expected behavior. |
| Node.js 20 deprecation | Actions run on Node.js 20, GitHub deprecating June 2026. Not a functional issue today. |
| Missing 5xx responses | Spec quality issue — several operations don't define error responses. Flagged for API team. |
| Schema `$ref` recommendations | Inline schemas could be extracted to reusable `$ref` components. Minor spec quality issue. |

---

## What the Customer's Ops Team Needs to Configure

| Item | Why It's Needed |
|---|---|
| Postman Enterprise license | Required for Spec Hub, governance, workspace linking |
| Runtime base URLs per environment | For environment configuration |
| Auth credentials per environment | JWT signing config, mTLS client cert + key |
| GitHub repo access with Actions enabled | For workflow execution |
| Org-level secrets | `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GH_FALLBACK_TOKEN` |
| Postman workspace admin user IDs | To grant team access during onboarding |

---

## Repository Structure
```
postman-loan-origination-api/
├── .github/workflows/
│   └── onboard-api.yml          ← Working onboarding workflow
├── .postman/
│   ├── config.json              ← Workspace ID, spec ID, collection IDs
│   └── resources.yaml           ← Resource mapping for safe reruns
├── postman/
│   ├── collections/
│   │   ├── [Baseline] loan-origination-api/
│   │   ├── [Smoke] loan-origination-api/
│   │   └── [Contract] loan-origination-api/
│   ├── environments/
│   │   ├── prod.postman_environment.json
│   │   ├── staging.postman_environment.json
│   │   └── dev.postman_environment.json
│   └── exports/
│       ├── [Baseline] loan-origination-api.postman_collection.json
│       ├── [Smoke] loan-origination-api.postman_collection.json
│       └── [Contract] loan-origination-api.postman_collection.json
├── specs/
│   └── loan-origination-api-openapi.yaml
└── README.md
```

---

## Collection Export Formats

The generated collections exist in two formats:

**YAML format** (`postman/collections/`): Native format produced by the `postman-repo-sync-action` for git sync. Git-diffable, maintained automatically by the workflow on every run.

**JSON format** (`postman/exports/`): Standard Postman Collection v2.1 exports, manually exported from the Postman workspace. Can be imported directly into any Postman client via **Import → Upload Files**.

---

## How I Used AI

AI assistance was used throughout this exercise in line with the stated policy: use it to accelerate, not to replace judgment.

### What AI helped with
- Cross-referencing the three action READMEs to understand how bootstrap and repo-sync chain together
- Drafting the initial workflow YAML structure
- First draft of this README — reviewed and edited by me
- Formatting tables and structuring presentation content

### What I wrote, decided, and validated myself
- Service selection — Loans as Service 2 because it has meaningful differences from Payments (JWT/mTLS auth, multipart file uploads, async underwriting) without changing the CI/CD pattern, making the delta easy to articulate
- All workflow design decisions — environment names pulled directly from the spec's `servers` block, one repo per service
- Every workflow run — triggered, watched, and validated against a real Postman Enterprise workspace. No mocked outputs.
- All consulting content — adaptation plan, working session design, 90-day roadmap, ROI math, handoff plan

### What I changed or overrode from AI suggestions
- Removed `system-env-map-json` and `governance-mapping-json` placeholder values — these require real Postman system environment UUIDs that don't exist in this exercise context. Fake values would cause silent failures.

---

## References

- [Postman API Onboarding Action](https://github.com/postman-cs/postman-api-onboarding-action)
- [Postman Bootstrap Action](https://github.com/postman-cs/postman-bootstrap-action)
- [Postman Repo Sync Action](https://github.com/postman-cs/postman-repo-sync-action)
- [Postman API Authentication](https://learning.postman.com/docs/developer/postman-api/authentication/)
- [Postman Spec Hub](https://learning.postman.com/docs/designing-and-developing-your-api/managing-apis/)
- [Postman CLI](https://learning.postman.com/docs/postman-cli/postman-cli-overview/)
