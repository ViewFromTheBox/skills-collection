---
name: report-bug
description: Runs a thorough interview process to report and plan a bug fix, then creates a detailed plan and accompanying bug issue in the repository. Asks clarifying and probing questions to nail down reproduction steps and root cause, analyzes the codebase to identify affected code paths, produces a fix plan with reproduction steps, root cause analysis, and implementation notes, and files the issue on GitHub or GitLab.
---

# Report Bug

Report a bug through a thorough interview process, then create a fix plan and issue in the repository.

## Step 1: Detect Platform and Codebase

### 1a: Detect the hosting platform

Determine whether this is a GitHub or GitLab repository:

1. Parse `git remote get-url origin`:
   - Contains `github.com` → **GitHub**
   - Contains `gitlab.com` or other GitLab domain → **GitLab**
2. Confirm with directory/CLI presence:
   - GitHub: `.github/` directory exists, `gh` CLI available and authenticated (`gh auth status`)
   - GitLab: `.gitlab/` directory exists, `glab` CLI available and authenticated (`glab auth status`)

If the platform can't be determined, ask the user.

### 1b: Explore the codebase

Silently explore the repository to ground the interview with real context:

1. **Identify the stack**: Check for `package.json`, `composer.json`, `go.mod`, `Cargo.toml`, `Package.swift`, `build.zig`, `pyproject.toml`, etc.
2. **Identify the framework**: Laravel, Next.js, Nuxt, SvelteKit, Tauri, etc.
3. **Scan architecture**: Directory structure, routing patterns, database layer, API patterns
4. **Check for existing conventions**:
   - Issue templates (`.github/ISSUE_TEMPLATE/` or `.gitlab/issue_templates/`)
   - Bug report templates specifically
   - Coding standards files (`.editorconfig`, linter configs, `.claude/rules`)
5. **Note relevant patterns**: Error handling, logging, validation, testing patterns

Use this context to ask sharper questions and trace the likely affected code paths.

## Step 2: Interview

Run a thorough, iterative interview to fully understand the bug. Ask 1–2 questions at a time using `AskUserQuestion`. The goal is to get a clear reproduction, understand the root cause, and plan the fix.

### Question categories

Work through these categories, adapting to the user's answers:

**What happened**:
- What did you expect to happen? What actually happened instead?
- When did this start? Was it working before? What changed?
- How severe is this? (Crash, data loss, wrong behavior, cosmetic, etc.)

**Reproduction**:
- Can you reproduce it reliably? What are the exact steps?
- Does it happen every time or intermittently?
- What environment? (Browser, OS, app version, specific user role, specific data state)
- Are there specific inputs or conditions that trigger it?

**Scope and impact**:
- Does it affect all users or specific ones? (Roles, permissions, account state)
- Are there workarounds?
- Is it blocking anything critical?

**Root cause investigation** (informed by codebase analysis):
- Based on the symptoms, which part of the codebase is likely involved?
- "I see the {component/route/model} handles this flow — does the bug seem related to {specific behavior}?"
- Could this be a data issue, a logic error, a race condition, a missing validation, etc.?
- Are there related recent changes (commits, PRs, deployments) that might have introduced this?

**Edge cases and related bugs**:
- Are there similar features that work correctly? What's different about this one?
- Does the bug cascade? (e.g., wrong calculation → wrong display → wrong export)
- Are there related bugs or symptoms you've noticed?

### Interview behavior

- **Get specific**: "It doesn't work" is not enough. Push for exact error messages, screenshots, URLs, user IDs, timestamps.
- **Reference the codebase**: "I see `OrderController@store` handles order creation and calls `InventoryService@reserve` — does the bug happen during creation or after?"
- **Trace the code path**: As the user describes the bug, mentally (or actually) trace the code path and ask targeted questions about specific decision points.
- **Recognize convergence**: Stop when you have clear repro steps, a strong hypothesis for the root cause, and enough context to plan the fix.
- **Don't over-interview**: Simple, obvious bugs don't need 15 questions. Read the room.

## Step 3: Investigate the Codebase

After the interview, dig into the code to strengthen the root cause analysis:

1. **Trace the affected code path**: Follow the flow from the entry point (route, command, event) through to where the bug manifests
2. **Read the relevant files**: Controllers, services, models, views, tests related to the buggy behavior
3. **Check for existing tests**: Are there tests that should have caught this? Are they missing or wrong?
4. **Check recent changes**: `git log --oneline -20 -- {affected_files}` to see if a recent commit introduced the issue
5. **Look for related code**: Similar patterns elsewhere that might have the same bug

