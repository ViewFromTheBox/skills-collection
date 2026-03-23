---
name: prepare-release
description: Prepares a package or app for release from a release/x.y.z branch. Lints, tests, verifies documentation, checks all version numbers match the branch version, audits dependencies for sync and security vulnerabilities, then creates a release PR/MR using the dedicated release template. Works on GitHub and GitLab.
---

# Prepare Release

Clean up and validate the current release branch for a new version release: lint, test, verify documentation, check version consistency across all files, audit dependencies, and create a release PR/MR using the dedicated release template.

## Step 1: Validate Branch State

1. **Verify on a release branch**:
   ```bash
   git branch --show-current
   ```
   The branch must match the pattern `release/x.y.z` (e.g. `release/1.2.0`, `release/2.0.0`).
   If not on a release branch, warn the user and stop.

2. **Extract the version number** from the branch name:
   - `release/1.2.0` → version `1.2.0`
   - This version will be used to verify all version references across the project.

3. **Check for uncommitted changes**:
   ```bash
   git status
   ```
   If there are uncommitted changes, warn the user. These should be committed or stashed before preparing the release.

4. **Determine the target branch**: Default to `main`, but allow the user to specify a different target (e.g. `develop`). Fall back to `master` if `main` doesn't exist.

## Step 2: Detect Platform

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

## Step 3: Version Number Verification

Verify the release version (`x.y.z` from the branch name) is consistent across ALL version references in the project.

### Auto-detect version locations

| File | What to Check |
|------|---------------|
| `composer.json` | `"version": "x.y.z"` field |
| `package.json` | `"version": "x.y.z"` field |
| `Cargo.toml` | `version = "x.y.z"` field |
| `go.mod` | Module version (if applicable) |
| `Package.swift` | Version constants or comments |
| `CHANGELOG.md` | Unreleased section should be renamed to the version with a date |
| `config/*.php` | Version constants (e.g. `'version' => '1.2.0'`) |
| `src/**/*.php` | Version constants in classes (e.g. `const VERSION = '1.2.0'`) |
| `*.swift` | Version strings or constants |
| `README.md` | Version badges, installation instructions with version numbers |
| `docs/installation.md` | Version references in install commands |
| `.env.example` | `APP_VERSION` or similar keys |
| `docker-compose.yml` | Image version tags |
| `Dockerfile` | Version labels or ARG values |
| GitHub Actions (`.github/workflows/*.yml`) | Version references in CI config |
| GitLab CI (`.gitlab-ci.yml`) | Version references in CI config |

### Process

1. **Scan all files** for version references matching the pattern `x.y.z` (semantic version strings)
2. **Compare each found version** against the release branch version
3. **Fix mismatches**: Update any version references that don't match the release version
4. **Report changes**: List all files where versions were updated

### Special cases

- **CHANGELOG.md**: If there's an `## [Unreleased]` or `## Unreleased` section, rename it to `## [{version}] - {date}` (using today's date in YYYY-MM-DD format). Add a new empty `## [Unreleased]` section above it.
- **Version constraints** (in dependencies): Do NOT update these — only update the project's own version declarations
- **Previous version references** in documentation (e.g. "Added in v1.1.0"): Do NOT update these — only update references to the "current" version

## Step 4: Dependency Audit

### 4a: Lock File Sync

Verify lock files are in sync with their manifests:

- **PHP**: `composer validate` — check composer.lock matches composer.json
- **Node**: Verify package-lock.json is in sync with package.json
- **Rust**: `cargo check` — verify Cargo.lock is consistent
- **Go**: `go mod verify` — verify go.sum integrity

If lock files are out of sync, run the appropriate update command:
```bash
# PHP
composer update --lock

# Node
npm install

# Rust
cargo update

# Go
go mod tidy
```

### 4b: Outdated Dependencies

Check for significantly outdated dependencies:

