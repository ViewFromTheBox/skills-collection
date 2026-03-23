---
name: audit-privacy
description: Deep privacy audit that scans the codebase for data collection, storage, sharing, and retention issues across web and mobile stacks. Generates a detailed report and files grouped issues using the bug template with privacy labels and the Awaiting Review milestone. Auto-detects stack (Laravel, React, Vue, Go, SwiftUI, UIKit, etc.) and platform (GitHub/GitLab).
context: fork
---

# Privacy Audit

Deep audit of the codebase for privacy violations, excessive data collection, improper data handling, and regulatory compliance gaps. Generates a detailed report and files one issue per privacy concern category in the remote repository.

## The Core Problem

Privacy violations erode user trust and create legal liability. Data collected without purpose, retained without policy, shared without consent, or logged without consideration adds up to a privacy posture that's difficult to remediate retroactively. This skill performs a systematic static analysis of how personal data flows through the codebase — collection, storage, processing, sharing, and retention — then turns every finding into a tracked issue.

## Phase 1: Discover the Codebase

1. **Identify the tech stack**:
   - Scan for file types and framework indicators:
     - `composer.json` with `laravel/framework` → Laravel (check for Sanctum, Passport, Socialite, Cashier, Nova)
     - `.jsx`, `.tsx` → React
     - `.vue` → Vue
     - `.swift` + SwiftUI imports → SwiftUI
     - `.swift` + UIKit imports → UIKit
     - `go.mod` → Go
     - `Dockerfile`, `docker-compose.yml` → Containerized app
     - `package.json` → Check for analytics, tracking, or ad SDKs
   - A project may contain multiple stacks (e.g. Laravel API + React SPA + iOS app)

2. **Map the data landscape**:
   - Identify user-facing forms and input collection points
   - Identify database models that store personal data
   - Identify authentication and user profile systems
   - Identify third-party integrations that receive or provide user data
   - Identify analytics, tracking, and telemetry systems
   - Identify logging infrastructure and what gets logged
   - Identify file upload and storage systems
   - Identify email/notification systems and what user data they reference
   - Identify payment processing and financial data handling
   - Identify API endpoints that expose or accept personal data

3. **Detect platform** (for issue creation):
   - Parse `git remote get-url origin`: `github.com` → GitHub, `gitlab.com` or self-hosted → GitLab
   - Confirm via `.github/` or `.gitlab/` directory presence
   - Verify CLI availability: `gh` (GitHub) or `glab` (GitLab)

## Phase 2: Personal Data Identification & Classification

### 2a: Data Inventory

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **PII in database schemas** | Major | Migration files and model definitions storing: names, email addresses, phone numbers, physical addresses, dates of birth, government IDs (SSN, passport), IP addresses, device identifiers |
| **Sensitive data categories** | Critical | Health/medical data, financial data (beyond payment processing), biometric data, racial/ethnic origin, political opinions, religious beliefs, sexual orientation, trade union membership — these require special handling under GDPR Article 9 / CCPA sensitive data |
| **Children's data** | Critical | Age collection or date of birth fields without age-gating logic (COPPA compliance), user registration without age verification |
| **Location data** | Major | GPS coordinates, geolocation API usage, IP-based location inference, address data beyond what's necessary |
| **Behavioral data** | Major | User activity tracking, click streams, page view logging, feature usage analytics, search history storage |

### 2b: Data Flow Mapping

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Data collection points** | Major | All forms, API endpoints, and SDK calls that collect personal data — verify each has a documented purpose |
| **Data passed to views/frontend** | Major | Backend passing more user data to frontend than needed (e.g. full user object sent to Blade/React when only name is displayed) |
| **Data in URLs** | Major | Personal data in query parameters, route parameters, or URL fragments (visible in logs, browser history, referrer headers) |
| **Data in local/session storage** | Major | PII stored in browser `localStorage`, `sessionStorage`, or cookies beyond session tokens |

## Phase 3: Consent & Transparency

### 3a: Consent Management

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing cookie consent** | Major | Analytics cookies, tracking pixels, or third-party scripts loaded without consent mechanism |
| **Pre-checked consent** | Major | Consent checkboxes defaulting to checked (violates GDPR — consent must be opt-in) |
| **Bundled consent** | Major | Single consent checkbox covering multiple unrelated purposes (consent must be granular) |
| **No consent records** | Major | Consent given by users not stored with timestamp, version, and scope (must be auditable) |
| **Missing consent for marketing** | Critical | Email/SMS marketing without explicit opt-in consent, newsletter signup without double opt-in |
| **Third-party SDK consent** | Major | Analytics SDKs (Google Analytics, Mixpanel, Segment, etc.) initialized before consent is obtained |