Document findings — these inform the plan and issue.

## Step 4: Ask About Output Format

Once the investigation is complete, ask the user what they want:

```
AskUserQuestion:
  "How should I deliver the bug report and fix plan?"
  Options:
    - "Plan file + issue" — Write a plan document in the repo AND create a bug issue referencing it
    - "Issue only" — Put everything in a well-structured bug issue
    - "Both, let me review the plan first" — Write the plan file, show it for review, then create the issue
```

### Plan file location

If the user wants a plan file:
1. Check if a plans directory already exists (`docs/plans/`, `docs/rfcs/`, `.github/plans/`)
2. If one exists, use it
3. If none exists, create `docs/plans/`
4. Name the file: `fix-{bug-slug}.md` (e.g. `docs/plans/fix-order-total-rounding.md`)

## Step 5: Write the Plan

Create a plan document with the following structure. Adapt the depth of implementation details based on whether the codebase was analyzed.

```markdown
# Bug Fix: {Brief Bug Description}

## Bug Summary

{1–2 sentences describing the bug — what's wrong and what the impact is.}

## Severity

{Critical / High / Medium / Low}
{Brief justification}

## Reproduction Steps

1. {Step 1}
2. {Step 2}
3. {Step 3}
4. ...

**Expected behavior**: {What should happen}
**Actual behavior**: {What actually happens}

**Environment**: {Browser, OS, app version, user role, or other relevant conditions}

## Root Cause Analysis

{Detailed explanation of why the bug occurs. Reference specific files, functions, and line numbers if the codebase was analyzed.}

### Affected Code Path

{Trace the flow from entry point to bug manifestation. Include file paths and function names.}

### Why It Breaks

{The specific logic error, missing validation, race condition, incorrect assumption, etc.}

## Fix Approach

### Recommended fix

{Describe the fix approach — what to change and why.}

### Files to modify

{Include this section ONLY if the codebase was analyzed. Otherwise, omit.}

- `{path/to/file}` — {What changes and why}
- ...

### Database changes

- {Migrations, data fixes — if applicable. Otherwise, omit.}

### Tests to add/update

- {New test cases that should be added to prevent regression}
- {Existing tests that need updating}

## Alternative Approaches

{If there are multiple ways to fix this, list them with pros/cons. Otherwise, omit.}

- **Option A**: {Description} — {Pros/cons}
- **Option B**: {Description} — {Pros/cons}

## Regression Risk

{What could this fix break? What areas need extra testing after the fix?}

## Open Questions

{Any unresolved questions — if none, omit this section.}
```

### Plan review

If the user chose "Both, let me review the plan first":
1. Write the plan file
2. Show it to the user for review
3. Incorporate any feedback
4. Only then proceed to create the issue

## Step 6: Create the Issue

### 6a: Ensure label exists

Auto-create the "bug" label if it doesn't exist:

**GitHub**:
```bash
gh label create "bug" --description "Something isn't working" --color "d73a4a" 2>/dev/null || true
```

**GitLab**:
```bash
glab label create "bug" --description "Something isn't working" --color "#d73a4a" 2>/dev/null || true
```

### 6b: Ensure milestone exists

Check for the "Awaiting Review" milestone and create if missing:

**GitHub**:
```bash
gh api repos/{owner}/{repo}/milestones --jq '.[] | select(.title=="Awaiting Review")' | head -1
# If not found:
gh api repos/{owner}/{repo}/milestones -f title="Awaiting Review" -f state="open" -f description="Issues awaiting review and triage"
```

**GitLab**:
```bash
glab api "projects/:id/milestones?title=Awaiting+Review" | head -1
# If not found:
glab api "projects/:id/milestones" -f title="Awaiting Review" -f description="Issues awaiting review and triage"
```

### 6c: Determine task breakdown

Based on the fix complexity:

**Simple fix** (≤5 implementation tasks): Use a checklist in the issue body.

**Complex fix** (>5 implementation tasks, multiple affected areas): Create a parent issue with an overview, then create child issues for each major task, linked back to the parent.

### 6d: Create the issue

#### Issue body structure (simple fix — checklist)

