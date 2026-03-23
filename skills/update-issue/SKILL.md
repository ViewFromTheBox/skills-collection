---
name: update-issue
description: Updates a linked issue with a progress comment summarizing commits since the last comment, updates labels to reflect current status, and checks off completed task list items in the issue body. Parses the issue number from the branch name or asks the user. Works on GitHub and GitLab.
---

# Update Issue

Post a progress comment on the linked issue summarizing work done since the last comment. Also update labels to reflect current status and check off completed task list items in the issue body.

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

## Step 2: Identify the Issue

1. **Parse from branch name**: Get the current branch name via `git branch --show-current`. Extract the issue number using common patterns:
   - `feature/123-description` → #123
   - `fix/123-description` → #123
   - `issue-123-description` → #123
   - `123-description` → #123
   - `bugfix/123` → #123
   - `chore/123-description` → #123
   - Any branch containing a numeric segment that could be an issue number

2. **Validate the issue exists**:
   - GitHub: `gh issue view {number} --json number,title,body,state,labels`
   - GitLab: `glab issue view {number}`
   - If the issue doesn't exist or is closed, warn the user.

3. **Fallback**: If no issue number can be parsed from the branch name, ask the user to provide the issue number or URL.

## Step 3: Determine the Time Window

1. **Fetch existing comments** on the issue:
   - GitHub: `gh issue view {number} --json comments --jq '.comments | sort_by(.createdAt) | last | .createdAt'`
   - GitLab: `glab api projects/{id}/issues/{number}/notes --jq 'sort_by(.created_at) | last | .created_at'`

2. **Set the start boundary**:
   - **If comments exist**: Use the timestamp of the most recent comment. All commits after this timestamp are included.
   - **If no comments exist**: Determine where the branch diverged from main/master:
     ```bash
     git merge-base main HEAD
     ```
     All commits from the merge-base to HEAD are included.

3. **Set the end boundary**: Current HEAD (now).

## Step 4: Gather Commits

1. **Get the commit list** within the time window:
   - If using a timestamp boundary (last comment):
     ```bash
     git log --after="{timestamp}" --format="%H %s" HEAD
     ```
   - If using merge-base boundary (no comments):
     ```bash
     git log {merge-base}..HEAD --format="%H %s"
     ```

2. **For each commit**, collect:
   - Short hash
   - Commit message (subject line)
   - Author
   - Timestamp

3. **Generate a narrative summary**:
   - Analyze the commit messages as a group
   - Write a concise 2-4 sentence summary of what was accomplished
   - Focus on the "what" and "why", not individual commits
   - Group related commits into logical chunks if there are many (e.g. "Added user search endpoint with filtering and pagination" rather than listing 5 individual commits)

## Step 5: Analyze Task List Completion

1. **Fetch the issue body**:
   - GitHub: `gh issue view {number} --json body --jq '.body'`
   - GitLab: `glab api projects/{id}/issues/{number} --jq '.description'`

2. **Extract task list items**: Find all `- [ ]` (unchecked) items in the issue body.

3. **Match commits to tasks**: For each unchecked task, analyze whether the commits in the time window address it:
   - Compare task descriptions against commit messages
   - Look for keyword overlap, related file changes, or semantic matches
   - Be conservative — only mark a task as completed if there's clear evidence in the commits

4. **Build the updated issue body**: For each task determined to be completed, change `- [ ]` to `- [x]`.

## Step 6: Update Labels

1. **Fetch current labels** on the issue:
   - GitHub: `gh issue view {number} --json labels --jq '.labels[].name'`
   - GitLab: `glab api projects/{id}/issues/{number} --jq '.labels[]'`

2. **Determine appropriate label updates**:
   - If the issue has no "in progress" label (or equivalent like "wip", "doing", "in-progress"), add one:
     - First check if any such label already exists in the repo
     - If not, auto-create it:
       - GitHub: `gh label create "in progress" --color "FBCA04" --description "Work is actively in progress"`
       - GitLab: `glab label create "in progress" --color "#FBCA04" --description "Work is actively in progress"`
   - If ALL task list items are now checked off, consider the issue near completion:
     - Add a "ready for review" label (auto-create if needed)
     - Remove the "in progress" label

3. **Apply label changes**:
   - GitHub: `gh issue edit {number} --add-label "in progress"`
   - GitLab: `glab issue edit {number} --label "in progress"`

## Step 7: Post Comment

Compose and post the progress comment.

### Comment Format

```markdown
## Progress Update

{narrative summary of work done}

### Commits since last update

| Hash | Message | Author | Date |
|------|---------|--------|------|
| `{short_hash}` | {message} | {author} | {date} |
| `{short_hash}` | {message} | {author} | {date} |

### Task Progress

- [x] {completed task 1} ✅ *completed in this update*
- [x] {completed task 2} ✅ *completed in this update*
- [ ] {remaining task 1}
- [ ] {remaining task 2}

**{n}/{total} tasks completed**
```

If there are no task list items in the issue body, omit the "Task Progress" section.

### Post the comment

- GitHub:
  ```bash
  gh issue comment {number} --body "{comment}"
  ```
- GitLab:
  ```bash
  glab issue note {number} --message "{comment}"
  ```

## Step 8: Update Issue Body

If any task list items were checked off in Step 5, update the issue body:

- GitHub:
  ```bash
  gh issue edit {number} --body "{updated_body}"
  ```
- GitLab:
  ```bash
  glab api -X PUT projects/{id}/issues/{number} -f description="{updated_body}"
  ```

## Step 9: Report Result

After the update is complete, report:

- **Issue**: link to the issue
- **Commits summarized**: count of commits in this update
- **Time window**: from (last comment date or branch start) to now
- **Tasks completed**: how many tasks were checked off (if applicable)
- **Labels updated**: what labels were added/removed
- **Comment link**: direct link to the new comment

Format as a concise summary:

```
Updated issue #123: Add user search endpoint
  12 commits summarized (Mar 1 – Mar 5)
  2/5 tasks completed in this update (4/5 total)
  Label added: in progress
  → https://github.com/user/repo/issues/123#issuecomment-456
```

## Important Notes

- **Branch detection**: The skill must be run from within the branch that's linked to the issue. If on `main` or `master`, warn the user.
- **Base branch**: When determining the merge-base, try `main` first, then fall back to `master` if `main` doesn't exist.
- **Conservative task matching**: Only check off tasks when commits clearly address them. A vague match is not enough — leave the task unchecked and let the user handle it.
- **No duplicate updates**: If there are no new commits since the last comment, report that and skip posting a comment.
- **Comment authorship**: The comment is posted as whoever is authenticated with `gh`/`glab`. This is expected — the user is running the skill themselves.
- **Large commit sets**: If there are 20+ commits, group them by theme in the narrative summary rather than listing each one individually. The commit table can be collapsed in a `<details>` block.
- **Platform auto-detect**: Same as all other skills — git remote URL + `.github/`/`.gitlab/` directory + `gh`/`glab` CLI verification.
