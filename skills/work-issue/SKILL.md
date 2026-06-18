---
name: work-issue
description: Takes an issue ID, fetches it from the remote repository, creates a typed branch, fully implements the issue using all available context (.claude/rules, code patterns, issue description), runs tests, commits, pushes, and creates a draft PR/MR with the default template. Works on GitHub and GitLab.
---

# Work Issue

Given an issue ID, fully implement it end-to-end: create a branch, write the code, run tests, pause for manual testing, then clean up, push, and open a draft PR/MR.

## Step 1: Detect Platform

1. Parse the git remote URL (`git remote get-url origin`):
   - Contains `github.com` → **GitHub**
   - Contains `gitlab.com` or a self-hosted GitLab domain → **GitLab**
2. Confirm by checking directory presence:
   - `.github/` directory → GitHub
   - `.gitlab/` directory → GitLab
3. Verify the required CLI tool is available:
   - GitHub → `gh` CLI must be installed and authenticated
   - GitLab → `glab` CLI must be installed and authenticated
4. If the CLI tool is missing, warn the user and stop.

## Step 2: Fetch the Issue

1. **Retrieve the full issue** using the provided issue ID:
   - **GitHub**:
     ```bash
     gh issue view {id} --json number,title,body,labels,comments,assignees,milestone,state
     ```
   - **GitLab**:
     ```bash
     glab issue view {id}
     ```
     Plus fetch comments:
     ```bash
     glab api projects/{project_id}/issues/{id}/notes
     ```

2. **Extract key information**:
   - **Title**: the issue title
   - **Body**: the full issue description (requirements, acceptance criteria, context)
   - **Labels**: all labels on the issue
   - **Comments**: all comments (may contain additional requirements, clarifications, or implementation hints)
   - **Task list**: any `- [ ]` items in the body (these are the deliverables)

3. **Validate**:
   - If the issue is closed, warn the user and stop.
   - If the issue is already assigned to someone else, warn the user but continue if they confirm.

## Step 3: Determine Issue Type

Map issue labels to a branch prefix. Check labels in this priority order:

| Label matches (case-insensitive) | Branch prefix |
|----------------------------------|---------------|
| `bug`, `bugfix`, `fix` | `bugfix/` |
| `feature`, `feature request` | `feature/` |
| `enhancement`, `improvement` | `enhancement/` |
| `chore`, `maintenance`, `task`, `dependencies` | `chore/` |
| `documentation`, `docs` | `docs/` |
| `refactor`, `tech debt` | `refactor/` |
| `performance` | `perf/` |
| `security` | `security/` |
| `accessibility` | `a11y/` |
| `privacy` | `privacy/` |

**Fallback** (no matching labels): analyze the issue title and body for keywords:
- Bug-related words (fix, broken, error, crash, regression) → `bugfix/`
- Feature-related words (add, new, implement, create, support) → `feature/`
- Enhancement-related words (improve, optimize, update, enhance) → `enhancement/`
- Default if nothing matches → `feature/`

## Step 4: Create Branch

1. **Stay on the current branch and ensure it's up to date**:
   ```bash
   git pull
   ```
   Branch from whatever branch you are currently on — do NOT switch to main/master.
   Record the current branch name as `{base_branch}` for use when creating the PR/MR later.

2. **Generate the branch name**:
   - Format: `{type}/{issue#}-{slug}`
   - Slug: derive from issue title, kebab-cased, lowercase, truncated to ~50 chars
   - Example: `feature/123-add-user-search-endpoint`
   - Remove special characters, collapse multiple hyphens

3. **Create and switch to the branch**:
   ```bash
   git checkout -b {branch_name}
   ```

## Step 5: Understand Context

Before writing any code, gather all available context to guide the implementation:

1. **Read project rules**: Scan `.claude/rules/` for all rule files (e.g. `laravel.md`, `tailwind.md`, `react.md`). These contain coding standards, patterns, and conventions established for the project.

2. **Understand project structure**: Examine the directory layout, key files, and architectural patterns:
   - Identify the framework and stack
   - Locate relevant source directories (controllers, models, views, components, etc.)
   - Find existing tests and testing patterns
   - Note any config files, environment setup, or build tooling

3. **Analyze existing code patterns**: Look at similar existing code to understand conventions:
   - How are similar features/fixes structured?
   - What naming conventions are used?
   - What testing patterns are established?
   - What dependencies are available?

4. **Parse the issue thoroughly**:
   - Read the issue body for requirements and acceptance criteria
   - Read all comments for clarifications, design decisions, or implementation hints
   - If there's a task list, use it as the implementation checklist
   - If there are linked issues or references, review them for additional context

## Step 6: Implement the Issue

This is the core of the skill — fully implement whatever the issue requires.

### General approach

