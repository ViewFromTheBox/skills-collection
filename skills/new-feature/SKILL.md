---
name: new-feature
description: Runs a thorough interview process to plan a new feature, then creates a detailed plan and accompanying issue in the repository. Asks clarifying and challenging questions to flesh out the idea, analyzes the codebase for context, produces a plan with user stories, acceptance criteria, and implementation steps, and files the issue on GitHub or GitLab with task breakdowns.
---

# New Feature

Plan a new feature through a thorough interview process, then create a plan and issue in the repository.

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
3. **Scan architecture**: Look at directory structure, routing patterns, database layer, API patterns
4. **Check for existing conventions**:
   - Issue templates (`.github/ISSUE_TEMPLATE/` or `.gitlab/issue_templates/`)
   - Existing plan documents (`docs/plans/`, `docs/rfcs/`, `.github/plans/`)
   - Coding standards files (`.editorconfig`, linter configs, `.claude/rules`)
5. **Note relevant patterns**: Authentication, authorization, state management, testing patterns, etc.

Use this context to ask sharper questions during the interview. Reference actual files and patterns when relevant.

## Step 2: Interview

Run a thorough, iterative interview to understand the feature. Ask 1–2 questions at a time using `AskUserQuestion`. The goal is to fully flesh out the idea before planning.

### Question categories

Work through these categories, adapting to the user's answers:

**Core understanding**:
- What is this feature? What problem does it solve?
- Who uses it? What triggers them to use it?
- Walk through a real use case end to end.

**Scope and boundaries**:
- What's the smallest useful version of this?
- What's explicitly out of scope for the first version?
- Are there related features this depends on or conflicts with?

**Behavior and edge cases**:
- What happens when {input is invalid / user cancels / data is missing}?
- Are there permission or access control concerns?
- Does this need to work offline / on mobile / across platforms?

**Technical grounding** (informed by codebase analysis):
- Does this touch existing {models / routes / components / APIs} in the codebase?
- Are there patterns in the codebase this should follow?
- Does this require new database tables, migrations, or schema changes?
- Are there third-party integrations or external APIs involved?

**Acceptance criteria**:
- How do we know this feature works correctly?
- What are the key scenarios to test?
- Are there performance or accessibility requirements?

### Interview behavior

- **Challenge weak assumptions**: If something sounds vague, push back. "What do you mean by 'users can manage their settings'? Which settings specifically?"
- **Reference the codebase**: "I see you have a `Policy` pattern for authorization — should this feature use the same pattern?"
- **Recognize convergence**: Stop when answers are consistent and confident, the scope is clear, and you could predict their answers to follow-up questions.
- **Don't over-interview**: If the feature is straightforward, 3–5 questions may be enough. If it's complex, go deeper. Read the room.

## Step 3: Ask About Output Format

Once the interview is complete, ask the user what they want:

```
AskUserQuestion:
  "How should I deliver the plan?"
  Options:
    - "Plan file + issue" — Write a plan document in the repo AND create an issue referencing it
    - "Issue only" — Put everything in a well-structured issue body
    - "Both, let me review the plan first" — Write the plan file, show it to me for review, then create the issue
```

### Plan file location

If the user wants a plan file:
1. Check if a plans directory already exists (`docs/plans/`, `docs/rfcs/`, `.github/plans/`)
2. If one exists, use it
3. If none exists, create `docs/plans/`
4. Name the file: `{feature-slug}.md` (e.g. `docs/plans/user-notification-preferences.md`)

## Step 4: Write the Plan

Create a plan document with the following structure. Adapt the depth of implementation details based on whether a codebase is present and analyzable.

```markdown
# Feature: {Feature Name}

## Problem Statement

{1–2 sentences describing the problem this feature solves.}

## Target User

{Who this is for and what triggers them to use it.}

## User Stories

- As a {user type}, I want to {action} so that {benefit}.
- As a {user type}, I want to {action} so that {benefit}.
- ...

## Scope

### In scope (v1)

- {Feature aspect 1}
- {Feature aspect 2}
- ...

### Out of scope

- {Explicitly excluded thing 1}
- {Explicitly excluded thing 2}
- ...

## Behavior

### Happy path

{Describe the main flow end to end.}

### Edge cases

- **{Edge case 1}**: {How it's handled}
- **{Edge case 2}**: {How it's handled}
- ...

## Acceptance Criteria

- [ ] {Criterion 1}
- [ ] {Criterion 2}
- ...

## Implementation Notes

{Include this section ONLY if a codebase is present and was analyzed. Otherwise, omit it entirely.}

### Files to create/modify

- `{path/to/file}` — {What changes and why}
- ...

### Database changes

- {New tables, columns, migrations — if applicable}

### API changes

- {New endpoints, modified responses — if applicable}

### Dependencies

- {New packages, services, or integrations — if applicable}

## Open Questions

- {Any unresolved questions from the interview — if none, omit this section}
```

