---
name: audit-accessibility
description: Deep accessibility audit that scans the codebase against WCAG 2.2 AA (configurable) for web projects and Apple HIG for mobile projects. Generates a detailed report and files grouped issues using the bug template with accessibility labels and the Awaiting Review milestone. Auto-detects stack (Blade, React, Vue, SwiftUI, UIKit) and platform (GitHub/GitLab).
context: fork
---

# Accessibility Audit

Deep audit of the codebase for accessibility violations. Generates a detailed report and files one issue per violated criterion in the remote repository.

## The Core Problem

Accessibility issues silently ship into production because there's no systematic audit step. Manual checks are inconsistent, and findings don't get tracked as actionable work items. This skill performs a thorough static analysis across the full UI stack — markup, styles, JS behavior, and ARIA usage — then turns every finding into a tracked issue.

## Configuration

- **Default standard**: WCAG 2.2 AA (web) / Apple HIG Accessibility (mobile)
- **Configurable**: The user can request a different WCAG level (A, AA, AAA) or version (2.0, 2.1, 2.2)
- **Severity tiers**: Critical (blocks users entirely), Major (significant barrier), Minor (best practice / enhancement)
- **Each finding also notes its WCAG criterion** (e.g. 1.1.1 Non-text Content) or HIG guideline reference

## Phase 1: Discover the Codebase

1. **Identify the tech stack**:
   - Scan for file types and framework indicators:
     - `.blade.php` → Laravel/Blade
     - `.jsx`, `.tsx` → React
     - `.vue` → Vue SFC
     - `.html` → Static HTML
     - `.swift` + SwiftUI imports → SwiftUI
     - `.swift` + UIKit imports → UIKit
     - `.css`, `.scss`, `tailwind.config.*` → CSS/Tailwind
   - A project may contain multiple stacks (e.g. Laravel + Blade + Tailwind + Alpine)

2. **Map the UI surface area**:
   - For web: inventory all templates, components, layouts, and pages
   - For mobile: inventory all views, view controllers, and SwiftUI scenes
   - Identify shared/reusable components vs. one-off pages
   - Note any existing accessibility utilities (sr-only classes, a11y helpers, accessibility modifiers)

3. **Detect platform** (for issue creation):
   - Parse `git remote get-url origin`: `github.com` → GitHub, `gitlab.com` or self-hosted → GitLab
   - Confirm via `.github/` or `.gitlab/` directory presence
   - Verify CLI availability: `gh` (GitHub) or `glab` (GitLab)

## Phase 2: Web Accessibility Audit

Run this phase for any web-rendering stack found (Blade, React, Vue, HTML).

### 2a: Semantic HTML & Structure

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Heading hierarchy** | 1.3.1 Info and Relationships | Skipped heading levels (h1 → h3), multiple h1s per page, headings used for styling only |
| **Landmark regions** | 1.3.1, 2.4.1 | Missing `<main>`, `<nav>`, `<header>`, `<footer>`, or equivalent ARIA roles. Pages without landmarks. |
| **Lists** | 1.3.1 | Navigation items not in `<ul>`/`<ol>`, definition lists misused |
| **Tables** | 1.3.1 | Data tables missing `<th>`, `scope`, or `<caption>`. Layout tables used for non-tabular content. |
| **Language** | 3.1.1, 3.1.2 | Missing `lang` attribute on `<html>`, missing `lang` on foreign-language passages |
| **Page titles** | 2.4.2 | Missing or generic `<title>` elements, SPA routes without title updates |

### 2b: Images & Non-text Content

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Alt text** | 1.1.1 Non-text Content | `<img>` without `alt`, decorative images missing `alt=""` or `role="presentation"`, meaningful images with empty/generic alt |
| **SVG accessibility** | 1.1.1 | `<svg>` without `<title>`, `aria-label`, or `aria-labelledby`. Inline SVG icons without text alternatives. |
| **Background images with meaning** | 1.1.1 | CSS `background-image` used for meaningful content without text alternative |
| **`<canvas>` fallback** | 1.1.1 | `<canvas>` elements without fallback content or ARIA descriptions |
| **Icon fonts** | 1.1.1 | Icon elements (e.g. `<i class="fa-*">`) without `aria-label` or `aria-hidden="true"` |