1. **Plan the changes**: Based on the issue requirements and codebase context, determine:
   - Which files need to be created or modified
   - The order of implementation (dependencies first)
   - What tests need to be written or updated

2. **Write the code**: Implement all changes needed to resolve the issue:
   - Follow the coding standards from `.claude/rules/`
   - Match existing code patterns and conventions
   - Write clean, well-structured code
   - Include appropriate comments where logic is non-obvious
   - Handle edge cases and error conditions

3. **Write/update tests**: Ensure the implementation is tested:
   - Add new tests for new functionality
   - Update existing tests if behavior changed
   - Follow the project's existing testing patterns and framework

4. **Verify completeness**: Check all requirements from the issue:
   - If there's a task list, ensure every item is addressed
   - If there are acceptance criteria, verify each one
   - If there are specific UI/UX requirements, implement them

### Implementation guidelines

- **Don't over-engineer**: Implement what the issue asks for, nothing more
- **Don't break existing functionality**: Be careful with changes that could have side effects
- **Follow existing patterns**: If the project uses Services, use Services. If it uses Actions, use Actions. Don't introduce new patterns unless the issue requires it.
- **Handle migrations**: If database changes are needed, create proper migration files
- **Handle config**: If new config values are needed, add them with sensible defaults
- **Handle routing**: If new routes are needed, follow the existing routing conventions

## Step 7: Run Tests

Execute the test suite to verify the implementation:

```bash
# Detect and run the appropriate test command
# Laravel apps:
php artisan test

# Laravel packages:
composer test

# Node/React/Vue projects:
npm test

# Go projects:
go test ./...

# Swift projects:
swift test

# Fallback:
./vendor/bin/phpunit  # PHP
./node_modules/.bin/jest  # JS
```

- **If tests PASS**: continue to the next step.
- **If tests FAIL**:
  - Analyze the failures
  - Attempt to fix the issues (up to 3 attempts)
  - If still failing after 3 attempts, **stop and report the failures**. Do NOT push broken code.
  - Include the test failure details in the report so the user can investigate.

## Step 8: Manual Testing Pause

Pause and ask the user to manually test the implementation before proceeding with code cleanup and review.

1. **Notify the user**: Present a summary of what was implemented and where to test it:
   - List the key files created or modified
   - Describe what functionality to test (based on the issue requirements)
   - Include any relevant URLs, commands, or steps needed to exercise the feature or verify the fix

2. **Wait for confirmation**: Ask the user to confirm that the implementation works as expected:
   - If the user confirms it works → proceed to Step 9
   - If the user reports issues → fix the reported problems, re-run tests (Step 7), and return to this step for another round of manual testing
   - Repeat until the user confirms the implementation is working correctly

3. **Do NOT proceed** to the next step until the user explicitly confirms the feature or fix is working.

## Step 9: Local Review Loop

Run the `/review` skill against the uncommitted diff (vs. `{base_branch}`) to get review feedback and iteratively clean up the code before committing.

1. **Invoke the `/review` skill**. The skill expects a PR number, but at this stage no PR exists yet — instead, gather the same inputs locally and have the reviewer analyze them:
   - Capture the diff: `git diff {base_branch}..HEAD` (committed changes) + `git diff` (uncommitted) + `git status --short` (new files).
   - Run the review reasoning yourself using the same rubric the `/review` skill applies: correctness, project conventions, performance, test coverage, security. Focus on the changes against `{base_branch}`.
   - Where `/review` would call `gh pr diff <number>`, use the local diff against `{base_branch}` instead.

2. **Analyze the feedback**: Apply any valid findings — fix correctness issues, naming problems, missing edge cases, missing tests, etc. Skip findings that are wrong, off-topic, or violate `.claude/rules/` — but note why in your turn output.

3. **Re-review after fixes** by re-running the same local-diff analysis.

4. **Repeat up to 3 passes total**. Stop iterating when:
   - The review surfaces no actionable feedback, OR
   - You've completed 3 review passes.

5. **Re-run tests** after applying review suggestions to ensure nothing was broken:
   - If tests fail, fix the issues (following the same retry logic from Step 7).
   - Do NOT proceed if tests are failing.

> **Why not the CodeRabbit CLI here?** The `cr` CLI is slow, flaky, and surfaces low-signal nits at the pre-PR stage when iteration speed matters most. The `/review` skill (or equivalent local analysis) is faster, runs in the same process, and produces higher-signal feedback you can act on immediately. CodeRabbit still runs automatically on the PR/MR after Step 11 (see Step 12), so its feedback is not lost — just deferred to where it's most useful.

## Step 10: Commit and Push

1. **Stage all changes**:
   ```bash
   git add -A
   ```

2. **Commit with a descriptive message**:
   ```bash
   git commit -m "{type}: {concise description}

   {longer description of what was implemented and why}

   Closes #{issue_number}"
   ```
   - The commit type matches the branch type (feat, fix, chore, docs, refactor, perf, etc.)
   - The description should be specific about what was done
   - Include `Closes #{issue_number}` for auto-close on merge

