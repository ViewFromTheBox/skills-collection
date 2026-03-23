---
name: create-issues
description: Takes a plan, spec, markdown file(s), or TODO list and creates issues in the remote repository. Supports individual .md files per issue, a single file with multiple issues, or structured plans/roadmaps. Auto-detects issue templates, labels, and platform (GitHub/GitLab). Defaults to Awaiting Review milestone with overrides via frontmatter.
---

# Create Issues

Parse markdown input — plans, specs, issue drafts, TODO lists — and file them as issues in the remote repository. No manual copying, no confirmation step. Just parse and create.

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

## Step 2: Discover Issue Templates

1. Scan for available issue templates:
   - **GitHub**: `.github/ISSUE_TEMPLATE/` — collect all YAML and Markdown templates. Parse each for `name`, `labels`, and field structure.
   - **GitLab**: `.gitlab/issue_templates/` — collect all Markdown templates. Note filenames as template names.
2. Build a template map:
   - Identify which templates exist (bug report, feature request, enhancement, task, etc.)
   - Note the required/optional fields in each template
   - Note default labels associated with each template

## Step 3: Parse Input

The skill accepts multiple input formats. Detect which one is provided and parse accordingly.

### 3a: Directory of Individual Markdown Files

**Detection**: User provides a path to a directory containing `.md` files.

**Parsing**:
- Each `.md` file becomes one issue
- Check each file for YAML frontmatter with optional metadata:
  ```yaml
  ---
  title: Issue title here
  template: bug          # Which issue template to use
  labels: [performance, backend]  # Additional labels
  milestone: "v2.0"      # Override default milestone
  assignee: username      # Assign to specific user
  ---
  ```
- If no frontmatter `title`, use the first `#` heading as the title
- The body of the markdown (after frontmatter) becomes the issue body
- If the file has no frontmatter at all, treat the entire content as the issue body and derive the title from the filename (e.g. `fix-login-bug.md` → "Fix login bug")

### 3b: Single Markdown File with Multiple Issues

**Detection**: User provides a path to a single `.md` file. The file contains multiple top-level headings (`#` or `##`).

**Parsing**:
- Split the file at each top-level heading — each section becomes one issue
- The heading text becomes the issue title
- Everything between that heading and the next heading becomes the issue body
- If the file has YAML frontmatter at the top, it applies as defaults for ALL issues (template, labels, milestone). Individual issues can override via inline markers (see below).
- Support inline metadata markers within each section:
  ```markdown
  ## Add user search endpoint
  <!-- template: feature -->
  <!-- labels: backend, api -->
  <!-- milestone: v2.0 -->
  <!-- assignee: jacob -->

  Description of the feature...
  ```

### 3c: Structured Plan / Spec / Roadmap

**Detection**: User provides a markdown document that isn't clearly pre-formatted as issues. It might be a project plan, a spec output from a planning session, a roadmap, or a TODO list.

**Parsing**:
- Analyze the document structure to identify actionable items:
  - Headings with descriptive titles → potential issues
  - Task lists (`- [ ] item`) → each unchecked item becomes an issue
  - Numbered lists of action items → each item becomes an issue
  - Sections with "TODO", "Action Items", "Tasks", "Next Steps" headings → parse children as issues
- For each extracted item:
  - Derive a clear, concise issue title
  - Include relevant context from the surrounding document as the issue body
  - If the item is under a category heading, use that as a label hint
- Be intelligent about granularity:
  - A single-line TODO item becomes a concise issue
  - A multi-paragraph section becomes a detailed issue with full context
  - Don't create issues from informational/background sections — only actionable items

### 3d: Inline / Conversational Input

**Detection**: User describes issues in the chat message itself rather than pointing to a file.

**Parsing**:
- Extract each distinct issue from the user's message
- Create title and body for each based on the description provided

## Step 4: Auto-Detect Issue Template

For each parsed issue, if no template is explicitly specified (via frontmatter or inline marker):

1. **Analyze the content** for keyword signals:
   - Bug indicators: "bug", "fix", "broken", "error", "crash", "regression", "incorrect", "fails", "doesn't work"
   - Feature indicators: "add", "new", "implement", "create", "introduce", "support for"
   - Enhancement indicators: "improve", "optimize", "refactor", "update", "enhance", "better"
   - Task/chore indicators: "update dependency", "migrate", "clean up", "remove", "configure"

2. **Match to available templates**:
   - Map the detected type to the closest available template
   - If no clear match, use the default template (or no template if none exists)

