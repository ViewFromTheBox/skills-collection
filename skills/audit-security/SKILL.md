---
name: audit-security
description: Deep security audit that scans the codebase for vulnerabilities, insecure patterns, and misconfigurations across web and mobile stacks. Generates a detailed report and files grouped issues using the bug template with security labels and the Awaiting Review milestone. Auto-detects stack (Laravel, React, Vue, Go, SwiftUI, UIKit, etc.) and platform (GitHub/GitLab).
context: fork
---

# Security Audit

Deep audit of the codebase for security vulnerabilities, insecure coding patterns, and misconfigurations at both the code level and architectural level. Generates a detailed report and files one issue per vulnerability category in the remote repository.

## The Core Problem

Security vulnerabilities accumulate silently — an unvalidated input here, a leaked secret there, a misconfigured CORS policy somewhere else. By the time they're exploited, the damage is done. This skill performs a systematic static analysis of code patterns, dependency risks, and architectural decisions, then turns every finding into a tracked issue.

## Phase 1: Discover the Codebase

1. **Identify the tech stack**:
   - Scan for file types and framework indicators:
     - `composer.json` with `laravel/framework` → Laravel (check for Sanctum, Passport, Fortify, Breeze)
     - `.jsx`, `.tsx` → React
     - `.vue` → Vue
     - `.swift` + SwiftUI imports → SwiftUI
     - `.swift` + UIKit imports → UIKit
     - `go.mod` → Go
     - `Dockerfile`, `docker-compose.yml` → Containerized app
     - `.env`, `.env.example` → Environment config
   - A project may contain multiple stacks (e.g. Laravel API + React SPA)

2. **Map the security surface area**:
   - Identify authentication system (Sanctum, Passport, Fortify, Breeze, JWT, OAuth, custom)
   - Identify authorization layer (Gates, Policies, middleware, RBAC)
   - Identify API layer (REST, GraphQL) and its authentication mechanism
   - Identify file upload handling
   - Identify external service integrations (payment gateways, email services, third-party APIs)
   - Identify secrets management approach (.env, vault, config)
   - Identify deployment configuration (Docker, CI/CD, server config)

3. **Detect platform** (for issue creation):
   - Parse `git remote get-url origin`: `github.com` → GitHub, `gitlab.com` or self-hosted → GitLab
   - Confirm via `.github/` or `.gitlab/` directory presence
   - Verify CLI availability: `gh` (GitHub) or `glab` (GitLab)

## Phase 2: Injection & Input Validation

### 2a: SQL Injection

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Raw queries with user input** | Critical | `DB::raw()`, `DB::select()`, `whereRaw()`, `orderByRaw()` concatenating user input instead of using parameter bindings |
| **String interpolation in queries** | Critical | Go: `fmt.Sprintf` building SQL strings. PHP: `"SELECT * FROM users WHERE id = $id"` |
| **Dynamic column/table names** | Major | User input used for column names, table names, or order direction without whitelist validation |
| **LIKE injection** | Minor | `LIKE '%{$input}%'` without escaping `%` and `_` wildcards in user input |

### 2b: Cross-Site Scripting (XSS)

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unescaped output** | Critical | Blade: `{!! $variable !!}` with user-controlled data. React: `dangerouslySetInnerHTML` with unsanitized input. Vue: `v-html` with user data. |
| **JavaScript context injection** | Critical | User data rendered inside `<script>` tags, `onclick` handlers, or `javascript:` URLs |
| **URL injection** | Major | User input used in `href`, `src`, `action` attributes without URL validation (allows `javascript:` protocol) |
| **SVG/HTML file uploads** | Major | Uploaded SVG/HTML files served without `Content-Type` sanitization (stored XSS vector) |
| **Missing CSP headers** | Major | No Content-Security-Policy header configured, or overly permissive policy with `unsafe-inline`, `unsafe-eval` |

### 2c: Command Injection

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Shell command execution** | Critical | `exec()`, `system()`, `shell_exec()`, `passthru()`, `proc_open()` with user input. Go: `exec.Command()` with unsanitized args. |
| **Process arguments** | Critical | User input passed as command arguments without escaping or whitelist validation |
| **Template injection** | Critical | User input rendered in server-side templates (Blade, Twig) without proper escaping, SSTI patterns |

### 2d: Path Traversal & File Inclusion

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Path traversal** | Critical | User input used in file paths (`Storage::get($userInput)`, `file_get_contents($path)`) without sanitizing `../` sequences |
| **Local file inclusion** | Critical | Dynamic `include`, `require`, or `view()` calls with user-controlled paths |
| **Unrestricted file upload path** | Major | Uploaded files stored with user-controlled filenames without sanitization |

