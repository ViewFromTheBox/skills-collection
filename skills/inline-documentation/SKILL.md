---
name: inline-documentation
description: Creates inline documentation consistent with WordPress inline documentation standards for PHP and JavaScript. Uses language-standard conventions for other languages (Go doc, Swift doc comments, Rustdoc, Zig doc comments, etc.). Covers file headers, functions, classes, hooks, constants, and inline comments.
---

# Inline Documentation

Add comprehensive inline documentation to the codebase. PHP and JavaScript follow WordPress inline documentation standards. Other languages follow their own ecosystem's documentation conventions.

## Step 1: Detect the Codebase

1. **Identify languages present**: Scan for file types:
   - `.php` → PHP (WordPress standards)
   - `.js`, `.mjs` → JavaScript (WordPress JSDoc standards)
   - `.ts`, `.tsx` → TypeScript (TSDoc conventions)
   - `.go` → Go (Go doc conventions)
   - `.swift` → Swift (Swift doc comment conventions)
   - `.rs` → Rust (Rustdoc conventions)
   - `.zig` → Zig (Zig doc comment conventions)
   - `.vue` → Vue SFC (JSDoc for `<script>`, WordPress standards for any PHP)
   - `.jsx`, `.tsx` → React (JSDoc/TSDoc conventions)
   - `.blade.php` → Blade (PHP WordPress standards for PHP sections)

2. **Scan for existing documentation patterns**: Check if the project already has inline docs to match style and coverage level.

3. **Identify what needs documentation**: Find all undocumented or under-documented:
   - Functions and methods
   - Classes, interfaces, traits, structs, enums
   - File headers
   - Constants and class properties
   - Hooks (actions and filters — PHP/WordPress)
   - Events (JavaScript)
   - Inline comments for non-obvious logic

## Step 2: PHP — WordPress PHPDoc Standards

Reference: https://developer.wordpress.org/coding-standards/inline-documentation-standards/php/

### Language & Grammar Rules

- **Third-person singular verbs**: Test by prefixing with "It" — "It retrieves the post" not "It retrieve the post"
- **Complete sentences** with periods for descriptions (except file header summaries which are titles)
- **Oxford commas** in all lists
- **Focus on "what" and "when"**, not "why"
- **No HTML/Markdown** in summaries — write "image tag" instead of "`<img>`"
- **Line wrapping**: Wrap text after 80 characters, maximum 120 characters total width

### File Headers

```php
/**
 * Summary (file title, not a sentence).
 *
 * Description with complete sentences.
 *
 * \@package Package_Name
 * \@subpackage Subpackage_Name
 * \@since x.x.x
 */
```

### Functions & Methods

DocBlocks must directly precede the element with no intervening code:

```php
/**
 * Summary. One sentence maximum, ending with a period.
 *
 * Description. Detailed explanation with complete sentences.
 * Can span multiple lines and paragraphs.
 *
 * \@since 1.0.0
 * \@since 1.2.0 Added the `$format` parameter.
 *
 * \@see related_function()
 * \@link https://external-reference.example.com
 *
 * \@global wpdb $wpdb WordPress database object.
 *
 * \@param string          $name     Required. The item name.
 * \@param int             $count    Optional. Number of items. Default 10.
 * \@param array|string    $args     {
 *     Optional. An array of arguments.
 *
 *     \@type string $key   Description of key. Default 'value'.
 *                          Accepts 'value1', 'value2'.
 *     \@type bool   $flag  Description of flag. Default false.
 * }
 * \@return array|false Array of items on success, false on failure.
 */
```

Key rules:
- `\@since` always uses three-digit format (e.g. `1.0.0`, `4.4.0`)
- `\@since MU (3.0.0)` for features ported from WordPress MU
- Multiple `\@since` tags for significant changes (new params, behavior changes, params becoming optional)
- `\@access private` only for private/internal APIs
- `\@param` marks optional parameters explicitly, includes defaults and accepted values
- `\@param` for arrays uses `\@type` sub-tags for array keys
- `\@return` lists all possible return types with descriptions
- `\@global` entries aligned by type and variable name

### Classes

```php
/**
 * Summary. One sentence, two-line maximum.
 *
 * Description with complete sentences explaining the class purpose.
 *
 * \@since 1.0.0
 *
 * \@see Related_Class
 */
class My_Class {
```

Class properties:

```php
/**
 * Description of the property.
 *
 * \@since 1.0.0
 * \@access private
 * \@var string
 */
private $property_name;
```

