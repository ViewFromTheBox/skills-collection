---
name: update-laravel
description: Compare the current Laravel version against the latest release, check for breaking changes via upgrade guide + codebase scan, then either auto-PR a clean update or file individual issues for each breaking change. Works for both Laravel apps and packages, on GitHub and GitLab.
---

# Update Laravel Dependencies

Automate updating `laravel/framework` and all first-party `laravel/*` packages. Detect breaking changes by fetching the official upgrade guide and scanning the codebase. If clean, open a PR/MR. If breaking, file one issue per affected change.

## Step 1: Detect Current State

1. Read `composer.json` to determine:
   - **Project type**: `"type": "project"` = app, `"type": "library"` = package
   - **Current `laravel/framework` version constraint** (e.g. `^11.0`)
   - **All other `laravel/*` dependencies** in both `require` and `require-dev` (sanctum, horizon, telescope, pennant, pulse, reverb, scout, socialite, etc.)
2. Read `composer.lock` to get the **exact installed versions** of each `laravel/*` package.

## Step 2: Check Latest Versions

1. Query the Packagist API for each `laravel/*` package found:
   ```
   https://repo.packagist.org/p2/laravel/framework.json
   ```
2. Extract the latest stable release version for each package.
3. Compare against the currently installed versions from `composer.lock`.
4. If ALL packages are already up to date, report that and **stop**.
5. Otherwise, compile the list of packages that need updating and their target versions.

## Step 3: Detect Platform

Auto-detect the git hosting platform:

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

## Step 4: Fetch and Analyze Upgrade Guide

1. Determine if the update crosses a **major version boundary** (e.g. 11.x → 12.x).
2. Fetch the Laravel upgrade guide for the target major version:
   ```
   https://laravel.com/docs/{major}.x/upgrade
   ```
   - Use WebFetch or curl to retrieve the guide content.
   - Cache the content for the duration of the session (don't re-fetch if already retrieved).
3. If the update is within the same major version (e.g. 11.32 → 11.45), also check the changelog for any notable changes, but these are rarely breaking.
4. Parse the upgrade guide to extract a structured list of breaking changes. For each change, identify:
   - **Category** (e.g. "Authentication", "Eloquent", "Routing", "Configuration")
   - **What changed** (API signature change, removed class, renamed method, config key change, etc.)
   - **Affected symbols** — specific class names, method names, config keys, artisan commands, or patterns to search for

## Step 5: Scan Codebase for Impact

For each breaking change extracted from the upgrade guide:

1. **Search the codebase** for usage of the affected symbols:
   - Grep for class names, method calls, config keys, facade references
   - Check `config/*.php` files for deprecated config keys
   - Check `routes/*.php` for routing changes
   - Check migration files for schema changes
   - Check service provider registrations
   - For packages: also check `src/`, `config/`, and `database/` directories
2. **Classify** each breaking change:
   - **Affects this project** — found matching usage in the codebase (record the file paths and line numbers)
   - **Not applicable** — no matching usage found
3. Compile results into two lists:
   - `applicable_breaks` — breaking changes that affect this project
   - `clean_changes` — breaking changes that don't apply

## Step 6a: Clean Path (No Applicable Breaking Changes)

If `applicable_breaks` is empty:

1. **Create branch**:
   ```bash
   git checkout -b update/laravel-{version} main
   ```
   Where `{version}` is the target `laravel/framework` version (e.g. `12.0.0`).

2. **Update composer.json**:
   - Update the version constraint for `laravel/framework`
   - Update version constraints for ALL other `laravel/*` dependencies to their latest compatible versions
   - For apps: update in `require` and/or `require-dev` as appropriate
   - For packages: update in `require` (be mindful of minimum version constraints for packages — use `^{major}.0` format)

3. **Run composer update**:
   ```bash
   composer update laravel/* --with-all-dependencies
   ```
   This updates both `composer.json` constraints AND `composer.lock`.

4. **Run the test suite**:
   ```bash
   # For apps:
   php artisan test
   # For packages:
   composer test
   # Fallback:
   ./vendor/bin/phpunit
   ```
   - If tests **fail**: stop, report the failures, do NOT push. The user needs to investigate.
   - If tests **pass**: continue to step 5.

5. **Commit and push**:
   ```bash
   git add composer.json composer.lock
   git commit -m "Update Laravel to {version}"
   git push -u origin update/laravel-{version}
   ```

6. **Create PR/MR**:
   - **GitHub**:
     ```bash
     gh pr create --title "Update Laravel to {version}" --template PULL_REQUEST_TEMPLATE.md
     ```
     If no template is found, use a default body summarizing which packages were updated and their old → new versions.
   - **GitLab**:
     ```bash
     glab mr create --title "Update Laravel to {version}" --description "$(cat .gitlab/merge_request_templates/Default.md)"
     ```
     If no template is found, use a default body.

   The PR/MR body should include:
   - List of all updated `laravel/*` packages with old → new versions
   - Confirmation that the upgrade guide was checked and no breaking changes apply
   - Confirmation that the test suite passed

## Step 6b: Breaking Path (Applicable Breaking Changes Found)

If `applicable_breaks` is not empty:

1. **Locate issue templates**:
   - GitHub: `.github/ISSUE_TEMPLATE/` directory — look for YAML or Markdown templates
   - GitLab: `.gitlab/issue_templates/` directory — look for Markdown templates
   - If no templates exist, use a sensible default format.

2. **Create one issue per applicable breaking change**:
   - **GitHub**:
     ```bash
     gh issue create --title "{title}" --body "{body}"
     ```
   - **GitLab**:
     ```bash
     glab issue create --title "{title}" --description "{body}"
     ```

3. Each issue should contain:
   - **Title**: `Laravel {version} Breaking Change: {category} — {short description}`
   - **Body**:
     - **What changed**: Description from the upgrade guide
     - **Affected files**: List of files and line numbers in this project that use the affected API
     - **Required action**: What needs to be modified to accommodate the change
     - **Laravel upgrade guide reference**: Link to the relevant section of the upgrade guide
     - **Priority context**: Whether this is a hard break (code will error) vs. a soft deprecation

4. After all issues are created, report a summary:
   - Total breaking changes found in the upgrade guide
   - How many affect this project (with issue links)
   - How many don't apply
   - Recommendation: resolve the issues first, then re-run this skill to perform the clean update

## Important Notes

- **Never force push** or modify existing branches without user confirmation.
- **Always work from `main`** as the base branch. If `main` doesn't exist, check for `master` and use that.
- **Package vs App differences**:
  - Packages use broader version constraints (e.g. `^11.0|^12.0` to support multiple Laravel versions)
  - Apps use specific constraints (e.g. `^12.0`)
  - Packages may need to update minimum PHP version requirements alongside Laravel bumps
- **First-party package compatibility**: When bumping `laravel/framework`, ensure all other `laravel/*` packages are bumped to versions compatible with the new framework version. The Packagist API provides `require` constraints for this.
- The `\@` symbol appears in PHP annotations — be aware when scanning code patterns.