### 2e: General Input Validation

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing request validation** | Major | Laravel controllers using `$request->all()`, `$request->input()` without Form Request validation or inline `$request->validate()` |
| **Mass assignment** | Critical | Eloquent models without `$fillable` or `$guarded`, or using `$guarded = []` |
| **Regex denial of service** | Minor | Complex regex patterns vulnerable to ReDoS (catastrophic backtracking) on user input |
| **Deserialization** | Critical | `unserialize()` on user input, `json_decode` without schema validation on untrusted data |
| **XML external entities** | Critical | XML parsing without disabling external entity loading (`LIBXML_NOENT` not disabled) |

## Phase 3: Authentication & Authorization

### 3a: Authentication

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Hardcoded credentials** | Critical | Passwords, API keys, tokens hardcoded in source code (not just .env) |
| **Weak password policy** | Major | No minimum length enforcement, missing complexity requirements in validation rules |
| **Missing brute force protection** | Major | Login endpoints without rate limiting (`ThrottleRequests` middleware, `RateLimiter`) |
| **Insecure password storage** | Critical | Passwords stored in plaintext, MD5, SHA1, or any non-bcrypt/argon2 hash |
| **Session fixation** | Major | Session ID not regenerated after login (`Session::regenerate()` missing post-authentication) |
| **Missing 2FA on sensitive actions** | Minor | Destructive or sensitive operations (password change, email change, account deletion) without re-authentication or 2FA |
| **JWT vulnerabilities** | Critical | JWT without signature verification, `alg: none` accepted, secrets in code, tokens without expiry |
| **OAuth misconfiguration** | Major | Missing state parameter validation, overly broad scopes, redirect URI not validated |

### 3b: Authorization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing authorization checks** | Critical | Controller actions that modify resources without `$this->authorize()`, Policy checks, or Gate checks |
| **IDOR vulnerabilities** | Critical | Routes using user-supplied IDs to access resources without verifying ownership (e.g. `/api/users/{id}/profile` without checking auth user matches) |
| **Privilege escalation** | Critical | Role/permission checks that can be bypassed, admin routes without proper middleware |
| **Missing middleware** | Major | Routes or route groups without `auth`, `verified`, or role-based middleware where expected |
| **Overly permissive policies** | Major | Policies that return `true` without proper checks, `before()` methods granting blanket admin access |

## Phase 4: Data Exposure & Secrets

### 4a: Secrets Management

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Secrets in source code** | Critical | API keys, database passwords, encryption keys, OAuth secrets hardcoded in PHP/JS/Go/Swift files |
| **Secrets in git history** | Critical | `.env` files committed to git (check `.gitignore`), secrets in old commits |
| **.env exposure** | Critical | `.env` not in `.gitignore`, `.env.example` containing real values, `.env` accessible via web |
| **Debug information leakage** | Major | `APP_DEBUG=true` patterns in production config, stack traces exposed to users, verbose error pages |
| **Logging sensitive data** | Major | Passwords, tokens, credit card numbers, PII logged in application logs |

### 4b: Data Exposure

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Overly broad API responses** | Major | API resources/transformers returning sensitive fields (password hashes, internal IDs, tokens, PII) that clients don't need |
| **Missing `$hidden` on models** | Major | Eloquent models without `$hidden` for sensitive attributes (password, remember_token, etc.) |
| **Information in error messages** | Major | Exception messages exposing database structure, file paths, or internal implementation details to end users |
| **GraphQL introspection** | Major | GraphQL schema introspection enabled in production, exposing full API surface |
| **Source maps in production** | Minor | JavaScript source maps deployed to production, exposing original source code |

### 4c: Mobile Data Security (Swift)

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Plaintext storage** | Critical | Sensitive data stored in `UserDefaults`, plist files, or unencrypted CoreData instead of Keychain |
| **Missing Keychain usage** | Major | Tokens, passwords, API keys not stored in Keychain or using `kSecAttrAccessible` inappropriately |
| **Insecure network config** | Critical | App Transport Security exceptions (`NSAllowsArbitraryLoads`) without justification |
| **Clipboard exposure** | Major | Sensitive data copied to clipboard without `UIPasteboard.general.setItems(_, options: [.expirationDate:])` |
| **Screenshot protection** | Minor | Sensitive screens not hidden in app switcher via `applicationWillResignActive` |
| **Certificate pinning** | Major | Missing SSL certificate pinning for API communication |