3. **Push to remote**:
   ```bash
   git push -u origin {branch_name}
   ```

## Step 11: Create Draft PR/MR

### Locate the PR/MR template

- **GitHub**: Check for `.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/` directory
- **GitLab**: Check for `.gitlab/merge_request_templates/Default.md` or `.gitlab/merge_request_templates/` directory

### Create the draft PR/MR

- **GitHub**:
  ```bash
  gh pr create \
    --title "{type}: {issue title}" \
    --body "{body}" \
    --draft \
    --base {base_branch}
  ```

- **GitLab**:
  ```bash
  glab mr create \
    --title "Draft: {type}: {issue title}" \
    --description "{body}" \
    --target-branch {base_branch} \
    --draft
  ```

### PR/MR body content

Use the repo's default template as the base structure. Fill in the template sections with:

- **Description/Summary**: What was implemented and why, referencing the issue
- **Changes made**: List of key changes (files created/modified, features added, bugs fixed)
- **Testing**: What tests were added/updated and how to verify manually
- **Closes #{issue_number}**: Auto-close link

If no template exists, use this default structure:

```markdown
## Summary

{Brief description of what this PR implements}

Closes #{issue_number}

## Changes

- {Key change 1}
- {Key change 2}
- {Key change 3}

## Testing

- {Test 1 added/updated}
- {Test 2 added/updated}

All tests passing.
```

## Step 12: PR/MR Review Pass

Run a final review pass against the open PR/MR using the `/review` skill, then proceed to Step 13.

1. **Invoke the `/review` skill with the PR/MR number**:
   - **GitHub**: `/review {pr_number}` — the skill fetches PR metadata and diff via `gh pr view` + `gh pr diff` and produces a structured review.
   - **GitLab**: `/review {mr_number}` — same flow via `glab mr view` + `glab mr diff`.

2. **Address actionable findings**:
   - Apply valid findings — fix correctness issues, missing edge cases, missing tests, etc.
   - Skip findings that are wrong, off-topic, or violate `.claude/rules/` — but note why in your turn output.
   - Re-run tests after edits (Step 7 logic). Do not push if tests fail.
   - Commit with a clear message (e.g. `chore: address review feedback`) and push to the same branch.

3. **Re-run `/review`** after the push. Repeat until the review surfaces no actionable feedback, OR up to 3 passes total. If still receiving actionable feedback after 3 passes, stop and report the open items to the user for manual triage.

> **Why not poll CodeRabbit here?** CodeRabbit runs automatically on every PR/MR creation and push, so its feedback still lands — but polling for it adds 1–3 minutes of dead time per pass and the agent process is brittle (timeouts, transient API failures, low-signal nits). The `/review` skill is synchronous, in-process, and produces higher-signal feedback you can act on immediately. CodeRabbit's automatic comments are still available for human review on the PR; address them in a follow-up commit if any are valid and you didn't already cover them via `/review`.

## Step 13: Report Result

After the draft PR/MR is created, report:

- **Issue**: link to the original issue
- **Branch**: the branch name created
- **PR/MR**: link to the draft PR/MR
- **Changes summary**: brief list of what was implemented
- **Tests**: confirmation that tests pass
- **Files changed**: count of files created/modified

Format as a concise summary:

```
Implemented issue #123: Add user search endpoint

Branch: feature/123-add-user-search-endpoint
PR: https://github.com/user/repo/pull/456 (draft)

Changes:
  - Created SearchController with index action
  - Added User::scopeSearch() query scope
  - Created SearchRequest form request with validation
  - Added GET /api/users/search route
  - Added 3 feature tests for search endpoint

8 files changed | Tests: ✅ all passing
```

## Important Notes

- **Draft PR only**: The PR/MR is always created as a draft. It's not ready for review until the user checks it.
- **Base branch**: Always branch from and target the current branch. Record it as `{base_branch}` in Step 4 and use it for the PR/MR target.
- **Respect .claude/rules**: The implementation MUST follow all rules and conventions defined in the project's `.claude/rules/` directory. These are non-negotiable coding standards.
- **Don't modify unrelated code**: Only change what's necessary to implement the issue. Don't refactor neighboring code, update dependencies, or fix unrelated issues unless the issue specifically asks for it.
- **Commit granularity**: Use a single commit for the implementation. If the issue is very large with clearly distinct phases, multiple commits are acceptable, but keep them logical and atomic.
- **Test failures are blockers**: Never push code that fails tests. If the implementation can't pass tests after 3 fix attempts, stop and report.
- **Platform auto-detect**: Same as all other skills — git remote URL + `.github/`/`.gitlab/` directory + `gh`/`glab` CLI verification.
