---
name: update-livewire
description: Compare the current Livewire version against the latest release, check for breaking changes via upgrade guide + codebase scan, then either auto-PR a clean update or file individual issues for each breaking change. Works for both Laravel apps and packages that depend on Livewire, on GitHub and GitLab.
---

# Update Livewire Dependencies

Automate updating `livewire/livewire` and related Livewire packages. Detect breaking changes by fetching the official upgrade guide and scanning the codebase. If clean, open a PR/MR. If breaking, file one issue per affected change.

## Step 1: Detect Current State

1. Read `composer.json` to determine:
   - **Project type**: `"type": "project"` = app, `"type": "library"` = package
   - **Current `livewire/livewire` version constraint** (e.g. `^3.0`)
   - **Related Livewire ecosystem packages** in both `require` and `require-dev` (e.g. `livewire/volt`, any community packages like `wire-elements/*`, `filament/*` that depend on Livewire)
2. Read `composer.lock` to get the **exact installed versions** of `livewire/livewire` and related packages.

## Step 2: Check Latest Versions

1. Query the Packagist API for `livewire/livewire` and any related first-party Livewire packages found:
   ```
   https://repo.packagist.org/p2/livewire/livewire.json
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

1. Determine if the update crosses a **major version boundary** (e.g. 3.x → 4.x).
2. Fetch the Livewire upgrade guide for the target major version:
   ```
   https://livewire.laravel.com/docs/upgrading
   ```
   - Use WebFetch or curl to retrieve the guide content.
   - Cache the content for the duration of the session (don't re-fetch if already retrieved).
   - Also check the Livewire changelog on GitHub for the specific version:
     ```
     https://github.com/livewire/livewire/releases
     ```
3. If the update is within the same major version (e.g. 3.4 → 3.5), also check the changelog for any notable changes, but these are rarely breaking.
4. Parse the upgrade guide to extract a structured list of breaking changes. For each change, identify:
   - **Category** (e.g. "Component Lifecycle", "Properties", "Validation", "JavaScript Hooks", "Wire Directives", "Alpine Integration", "File Uploads", "Events")
   - **What changed** (API signature change, removed method, renamed directive, behavior change, etc.)
   - **Affected symbols** — specific class names, method names, Blade directives, wire: attributes, Alpine interop patterns, or JavaScript hooks to search for

### Livewire-Specific Patterns to Check

When scanning for breaking changes, pay special attention to:

- **PHP component files**: Classes extending `Livewire\Component` or `Livewire\Form`
  - Lifecycle hooks (`mount`, `hydrate`, `dehydrate`, `updating*`, `updated*`)
  - Property declarations and `#[Rule]` / `#[Validate]` attributes
  - `$this->dispatch()`, `$this->emit()` (v2 → v3 migration changed emit to dispatch)
  - `#[Computed]`, `#[Lazy]`, `#[On]`, `#[Layout]`, `#[Title]` attributes
  - `WithFileUploads`, `WithPagination` traits
- **Blade templates**: Files in `resources/views/livewire/` and any `.blade.php` files
  - `wire:model` modifiers (`.defer`, `.lazy`, `.live`, `.blur`, `.throttle`)
  - `wire:click`, `wire:submit`, `wire:keydown`, and other action directives
  - `wire:loading`, `wire:dirty`, `wire:offline` directives
  - `\@livewire` and `\@livewireStyles` / `\@livewireScripts` directives (vs `\@persist`, `\@teleport`)
  - `<livewire:component-name />` tag syntax
  - `$wire`, `\@entangle`, `\@this` JavaScript interop
- **JavaScript files**: `resources/js/**/*.js`
  - `Livewire.on()`, `Livewire.hook()`, `Livewire.dispatch()`
  - `window.livewire` (v2 global) vs `window.Livewire` (v3 global)
  - Custom JavaScript hooks and plugin registrations
- **Config files**: `config/livewire.php`
  - Layout configuration, temporary upload directory, class namespace
- **Route files**: `routes/web.php`
  - `Route::livewire()` (v2) or full-page component routing

## Step 5: Scan Codebase for Impact

For each breaking change extracted from the upgrade guide:

1. **Search the codebase** for usage of the affected symbols:
   - Grep PHP files for class names, method calls, trait usage, attribute usage
   - Grep Blade files for `wire:` directives, `\@livewire` directives, `$wire` usage, `\@entangle` usage
   - Grep JS files for `Livewire.on()`, `Livewire.hook()`, `window.livewire`
   - Check `config/livewire.php` for deprecated config keys
   - For packages: also check `src/`, `resources/views/`, and `config/` directories
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
   git checkout -b update/livewire-{version} main
   ```
   Where `{version}` is the target `livewire/livewire` version (e.g. `4.0.0`).

2. **Update composer.json**:
   - Update the version constraint for `livewire/livewire`
   - Update version constraints for related Livewire packages (e.g. `livewire/volt`) to their latest compatible versions
   - For apps: update in `require` and/or `require-dev` as appropriate
   - For packages: update in `require` (use `^{major}.0` format for broader compatibility)

3. **Run composer update**:
   ```bash
   composer update livewire/* --with-all-dependencies
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
   git commit -m "Update Livewire to {version}"
   git push -u origin update/livewire-{version}
   ```

6. **Create PR/MR**:
   - **GitHub**:
     ```bash
     gh pr create --title "Update Livewire to {version}" --template PULL_REQUEST_TEMPLATE.md
     ```
     If no template is found, use a default body summarizing which packages were updated and their old → new versions.
   - **GitLab**:
     ```bash
     glab mr create --title "Update Livewire to {version}" --description "$(cat .gitlab/merge_request_templates/Default.md)"
     ```
     If no template is found, use a default body.

   The PR/MR body should include:
   - List of all updated Livewire packages with old → new versions
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
   - **Title**: `Livewire {version} Breaking Change: {category} — {short description}`
   - **Body**:
     - **What changed**: Description from the upgrade guide
     - **Affected files**: List of files and line numbers in this project that use the affected API
     - **Required action**: What needs to be modified to accommodate the change
     - **Livewire upgrade guide reference**: Link to the relevant section of the upgrade guide
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
  - Packages use broader version constraints (e.g. `^3.0|^4.0` to support multiple Livewire versions)
  - Apps use specific constraints (e.g. `^4.0`)
  - Packages may need to update minimum Laravel version requirements alongside Livewire bumps since Livewire has a minimum Laravel version dependency
- **Livewire + Laravel compatibility**: Livewire major versions require specific minimum Laravel versions (e.g. Livewire 3 requires Laravel 10+). When bumping Livewire, verify the current Laravel version satisfies the new Livewire version's requirements. If not, warn the user that Laravel must be updated first.
- **Alpine.js coupling**: Livewire ships with Alpine.js bundled. Major Livewire updates may also bump the bundled Alpine version, which can affect custom Alpine components or plugins. Flag any `\@alpinejs` or custom Alpine directives in the codebase as potentially affected during major version bumps.
- All version bump types (major, minor, patch) get the same full analysis — always run the upgrade guide check + codebase scan.