## Phase 5: Transport & Network Security

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing HTTPS enforcement** | Critical | HTTP URLs for API endpoints, missing `ForceScheme('https')`, no HSTS header |
| **Insecure cookie flags** | Major | Session cookies without `Secure`, `HttpOnly`, `SameSite` flags |
| **CORS misconfiguration** | Critical | `Access-Control-Allow-Origin: *` with credentials, overly permissive origins, wildcard in allowed headers |
| **Missing CSRF protection** | Critical | State-changing endpoints without CSRF tokens, `\@csrf` missing in Blade forms, SPA without CSRF cookie setup |
| **Insecure TLS configuration** | Major | TLS 1.0/1.1 support, weak cipher suites, missing certificate validation in HTTP clients |
| **Open redirects** | Major | Redirect URLs constructed from user input without domain whitelist validation |
| **Missing security headers** | Major | Missing `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` headers |
| **WebSocket security** | Major | WebSocket connections without authentication, missing origin validation |

## Phase 6: Dependency & Supply Chain Security

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Known vulnerable dependencies** | Critical | Check `composer.lock` / `package-lock.json` / `go.sum` for packages with known CVEs (cross-reference with advisory databases) |
| **Outdated security packages** | Major | Authentication/encryption packages significantly behind latest versions |
| **Unnecessary dependencies** | Minor | Dev dependencies in production, unused packages that expand attack surface |
| **Lock file integrity** | Major | Missing `composer.lock` / `package-lock.json` / `go.sum` (builds not reproducible, vulnerable to supply chain attacks) |
| **Dependency confusion** | Minor | Private package names that could conflict with public registry packages |

## Phase 7: Cryptography & Randomness

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Weak hashing algorithms** | Critical | MD5, SHA1 used for security purposes (password hashing, token generation, integrity checks) |
| **Insecure random generation** | Critical | `rand()`, `mt_rand()`, `Math.random()`, `math/rand` used for security tokens instead of `random_bytes()`, `crypto/rand`, `SecRandomCopyBytes` |
| **Hardcoded encryption keys** | Critical | `APP_KEY`, encryption keys, or signing secrets hardcoded instead of environment-sourced |
| **ECB mode usage** | Critical | AES-ECB or other insecure cipher modes instead of AES-GCM or AES-CBC with HMAC |
| **Missing encryption at rest** | Major | Sensitive database fields (PII, payment data) stored without column-level encryption |

## Phase 8: File Upload & Storage Security

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unrestricted file types** | Critical | File uploads without MIME type validation, relying only on extension checking |
| **Executable uploads** | Critical | Upload directories within web root, uploaded files served without `Content-Disposition: attachment` |
| **Missing file size limits** | Major | No `upload_max_filesize` validation, missing validation rules for file size |
| **Insecure storage permissions** | Major | Uploaded files with world-readable permissions, public disk used for private files |
| **Missing virus/malware scanning** | Minor | File uploads without any content inspection for malware |
| **Image processing vulnerabilities** | Major | Image manipulation without input validation (ImageMagick/GD vulnerabilities) |

## Phase 9: Compile Findings

1. **Deduplicate**: If the same vulnerability pattern appears across multiple instances of a shared module, group them.
2. **Classify each finding**:
   - **Severity**: Critical / Major / Minor
     - **Critical**: Exploitable vulnerability that could lead to data breach, unauthorized access, or remote code execution (SQL injection, XSS with user data, hardcoded secrets, missing auth checks, mass assignment)
     - **Major**: Significant security weakness that increases attack surface or weakens defenses (missing security headers, CORS misconfiguration, weak session config, overly broad API responses, missing rate limiting)
     - **Minor**: Best practice improvement, defense-in-depth hardening (missing 2FA on settings, source maps in production, unnecessary dependencies, screenshot protection)
   - **OWASP category**: Map to relevant OWASP Top 10 category where applicable (A01-A10)
   - **CWE reference**: Include Common Weakness Enumeration ID where applicable
   - **Affected files**: List of file paths and line numbers
   - **Exploitability**: Brief note on how this could be exploited and by whom (unauthenticated attacker, authenticated user, admin, etc.)
   - **Remediation**: Specific recommendation for how to fix it

3. **Group by vulnerability category**: All instances of the same vulnerability type go into a single group. This becomes one issue.

## Phase 10: Generate Report

Create `SECURITY_AUDIT.md` in the project root with the following structure:

```markdown
# Security Audit Report

**Date**: {date}
**Stack**: {detected stack}
**OWASP Top 10 Coverage**: {list which categories were checked}

## Summary

| Severity | Count |
|----------|-------|
| Critical | {n}   |
| Major    | {n}   |
| Minor    | {n}   |
| **Total** | **{n}** |

## Critical Findings

### {Category}: {Title}

**Severity**: Critical
**OWASP**: {A01-A10 category} | **CWE**: {CWE-ID}

{Description of the vulnerability and why it's dangerous}

**Exploitability**: {who can exploit this and how}

**Affected locations**:
- `{file}:{line}` — {brief context}
- `{file}:{line}` — {brief context}

**Recommended fix**: {specific remediation steps with code examples}

---

## Major Findings
{same structure}

## Minor Findings
{same structure}
```

