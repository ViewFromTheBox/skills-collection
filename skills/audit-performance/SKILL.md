---
name: audit-performance
description: Deep performance audit that scans the codebase for code-level and architectural performance issues across web and mobile stacks. Generates a detailed report and files grouped issues using the bug template with performance labels and the Awaiting Review milestone. Auto-detects stack (Laravel, React, Vue, SwiftUI, UIKit, Go, etc.) and platform (GitHub/GitLab).
context: fork
---

# Performance Audit

Deep audit of the codebase for performance issues at both the code level and architectural level. Generates a detailed report and files one issue per performance category in the remote repository.

## The Core Problem

Performance problems compound silently — an N+1 query here, a missing cache there, an unbounded loop somewhere else. By the time they surface as user-visible slowness, they're entangled across the codebase. This skill performs a systematic static analysis of code patterns and architectural decisions, then turns every finding into a tracked issue.

## Phase 1: Discover the Codebase

1. **Identify the tech stack**:
   - Scan for file types and framework indicators:
     - `composer.json` with `laravel/framework` → Laravel (check for Eloquent, Blade, Livewire, Inertia)
     - `.jsx`, `.tsx` → React
     - `.vue` → Vue
     - `.swift` + SwiftUI imports → SwiftUI
     - `.swift` + UIKit imports → UIKit
     - `go.mod` → Go
     - `tailwind.config.*` → Tailwind CSS
     - `vite.config.*`, `webpack.config.*` → JS build pipeline
     - `docker-compose.yml`, `Dockerfile` → Containerized app
   - A project may contain multiple stacks (e.g. Laravel + Inertia + React + Tailwind)

2. **Map the architecture**:
   - Identify database layer (Eloquent, raw SQL, DBAL, CoreData, SQLite)
   - Identify caching layer (Redis, Memcached, file cache, application-level caching)
   - Identify queue/job system (Laravel queues, Horizon, Sidekiq, etc.)
   - Identify API layer (REST controllers, GraphQL, API resources)
   - Identify frontend build system (Vite, Webpack, asset pipeline)
   - Identify external service integrations (HTTP clients, SDKs, third-party APIs)

3. **Detect platform** (for issue creation):
   - Parse `git remote get-url origin`: `github.com` → GitHub, `gitlab.com` or self-hosted → GitLab
   - Confirm via `.github/` or `.gitlab/` directory presence
   - Verify CLI availability: `gh` (GitHub) or `glab` (GitLab)

## Phase 2: Database & Query Performance

### 2a: ORM / Query Patterns

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **N+1 queries** | Critical | Accessing relationships in loops without eager loading. Eloquent: `foreach ($posts as $post) { $post->author->name }` without `with('author')`. |
| **Missing eager loading** | Critical | Controller/service methods that load models with relationships accessed in views/serializers but no `with()`, `load()`, or `$with` property |
| **Unbounded queries** | Critical | `Model::all()`, `DB::table()->get()`, or queries without `limit()`/`paginate()` on tables that could grow large |
| **Select * patterns** | Major | Queries not using `select()` to limit columns, especially on wide tables or when serializing to JSON |
| **Missing where clauses on updates/deletes** | Critical | `Model::update()` or `Model::delete()` patterns that could affect more rows than intended |
| **Raw queries in loops** | Critical | `DB::select()` or `DB::statement()` inside foreach/while loops |
| **Repeated identical queries** | Major | Same query executed multiple times in a single request cycle (should be cached or deduplicated) |
| **Chunking large datasets** | Major | Processing large result sets without `chunk()`, `lazy()`, or `cursor()` |

### 2b: Database Schema & Indexing

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing indexes on foreign keys** | Major | Migration files creating foreign key columns without corresponding index (Laravel auto-indexes `foreignId()` but not manual `unsignedBigInteger` + `foreign()`) |
| **Missing indexes on queried columns** | Major | Columns used in `where()`, `orderBy()`, `groupBy()` that don't have indexes (cross-reference migrations with query patterns) |
| **Missing composite indexes** | Minor | Queries filtering on multiple columns that would benefit from a composite index |
| **Expensive column types** | Minor | `TEXT`/`BLOB` columns that could be `VARCHAR`, JSON columns queried with `whereJsonContains()` without generated columns |
| **Missing soft delete index** | Minor | Models using `SoftDeletes` without an index on `deleted_at` |