### Hooks (Actions & Filters)

```php
/**
 * Filters the post title before display.
 *
 * Allows modification of the post title in the loop.
 *
 * \@since 1.0.0
 * \@since 2.0.0 Added the `$post_id` parameter.
 *
 * \@param string $title   The post title.
 * \@param int    $post_id The post ID.
 * \@return string The filtered post title.
 */
$title = apply_filters( 'my_plugin_post_title', $title, $post_id );
```

```php
/**
 * Fires after a post is saved to the database.
 *
 * \@since 1.0.0
 *
 * \@param int     $post_id The post ID.
 * \@param WP_Post $post    The post object.
 */
do_action( 'my_plugin_after_save_post', $post_id, $post );
```

For dynamic hooks, document the pattern:
```php
/**
 * Filters the {$post_type} archive title.
 *
 * The dynamic portion of the hook name, `$post_type`,
 * refers to the post type slug.
 *
 * \@since 1.0.0
 *
 * \@param string $title The archive title.
 */
$title = apply_filters( "my_plugin_{$post_type}_archive_title", $title );
```

### Constants

```php
/**
 * Description of the constant.
 *
 * \@since 1.0.0
 * \@var string
 */
define( 'MY_CONSTANT', 'value' );
```

### Requires and Includes

```php
/**
 * Summary description of the included file.
 */
require_once ABSPATH . 'file-name.php';
```

### Inline Comments

- Single-line: `// Brief explanation of non-obvious code.`
- Multi-line: `/* Longer explanation spanning multiple lines. */`
- Should explain non-obvious logic, edge cases, or workarounds
- Do NOT state the obvious — `// Increment counter` above `$i++` adds no value

## Step 3: JavaScript — WordPress JSDoc Standards

Reference: https://developer.wordpress.org/coding-standards/inline-documentation-standards/javascript/

### Language & Grammar Rules

Same as PHP: third-person singular, complete sentences, Oxford commas, no markup in summaries.

### Functions

```javascript
/**
 * Summary. (use period)
 *
 * Description. (use period)
 *
 * \@since      1.0.0
 * \@deprecated 2.0.0 Use newFunction() instead.
 * \@access     private
 *
 * \@class
 * \@augments parent
 * \@mixes    mixin
 *
 * \@alias    realName
 * \@memberof namespace
 *
 * \@see  Function/class relied on
 * \@link URL
 *
 * \@fires   eventName
 * \@fires   className#eventName
 * \@listens event:eventName
 *
 * \@param {string}   name             The item name.
 * \@param {number}   [count=10]       Optional. Number of items.
 * \@param {Object}   options          The options object.
 * \@param {string}   options.key      Description of key.
 * \@param {boolean}  options.flag     Description of flag.
 *
 * \@yield {type} Yielded value description.
 *
 * \@return {Array|boolean} Return value description.
 */
```

Key rules:
- Types in curly braces: `{string}`, `{number}`, `{Object}`, `{Array}`, `{boolean}`
- Optional params in square brackets: `[param]`, `[param=default]`
- Object properties use dot notation: `options.key`
- `\@since` uses three-digit format: `1.0.0`
- `\@fires` and `\@listens` for event documentation
- Markdown permitted in long descriptions
- No HTML/markdown in `\@param` and `\@return` descriptions

### Classes (Backbone-style)

Use `\@lends` before the class definition object. Document `initialize` with:
- `\@constructs namespace.Class`
- `\@augments Parent`
- `\@mixes mixin`
- `\@requires` for module dependencies

### File Headers

```javascript
/**
 * Summary (file title, not a sentence).
 *
 * Description with complete sentences.
 *
 * \@package Package_Name
 * \@since   1.0.0
 */
```

### Inline Comments

- Single-line: `// Brief explanation.`
- Multi-line: `/* Longer explanation. */`
- Reserve `/** */` for formal doc blocks only

## Step 4: TypeScript — TSDoc Conventions

Follow TSDoc standard (builds on JSDoc with TypeScript-specific features):

```typescript
/**
 * Retrieves a user by their unique identifier.
 *
 * \@param id - The unique user identifier.
 * \@returns The user object, or undefined if not found.
 *
 * \@throws {@link NotFoundError}
 * Thrown if the user ID is invalid.
 *
 * \@example
 * ```typescript
 * const user = getUser(123);
 * ```
 *
 * \@since 1.0.0
 */
function getUser(id: number): User | undefined {
```

