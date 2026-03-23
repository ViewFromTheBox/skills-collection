---
name: update-documentation
description: Updates all documentation in the docs/ directory based on the current codebase state. Diffs code against existing docs and patches stale sections while preserving custom prose. Scaffolds missing standard doc files. Auto-generates and verifies API references from doc comments. Uses platform-appropriate link formatting (GitHub/GitLab). All docs use YAML frontmatter with title.
---

# Update Documentation

Scan the codebase, compare it to existing documentation, update stale sections, scaffold missing standard doc files, and ensure the API reference reflects the actual public API. All docs are markdown files with YAML frontmatter.

## Step 1: Detect Platform

1. Parse the git remote URL (`git remote get-url origin`):
   - Contains `github.com` → **GitHub**
   - Contains `gitlab.com` or a self-hosted GitLab domain → **GitLab**
2. Confirm by checking directory presence:
   - `.github/` directory → GitHub
   - `.gitlab/` directory → GitLab

This determines link formatting:
- **GitHub**: relative links like `[Configuration](./configuration.md)`, anchors like `[section](#section-name)`
- **GitLab**: relative links like `[Configuration](configuration.md)`, anchors like `[section](configuration.md#section-name)` — GitLab requires the filename prefix even for same-page anchors in some rendering contexts

## Step 2: Discover Project & Docs Structure

### 2a: Identify project type

- **Laravel package** (`composer.json` with `"type": "library"`): Standard doc structure expected:
  - `docs/index.md` — overview and quick start
  - `docs/installation.md` — install steps, requirements, service provider, config publishing
  - `docs/configuration.md` — config options explained
  - `docs/usage.md` — common usage patterns and examples
  - `docs/api-reference.md` — public API documentation
  - `docs/troubleshooting.md` — common issues and solutions

- **Laravel app** (`composer.json` with `"type": "project"`): Doc structure varies, but common pages:
  - `docs/index.md` — project overview
  - `docs/setup.md` — local development setup
  - `docs/architecture.md` — project structure and patterns
  - `docs/api.md` — API endpoints reference (if applicable)
  - `docs/deployment.md` — deployment process

- **Go project** (`go.mod`): Common structure:
  - `docs/index.md` — overview
  - `docs/installation.md` — install and build
  - `docs/usage.md` — CLI usage or library usage
  - `docs/api-reference.md` — exported types and functions
  - `docs/configuration.md` — config options (if applicable)

- **React/Vue/Node project** (`package.json`): Common structure:
  - `docs/index.md` — overview
  - `docs/getting-started.md` — setup and installation
  - `docs/components.md` — component reference (if component library)
  - `docs/api-reference.md` — API documentation
  - `docs/configuration.md` — config options

- **Swift project** (`.swift` files, `Package.swift`): Common structure:
  - `docs/index.md` — overview
  - `docs/installation.md` — SPM/CocoaPods setup
  - `docs/usage.md` — usage patterns
  - `docs/api-reference.md` — public API

### 2b: Inventory existing docs

1. Scan the `docs/` directory for all `.md` files
2. For each file, read the content and extract:
   - Frontmatter (title and any other fields)
   - Section headings and their content
   - Code examples
   - Links (internal and external)
3. Build a map of what each doc currently covers

## Step 3: Scan the Codebase

Analyze the current state of the source code to build a comprehensive picture of what the docs should describe.

### 3a: Public API surface

- **PHP/Laravel**: Scan `src/` for public classes, methods, traits, interfaces, facades
  - Parse PHPDoc blocks (`/** ... */`) for descriptions, params, return types, exceptions
  - Identify published config keys from `config/*.php`
  - Identify Artisan commands from `Commands/` directory
  - Identify published migrations, views, and assets
  - Identify route definitions and middleware
  - Identify events, listeners, and observers
  - Identify Blade components and directives

- **Go**: Scan for exported types, functions, methods, interfaces, constants
  - Parse Go doc comments
  - Identify CLI flags and commands (if CLI app)
  - Identify config struct fields