### 2c: Go / Non-ORM Database Patterns

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unclosed rows** | Critical | `sql.Query()` / `sql.QueryRow()` results not closed with `rows.Close()` (resource leak) |
| **Connection pool misconfiguration** | Major | Missing `SetMaxOpenConns()`, `SetMaxIdleConns()`, `SetConnMaxLifetime()` on `sql.DB` |
| **Query building via string concatenation** | Critical | SQL injection risk AND performance issue (no prepared statement reuse) |
| **Missing context propagation** | Major | Database calls without `context.Context` parameter (can't timeout/cancel) |

## Phase 3: Backend Performance

### 3a: PHP / Laravel Specific

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing caching** | Major | Expensive computations or external API calls without `Cache::remember()` or similar caching |
| **Config/route not cached** | Major | Production deployments without `config:cache`, `route:cache`, `view:cache`, `event:cache` |
| **Synchronous work in requests** | Major | Email sending, file processing, API calls, or heavy computation that should be dispatched to a queue job |
| **Missing queue usage** | Major | Operations that take > 1 second blocking the HTTP response instead of using queue jobs |
| **Inefficient collection operations** | Major | Using `Collection` methods where query-level operations would be better (`$users->filter()` vs `User::where()`) |
| **Overly broad model events** | Minor | Model observers/events triggering expensive operations on every save/update |
| **Missing API resource pagination** | Critical | API endpoints returning unbounded collections without pagination |
| **Session driver** | Minor | Using `file` or `database` session driver in production (should use `redis` or `memcached`) |
| **Redundant middleware** | Minor | Middleware stacks with unnecessary layers on API routes (e.g. session, CSRF on stateless API) |
| **Debug mode in production** | Critical | `APP_DEBUG=true` patterns, debug bars, telescope in production without proper gating |

### 3b: Go Specific

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Goroutine leaks** | Critical | Goroutines started without cancellation context or done channel, unbuffered channels without readers |
| **Mutex contention** | Major | `sync.Mutex` protecting large critical sections, missing `sync.RWMutex` for read-heavy workloads |
| **Unnecessary allocations** | Major | String concatenation in loops (use `strings.Builder`), `append()` without pre-allocated capacity, `[]byte` ↔ `string` conversions in hot paths |
| **Missing context timeout** | Major | HTTP handlers or external calls without `context.WithTimeout()` or `context.WithDeadline()` |
| **Inefficient JSON handling** | Minor | `encoding/json` in hot paths without considering `json-iterator` or code generation, unmarshaling into `interface{}` |
| **Unshared `regexp.Compile`** | Major | `regexp.Compile()` or `regexp.MustCompile()` called inside functions instead of package-level `var` |

### 3c: General Backend

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unbounded loops** | Critical | Loops without break conditions that depend on external data, while(true) patterns without exits |
| **String concatenation in loops** | Major | Building strings via `+=` or `.` in loops instead of using builders/buffers |
| **Synchronous HTTP calls** | Major | External API calls that block request handling without timeouts, retries, or circuit breakers |
| **Missing timeouts on external calls** | Critical | HTTP clients, database connections, or external service calls without configured timeouts |
| **Large file operations in memory** | Major | Reading entire files into memory (`file_get_contents`, `ioutil.ReadAll`) for large files instead of streaming |
| **Missing rate limiting** | Major | API endpoints without rate limiting that could be abused |
| **Logging in hot paths** | Minor | Verbose logging inside tight loops or frequently called functions |

## Phase 4: Frontend Performance

### 4a: JavaScript / React / Vue

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Bundle size** | Major | Importing entire libraries when only specific modules are needed (e.g. `import _ from 'lodash'` vs `import debounce from 'lodash/debounce'`) |
| **Missing code splitting** | Major | All routes loaded eagerly, no `React.lazy()` / dynamic `import()` / Vue async components |
| **Unoptimized re-renders** | Major | React: missing `useMemo`, `useCallback`, `React.memo` on expensive components. Vue: missing `computed` for derived state, unnecessary watchers. |
| **Memory leaks** | Critical | Event listeners, intervals, subscriptions not cleaned up in `useEffect` cleanup / `onUnmounted` |
| **Large list rendering** | Major | Rendering hundreds/thousands of DOM elements without virtualization (`react-window`, `vue-virtual-scroller`) |
| **Unkeyed list items** | Major | `v-for` without `:key`, `.map()` without `key` prop — forces full re-render |
| **Blocking main thread** | Critical | Heavy computation in render cycle, synchronous loops processing large datasets on the UI thread |
| **Unoptimized images** | Major | Images without `loading="lazy"`, missing `width`/`height` attributes (layout shift), uncompressed formats |
| **Missing debounce/throttle** | Major | Scroll, resize, input handlers firing on every event without debouncing |
| **Excessive watchers** | Minor | Vue: many `watch()` calls that could be consolidated or replaced with `computed` |

### 4b: CSS / Tailwind

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Unused CSS** | Minor | Large stylesheets without PurgeCSS/Tailwind purge configured for production |
| **Expensive selectors** | Minor | Deeply nested selectors, universal selectors (`*`), attribute selectors in hot render paths |
| **Layout thrashing patterns** | Major | JavaScript reading layout properties (offsetHeight, getBoundingClientRect) then writing styles in the same frame |
| **Missing will-change** | Minor | Frequently animated elements without `will-change` hints for browser optimization |
| **Excessive animations** | Minor | CSS animations on many elements simultaneously without `\@media (prefers-reduced-motion)` check or GPU-composited properties only (transform, opacity) |
| **Web font loading** | Minor | Fonts without `font-display: swap` or `optional`, render-blocking font loads |

### 4c: Asset Pipeline

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing minification** | Major | Build config without minification for production (Vite handles this by default, but check Webpack configs) |
| **Missing compression** | Major | No gzip/brotli compression configuration in server or build |
| **Unoptimized source maps** | Minor | Full source maps shipped in production builds |
| **Missing tree shaking** | Major | CommonJS imports preventing tree shaking, `sideEffects` not configured in package.json |
| **Missing CDN hints** | Minor | No `<link rel="preconnect">` for third-party domains, no `<link rel="preload">` for critical assets |

## Phase 5: Mobile Performance (Swift)

### 5a: SwiftUI

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Excessive view redraws** | Critical | `\@State`, `\@ObservedObject`, `\@EnvironmentObject` triggering unnecessary view updates, missing `EquatableView` |
| **Heavy computation in body** | Critical | Expensive operations (sorting, filtering, network calls) directly in `var body` instead of `onAppear` or cached properties |
| **Missing lazy containers** | Major | `VStack`/`HStack` with many children that should use `LazyVStack`/`LazyHStack` |
| **Image loading** | Major | Loading large images without `.resizable()` and proper sizing, missing `AsyncImage` for remote images |
| **Missing task cancellation** | Major | `.task { }` blocks without checking `Task.isCancelled` for long-running work |

### 5b: UIKit

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Main thread violations** | Critical | Network calls, file I/O, or heavy computation on the main thread (not dispatched to background queue) |
| **Missing cell reuse** | Critical | `UITableView`/`UICollectionView` not using `dequeueReusableCell`, creating new cells in `cellForRowAt` |
| **Image caching** | Major | Downloading images without caching (missing `NSCache` or library like Kingfisher/SDWebImage) |
| **Retain cycles** | Critical | Closures capturing `self` strongly in async callbacks, delegates not declared as `weak` |
| **Auto Layout performance** | Major | Complex constraint hierarchies that cause layout thrashing, constraints modified in tight loops |

## Phase 6: Architecture-Level Performance

| Check | Severity | What to Look For |
|-------|----------|------------------|
| **Missing caching strategy** | Major | No Redis/Memcached layer for frequently accessed data, no HTTP cache headers on API responses |
| **Missing CDN for static assets** | Minor | Static files served directly from app server without CDN configuration |
| **Missing database connection pooling** | Major | New database connections per request instead of persistent connections or connection pool |
| **Missing queue for async work** | Major | Background tasks (emails, reports, webhooks) processed synchronously in HTTP requests |
| **Missing pagination everywhere** | Critical | List endpoints without pagination support (both API and database query level) |
| **Monolithic responses** | Major | API endpoints returning deeply nested full objects when clients only need a subset |
| **Missing health/readiness probes** | Minor | Containerized apps without health check endpoints for orchestrator load balancing |
| **N+1 at the API level** | Major | Frontend making multiple sequential API calls that could be batched or combined into a single endpoint |
| **Missing request deduplication** | Minor | Same API request fired multiple times on page load (race conditions, missing state management) |

## Phase 7: Compile Findings

1. **Deduplicate**: If the same pattern appears across multiple instances of a shared module, group them.
2. **Classify each finding**:
   - **Severity**: Critical / Major / Minor
     - **Critical**: Causes outages, data loss, or severe degradation under normal load (N+1 in loops, unbounded queries, memory leaks, goroutine leaks, main thread blocking)
     - **Major**: Noticeable performance degradation, scales poorly (missing caching, no eager loading, bundle bloat, missing pagination, synchronous external calls)
     - **Minor**: Best practice improvement, marginal gains (selector optimization, logging in hot paths, missing CDN hints, font loading)
   - **Category**: Database, Backend, Frontend, Mobile, Architecture
   - **Affected files**: List of file paths and line numbers
   - **Estimated impact**: Brief note on how this affects response time, memory, CPU, or user experience
   - **Remediation**: Specific recommendation for how to fix it

3. **Group by performance rule/category**: All instances of the same issue type go into a single group. This becomes one issue.

## Phase 8: Generate Report

Create `PERFORMANCE_AUDIT.md` in the project root with the following structure:

```markdown
# Performance Audit Report

**Date**: {date}
**Stack**: {detected stack}

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
**Area**: {Database / Backend / Frontend / Mobile / Architecture}

{Description of the issue and why it causes performance problems}

**Estimated impact**: {e.g. "Each N+1 adds ~50ms per loop iteration; at 100 items this adds 5 seconds to page load"}

**Affected locations**:
- `{file}:{line}` — {brief context}
- `{file}:{line}` — {brief context}

**Recommended fix**: {specific remediation steps with code examples where helpful}

---

## Major Findings
{same structure}

## Minor Findings
{same structure}
```

## Phase 9: File Issues

### 9a: Setup

1. **Verify the performance label exists** in the repo:
   - GitHub: `gh label list --search "performance"`
   - GitLab: `glab label list --search "performance"`
   - If it doesn't exist, **create it automatically**:
     - GitHub: `gh label create "performance" --color "E86D0A" --description "Performance issue"`
     - GitLab: `glab label create "performance" --color "#E86D0A" --description "Performance issue"`

2. **Locate the "Awaiting Review" milestone**:
   - GitHub: `gh api repos/{owner}/{repo}/milestones --jq '.[] | select(.title=="Awaiting Review") | .number'`
   - GitLab: `glab api projects/{id}/milestones --jq '.[] | select(.title=="Awaiting Review") | .id'`
   - If not found, warn the user and file issues without a milestone.

3. **Locate the bug issue template**:
   - GitHub: `.github/ISSUE_TEMPLATE/` — look for a bug report template (YAML or Markdown)
   - GitLab: `.gitlab/issue_templates/` — look for a bug report template
   - If no template exists, use a sensible default format.

### 9b: Create Issues

Create **one issue per performance rule/category violated**:

- **GitHub**:
  ```bash
  gh issue create \
    --title "{title}" \
    --body "{body}" \
    --label "performance" \
    --label "bug" \
    --milestone "Awaiting Review"
  ```

- **GitLab**:
  ```bash
  glab issue create \
    --title "{title}" \
    --description "{body}" \
    --label "performance,bug" \
    --milestone "Awaiting Review"
  ```

Each issue should contain:

- **Title**: `Perf: {Category} — {short description}`
  - Example: `Perf: Database — N+1 queries in PostController index`
  - Example: `Perf: Frontend — Missing code splitting on route components`
  - Example: `Perf: SwiftUI — Heavy computation in view body`
- **Body** (using the bug template structure):
  - **Severity**: Critical / Major / Minor
  - **Area**: Database / Backend / Frontend / Mobile / Architecture
  - **Description**: What the issue is and why it degrades performance
  - **Estimated impact**: How this affects response time, memory, CPU, or user experience
  - **Affected locations**: List of files and line numbers with brief context for each
  - **Steps to reproduce**: How to observe the performance issue (e.g. "Load the posts index page with 100+ posts and observe query count" or "Profile the settings screen with Instruments and observe main thread usage")
  - **Expected behavior**: What the performance should look like after the fix
  - **Recommended fix**: Specific code-level remediation steps with examples
  - **Resources**: Links to relevant documentation (Laravel optimization guide, React performance docs, Apple Instruments guide, etc.)

### 9c: Report Summary

After all issues are created, report:
- Total findings by severity and area
- Number of issues created (with links)
- Link to the `PERFORMANCE_AUDIT.md` report
- Top 3 highest-impact areas to address first (prioritized by estimated user impact)

## Important Notes

- **Static analysis only** — this skill reads source code, it does not profile running applications. Estimated impact is based on pattern recognition, not measured timings. Flag issues with confident recommendations but note that profiling should confirm priorities.
- **Component-aware**: If a performance issue is in a shared service, repository, or component, note that fixing it fixes all callers. Prioritize shared code fixes in the report.
- **Framework idioms**: Respect framework-specific patterns:
  - Laravel: check Eloquent relationships, scopes, caching facades, queue dispatch, config caching
  - React: check hooks dependencies, memo usage, context provider placement, suspense boundaries
  - Vue: check computed vs methods vs watchers, `v-once`, `v-memo`, `shallowRef`
  - Go: check goroutine lifecycle, channel usage, sync primitives, context propagation
  - Swift: check async/await, MainActor, lazy properties, Instruments-friendly patterns
- **Don't flag framework defaults**: If the framework handles optimization internally (e.g. Vite's default minification, Laravel's query builder parameterization), don't flag it. Focus on developer code that undermines or misses the framework's built-in optimizations.
- The `\@` symbol appears in PHP/Blade directives, Swift attributes (`\@MainActor`, `\@State`, `\@Environment`), CSS at-rules (`\@media`), and Java annotations — be aware when scanning code patterns.