```markdown
## Bug Description

{From the plan — what's wrong and what the impact is.}

## Severity

{Critical / High / Medium / Low}

## Reproduction Steps

1. {Step 1}
2. {Step 2}
3. ...

**Expected behavior**: {What should happen}
**Actual behavior**: {What actually happens}
**Environment**: {Relevant environment details}

## Root Cause

{Summary of the root cause analysis — reference specific files/functions if known.}

## Fix Approach

{From the plan — recommended fix approach.}

## Acceptance Criteria

- [ ] {Bug no longer reproduces with the original repro steps}
- [ ] {Edge case 1 is handled}
- [ ] {Regression tests added}
- [ ] ...

## Implementation Tasks

- [ ] {Task 1}
- [ ] {Task 2}
- [ ] {Task 3}

## Plan

{Link to plan file if one was created, otherwise omit}
```

#### Issue body structure (complex fix — parent issue)

```markdown
## Bug Description

{From the plan}

## Severity

{Critical / High / Medium / Low}

## Reproduction Steps

1. {Step 1}
2. {Step 2}
3. ...

**Expected behavior**: {What should happen}
**Actual behavior**: {What actually happens}
**Environment**: {Relevant environment details}

## Root Cause

{Summary of the root cause analysis}

## Fix Approach

{Recommended fix approach}

## Acceptance Criteria

- [ ] {Bug no longer reproduces}
- [ ] {Regression tests added}

## Sub-issues

- [ ] #{child_issue_1} — {Brief description}
- [ ] #{child_issue_2} — {Brief description}
- [ ] #{child_issue_3} — {Brief description}

## Plan

{Link to plan file if one was created, otherwise omit}
```

#### Creating the issue

**GitHub**:
```bash
gh issue create \
  --title "Bug: {Brief bug description}" \
  --body "$ISSUE_BODY" \
  --label "bug" \
  --milestone "Awaiting Review"
```

**GitLab**:
```bash
glab issue create \
  --title "Bug: {Brief bug description}" \
  --description "$ISSUE_BODY" \
  --label "bug" \
  --milestone "Awaiting Review"
```

### 6e: Create child issues (complex fixes only)

For each major fix task, create a child issue:

**GitHub**:
```bash
gh issue create \
  --title "{Task title}" \
  --body "Parent: #{parent_issue_number}

## Task

{Detailed description of what this task involves}

## Acceptance Criteria

- [ ] {Task-specific criterion 1}
- [ ] {Task-specific criterion 2}

## Implementation Notes

{File-level details if codebase was analyzed}" \
  --label "bug" \
  --milestone "Awaiting Review"
```

After creating all child issues, **update the parent issue body** to include the actual issue numbers in the sub-issues checklist.

**GitLab**: Same pattern using `glab issue create`. If the GitLab instance supports task/sub-issue relationships, use those. Otherwise, use the checklist-with-links pattern.

## Step 7: Report Results

After creating everything, report:

```
Bug reported: {Brief bug description}
Severity: {Critical / High / Medium / Low}

Plan: {path/to/plan/file.md — if created}
Issue: #{issue_number} — {issue_url}
{Sub-issues: #{child_1}, #{child_2}, ... — if created}

Labels: bug
Milestone: Awaiting Review
```

## Important Notes

- **Reproduction is king**: A bug without clear repro steps is nearly useless. Push hard for specifics during the interview.
- **Reference the codebase**: When the repo is available, trace the actual code path. "The bug is in the order flow" becomes "`OrderController@store` calls `InventoryService@reserve` which doesn't handle the case where quantity is zero."
- **Don't fix the bug**: This skill plans and files issues. It does NOT create branches, write code, or open PRs. That's what `work-issue` is for.
- **Respect existing templates**: If the repo has a bug report issue template, use it as the base structure instead of the default structure above. Merge the plan content into the template format.
- **Severity is important**: Always include severity. It helps with triage and prioritization.
- **Regression tests matter**: Always include "add regression tests" as an acceptance criterion. Bugs that get fixed without tests tend to come back.
- **Plan file is optional**: Not every bug needs a plan document. Simple, well-understood bugs can live entirely in the issue body.
- **Child issues are for complex fixes**: Don't create 2 child issues — that's overhead. Use child issues when there are 6+ distinct tasks that could be worked on independently.
- **\@mentions and assignments**: Don't assign issues or \@mention users unless the user explicitly asks. The "Awaiting Review" milestone signals that triage is needed.
- **Commit the plan file**: If a plan file is created, commit it to the current branch:
  ```bash
  git add docs/plans/fix-{bug-slug}.md
  git commit -m "docs: add fix plan for {bug-description}"
  ```
  Do NOT push unless the user asks.