## Phase 11: File Issues

### 11a: Setup

1. **Verify the security label exists** in the repo:
   - GitHub: `gh label list --search "security"`
   - GitLab: `glab label list --search "security"`
   - If it doesn't exist, **create it automatically**:
     - GitHub: `gh label create "security" --color "D93F0B" --description "Security vulnerability"`
     - GitLab: `glab label create "security" --color "#D93F0B" --description "Security vulnerability"`

2. **Locate the "Awaiting Review" milestone**:
   - GitHub: `gh api repos/{owner}/{repo}/milestones --jq '.[] | select(.title=="Awaiting Review") | .number'`
   - GitLab: `glab api projects/{id}/milestones --jq '.[] | select(.title=="Awaiting Review") | .id'`
   - If not found, warn the user and file issues without a milestone.

3. **Locate the bug issue template**:
   - GitHub: `.github/ISSUE_TEMPLATE/` — look for a bug report template (YAML or Markdown)
   - GitLab: `.gitlab/issue_templates/` — look for a bug report template
   - If no template exists, use a sensible default format.

### 11b: Create Issues

Create **one issue per vulnerability category**:

- **GitHub**:
  ```bash
  gh issue create \
    --title "{title}" \
    --body "{body}" \
    --label "security" \
    --label "bug" \
    --milestone "Awaiting Review"
  ```

- **GitLab**:
  ```bash
  glab issue create \
    --title "{title}" \
    --description "{body}" \
    --label "security,bug" \
    --milestone "Awaiting Review"
  ```

Each issue should contain:

- **Title**: `Security: {Category} — {short description}`
  - Example: `Security: SQL Injection — Raw queries with user input in ReportController`
  - Example: `Security: Authentication — Missing brute force protection on login endpoint`
  - Example: `Security: Data Exposure — API resources returning password hashes`
- **Body** (using the bug template structure):
  - **Severity**: Critical / Major / Minor
  - **OWASP**: Category reference (e.g. A03:2021 Injection)
  - **CWE**: CWE ID and name where applicable
  - **Description**: What the vulnerability is and why it's dangerous
  - **Exploitability**: Who can exploit this (unauthenticated, authenticated, admin) and a brief attack scenario
  - **Affected locations**: List of files and line numbers with brief context for each
  - **Steps to reproduce**: How to verify the vulnerability exists (e.g. "Send a POST to /api/search with `query='; DROP TABLE users;--`")
  - **Expected behavior**: What the secure behavior should be
  - **Recommended fix**: Specific code-level remediation steps with examples
  - **Resources**: Links to OWASP guidance, CWE entry, or framework security documentation

### 11c: Report Summary

After all issues are created, report:
- Total findings by severity and OWASP category
- Number of issues created (with links)
- Link to the `SECURITY_AUDIT.md` report
- Top 3 highest-risk findings to address immediately (prioritized by exploitability and impact)

## Important Notes

- **Static analysis only** — this skill reads source code, it does not perform penetration testing or dynamic scanning. Some vulnerabilities require runtime testing to confirm. Flag issues with confident recommendations but note where dynamic testing should verify.
- **Component-aware**: If a vulnerability is in shared middleware, a base controller, or a service, note that fixing it fixes all dependents. Prioritize shared code fixes.
- **Framework idioms**: Respect framework-specific security patterns:
  - Laravel: check middleware stacks, Form Requests, Policies, Sanctum/Passport config, encryption helpers, CSRF
  - React/Vue: check `dangerouslySetInnerHTML`/`v-html`, CSP compatibility, XSS in JSX/templates
  - Go: check `html/template` vs `text/template`, `crypto/rand` vs `math/rand`, context propagation, TLS config
  - Swift: check Keychain usage, ATS config, certificate pinning, biometric auth, data protection classes
- **Don't flag framework defaults**: If the framework handles security internally (e.g. Laravel's automatic CSRF, Eloquent's parameter binding, Go's `html/template` auto-escaping), don't flag it. Focus on developer code that bypasses or weakens built-in protections.
- **Sensitive report handling**: The `SECURITY_AUDIT.md` report contains vulnerability details. Remind the user to consider whether this should be committed to the repo or kept separately, especially for public repositories.
- The `\@` symbol appears in PHP/Blade directives (`\@csrf`), Swift attributes, Java annotations, and CSS at-rules — be aware when scanning code patterns.
