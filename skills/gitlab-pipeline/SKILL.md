---
name: gitlab-pipeline
description: Diagnoses and fixes failed GitLab CI/CD pipelines. Auto-detects the latest failed pipeline on the current branch, analyzes job logs to identify root causes (CI config errors, test failures, lint issues, build failures, Docker problems, deployment errors), fixes the code or CI configuration, verifies locally, then pushes to re-trigger the pipeline. Up to 5 fix attempts before stopping and reporting.
---

# GitLab Pipeline

Diagnose and fix failed GitLab CI/CD pipelines: find the failed pipeline, analyze job logs, identify root causes, fix the issue, verify locally, and push to re-trigger.

## Step 1: Validate Environment

1. **Verify this is a GitLab repository**:
   - Parse `git remote get-url origin` — must contain `gitlab.com` or a self-hosted GitLab domain
   - Confirm `.gitlab/` directory or `.gitlab-ci.yml` exists

2. **Verify `glab` CLI is installed and authenticated**:
   ```bash
   glab auth status
   ```
   If `glab` is missing or not authenticated, warn the user and stop.

3. **Get the current branch**:
   ```bash
   git branch --show-current
   ```

4. **Check for uncommitted changes**:
   ```bash
   git status
   ```
   If there are uncommitted changes, warn the user — these should be committed or stashed before fixing pipeline issues.

## Step 2: Find the Failed Pipeline

### 2a: Auto-detect from current branch

```bash
glab ci list --status failed -P $(glab repo view --json fullPath -q .fullPath) 2>/dev/null | head -20
```

Or use the API directly:

```bash
glab api "projects/:id/pipelines?ref=$(git branch --show-current)&status=failed&per_page=5"
```

Look for the most recent failed pipeline on the current branch.

### 2b: Fallback

If no failed pipeline is found on the current branch:
- Check if there's a failed pipeline on the default branch that might be relevant
- If still nothing, ask the user for a pipeline ID or URL

### 2c: Get pipeline details

Once the pipeline is identified, retrieve:
- Pipeline ID, status, and ref (branch)
- Created/updated timestamps
- The commit SHA that triggered it

```bash
glab api "projects/:id/pipelines/{pipeline_id}"
```

## Step 3: Analyze Pipeline Jobs

### 3a: Fetch all jobs

```bash
glab api "projects/:id/pipelines/{pipeline_id}/jobs?per_page=100"
```

### 3b: Map job relationships

Build a picture of the full pipeline:
1. **Group jobs by stage** (e.g. build → test → lint → deploy)
2. **Identify failed jobs** — these are the primary targets
3. **Check upstream jobs** — a passing job may have produced warnings or artifacts that caused downstream failure
4. **Check downstream jobs** — jobs that were skipped or cancelled due to the failure

### 3c: Prioritize failed jobs

Order failed jobs for analysis:
1. **Earliest stage first** — a build failure is the root cause, not the downstream test failure
2. **Within the same stage** — alphabetical or by job ID

## Step 4: Diagnose Each Failed Job

For each failed job (starting with the earliest stage):

### 4a: Download the job log

```bash
glab api "projects/:id/jobs/{job_id}/trace" > /tmp/job_{job_id}.log
```

### 4b: Categorize the failure

Read the job log and classify the failure type:

| Category | Indicators |
|----------|------------|
| **CI config error** | YAML syntax errors, unknown keywords, invalid `rules:`, missing `image:`, variable expansion failures |
| **Test failure** | Test framework output (PHPUnit, Jest, pytest, go test, cargo test, swift test), assertion errors, test counts with failures |
| **Lint failure** | Linter output (ESLint, Pint, PHP CS Fixer, gofmt, rustfmt, SwiftLint), style violations, formatting errors |
| **Build/compile failure** | Compiler errors, missing imports, type errors, undefined references, `npm run build` failures |
| **Docker failure** | Dockerfile errors, image pull failures, registry auth issues, layer build failures, `docker compose` errors |
| **Deployment failure** | SSH errors, permission denied, service restart failures, health check failures, rollback triggers |