### 3b: Privacy Notices

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing privacy policy link** | Major | Registration forms, data collection pages, or app onboarding without link to privacy policy |
| **Data collection without notice** | Major | Background data collection (analytics, error tracking, device info) without disclosure in privacy policy |
| **Missing purpose specification** | Minor | Data collected without a clear, documented purpose in code comments or documentation |

## Phase 4: Data Minimization & Purpose Limitation

### 4a: Collection Minimization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Excessive form fields** | Major | Registration or profile forms collecting data not necessary for the service (e.g. date of birth for a note-taking app, phone number when only email is needed) |
| **Unnecessary data in API requests** | Major | API endpoints accepting more personal data fields than they need to function |
| **Over-broad database columns** | Minor | Database tables with personal data columns that are never read or displayed anywhere in the codebase |
| **Full objects passed when subset needed** | Major | Entire user models passed to services, jobs, or views when only specific non-sensitive fields are needed |

### 4b: Purpose Limitation

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Data repurposing** | Major | User data collected for one purpose used for another (e.g. support email used for marketing, purchase history used for profiling) |
| **Analytics on sensitive data** | Critical | Analytics or reporting aggregating sensitive personal data without anonymization |
| **Training data concerns** | Major | User-generated content or personal data fed to ML models or AI services without disclosure |

## Phase 5: Data Storage & Protection

### 5a: Encryption & Storage

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unencrypted PII in database** | Major | Sensitive fields (SSN, government ID, health data, financial data) stored without column-level encryption. Laravel: check for `Crypt::encrypt()`, `$casts` with `encrypted` |
| **Plaintext in backups** | Major | Database backup scripts without encryption, backup files stored without access controls |
| **Unencrypted file storage** | Major | User-uploaded documents containing PII stored without encryption at rest |
| **Cache containing PII** | Major | Personal data cached in Redis/Memcached/file cache without TTL or encryption consideration |

### 5b: Mobile Data Storage (Swift)

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **PII in UserDefaults** | Critical | Personal data stored in `UserDefaults` or plist files (not encrypted, included in backups) |
| **Missing Data Protection** | Major | Files containing PII without `FileProtectionType.complete` or `completeUnlessOpen` |
| **CoreData without encryption** | Major | CoreData stores with personal data not using encrypted persistent stores |
| **Unprotected Keychain items** | Major | Keychain items with `kSecAttrAccessibleAlways` instead of more restrictive access levels |
| **iCloud sync of PII** | Major | Personal data synced to iCloud without user awareness or opt-out, `NSUbiquitousKeyValueStore` with PII |
| **Analytics SDK data** | Major | Analytics SDKs configured to collect device identifiers, location, or personal data without user opt-in |

### 5c: Logging & Monitoring

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **PII in application logs** | Critical | Names, emails, phone numbers, addresses, IPs, or other PII written to log files via `Log::info()`, `console.log()`, `log.Printf()`, `print()` |
| **PII in error tracking** | Major | Error reporting services (Sentry, Bugsnag, Raygun) capturing PII in exception context, breadcrumbs, or user context |
| **PII in debug output** | Major | Debug/dump statements (`dd()`, `var_dump()`, `console.log()`) outputting personal data that could reach production |
| **Access logs with PII** | Major | Web server access logs capturing PII in URL parameters or POST bodies |
| **Audit trail completeness** | Minor | Access to personal data not logged for accountability (who accessed what, when) |

## Phase 6: Data Sharing & Third Parties

### 6a: Third-Party Data Transfers

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Analytics integrations** | Major | Google Analytics, Mixpanel, Segment, Amplitude, etc. receiving PII (user IDs, emails, names) instead of anonymized identifiers |
| **Social login data** | Major | OAuth/social login requesting more scopes than necessary, storing social profile data beyond what's needed for authentication |
| **Payment processor data** | Major | Sending more customer data to payment processors (Stripe, PayPal) than required for transactions |
| **Email service data** | Major | Email/SMS services (Mailgun, SendGrid, Twilio) receiving and storing more user data than necessary for delivery |
| **Advertising/tracking pixels** | Critical | Facebook Pixel, Google Ads, TikTok Pixel, or similar tracking scripts sending PII or browsing behavior without consent |
| **CDN/proxy exposure** | Minor | CDN or reverse proxy configurations that log or cache responses containing PII |
| **Cross-origin data leakage** | Major | CORS configuration allowing PII-containing API responses to be read by third-party domains |

### 6b: API Data Exposure

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Over-exposed API responses** | Major | API endpoints returning more personal data fields than the consuming client needs |
| **Missing field filtering** | Major | No API resource/transformer layer to control which fields are exposed — raw model serialization |
| **Public API without auth** | Critical | Endpoints exposing personal data accessible without authentication |
| **Enumeration vulnerabilities** | Major | Sequential IDs or predictable patterns allowing user data enumeration (e.g. `/api/users/1`, `/api/users/2`) |
| **GraphQL over-fetching** | Major | GraphQL schema exposing sensitive user fields without field-level authorization |