3. **Apply template structure**:
   - If the issue body doesn't already match the template's structure, reformat it to fit the template's sections
   - Preserve all original content while fitting it into the template format
   - Fill in template fields that can be inferred from the content (e.g. "Steps to Reproduce" from a bug description)
   - Leave template fields empty (with a placeholder comment) if the content doesn't provide enough information

## Step 5: Resolve Metadata

For each issue, resolve metadata in this priority order (highest wins):

### Labels
1. **Per-issue override**: frontmatter `labels` or inline `<!-- labels: ... -->` marker
2. **Template default labels**: labels associated with the matched issue template
3. **Auto-detected from content**: infer labels from keywords (e.g. "database" → `database`, "UI" → `frontend`)
4. **Batch default**: frontmatter at the top of a multi-issue file

For any label that doesn't exist in the repo, **auto-create it**:
- GitHub: `gh label create "{name}" --color "{auto-pick}" --description ""`
- GitLab: `glab label create "{name}" --color "#{auto-pick}" --description ""`
- Pick sensible colors based on category (bugs → red, features → blue, etc.)

### Milestone
1. **Per-issue override**: frontmatter `milestone` or inline `<!-- milestone: ... -->` marker
2. **Batch default**: frontmatter at the top of a multi-issue file
3. **Skill default**: "Awaiting Review" milestone
- If the specified milestone doesn't exist, warn and create without milestone

### Assignee
1. **Per-issue override**: frontmatter `assignee` or inline `<!-- assignee: ... -->` marker
2. **Batch default**: frontmatter at the top of a multi-issue file
3. **Skill default**: unassigned

## Step 6: Create Issues

File all issues immediately — no confirmation step.

### GitHub
```bash
gh issue create \
  --title "{title}" \
  --body "{body}" \
  --label "{label1},{label2}" \
  --milestone "Awaiting Review" \
  --assignee "{assignee}"
```

### GitLab
```bash
glab issue create \
  --title "{title}" \
  --description "{body}" \
  --label "{label1},{label2}" \
  --milestone "Awaiting Review" \
  --assignee "{assignee}"
```

For each issue created, capture the issue URL/number from the CLI output.

## Step 7: Report Results

After all issues are created, report:

- **Total issues created**: count with links to each
- **Breakdown by template**: how many bugs, features, enhancements, etc.
- **Breakdown by label**: which labels were applied and how often
- **Labels created**: list any new labels that were auto-created
- **Failures**: any issues that failed to create (with error details)

Format as a concise summary table:

```
Created 12 issues:

| # | Title | Template | Labels | Milestone | Link |
|---|-------|----------|--------|-----------|------|
| 1 | Fix login redirect loop | Bug | bug, auth | Awaiting Review | #142 |
| 2 | Add user search endpoint | Feature | feature, api | v2.0 | #143 |
...

New labels created: api, auth
```

## Input Format Reference

### Frontmatter (per-file or batch defaults)
```yaml
---
title: Issue title             # Optional, falls back to first heading or filename
template: bug                  # Optional, auto-detected if not specified
labels: [bug, backend]         # Optional, merged with auto-detected labels
milestone: "Awaiting Review"   # Optional, defaults to "Awaiting Review"
assignee: username             # Optional, defaults to unassigned
---
```

### Inline Markers (per-issue in multi-issue files)
```markdown
## Issue Title Here
<!-- template: feature -->
<!-- labels: frontend, ux -->
<!-- milestone: v2.0 -->
<!-- assignee: jacob -->

Issue body content...
```

### Plan / TODO Format
```markdown
# Project Plan: User Dashboard

## Phase 1: Core Features
- [ ] Add user profile page with avatar upload
- [ ] Implement activity feed with real-time updates
- [ ] Create notification preferences panel

## Phase 2: Analytics
- [ ] Build usage statistics dashboard
- [ ] Add export to CSV functionality
```

All three formats (and combinations) are valid input.

## Important Notes

- **Idempotency**: The skill does NOT check for duplicate issues. If run twice on the same input, it will create duplicates. This is by design — the user controls when to run it.
- **Template flexibility**: If an issue's content doesn't perfectly fit a template, include all content and leave unmatched template sections with placeholder comments rather than discarding information.
- **Large batches**: For batches of 20+ issues, create them sequentially with a brief pause between to avoid API rate limits.
- **Markdown preservation**: Preserve markdown formatting (code blocks, lists, bold, links) in issue bodies — both GitHub and GitLab render markdown natively.
- **Platform auto-detect**: Same as all other skills — git remote URL + `.github/`/`.gitlab/` directory + `gh`/`glab` CLI verification.