### 4c: Extract the root cause

From the job log, identify:
- **The exact error message(s)**
- **The file(s) and line number(s)** involved
- **The command that failed** and its exit code
- **Any relevant context** (environment variables, runner tags, Docker image version)

### 4d: Check for environment vs code issues

Distinguish between:
- **Code issues** (fixable): syntax errors, failing tests, lint violations, type errors
- **Environment issues** (may not be fixable from code): runner unavailable, Docker registry down, expired credentials, network timeouts
- **CI config issues** (fixable): `.gitlab-ci.yml` syntax, wrong image tags, incorrect variable references

If the issue is purely environmental (e.g. runner down, registry timeout), report it and stop — these can't be fixed from code.

## Step 5: Fix the Issue

### 5a: CI config errors

Read `.gitlab-ci.yml` (and any included files referenced by `include:`) and fix:
- YAML syntax errors
- Invalid job definitions (missing `script:`, invalid `rules:` syntax, etc.)
- Incorrect image references
- Wrong variable names or missing variables
- Broken `include:` references
- Invalid `needs:` or `dependencies:` references

### 5b: Test failures

1. Read the failing test file(s) and the source code they test
2. Determine if the test is wrong (outdated assertion) or the code is wrong (regression)
3. Fix the code or the test as appropriate
4. **Do NOT delete or skip failing tests** — fix the root cause

### 5c: Lint failures

1. Identify the linter and the violated rules
2. Run the linter locally with auto-fix:
   - PHP (Pint): `./vendor/bin/pint`
   - PHP (CS Fixer): `./vendor/bin/php-cs-fixer fix`
   - JavaScript (ESLint): `npx eslint --fix .`
   - JavaScript (Prettier): `npx prettier --write .`
   - Go: `gofmt -w .`
   - Rust: `cargo fmt`
   - Swift: `swiftformat .` / `swiftlint --fix`
3. For issues that can't be auto-fixed, manually fix the code

### 5d: Build/compile failures

1. Read the error output to identify the failing file(s) and error(s)
2. Fix missing imports, type mismatches, undefined references, etc.
3. Run the build command locally to verify

### 5e: Docker failures

1. Read the Dockerfile and/or `docker-compose.yml`
2. Fix common issues:
   - Base image tag doesn't exist → update to valid tag
   - `COPY` references missing files → fix paths
   - Build arg not passed → add to CI config
   - Multi-stage build reference errors → fix stage names
3. If the failure is a registry/auth issue, report it as environmental

### 5f: Deployment failures

1. Read deployment scripts and CI job configuration
2. Fix common issues:
   - Wrong environment variables or secrets references
   - Incorrect deployment paths or targets
   - Health check URL/command errors
   - Missing deployment dependencies
3. If the failure is infrastructure-related (SSH key expired, server down), report it as environmental

## Step 6: Verify Locally

Before pushing, verify the fix works locally. Match the verification to the failure type:

| Failure Type | Local Verification |
|-------------|-------------------|
| CI config error | `python3 -c "import yaml; yaml.safe_load(open('.gitlab-ci.yml'))"` for syntax; visual inspection for semantic issues |
| Test failure | Run the test suite (e.g. `php artisan test`, `npm test`, `go test ./...`, `cargo test`, `swift test`) |
| Lint failure | Run the linter (e.g. `./vendor/bin/pint --test`, `npx eslint .`, `gofmt -l .`) |
| Build failure | Run the build command (e.g. `npm run build`, `go build ./...`, `cargo build`, `swift build`) |
| Docker failure | `docker build .` if Docker is available locally; otherwise skip local verify and note it |
| Deployment failure | Verify config/script syntax; full deploy verification isn't possible locally |

- **If local verification passes**: proceed to commit and push.
- **If local verification fails**: go back to Step 5 and try again (up to 5 total attempts).

## Step 7: Commit and Push

1. **Stage the changes**:
   ```bash
   git add -A
   ```