## Phase 7: Data Retention & Deletion

### 7a: Retention Policies

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **No retention policy** | Major | Personal data stored indefinitely without defined retention periods — no automated cleanup |
| **Soft deletes without purge** | Major | `SoftDeletes` trait on user-related models without a scheduled job to permanently purge after retention period |
| **Log retention** | Major | Application logs containing PII retained indefinitely without rotation or purge schedule |
| **Session data retention** | Minor | Expired sessions not cleaned up, session data containing PII persisted beyond session lifetime |
| **Backup retention** | Minor | Database backups containing PII retained beyond the defined backup retention window |

### 7b: Right to Deletion (Right to be Forgotten)

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **No account deletion mechanism** | Critical | No way for users to request account deletion, or no code path that handles full data removal |
| **Incomplete deletion** | Critical | Account deletion that removes the user record but leaves personal data in: related tables, file storage, cache, logs, third-party services, backups |
| **Missing cascade deletes** | Major | Foreign key relationships without cascade delete or explicit cleanup when user is deleted |
| **Third-party data deletion** | Major | No mechanism to request data deletion from third-party services when user deletes account (analytics, email providers, etc.) |
| **Data export capability** | Major | No mechanism for users to export their personal data (GDPR Article 20 — data portability) |

## Phase 8: Mobile Privacy (Apple-Specific)

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **App Tracking Transparency** | Critical | Accessing IDFA (`ASIdentifierManager`) or tracking user across apps/websites without `ATTrackingManager.requestTrackingAuthorization()` |
| **Privacy Nutrition Labels** | Major | App collecting data categories not declared in App Store privacy labels (cross-reference code with declared data types) |
| **Location permission scope** | Major | Requesting `Always` location permission when `WhenInUse` would suffice, no explanation in `NSLocationWhenInUseUsageDescription` |
| **Camera/microphone/contacts access** | Major | Accessing camera, microphone, photo library, or contacts without clear purpose strings in `Info.plist` |
| **Clipboard access** | Major | Reading `UIPasteboard` contents without user-initiated action (iOS 14+ shows clipboard access notification) |
| **Device fingerprinting** | Major | Collecting device model, OS version, screen size, installed fonts, or carrier info for fingerprinting purposes |
| **Background location** | Critical | Background location tracking enabled without clear user benefit and disclosure |

## Phase 9: Compile Findings

