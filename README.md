# postman-loan-origination-api
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
- All workflow design decisions — environment names pulled directly from the spec's servers block, one repo per service
- Every workflow run — triggered, watched, and validated against a real Postman Enterprise workspace. No mocked outputs.
- All consulting content — adaptation plan, working session design, 90-day roadmap, ROI math, handoff plan

### What I changed or overrode from AI suggestions
- Removed system-env-map-json and governance-mapping-json placeholder values — these require real Postman system environment UUIDs that don't exist in this exercise context. Fake values would cause silent failures.

---

## Collection Export Formats

The generated collections exist in two formats in this repo:

**YAML format** (`postman/collections/`): Native format produced by the `postman-repo-sync-action` for git sync. Each collection is broken into individual request files — readable diffs in version control.

**JSON format** (`postman/exports/`): Standard Postman Collection v2.1 exports, manually exported from the Postman workspace. Can be imported directly into any Postman client via **Import → Upload Files**.

Both formats represent the same collections. YAML is maintained automatically by the workflow on every run. JSON files are provided for direct import convenience.