- **JavaScript/TypeScript**: Scan for exported functions, classes, components, hooks, types
  - Parse JSDoc blocks
  - Identify props/emits (Vue), props/hooks (React)
  - Identify exported constants and config

- **Swift**: Scan for public/open classes, structs, enums, protocols, functions
  - Parse Swift doc comments (`///` and `/** */`)
  - Identify public initializers and their parameters

### 3b: Configuration options

- Read config files (`config/*.php`, `.env.example`, `tailwind.config.*`, `vite.config.*`, etc.)
- Extract all configurable options with their defaults and descriptions
- Note any environment variables referenced

### 3c: Installation requirements

- Read `composer.json` / `package.json` / `go.mod` / `Package.swift` for:
  - Minimum language/runtime version requirements
  - Key dependencies
  - Required PHP extensions, Node version, Go version, etc.

### 3d: Testing patterns

- Identify test framework and how to run tests
- Note any setup steps required for testing

## Step 4: Diff and Patch Existing Docs

For each existing doc file, compare the codebase analysis against the current documentation:

### 4a: Identify stale sections

- **API signatures changed**: Method renamed, parameters added/removed, return type changed, but docs still show the old signature
- **Config options changed**: New config keys added, old ones removed or renamed, default values changed
- **Installation steps outdated**: Minimum version bumped, dependencies changed, new required steps
- **Code examples broken**: Example code references methods, classes, or patterns that no longer exist
- **Missing features**: New public API, commands, routes, or components not documented at all
- **Removed features**: Docs describe features or APIs that have been removed from the codebase

### 4b: Patch stale sections

For each stale section:

1. **Preserve custom prose**: Do NOT rewrite human-authored explanations, tutorials, or narrative text that is still accurate. Only update the parts that are factually wrong or outdated.
2. **Update signatures and examples**: Replace outdated method signatures, config keys, or code examples with current ones.
3. **Add missing items**: Insert documentation for new features, methods, config options, etc. in the appropriate section, matching the existing style and level of detail.
4. **Mark removed items**: If a documented feature no longer exists, remove its documentation or add a deprecation notice if the feature was recently removed.
5. **Fix broken links**: Update internal links that point to renamed or moved sections/files.

### 4c: Preserve document style

- Match the existing writing style (formal/informal, terse/verbose)
- Match heading levels and section organization
- Match code example format (inline vs fenced blocks, language tags)
- Match list styles (bullets vs numbers)

## Step 5: Scaffold Missing Doc Files

For any standard doc files that don't exist (based on the project type identified in Step 2):

1. **Create the file** in `docs/` with appropriate frontmatter:
   ```yaml
   ---
   title: {Page Title}
   ---
   ```

2. **Generate content** based on the codebase analysis:
   - Write substantive documentation, not just stubs or placeholders
   - Include real code examples derived from the actual codebase
   - Follow the style of existing doc files if any exist
   - Use platform-appropriate link formatting

### Standard doc file content guidelines

**index.md**:
- Package/project name and brief description (from composer.json/package.json description)
- Key features list
- Quick start example (minimal code to get started)
- Links to other doc pages

**installation.md**:
- Requirements (PHP/Node/Go/Swift version, extensions, etc.)
- Install command (`composer require`, `npm install`, etc.)
- Framework-specific setup (service provider registration, config publishing, migrations)
- Environment variables needed

**configuration.md**:
- All config options with descriptions, types, and defaults
- Example config file with comments
- Environment variable overrides

**usage.md**:
- Common use cases with code examples
- Step-by-step guides for primary features
- Tips and best practices

**api-reference.md**:
- Generated from code (see Step 6)

**troubleshooting.md**:
- Common errors and solutions
- FAQ items
- Debugging tips

## Step 6: Generate and Verify API Reference

### 6a: Generate from doc comments

Parse source code doc comments to build the API reference:

- **PHP (PHPDoc)**:
  ```php
  /**
   * Create a new user.
   *
   * \@param string $name The user's name
   * \@param string $email The user's email
   * \@return User The newly created user
   * \@throws ValidationException If validation fails
   */
  public function create(string $name, string $email): User
  ```
  Becomes:
  ```markdown
  ### `create(string $name, string $email): User`

  Create a new user.

  **Parameters:**
  - `$name` (string) — The user's name
  - `$email` (string) — The user's email

  **Returns:** `User` — The newly created user

  **Throws:** `ValidationException` — If validation fails
  ```

- **Go doc comments**: Parse `//` comments above exported symbols
- **JSDoc**: Parse `/** */` blocks with `\@param`, `\@returns`, `\@example`
- **Swift doc comments**: Parse `///` and `/** */` with `- Parameter:`, `- Returns:`, `- Throws:`

### 6b: Verify existing API reference

If `api-reference.md` already exists:

1. Compare documented API against actual public API
2. **Missing methods/classes**: Add documentation for undocumented public API
3. **Removed methods/classes**: Remove or mark as deprecated
4. **Changed signatures**: Update parameters, return types, descriptions
5. **Preserve custom examples**: If someone added usage examples beyond the auto-generated content, keep them

### 6c: Organize the API reference

Group the API reference logically:

- By class/module (for OOP codebases)
- By feature area (for functional codebases)
- Alphabetically within each group
- Include a table of contents at the top for large APIs

## Step 7: Validate Links and Cross-References

1. **Check internal links**: Verify all `[text](./file.md)` links point to files that exist
2. **Check anchor links**: Verify `#section-name` anchors match actual headings
3. **Fix broken links**: Update any links that point to renamed or moved files/sections
4. **Platform formatting**: Ensure all links use the correct format for the detected platform:
   - GitHub: `[text](./file.md)`, `[text](./file.md#anchor)`
   - GitLab: `[text](file.md)`, `[text](file.md#anchor)`

## Step 8: Ensure Consistent Frontmatter

Every doc file must have YAML frontmatter with at least a `title` field:

```yaml
---
title: Installation
---
```

For existing files:
- If frontmatter exists, verify the title is accurate
- If frontmatter is missing, add it with a title derived from the first heading or filename

For new files:
- Always include frontmatter with an appropriate title

## Step 9: Report Results

After all updates are complete, report:

- **Files updated**: list of existing doc files that were modified, with a summary of what changed
- **Files created**: list of new doc files that were scaffolded
- **API reference changes**: count of methods/classes added, updated, or removed
- **Links fixed**: count of broken links that were repaired
- **Stale sections patched**: count of outdated sections that were updated

Format as a concise summary:

```
Documentation updated:

Updated:
  - docs/configuration.md — Added 3 new config options, updated 1 default value
  - docs/api-reference.md — Added 5 new methods, updated 2 signatures, removed 1 deprecated method
  - docs/installation.md — Updated minimum PHP version to 8.2

Created:
  - docs/troubleshooting.md — Generated with 8 common issues from codebase analysis

Links: 2 broken links fixed
Frontmatter: 1 missing title added
```

## Important Notes

- **Diff and patch, don't regenerate**: The core principle is to update what's stale while preserving what's accurate. Never rewrite an entire doc file from scratch if it already exists — human-authored explanations, tutorials, and examples that are still correct should be preserved.
- **Real content, not stubs**: When scaffolding new doc files, write substantive documentation with real code examples from the codebase. Empty files or placeholder text ("TODO: add documentation here") are not acceptable.
- **Code examples must work**: Every code example in the docs should be derived from or verified against the actual codebase. Don't include examples that reference non-existent methods, classes, or config keys.
- **Respect existing organization**: If docs already have a non-standard organization, work within it rather than reorganizing. Only apply the standard structure when scaffolding missing files.
- **Language awareness**: Match the doc comment format to the language (PHPDoc for PHP, JSDoc for JS/TS, Go doc for Go, Swift doc for Swift). Don't mix formats.
- The `\@` symbol appears in PHPDoc tags (`\@param`, `\@return`, `\@throws`), JSDoc tags (`\@param`, `\@returns`, `\@example`), and Blade directives — be aware when parsing and writing doc content.