1. **Deduplicate**: If the same privacy pattern appears across multiple instances of a shared module, group them.
2. **Classify each finding**:
   - **Severity**: Critical / Major / Minor
     - **Critical**: Direct regulatory violation or high-risk data exposure (PII in logs without controls, missing deletion mechanism, tracking without consent, sensitive data without encryption, children's data without age-gating)
     - **Major**: Significant privacy gap that increases risk or erodes user trust (excessive data collection, missing consent records, over-broad API responses, third-party data leakage, no retention policy)
     - **Minor**: Best practice improvement, defense-in-depth (missing purpose documentation, backup retention, CDN caching, audit trail gaps)
   - **Regulatory reference**: Map to relevant regulation where applicable:
     - GDPR articles (e.g. Art. 5 — data minimization, Art. 6 — lawful basis, Art. 17 — right to erasure)
     - CCPA/CPRA sections
     - Apple App Store Review Guidelines (privacy sections)
   - **Affected files**: List of file paths and line numbers
   - **Data categories involved**: What type of personal data is affected (identifiers, contact info, financial, health, behavioral, etc.)
   - **Remediation**: Specific recommendation for how to fix it

3. **Group by privacy concern category**: All instances of the same concern type go into a single group. This becomes one issue.

## Phase 10: Generate Report

Create `PRIVACY_AUDIT.md` in the project root with the following structure:

```markdown
# Privacy Audit Report

**Date**: {date}
**Stack**: {detected stack}
**Regulatory Context**: {GDPR / CCPA / Apple App Store / applicable regulations}

## Summary

| Severity | Count |
|----------|-------|
| Critical | {n}   |
| Major    | {n}   |
| Minor    | {n}   |
| **Total** | **{n}** |

## Data Inventory

| Data Category | Collection Points | Storage Location | Third-Party Recipients | Retention Policy |
|--------------|-------------------|-----------------|----------------------|-----------------|
| {e.g. Email} | {registration form, API} | {users table} | {Mailgun, Segment} | {indefinite — needs policy} |

## Critical Findings

### {Category}: {Title}

**Severity**: Critical
**Regulation**: {GDPR Art. X / CCPA § X / App Store Guideline X.X}
**Data Categories**: {what personal data is involved}

{Description of the privacy concern and why it matters}

**Affected locations**:
- `{file}:{line}` — {brief context}
- `{file}:{line}` — {brief context}

**Recommended fix**: {specific remediation steps}

---

## Major Findings
{same structure}

## Minor Findings
{same structure}
```

## Phase 11: File Issues

### 11a: Setup

1. **Verify the privacy label exists** in the repo:
   - GitHub: `gh label list --search "privacy"`
   - GitLab: `glab label list --search "privacy"`
   - If it doesn't exist, **create it automatically**:
     - GitHub: `gh label create "privacy" --color "0E8A16" --description "Privacy concern"`
     - GitLab: `glab label create "privacy" --color "#0E8A16" --description "Privacy concern"`

2. **Locate the "Awaiting Review" milestone**:
   - GitHub: `gh api repos/{owner}/{repo}/milestones --jq '.[] | select(.title=="Awaiting Review") | .number'`
   - GitLab: `glab api projects/{id}/milestones --jq '.[] | select(.title=="Awaiting Review") | .id'`
   - If not found, warn the user and file issues without a milestone.

3. **Locate the bug issue template**:
   - GitHub: `.github/ISSUE_TEMPLATE/` — look for a bug report template (YAML or Markdown)
   - GitLab: `.gitlab/issue_templates/` — look for a bug report template
   - If no template exists, use a sensible default format.

### 11b: Create Issues

Create **one issue per privacy concern category**:

- **GitHub**:
  ```bash
  gh issue create \
    --title "{title}" \
    --body "{body}" \
    --label "privacy" \
    --label "bug" \
    --milestone "Awaiting Review"
  ```

- **GitLab**:
  ```bash
  glab issue create \
    --title "{title}" \
    --description "{body}" \
    --label "privacy,bug" \
    --milestone "Awaiting Review"
  ```

Each issue should contain:

- **Title**: `Privacy: {Category} — {short description}`
  - Example: `Privacy: Data Minimization — Registration form collects unnecessary fields`
  - Example: `Privacy: Logging — PII written to application logs without redaction`
  - Example: `Privacy: Retention — No automated purge for soft-deleted user records`
  - Example: `Privacy: Third-Party — Analytics SDK initialized before consent`
- **Body** (using the bug template structure):
  - **Severity**: Critical / Major / Minor
  - **Regulation**: Relevant GDPR article, CCPA section, or App Store guideline
  - **Data categories**: What personal data is involved
  - **Description**: What the privacy concern is and the risk to users
  - **Affected locations**: List of files and line numbers with brief context for each
  - **Steps to reproduce**: How to observe the privacy issue (e.g. "Register a new account and inspect the database — date of birth is stored but never displayed or used" or "Enable Charles Proxy and observe data sent to analytics on app launch before consent screen")
  - **Expected behavior**: What the privacy-respecting behavior should be
  - **Recommended fix**: Specific code-level remediation steps
  - **Resources**: Links to relevant GDPR guidance, CCPA requirements, Apple privacy documentation, or ICO/CNIL guidance

### 11c: Report Summary

After all issues are created, report:
- Total findings by severity and data category
- Number of issues created (with links)
- Link to the `PRIVACY_AUDIT.md` report
- Data inventory summary (what personal data is collected, where it's stored, who it's shared with)
- Top 3 highest-risk findings to address first (prioritized by regulatory exposure and user impact)

## Important Notes

- **Static analysis only** — this skill reads source code, it does not perform runtime traffic analysis or dynamic data flow tracing. Some privacy issues (like actual data sent to third parties at runtime) require network monitoring to confirm. Flag issues with confident recommendations but note where dynamic testing should verify.
- **Regulatory guidance, not legal advice** — this audit references GDPR, CCPA, and App Store guidelines for context but does not constitute legal compliance certification. Recommend consulting a privacy professional for formal compliance assessment.
- **Component-aware**: If a privacy issue is in a shared service (e.g. a logging middleware that captures PII globally), note that fixing it fixes all dependents. Prioritize shared code fixes.
- **Framework idioms**: Respect framework-specific privacy patterns:
  - Laravel: check `$hidden`/`$visible` on models, `Log` facade usage, Form Requests, API Resources, event broadcasting, queue payloads
  - React/Vue: check `localStorage`/`sessionStorage` usage, analytics SDK initialization, form data handling, state management stores
  - Go: check logging libraries, context propagation of user data, HTTP middleware that captures request data
  - Swift: check `Info.plist` usage descriptions, Keychain vs UserDefaults, ATT framework, privacy manifest (`PrivacyInfo.xcprivacy`)
- **Don't flag framework defaults**: If the framework handles privacy correctly by default (e.g. Laravel's `$hidden` on the User model for password/remember_token), don't flag it. Focus on developer code that exposes or mishandles personal data beyond defaults.
- The `\@` symbol appears in PHP/Blade directives, Swift attributes (`\@Environment`, `\@AppStorage`), and CSS at-rules — be aware when scanning code patterns.