### Plan review

If the user chose "Both, let me review the plan first":
1. Write the plan file
2. Show it to the user for review
3. Incorporate any feedback
4. Only then proceed to create the issue

## Step 5: Create the Issue

### 5a: Ensure label exists

Auto-create the "enhancement" label if it doesn't exist:

**GitHub**:
```bash
gh label create "enhancement" --description "New feature or request" --color "a2eeef" 2>/dev/null || true
```

**GitLab**:
```bash
glab label create "enhancement" --description "New feature or request" --color "#a2eeef" 2>/dev/null || true
```

### 5b: Ensure milestone exists

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

### 5c: Determine task breakdown

Based on the feature complexity:

**Small feature** (≤5 implementation tasks): Use a checklist in the issue body.

**Large feature** (>5 implementation tasks): Create a parent issue with an overview, then create child issues for each major task, linked back to the parent.

### 5d: Create the issue

#### Issue body structure (small feature — checklist)

```markdown
## Problem Statement

{From the plan}

## User Stories

{From the plan}

## Scope

### In scope (v1)
{From the plan}

### Out of scope
{From the plan}

## Acceptance Criteria

- [ ] {Criterion 1}
- [ ] {Criterion 2}

## Implementation Tasks

- [ ] {Task 1}
- [ ] {Task 2}
- [ ] {Task 3}

## Plan

{Link to plan file if one was created, otherwise omit}
```

#### Issue body structure (large feature — parent issue)

```markdown
## Problem Statement

{From the plan}

## User Stories

{From the plan}

## Scope

### In scope (v1)
{From the plan}

### Out of scope
{From the plan}

## Acceptance Criteria

- [ ] {Criterion 1}
- [ ] {Criterion 2}

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
  --title "Feature: {Feature Name}" \
  --body "$ISSUE_BODY" \
  --label "enhancement" \
  --milestone "Awaiting Review"
```

**GitLab**:
```bash
glab issue create \
  --title "Feature: {Feature Name}" \
  --description "$ISSUE_BODY" \
  --label "enhancement" \
  --milestone "Awaiting Review"
```

### 5e: Create child issues (large features only)

For each major implementation task, create a child issue:

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
  --label "enhancement" \
  --milestone "Awaiting Review"
```

After creating all child issues, **update the parent issue body** to include the actual issue numbers in the sub-issues checklist.

**GitLab**: Same pattern using `glab issue create`. If the GitLab instance supports task/sub-issue relationships, use those. Otherwise, use the checklist-with-links pattern.

## Step 6: Report Results

After creating everything, report:

```
Feature planned: {Feature Name}

Plan: {path/to/plan/file.md — if created}
Issue: #{issue_number} — {issue_url}
{Sub-issues: #{child_1}, #{child_2}, ... — if created}

Labels: enhancement
Milestone: Awaiting Review
```

## Important Notes

- **Interview quality matters**: The whole point of this skill is to make the feature as well-planned as possible before implementation starts. Don't rush the interview.
- **Reference the codebase**: When the repo is available, ground questions and implementation notes in actual code. "I see you use Livewire for interactivity — should this feature follow the same pattern?"
- **Don't implement**: This skill plans and files issues. It does NOT create branches, write code, or open PRs. That's what `work-issue` is for.
- **Respect existing templates**: If the repo has issue templates, use them as the base structure instead of the default structure above. Merge the plan content into the template format.
- **Plan file is optional**: Not every feature needs a plan document. Small, well-defined features can live entirely in the issue body.
- **Child issues are for large features**: Don't create 2 child issues — that's overhead. Use child issues when there are 6+ distinct implementation tasks that could be worked on independently.
- **\@mentions and assignments**: Don't assign issues or \@mention users unless the user explicitly asks. The "Awaiting Review" milestone signals that triage is needed.
- **Commit the plan file**: If a plan file is created, commit it to the current branch:
  ```bash
  git add docs/plans/{feature-slug}.md
  git commit -m "docs: add feature plan for {feature-name}"
  ```
  Do NOT push unless the user asks.
