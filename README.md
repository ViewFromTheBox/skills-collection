# Skills Collection

A curated collection of custom skills for AI-assisted development workflows, code auditing, issue management, documentation, and more.

Currently built for [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code/overview) (Anthropic's command-line tool for agentic coding) and [Claude.ai](https://claude.ai) via the [skills feature](https://docs.anthropic.com/en/docs/build-with-claude/claude-code/skills). Each skill is a self-contained folder with a `SKILL.md` file that tells the AI how to perform a specific task.

## What Are Skills?

Skills are markdown instruction files that teach Claude how to perform repeatable tasks. When you add them to your Claude environment, they're automatically loaded when relevant — so you can say things like "work on issue #42" or "audit this codebase for security vulnerabilities" and Claude knows exactly what to do.

Each skill lives in its own folder with a `SKILL.md` file containing:

- **YAML frontmatter** with a `name` and `description` (used for matching your request to the right skill)
- **Step-by-step instructions** that Claude follows to complete the task

## Skills

### Code Auditing

| Skill | Description |
| --- | --- |
| [audit-accessibility](skills/audit-accessibility/) | Deep accessibility audit against WCAG 2.2 AA for web and Apple HIG for mobile projects |
| [audit-performance](skills/audit-performance/) | Deep performance audit for code-level and architectural issues across web and mobile stacks |
| [audit-privacy](skills/audit-privacy/) | Scans for data collection, storage, sharing, and retention issues |
| [audit-security](skills/audit-security/) | Deep security audit for vulnerabilities, insecure patterns, and misconfigurations |

### Issue & PR Workflow

| Skill | Description |
| --- | --- |
| [create-issue](skills/create-issue/) | Creates a single issue from a plan, markdown file, or prompt (GitHub & GitLab) |
| [create-issues](skills/create-issues/) | Batch creates issues from a spec, plan, or markdown file(s) (GitHub & GitLab) |
| [work-issue](skills/work-issue/) | Takes an issue ID, creates a branch, implements it, runs tests, pushes, and opens a draft PR/MR |
| [update-issue](skills/update-issue/) | Updates a linked issue with progress comments, label changes, and task list check-offs |
| [prepare-pull-request](skills/prepare-pull-request/) | Lints, tests, checks docs, and opens a draft PR on GitHub |
| [prepare-merge-request](skills/prepare-merge-request/) | Lints, tests, checks docs, and opens a draft MR (GitHub & GitLab) |
| [prepare-release](skills/prepare-release/) | Prepares a release branch: lint, test, verify docs, audit deps, create release PR/MR |

### CI/CD & Pipelines

| Skill | Description |
| --- | --- |
| [gitlab-pipeline](skills/gitlab-pipeline/) | Diagnoses and fixes failed GitLab CI/CD pipelines (up to 5 fix attempts) |

### Framework Updates

| Skill | Description |
| --- | --- |
| [update-laravel](skills/update-laravel/) | Compares current Laravel version against latest, scans for breaking changes via the upgrade guide, then auto-PRs a clean update or files individual issues for each breaking change (GitHub & GitLab) |
| [update-livewire](skills/update-livewire/) | Compares current Livewire version against latest, scans for breaking changes via the upgrade guide, then auto-PRs a clean update or files individual issues for each breaking change (GitHub & GitLab) |

### Documentation

| Skill | Description |
| --- | --- |
| [inline-documentation](skills/inline-documentation/) | Creates inline docs using WordPress standards for PHP/JS, language-standard conventions for others |
| [update-documentation](skills/update-documentation/) | Updates all docs in `docs/` based on current codebase state |

### Planning & Bug Reporting

| Skill | Description |
| --- | --- |
| [new-feature](skills/new-feature/) | Runs an interview process to plan a feature, then creates a detailed issue |
| [report-bug](skills/report-bug/) | Runs an interview process to report and plan a bug fix, then creates an issue |

## Skills From Others

These are skills built by other developers that I use and recommend. They're not included in this repo, but are worth checking out.

<!-- Add links to skills you use from others here -->
<!-- Example: -->
<!-- | Skill | Description | Source | -->
<!-- | --- | --- | --- | -->
<!-- | [skill-name](https://github.com/...) | What it does | [@author](https://github.com/...) | -->

## Installation

### For Claude.ai

1. Go to **Settings** → **Skills** in Claude.ai
2. Upload individual skill folders or point to this repository

### For Claude Code

Copy the skills you want into your Claude Code skills directory:

```bash
# Clone this repo
git clone https://github.com/ViewFromTheBox/skills-collection.git

# Copy all skills
cp -r skills-collection/skills/* ~/.claude/skills/

# Or copy individual skills
cp -r skills-collection/skills/work-issue ~/.claude/skills/
```

### Per-Project Usage

You can also drop skills into a specific project's `.claude/skills/` directory for project-scoped availability:

```bash
cp -r skills-collection/skills/audit-security /path/to/your/project/.claude/skills/
```

## Repo Structure

```
skills-collection/
├── README.md
├── LICENSE
└── skills/
    ├── audit-accessibility/
    │   └── SKILL.md
    ├── audit-performance/
    │   └── SKILL.md
    ├── create-issue/
    │   └── SKILL.md
    ├── work-issue/
    │   └── SKILL.md
    └── ... (18 skills total)
```

Each skill is self-contained in its own folder. Some skills may include additional supporting files alongside `SKILL.md`.

## Writing Your Own Skills

If you want to create your own skills, the format is straightforward:

1. Create a folder with a descriptive name (e.g. `my-skill/`)
2. Add a `SKILL.md` with YAML frontmatter and step-by-step instructions:

```markdown
---
name: my-skill
description: A short description of what this skill does and when it should trigger.
---

# My Skill

Instructions that Claude will follow when this skill is activated.

## Step 1: Do something

1. First, do this
2. Then, do that
```

The `description` field is especially important — it's what Claude uses to decide when to activate the skill based on your request.

## Contributing

Found a bug in a skill? Have an improvement? Contributions are welcome. Please open an issue or submit a pull request.

## About

Built and maintained by [Jacob Martella](https://github.com/ViewFromTheBox), a Laravel developer building open-source tools for the web. These skills are part of my daily development workflow and power projects like [ArtisanPack UI](https://github.com/ArtisanPack-UI).

## License

This project is open-sourced software licensed under the [MIT License](LICENSE).