```bash
# PHP
composer outdated --direct

# Node
npm outdated

# Rust
cargo outdated  # if installed

# Go
go list -u -m all
```

Flag any dependencies that are more than one major version behind. Include in the report but do NOT auto-update (that's a separate concern from the release).

### 4c: Security Vulnerability Scan

```bash
# PHP
composer audit

# Node
npm audit

# Rust
cargo audit  # if installed

# Go
govulncheck ./...  # if installed
```

- **If critical vulnerabilities are found**: warn the user prominently in the report. Do NOT block the release — the user decides whether to proceed.
- **If moderate/low vulnerabilities are found**: note them in the report.

## Step 5: Lint with Auto-Fix

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
4. **Stage lint fixes**: Don't commit yet — accumulate all fixes for a single cleanup commit

## Step 6: Run Tests

Execute the full test suite:

```bash
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
  - If still failing after 3 attempts, **stop and report the failures**. Do NOT proceed with a release PR/MR that has failing tests.

## Step 7: Check and Fix Inline Documentation

Review all source files for inline documentation quality. Follow the standards from the inline-documentation skill:

- **PHP and JavaScript**: WordPress PHPDoc/JSDoc standards (third-person singular, `\@since` tags, complete parameter documentation)
- **Other languages**: Native conventions (Go doc, Swift doc, Rustdoc, TSDoc, Zig doc)

### Release-specific documentation checks

- Verify `\@since` tags on new features reference the correct release version
- Verify `\@deprecated` tags reference the correct deprecation version
- Verify no `\@since Unknown` tags remain — all must have the proper version

## Step 8: Check and Fix docs/ Directory

If a `docs/` directory exists, verify it's in sync with the current codebase:

- **API reference**: Ensure all public API is documented and matches current signatures
- **Configuration docs**: Ensure all config options are documented with correct defaults
- **Installation docs**: Ensure version requirements and install commands are accurate
- **Changelog/upgrade guide**: Ensure the release is properly documented
- **Link validation**: Verify all internal links point to valid files and anchors
- **Platform link formatting**: GitHub-style (`[text](./file.md)`) or GitLab-style (`[text](file.md)`) based on detected platform

## Step 9: Commit Cleanup Changes

If any changes were made (version bumps, lint fixes, documentation updates, lock file sync):

1. **Stage all changes**:
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
   git commit -m "chore: prepare release {version}

   - Updated version numbers to {version}
   - Auto-fixed linting issues ({linter names})
   - Added/updated inline documentation
   - Updated docs/ to reflect current codebase
   - Synced lock files
   - All tests passing"
   ```

## Step 10: Push to Remote

```bash
git push -u origin release/{version}
```

If the branch already exists on the remote:
```bash
git push
```

## Step 11: Create Release PR/MR

### Locate the release template

- **GitHub**: Look for a release-specific template:
  1. `.github/PULL_REQUEST_TEMPLATE/release.md`
  2. `.github/PULL_REQUEST_TEMPLATE/Release.md`
  3. Fall back to `.github/PULL_REQUEST_TEMPLATE.md` (default)
- **GitLab**: Look for a release-specific template:
  1. `.gitlab/merge_request_templates/Release.md`
  2. `.gitlab/merge_request_templates/release.md`
  3. Fall back to `.gitlab/merge_request_templates/Default.md`

### Gather release information

1. **Get all commits** in the release branch:
   ```bash
   git log main..HEAD --format="%h %s"
   ```

2. **Get the full diff summary**:
   ```bash
   git diff --stat main...HEAD
   ```

3. **Extract changelog entries**: If CHANGELOG.md exists, pull the entries for this version.

4. **Summarize the release**: Write a narrative of what this release includes.

### Create the release PR/MR

- **GitHub**:
  ```bash
  gh pr create \
    --title "Release {version}" \
    --body "{body}" \
    --base {target_branch}
  ```

- **GitLab**:
  ```bash
  glab mr create \
    --title "Release {version}" \
    --description "{body}" \
    --target-branch {target_branch}
  ```

Note: Release PRs/MRs are NOT created as drafts — they're ready for final review.

### PR/MR body content

Use the release template as the base structure. Fill in with:

- **Version**: The release version number
- **Summary**: Narrative of what's included in this release
- **Changelog**: Entries from CHANGELOG.md for this version (or a generated summary from commits)
- **Checklist**:
  - Version numbers verified and consistent across all files
  - Dependencies audited (lock sync, outdated, security)
  - All linting issues auto-fixed
  - Full test suite passing
  - Inline documentation verified and up to date
  - docs/ directory in sync with codebase
  - CHANGELOG.md updated with release date
- **Dependency audit results**: Summary of outdated and vulnerable dependencies (if any)

If no release template exists, use this default structure:

```markdown
## Release {version}

{Narrative summary of this release}

### Changelog

{Changelog entries for this version}

### Checklist

- [x] Version numbers consistent across all files ({version})
- [x] Dependencies audited — lock files in sync
- [x] Security scan: {n critical, n moderate, n low vulnerabilities} (or "no vulnerabilities found")
- [x] Code linted and auto-formatted ({linter names})
- [x] All tests passing ({n} tests)
- [x] Inline documentation verified and updated
- [x] docs/ directory in sync with codebase
- [x] CHANGELOG.md updated with release date

### Dependency Notes

{Any outdated or vulnerable dependencies to be aware of}
```

## Step 12: Report Results

After the release PR/MR is created, report:

- **PR/MR**: link to the release PR/MR
- **Version**: the release version
- **Branch**: `release/{version}` → `{target_branch}`
- **Version checks**: how many files had version updates, any mismatches found
- **Dependencies**: lock sync status, outdated count, vulnerability count
- **Lint**: which linters ran, how many issues were auto-fixed, any remaining issues
- **Tests**: pass/fail with count
- **Documentation**: what was added or updated
- **Warnings**: any issues that need manual attention before merging

Format as a concise summary:

```
Release PR created: release/1.2.0 → main
→ https://github.com/user/repo/pull/456

Release {version} preparation:
  Versions: Updated 4 files to 1.2.0 (composer.json, config/app.php, README.md, CHANGELOG.md)
  Dependencies: Lock files in sync | 2 outdated (non-critical) | 0 vulnerabilities
  Lint: Pint auto-fixed 8 issues
  Tests: ✅ 124 tests passing
  Docs: Updated api-reference.md, verified 12 @since tags

⚠️ Warnings:
  - 2 outdated dependencies: laravel/sanctum (3.x → 4.x), league/flysystem (3.x → 3.y)

Ready for final review and merge.
```

## Important Notes

- **NOT a draft**: Release PRs/MRs are created as regular (non-draft) PRs — they're ready for final review.
- **Target branch is configurable**: Default to `main`, but the user can specify a different target. Fall back to `master` if `main` doesn't exist.
- **Version consistency is critical**: Every version reference in the project must match the release branch version. This is the primary validation step.
- **Security vulnerabilities warn, don't block**: The release is the user's decision. Report vulnerabilities prominently but don't refuse to create the PR/MR.
- **Test failures DO block**: Never create a release PR/MR with failing tests.
- **CHANGELOG.md**: If it exists, the `[Unreleased]` section must be converted to the release version with today's date. This is non-negotiable for a release.
- **Don't modify feature code**: Only version numbers, lint fixes, documentation, and lock files are changed. No refactoring or logic changes.
- **Respect existing linter and editor configs**: Use the project's configuration as-is.
- **Platform auto-detect**: Same as all other skills — git remote URL + `.github/`/`.gitlab/` directory + `gh`/`glab` CLI verification.
- The `\@` symbol appears in doc tags (`\@since`, `\@deprecated`, `\@param`, etc.) — handle carefully when verifying documentation.