2. **Check if there are changes to commit**:
   ```bash
   git diff --staged --quiet
   ```
   If no changes, the issue might be environmental — report and stop.

3. **Commit with a descriptive message**:
   ```bash
   git commit -m "fix: resolve pipeline failure in {stage}/{job_name}

   - {Brief description of what was wrong}
   - {Brief description of what was fixed}
   - Pipeline: #{pipeline_id}"
   ```

4. **Push to trigger a new pipeline**:
   ```bash
   git push
   ```

## Step 8: Monitor Re-triggered Pipeline

After pushing:

1. **Get the new pipeline ID**:
   ```bash
   glab api "projects/:id/pipelines?ref=$(git branch --show-current)&per_page=1&order_by=id&sort=desc"
   ```

2. **Report the new pipeline** — provide the link so the user can monitor it.

3. **Do NOT wait for the pipeline to complete** — pipelines can take a long time. Report the fix and the new pipeline link.

## Step 9: Handle Multiple Failures

If the pipeline has failures in multiple jobs across different stages:

1. **Fix the earliest-stage failure first** — later failures may be caused by the earlier one
2. **Push and check if later failures are resolved** by the same fix
3. **If multiple independent failures exist**, fix them all in one commit if possible
4. **Track each fix attempt** — up to 5 total attempts across all failures

## Step 10: Report Results

After fixing (or after exhausting 5 attempts), report:

```
Pipeline fix: #{pipeline_id} on {branch}
→ {gitlab_pipeline_url}

Diagnosis:
  Failed jobs: {job1} ({stage}), {job2} ({stage})
  Root cause: {description of what was wrong}
  Category: {CI config | test | lint | build | Docker | deployment}

Fix applied:
  Files changed: {list of files modified}
  Commit: {commit_hash} — {commit message}
  New pipeline: #{new_pipeline_id}
  → {new_pipeline_url}

Local verification: ✅ {command} passed

⚠️ Notes:
  - {Any environmental issues that couldn't be fixed}
  - {Any warnings from the fix}
```

If the fix failed after 5 attempts:

```
Pipeline fix FAILED: #{pipeline_id} on {branch}
→ {gitlab_pipeline_url}

Diagnosis:
  Failed jobs: {job1} ({stage}), {job2} ({stage})
  Root cause: {description of what was wrong}
  Category: {CI config | test | lint | build | Docker | deployment}

Attempted fixes (5 attempts):
  1. {What was tried and why it didn't work}
  2. {What was tried and why it didn't work}
  ...

Remaining errors:
  {Error details for manual investigation}

Suggested next steps:
  - {Specific suggestions for manual resolution}
```

## Important Notes

- **GitLab only**: This skill is for GitLab CI/CD pipelines. For GitHub Actions, use a separate skill.
- **5 attempts max**: Try up to 5 times to fix the pipeline. After that, stop and report — infinite loops help nobody.
- **Don't skip tests**: If tests fail, fix the tests or the code. Never skip, ignore, or mark tests as expected-to-fail as a "fix".
- **Environmental issues**: If the failure is due to runner availability, registry outages, expired secrets, or network issues, report it clearly and stop. These can't be fixed from code.
- **Don't modify unrelated code**: Only change what's necessary to fix the pipeline failure. No refactoring, no "while I'm here" changes.
- **CI config changes are sensitive**: Changes to `.gitlab-ci.yml` affect the entire pipeline. Be conservative — fix the specific issue without restructuring jobs or stages.
- **Included CI files**: GitLab supports `include:` directives. If the CI config includes external files (local or remote), check those too when diagnosing CI config errors.
- **Job artifacts**: Some failures leave artifacts (test reports, coverage data, build outputs). Check job artifacts via the API if the log alone isn't sufficient.
- **Protected branches**: If the current branch is protected, `git push` may fail. Report this and let the user handle it.
- **Merge train pipelines**: If the failed pipeline is a merge train pipeline, note this — the fix should go to the source branch, not the merge ref.