### 2c: Forms & Interactive Elements

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Form labels** | 1.3.1, 3.3.2 | `<input>`, `<select>`, `<textarea>` without associated `<label>` (via `for`/`id` or wrapping), missing `aria-label` or `aria-labelledby` |
| **Error identification** | 3.3.1 | Form errors not programmatically associated with fields, missing `aria-describedby` for error messages, missing `aria-invalid` |
| **Required fields** | 3.3.2 | Required fields without `required` attribute or `aria-required="true"`, visual-only required indicators (asterisk without programmatic marking) |
| **Autocomplete** | 1.3.5 | Identity fields (name, email, address, phone, cc) missing `autocomplete` attributes |
| **Button labeling** | 4.1.2 | `<button>` with only an icon and no accessible name, `<a>` styled as button without `role="button"` |
| **Custom controls** | 4.1.2 | Custom dropdowns, toggles, tabs, modals missing appropriate ARIA roles, states, and properties |

### 2d: Keyboard & Focus Management

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Focus visibility** | 2.4.7, 2.4.11 | CSS `outline: none` or `outline: 0` without replacement focus indicator, missing `focus-visible` styles |
| **Tab order** | 2.4.3 | Positive `tabindex` values (> 0), interactive elements with `tabindex="-1"` that should be focusable |
| **Keyboard traps** | 2.1.2 | Modals/dialogs without focus trapping and escape key handling, custom components that capture focus |
| **Skip links** | 2.4.1 | Missing "skip to content" link, skip link not the first focusable element |
| **Click handlers without keyboard** | 2.1.1 | `onClick` on non-interactive elements (`<div>`, `<span>`) without `onKeyDown`/`onKeyUp`, `role`, and `tabindex` |
| **Mouse-only interactions** | 2.5.1 | Drag-and-drop without keyboard alternative, hover-only tooltips without focus trigger |

### 2e: Color & Visual Design

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Color contrast (text)** | 1.4.3 | Hardcoded color values in CSS/Tailwind that may not meet 4.5:1 ratio (normal text) or 3:1 (large text). Flag suspicious combinations for manual review. |
| **Color contrast (UI)** | 1.4.11 | Interactive component borders, icons, and graphical elements that may not meet 3:1 ratio against background |
| **Color as sole indicator** | 1.4.1 | Error states, required fields, active states, or status indicators using only color (no icon, text, or pattern) |
| **Text resizing** | 1.4.4 | Font sizes in `px` instead of `rem`/`em`, containers with fixed heights that may clip enlarged text |
| **Reflow** | 1.4.10 | Fixed-width layouts, horizontal scrolling at 320px viewport, content overflow issues |

### 2f: CSS/Tailwind Specific

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Reduced motion** | 2.3.1, 2.3.3 | CSS animations/transitions without `\@media (prefers-reduced-motion: reduce)` alternative |
| **Screen reader utilities** | — | Available `sr-only` class but not used where needed (icon-only buttons, etc.) |
| **Focus styles in Tailwind** | 2.4.7 | Missing `focus:` or `focus-visible:` variants on interactive elements |
| **Hidden content** | — | `display: none` or `hidden` on content that should be screen-reader accessible (use `sr-only` instead), or `aria-hidden="true"` on content that should be visible |
| **Print styles** | — | Critical content hidden in print stylesheets |

### 2g: Dynamic Content & JavaScript

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Live regions** | 4.1.3 | Dynamic content updates (AJAX, SPA route changes, toast notifications) without `aria-live`, `role="alert"`, or `role="status"` |
| **SPA navigation** | 2.4.2, 4.1.3 | Client-side route changes without focus management or page title updates |
| **Timeout warnings** | 2.2.1 | Session timeouts without user warning or extension option |
| **Auto-playing media** | 1.4.2 | Audio/video that auto-plays without pause/stop/mute controls |
| **Alpine.js / Livewire** | 4.1.2 | Dynamic Livewire/Alpine components that add/remove DOM without managing focus or announcing changes |

### 2h: ARIA Usage Quality

| Check | WCAG Criterion | What to Look For |
|-------|---------------|------------------|
| **Invalid ARIA** | 4.1.2 | Nonexistent ARIA attributes, invalid `role` values, `aria-*` on elements that don't support them |
| **Redundant ARIA** | — | `role="button"` on `<button>`, `role="link"` on `<a>`, `aria-label` duplicating visible text |
| **Missing required ARIA** | 4.1.2 | Roles without required properties (e.g. `role="slider"` without `aria-valuenow`, `role="combobox"` without `aria-expanded`) |
| **ARIA hidden misuse** | — | `aria-hidden="true"` on focusable elements, hiding content that sighted users can see and interact with |

