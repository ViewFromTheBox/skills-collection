---
name: prune-branches
description: Prunes local git branches that are not in active development and have no corresponding remote tracking branch. Use this skill when the user wants to clean up stale local branches, remove merged branches, or tidy their git workspace.
context: fork
---

# Prune Local Git Branches

Remove local branches that are no longer tracked on the remote and are not in active development.

## Prerequisites

- Must be inside a git repository
- Must have a remote configured (typically `origin`)

## Procedure

### Step 1: Fetch and prune remote tracking references

Run `git fetch --prune` to sync remote tracking branches and remove stale references.

### Step 2: Identify the default branch

Determine the default branch (main, master, develop, etc.) by checking:
```
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```
If that fails, check which of `main` or `master` exists locally. This branch is always protected from deletion.

### Step 3: Identify the current branch

Run `git branch --show-current`. The current branch is always protected from deletion (git won't allow it anyway).

### Step 4: Gather candidate branches

List all local branches and compare against remote tracking branches:

```bash
git branch -vv
```

A branch is a **candidate for deletion** if it meets ALL of these criteria:
1. It is NOT the default branch
2. It is NOT the currently checked-out branch
3. Its remote tracking branch is **gone** (shown as `[origin/...: gone]` in `git branch -vv`) OR it has **no remote tracking branch at all**

### Step 5: Check for unmerged work

For each candidate branch, check if it has been merged into the default branch:
```bash
git branch --merged <default-branch>
```

Categorize candidates into:
- **Safe to delete**: Merged into the default branch AND remote is gone/missing
- **Unmerged but remote gone**: Has commits not in the default branch, but remote tracking branch is gone

### Step 6: Present findings to the user

Display a summary table:

| Branch | Status | Unmerged Commits | Recommendation |
|--------|--------|-----------------|----------------|

- For **safe to delete** branches: recommend deletion
- For **unmerged but remote gone** branches: warn the user and show the number of unmerged commits. Ask before deleting.

### Step 7: Delete approved branches

- Delete safe (merged) branches with: `git branch -d <branch>`
- Delete unmerged branches only with explicit user approval using: `git branch -D <branch>`

### Important Safety Rules

- NEVER delete the default branch or the current branch
- NEVER force-delete unmerged branches without explicit user confirmation
- Always show what will be deleted BEFORE deleting
- If there are no branches to prune, report that the workspace is clean
