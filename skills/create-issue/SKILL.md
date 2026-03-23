---
name: create-issue
description: Creates a single issue in the remote repository from a plan, markdown file, or prompt. Auto-detects issue template, labels, and platform (GitHub/GitLab). Defaults to Awaiting Review milestone with overrides via frontmatter or user instruction.
---

# Create Issue

Create a single issue in the remote repository from any input — a markdown file, a plan excerpt, or a conversational prompt. Auto-detects the best issue template, resolves metadata, and files immediately.

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

The skill accepts multiple input formats for a single issue. Detect which one is provided and parse accordingly.

### 3a: Markdown File

**Detection**: User provides a path to a `.md` file.

**Parsing**:
- Check for YAML frontmatter with optional metadata:
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

### 3b: Plan Excerpt

**Detection**: User provides a section from a plan, spec, or roadmap — either as a file reference with a specific section or pasted directly.

**Parsing**:
- Extract the actionable item from the plan context
- Derive a clear, concise issue title from the heading or first line
- Include relevant surrounding context from the plan as the issue body
- If the plan section has sub-items or a checklist, include those as a task list in the issue body

### 3c: Conversational Prompt

**Detection**: User describes the issue in the chat message itself.

**Parsing**:
- Extract the issue details from the user's description
- Formulate a clear, concise title
- Structure the description as a well-formatted issue body
- If the user mentions specific details (steps to reproduce, expected behavior, etc.), organize them into appropriate sections

## Step 4: Auto-Detect Issue Template

If no template is explicitly specified (via frontmatter or user instruction):

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
   - Fill in template fields that can be inferred from the content
   - Leave template fields empty (with a placeholder comment) if the content doesn't provide enough information

## Step 5: Resolve Metadata

Resolve metadata in this priority order (highest wins):

### Labels
1. **Explicit override**: frontmatter `labels` or user instruction
2. **Template default labels**: labels associated with the matched issue template
3. **Auto-detected from content**: infer labels from keywords (e.g. "database" → `database`, "UI" → `frontend`)

For any label that doesn't exist in the repo, **auto-create it**:
- GitHub: `gh label create "{name}" --color "{auto-pick}" --description ""`
- GitLab: `glab label create "{name}" --color "#{auto-pick}" --description ""`
- Pick sensible colors based on category (bugs → red, features → blue, enhancements → green, etc.)

### Milestone
1. **Explicit override**: frontmatter `milestone` or user instruction
2. **Skill default**: "Awaiting Review" milestone
- If the specified milestone doesn't exist, warn and create without milestone

### Assignee
1. **Explicit override**: frontmatter `assignee` or user instruction
2. **Skill default**: unassigned

## Step 6: Create Issue

File the issue immediately — no confirmation step.

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

Capture the issue URL/number from the CLI output.

## Step 7: Report Result

After the issue is created, report:

- **Issue link**: direct URL to the created issue
- **Title**: the issue title
- **Template used**: which template was applied
- **Labels**: which labels were assigned (note any that were auto-created)
- **Milestone**: which milestone was assigned
- **Assignee**: who it was assigned to (or "unassigned")

Format as a concise summary:

```
Created issue #142: Fix login redirect loop
Template: Bug Report | Labels: bug, auth | Milestone: Awaiting Review
→ https://github.com/user/repo/issues/142
```

## Important Notes

- **Template flexibility**: If the issue content doesn't perfectly fit a template, include all content and leave unmatched template sections with placeholder comments rather than discarding information.
- **Markdown preservation**: Preserve markdown formatting (code blocks, lists, bold, links) in the issue body — both GitHub and GitLab render markdown natively.
- **Smart title generation**: When deriving a title from a prompt or plan, keep it concise (under 80 characters), action-oriented, and specific. Avoid generic titles like "Bug fix" or "New feature."
- **Platform auto-detect**: Same as all other skills — git remote URL + `.github/`/`.gitlab/` directory + `gh`/`glab` CLI verification.