Key rules:
- Types are NOT in doc comments (TypeScript provides them)
- Use `\@param name - Description` (hyphen separator, no type)
- Use `\@returns` (not `\@return`)
- Use `\@throws` with `{@link ErrorClass}` for exceptions
- Use `\@example` with fenced code blocks
- `\@since` for versioning (following WordPress convention)

## Step 5: Go — Go Doc Conventions

Follow standard Go documentation conventions:

```go
// UserService provides methods for managing user accounts.
//
// It handles creation, retrieval, updating, and deletion
// of user records in the database.
type UserService struct {
```

```go
// GetByID retrieves a user by their unique identifier.
//
// It returns ErrNotFound if no user exists with the given ID.
// The returned User includes all profile fields but excludes
// the password hash.
func (s *UserService) GetByID(ctx context.Context, id int64) (*User, error) {
```

Key rules:
- Comments start with the name of the thing being documented
- Use `//` comments (not `/* */` blocks) — Go convention
- First sentence is the summary, appears in `go doc` output
- No tags (`\@param`, `\@return`, etc.) — Go doc doesn't use them
- Document parameters and return values in prose
- Document errors that can be returned
- Package-level doc comment goes in `doc.go` or the main package file

Package documentation:
```go
// Package users provides user account management functionality.
//
// It supports creating, retrieving, updating, and deleting
// user records with role-based access control.
package users
```

## Step 6: Swift — Swift Doc Comment Conventions

Follow Apple's Swift documentation markup:

```swift
/// Retrieves a user by their unique identifier.
///
/// This method queries the local database first, falling back
/// to the remote API if the user is not cached locally.
///
/// - Parameter id: The unique user identifier.
/// - Returns: The user object.
/// - Throws: `NetworkError.timeout` if the remote request exceeds 30 seconds.
///
/// ## Example
/// ```swift
/// let user = try await userService.getUser(id: 123)
/// ```
///
/// - Since: 1.0.0
func getUser(id: Int) async throws -> User {
```

Key rules:
- Use `///` line comments (preferred) or `/** */` block comments
- First paragraph is the summary
- `- Parameter name: Description` for single params
- `- Parameters:` block with indented `- name: Description` for multiple
- `- Returns: Description`
- `- Throws: Description`
- `- Note:`, `- Warning:`, `- Important:` for callouts
- `- Since:` for versioning
- Supports Markdown in descriptions
- `## Section` headings for organizing longer docs

Classes and structs:
```swift
/// A service for managing user accounts.
///
/// `UserService` provides CRUD operations for user records
/// with built-in caching and offline support.
///
/// - Since: 1.0.0
class UserService {
```

Properties:
```swift
/// The maximum number of retry attempts for failed requests.
///
/// - Since: 1.0.0
let maxRetries: Int
```

## Step 7: Rust — Rustdoc Conventions

Follow standard Rustdoc conventions:

```rust
/// Retrieves a user by their unique identifier.
///
/// Returns `None` if no user exists with the given ID.
///
/// # Arguments
///
/// * `id` - The unique user identifier.
///
/// # Returns
///
/// The user record, or `None` if not found.
///
/// # Errors
///
/// Returns [`DatabaseError`] if the connection fails.
///
/// # Examples
///
/// ```
/// let user = service.get_user(123)?;
/// assert_eq!(user.name, "Alice");
/// ```
///
/// # Panics
///
/// Panics if the database connection pool is not initialized.
pub fn get_user(&self, id: u64) -> Result<Option<User>, DatabaseError> {
```

Key rules:
- Use `///` for item documentation, `//!` for module/crate documentation
- First paragraph is the summary
- Use `# Heading` sections: `# Arguments`, `# Returns`, `# Errors`, `# Panics`, `# Examples`, `# Safety`
- Code examples in ```` ``` ```` blocks are compiled and run as tests by default
- Use `` [`Type`] `` for cross-references (auto-linked by Rustdoc)
- Markdown is fully supported
- `# Safety` section required for `unsafe` functions

Module documentation:
```rust
//! User account management.
//!
//! This module provides types and functions for creating,
//! retrieving, updating, and deleting user records.
```

## Step 8: Zig — Zig Doc Comment Conventions

Follow Zig's documentation comment conventions:

```zig
/// Retrieves a user by their unique identifier.
///
/// Returns `null` if no user exists with the given ID.
/// Returns an error if the database connection fails.
pub fn getUser(self: *UserService, id: u64) !?User {
```

Key rules:
- Use `///` for doc comments (three slashes)
- Use `//!` for top-level/container doc comments
- First paragraph is the summary
- Document parameters and return values in prose
- No tag system — describe behavior in natural language
- Use backticks for code references: `` `parameter_name` ``, `` `null` ``, `` `error` ``
- Zig's doc generator uses these comments

## Step 9: Vue SFC

For Vue Single File Components, document both `<script>` and template sections:

```vue
<script setup lang="ts">
/**
 * UserProfile displays the user's profile information.
 *
 * \@since 1.0.0
 */

/**
 * The user ID to display.
 *
 * \@since 1.0.0
 */
const props = defineProps<{
  userId: number
}>()

/**
 * Emitted when the user clicks the edit button.
 *
 * \@since 1.0.0
 */
const emit = defineEmits<{
  (e: 'edit', userId: number): void
}>()
</script>
```

Follow JSDoc/TSDoc for the script section based on whether it's JavaScript or TypeScript.

## Step 10: Apply Documentation

### Process

1. **Scan for undocumented code**: Find all functions, classes, methods, properties, constants, hooks, and events that lack documentation.

2. **Scan for under-documented code**: Find existing doc comments that are missing required elements:
   - Missing `\@since` tags
   - Missing `\@param` for parameters
   - Missing `\@return` for non-void functions
   - Missing descriptions
   - Incomplete hook documentation

3. **Add missing documentation**: Write doc comments following the appropriate language standard:
   - Read the code to understand what it does
   - Write accurate summaries and descriptions
   - Document all parameters with types and descriptions
   - Document return values with all possible types
   - Add `\@since` tags (use the current project version from composer.json/package.json, or ask if unknown)
   - Document hooks with their parameters and filter return values
   - Add inline comments for non-obvious logic

4. **Fix existing documentation**: Update doc comments that are:
   - Inaccurate (description doesn't match what the code does)
   - Incomplete (missing params, missing return, missing since)
   - Poorly formatted (wrong tag order, missing periods, incorrect verb form)

5. **Do NOT add trivial documentation**: Skip obvious cases:
   - Simple getters/setters with self-evident names
   - Single-line helper functions with clear names
   - Code that is already self-documenting

### Version detection for \@since tags

- **PHP/Laravel**: Read version from `composer.json` `"version"` field, or the latest git tag
- **JavaScript/Node**: Read version from `package.json` `"version"` field
- **Swift**: Read version from git tags or `Package.swift`
- **Go**: Read version from git tags
- **Rust**: Read version from `Cargo.toml` `version` field
- If no version can be determined, use `\@since Unknown` and warn the user

## Step 11: Report Results

After documentation is complete, report:

- **Files documented**: count of files that were modified
- **Functions/methods documented**: count of new doc blocks added
- **Doc blocks updated**: count of existing doc blocks that were fixed
- **Hooks documented**: count of action/filter doc blocks added (PHP)
- **Inline comments added**: count of inline comments for non-obvious logic
- **Warnings**: any `\@since Unknown` tags that need version numbers

Format as a concise summary:

```
Inline documentation updated:

PHP (WordPress standards):
  - 24 functions documented, 8 doc blocks updated
  - 6 hooks documented (3 filters, 3 actions)
  - 12 inline comments added

JavaScript (WordPress JSDoc standards):
  - 15 functions documented, 3 doc blocks updated
  - 4 event handlers documented

Warnings:
  - 3 items tagged @since Unknown — version could not be determined
```

## Important Notes

- **WordPress standards for PHP and JS are non-negotiable**: Follow the WordPress inline documentation standards exactly for PHP and JavaScript files. Do not deviate from the formatting, tag order, or grammar rules.
- **Language-native for everything else**: Go, Swift, Rust, Zig, and TypeScript follow their own ecosystem conventions. Don't force WordPress-style tags into languages that don't use them.
- **Accuracy over completeness**: Every doc comment must be accurate. An inaccurate doc comment is worse than no doc comment. Read the code carefully before writing documentation.
- **Don't document the obvious**: `$i++` doesn't need `// Increment counter`. Focus inline comments on non-obvious logic, workarounds, edge cases, and business rules.
- **Preserve existing documentation**: If a function already has accurate documentation, don't rewrite it. Only add missing elements or fix inaccuracies.
- The `\@` symbol appears frequently in doc tags (`\@param`, `\@return`, `\@since`, `\@throws`, etc.) — handle carefully in all languages.