## Phase 3: Mobile Accessibility Audit (Apple HIG)

Run this phase for SwiftUI or UIKit projects.

### 3a: VoiceOver Support

| Check | Guideline | What to Look For |
|-------|-----------|------------------|
| **Accessibility labels** | HIG: Accessibility | Views and controls without `.accessibilityLabel()` (SwiftUI) or `accessibilityLabel` (UIKit) |
| **Accessibility hints** | HIG: Accessibility | Interactive elements without `.accessibilityHint()` describing the result of the action |
| **Accessibility traits** | HIG: Accessibility | Missing `.accessibilityAddTraits()` — buttons not marked as `.isButton`, headers not marked as `.isHeader` |
| **Accessibility value** | HIG: Accessibility | Sliders, progress bars, steppers without `.accessibilityValue()` |
| **Grouping** | HIG: Accessibility | Related elements not grouped with `.accessibilityElement(children: .combine)` or `shouldGroupAccessibilityChildren` |
| **Ordering** | HIG: Accessibility | Illogical VoiceOver traversal order, missing `.accessibilitySortPriority()` |
| **Custom actions** | HIG: Accessibility | Swipe actions or context menus without `.accessibilityCustomAction()` alternatives |

### 3b: Visual Accessibility

| Check | Guideline | What to Look For |
|-------|-----------|------------------|
| **Dynamic Type** | HIG: Typography | Hardcoded font sizes instead of `.font(.body)` or `UIFont.preferredFont(forTextStyle:)`, missing `\@ScaledMetric` for custom dimensions |
| **Color contrast** | HIG: Color | Custom colors that may not meet contrast ratios, missing dark mode alternatives |
| **Color as sole indicator** | HIG: Color | Status or state communicated only through color without shape/icon/text |
| **Bold text support** | HIG: Accessibility | Not responding to `UIAccessibility.isBoldTextEnabled` or `\@Environment(\.legibilityWeight)` |
| **Reduce motion** | HIG: Motion | Animations without checking `UIAccessibility.isReduceMotionEnabled` or `\@Environment(\.accessibilityReduceMotion)` |
| **Reduce transparency** | HIG: Accessibility | Blur effects without checking `UIAccessibility.isReduceTransparencyEnabled` |

### 3c: Interaction Accessibility

| Check | Guideline | What to Look For |
|-------|-----------|------------------|
| **Tap target size** | HIG: Layout | Touch targets smaller than 44×44 points, buttons with `frame()` modifiers setting small sizes |
| **Custom gestures** | HIG: Gestures | Custom gestures without accessibility alternatives |
| **Timeout** | HIG: Accessibility | Timed interactions without accessibility accommodations |
| **Focus management** | HIG: Accessibility | Modal presentations not moving VoiceOver focus, missing `.accessibilityFocused()` binding |

## Phase 4: Compile Findings

1. **Deduplicate**: If the same violation appears across multiple instances of a shared component, group them.
2. **Classify each finding**:
   - **Severity**: Critical / Major / Minor
     - **Critical**: Users with disabilities cannot access core functionality (missing form labels, keyboard traps, no alt text on informational images, zero focus indication)
     - **Major**: Significant barrier but workaround exists (heading hierarchy broken, missing landmarks, poor contrast, missing live regions)
     - **Minor**: Best practice / enhancement (redundant ARIA, px font sizes, missing autocomplete, suboptimal VoiceOver ordering)
   - **WCAG Criterion** (web) or **HIG Guideline** (mobile) reference
   - **Affected files**: List of file paths and line numbers
   - **Remediation**: Specific recommendation for how to fix it

3. **Group by WCAG criterion / HIG guideline**: All instances of the same violated rule go into a single group. This becomes one issue.

## Phase 5: Generate Report

Create `ACCESSIBILITY_AUDIT.md` in the project root with the following structure:

```markdown
# Accessibility Audit Report

**Date**: {date}
**Standard**: WCAG 2.2 AA / Apple HIG Accessibility
**Stack**: {detected stack}

## Summary

| Severity | Count |
|----------|-------|
| Critical | {n}   |
| Major    | {n}   |
| Minor    | {n}   |
| **Total** | **{n}** |

## Critical Findings

### {WCAG Criterion / HIG Guideline}: {Title}

**Severity**: Critical
**WCAG**: {criterion number and name} | **Level**: {A/AA/AAA}

{Description of the violation}

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

## Phase 6: File Issues

### 6a: Setup

1. **Verify the accessibility label exists** in the repo:
   - GitHub: `gh label list --search "accessibility"`
   - GitLab: `glab label list --search "accessibility"`
   - If it doesn't exist, **create it automatically**:
     - GitHub: `gh label create "accessibility" --color "6A0DAD" --description "Accessibility issue"`
     - GitLab: `glab label create "accessibility" --color "#6A0DAD" --description "Accessibility issue"`

2. **Locate the "Awaiting Review" milestone**:
   - GitHub: `gh api repos/{owner}/{repo}/milestones --jq '.[] | select(.title=="Awaiting Review") | .number'`
   - GitLab: `glab api projects/{id}/milestones --jq '.[] | select(.title=="Awaiting Review") | .id'`
   - If not found, warn the user and file issues without a milestone.

3. **Locate the bug issue template**:
   - GitHub: `.github/ISSUE_TEMPLATE/` — look for a bug report template (YAML or Markdown)
   - GitLab: `.gitlab/issue_templates/` — look for a bug report template
   - If no template exists, use a sensible default format.

### 6b: Create Issues

Create **one issue per violated WCAG criterion / HIG guideline**:

- **GitHub**:
  ```bash
  gh issue create \
    --title "{title}" \
    --body "{body}" \
    --label "accessibility" \
    --label "bug" \
    --milestone "Awaiting Review"
  ```

- **GitLab**:
  ```bash
  glab issue create \
    --title "{title}" \
    --description "{body}" \
    --label "accessibility,bug" \
    --milestone "Awaiting Review"
  ```

Each issue should contain:

- **Title**: `A11y: {WCAG Criterion / HIG Guideline} — {short description}`
  - Example: `A11y: WCAG 1.1.1 — Images missing alt text`
  - Example: `A11y: HIG VoiceOver — Missing accessibility labels on navigation controls`
- **Body** (using the bug template structure):
  - **Severity**: Critical / Major / Minor
  - **Standard reference**: WCAG criterion number, name, and level (or HIG guideline)
  - **Description**: What the violation is and why it matters for users
  - **Affected locations**: List of files and line numbers with brief context for each
  - **Steps to reproduce**: How to observe the issue (e.g. "Navigate with keyboard to the login form" or "Enable VoiceOver and swipe through the settings screen")
  - **Expected behavior**: What should happen for accessibility compliance
  - **Recommended fix**: Specific code-level remediation steps
  - **Resources**: Link to relevant WCAG understanding doc or Apple HIG section

### 6c: Report Summary

After all issues are created, report:
- Total findings by severity
- Number of issues created (with links)
- Link to the `ACCESSIBILITY_AUDIT.md` report
- Top 3 highest-impact areas to address first

## Important Notes

- **Static analysis only** — this skill reads source code, it does not render pages or run a browser. Some checks (like actual computed color contrast) require manual verification. Flag suspicious color combinations for review rather than making definitive pass/fail claims.
- **Component-aware**: If a violation is in a shared component, note that fixing the component fixes all instances. Prioritize shared component fixes in the report.
- **Framework idioms**: Respect framework-specific patterns:
  - React: check JSX `aria-*` props, `htmlFor` instead of `for`, `className` patterns
  - Vue: check template `v-bind` ARIA, `:aria-label`, dynamic attributes
  - Blade: check `{!! !!}` raw output for missing accessibility attributes, `\@livewire` components
  - SwiftUI: check modifier chains for `.accessibility*()` modifiers
  - UIKit: check `IBOutlet` connections and programmatic `accessibility*` property assignments
- **Don't flag what's already handled**: If the project uses an accessibility-aware component library (e.g. Headless UI, Radix, shadcn/ui), don't flag issues that the library handles internally. Focus on how the library components are used (correct props, labels, etc.).
- The `\@` symbol appears in Blade directives (`\@livewire`), CSS at-rules (`\@media`), and Swift attributes (`\@Environment`, `\@ScaledMetric`) — be aware when scanning code patterns.
