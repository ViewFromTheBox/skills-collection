---
name: prepare-pull-request
description: Cleans up a branch before opening a GitHub pull request. Auto-detects and runs linters with auto-fix, runs the full test suite, checks and fixes inline documentation and docs/ sync, commits any cleanup changes, then creates a draft pull request using the default template. GitHub-specific — uses the gh CLI.
---

# Prepare Pull Request

Clean up the current branch for review: lint, test, verify documentation, commit any fixes, and create a draft pull request on GitHub. This is the final step before code review.

## Step 1: Validate Branch State

1. **Verify not on main/master**:
   ```bash
   git branch --show-current
   ```
   If on `main` or `master`, warn the user and stop — there's nothing to prepare.

2. **Check for uncommitted changes**:
   ```bash
   git status
   ```
   If there are uncommitted changes, warn the user. These will be included in the cleanup commit if the user confirms, or they should be committed/stashed first.

3. **Determine the base branch**: Try `main`, fall back to `master`.

4. **Get the branch diff scope**: Identify all files changed in this branch vs the base:
   ```bash
   git diff --name-only main...HEAD
   ```
   This is the set of files that need linting, testing, and documentation checks.

## Step 2: Verify GitHub Environment

1. Confirm this is a GitHub repository:
   - Parse `git remote get-url origin` — must contain `github.com`
   - Confirm `.github/` directory exists
2. Verify `gh` CLI is installed and authenticated:
   ```bash
   gh auth status
   ```
3. If `gh` is missing or not authenticated, warn the user and stop.

## Step 3: Lint with Auto-Fix

Auto-detect the project's linter(s) and run them with auto-fix mode.

### Linter Detection

| Indicator | Linter | Auto-fix Command |
|-----------|--------|------------------|
| `pint.json` or `composer.json` has `laravel/pint` | Laravel Pint | `./vendor/bin/pint` |
| `.php-cs-fixer.php` or `.php-cs-fixer.dist.php` | PHP CS Fixer | `./vendor/bin/php-cs-fixer fix` |
| `phpstan.neon` or `phpstan.neon.dist` | PHPStan | `./vendor/bin/phpstan analyse` (no auto-fix, report only) |
| `.eslintrc.*` or `eslint.config.*` or `package.json` has `eslint` | ESLint | `npx eslint --fix .` |
| `prettier` in `package.json` or `.prettierrc.*` | Prettier | `npx prettier --write .` |
| `biome.json` | Biome | `npx biome check --apply .` |
| `go.mod` | Go | `gofmt -w .` and `go vet ./...` |
| `.swiftformat` or `Package.swift` | SwiftFormat | `swiftformat .` |
| `.swiftlint.yml` | SwiftLint | `swiftlint --fix` |
| `rustfmt.toml` or `Cargo.toml` | Rustfmt | `cargo fmt` |
| `clippy` (Rust) | Clippy | `cargo clippy --fix --allow-dirty` (report warnings) |

### Process

1. **Run all detected linters** in auto-fix mode
2. **Track what changed**: After each linter run, check `git diff` to see what was modified
3. **Report unfixable issues**: If the linter reports issues it can't auto-fix, collect them for the final report
4. **Stage lint fixes**: Don't commit yet — accumulate all fixes for a single cleanup commit at the end

## Step 4: Run Tests

Execute the test suite to verify everything works after lint fixes:

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

# Rust projects:
cargo test

# Fallback:
./vendor/bin/phpunit  # PHP
./node_modules/.bin/jest  # JS
```

- **If tests PASS**: continue to the next step.
- **If tests FAIL**:
  - Analyze the failures
  - Attempt to fix the issues (up to 3 attempts)
  - If still failing after 3 attempts, **stop and report the failures**. Do NOT proceed with a pull request that has failing tests.
  - Include the test failure details in the report so the user can investigate.

## Step 5: Check and Fix Inline Documentation

Review all files changed in this branch for inline documentation quality. Follow the standards from the inline-documentation skill:

### 5a: PHP and JavaScript (WordPress Standards)

For all changed `.php` and `.js` files:

- **Missing doc blocks**: Add documentation for any undocumented functions, methods, classes, properties, constants, hooks
- **WordPress PHPDoc format**: Third-person singular, complete sentences, `\@since` with three-digit version, `\@param`/`\@return` for all parameters and return values, hook documentation for actions/filters
- **WordPress JSDoc format**: Same grammar rules, types in curly braces, optional params in square brackets

### 5b: Other Languages (Native Conventions)

For changed files in other languages:

- **TypeScript**: TSDoc conventions
- **Go**: Go doc conventions (comments starting with symbol name)
- **Swift**: Swift doc comments (`///` with `- Parameter:`, `- Returns:`, `- Throws:`)
- **Rust**: Rustdoc (`///` with `# Sections`)
- **Zig**: Zig doc comments (`///`)
- **Vue SFC**: JSDoc/TSDoc in script section

### 5c: Version Detection for \@since

- Read version from `composer.json`, `package.json`, `Cargo.toml`, or latest git tag
- Apply to all new `\@since` tags

## Step 6: Check and Fix docs/ Directory

If a `docs/` directory exists, verify it's in sync with the current codebase state:

### 6a: Check for Stale Documentation

Compare the code changes in this branch against the existing documentation:

- **API changes**: If public methods/classes were added, changed, or removed, check that `docs/api-reference.md` reflects this
- **Config changes**: If config options were added or changed, check `docs/configuration.md`
- **New features**: If significant new functionality was added, check relevant doc pages
- **Installation changes**: If dependencies or requirements changed, check `docs/installation.md`

### 6b: Fix Stale Sections

For any documentation that's out of sync:

- **Diff and patch**: Update only the sections that are stale, preserving accurate custom prose
- **Add missing sections**: If new features aren't documented, add them in the appropriate place
- **Fix broken references**: Update any code examples or API references that don't match current code
- **Scaffold missing files**: If the project type has a standard doc structure and files are missing, create them

### 6c: Link Validation

- Verify all internal links in docs still point to valid files and anchors
- Use GitHub-style relative links: `[text](./file.md)`, `[text](./file.md#anchor)`

## Step 7: Commit Cleanup Changes

If any changes were made by linting, documentation fixes, or test fixes:

1. **Stage all cleanup changes**:
   ```bash
   git add -A
   ```

2. **Check if there are actually changes to commit**:
   ```bash
   git diff --staged --quiet
   ```
   If no staged changes, skip the commit.

3. **Commit with a descriptive message**:
   ```bash
   git commit -m "chore: lint, test, and documentation cleanup

   - Auto-fixed linting issues ({linter names})
   - Added/updated inline documentation
   - Updated docs/ to reflect current codebase
   - All tests passing"
   ```

## Step 8: Push to Remote

```bash
git push -u origin {branch_name}
```

If the branch already exists on the remote, just push:
```bash
git push
```

## Step 9: Create Draft Pull Request

### Determine the issue reference

Parse the branch name for an issue number:
- `feature/123-description` → #123
- `bugfix/456-description` → #456
- etc.

### Locate the pull request template

Check for `.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/` directory. If multiple templates exist in the directory, use `default.md` or the first available template.

### Gather branch changes for the PR body

1. **Get all commits** in the branch:
   ```bash
   git log main..HEAD --format="%h %s"
   ```

2. **Get the full diff summary**:
   ```bash
   git diff --stat main...HEAD
   ```

3. **Summarize the changes**: Write a concise narrative of what this branch accomplishes, based on commits and the diff.

### Create the draft pull request

```bash
gh pr create \
  --title "{type}: {description}" \
  --body "{body}" \
  --draft \
  --base main
```

### Pull request body content

Use the repo's pull request template as the base structure. Fill in the template sections with:

- **Description/Summary**: Narrative of what the branch implements, referencing the issue
- **Changes made**: List of key changes (files created/modified, features added, bugs fixed)
- **Testing**: Confirmation that tests pass, with count of tests run
- **Documentation**: Note that inline docs and docs/ were verified and updated
- **Linting**: Note that code was auto-formatted with the project's linter(s)
- **Closes #{issue_number}**: Auto-close link (if issue number was found in branch name)

If no template exists, use this default structure:

```markdown
## Summary

{Narrative summary of what this branch implements}

Closes #{issue_number}

## Changes

- {Key change 1}
- {Key change 2}
- {Key change 3}

## Checklist

- [x] Code linted and auto-formatted ({linter names})
- [x] All tests passing ({n} tests)
- [x] Inline documentation verified and updated
- [x] docs/ directory in sync with codebase
- [ ] Ready for review
```

## Step 10: Report Results

After the draft pull request is created, report:

- **Pull request**: link to the draft PR
- **Branch**: the branch name
- **Lint**: which linters ran, how many issues were auto-fixed, any remaining issues
- **Tests**: pass/fail with count
- **Documentation**: what was added or updated (inline docs and docs/)
- **Commits**: count of total commits in the branch, plus the cleanup commit if one was made
- **Unfixable issues**: any linting or documentation issues that need manual attention

Format as a concise summary:

```
Draft PR created: feature/123-add-user-search → main
→ https://github.com/user/repo/pull/456

Cleanup summary:
  Lint: Pint auto-fixed 12 issues, ESLint auto-fixed 4 issues
  Tests: ✅ 87 tests passing
  Docs: Added 6 inline doc blocks, updated docs/api-reference.md
  Cleanup commit: chore: lint, test, and documentation cleanup

No unfixable issues found — ready for review.
```

## Important Notes

- **GitHub only**: This skill is specifically for GitHub repositories using the `gh` CLI. For GitLab, use prepare-merge-request instead.
- **Draft only**: The pull request is always created as a draft. The user marks it ready for review after checking.
- **Base branch**: Always target `main`. If `main` doesn't exist, use `master`.
- **Test failures are blockers**: Never create a pull request with failing tests. Fix them or stop and report.
- **Cleanup commit is separate**: The lint/doc cleanup commit is distinct from the feature work so reviewers can see what was auto-fixed vs manually written.
- **Don't modify feature code**: Linting and documentation are the only changes made. Do not refactor, optimize, or alter the logic of the feature code.
- **Respect .editorconfig and linter configs**: Use the project's existing linter configuration. Don't override rules or add new ones.
- **GitHub Actions awareness**: If the repo has GitHub Actions CI workflows, note in the PR body that CI will run automatically. Do not attempt to trigger or wait for CI.
