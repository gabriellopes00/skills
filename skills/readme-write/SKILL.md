---
name: readme-write
description: Creates or revises README files for software projects, covering project-type detection, templates (library, CLI, web app, API), section structure, onboarding flow, examples, badges, contribution guidance, writing style, and advanced GitHub Flavored Markdown. Use when creating or revising a README, project documentation, or getting-started guides.
---

# README Write

Produce a README that helps visitors decide quickly whether to use the project and how to get started. Combines project analysis and templates with onboarding-first structure, style rules, and a final checklist.

## Goal

A good README answers, in order:

1. What is this project?
2. Why should I use it?
3. How do I run it right now?
4. How do I configure common cases?
5. How do I contribute?

## Workflow

1. **Identify audience and primary use case.** Developers want technical details and API references; end users want features, benefits, and screenshots; contributors want architecture, setup, and testing.
2. **Analyze the project** (see below) to collect name, description, language/framework, features, and dependencies.
3. **Select a template** by project type (see below; full templates in [templates.md](references/templates.md)).
4. **Write a short, value-first opening**: title, one-line/one-paragraph description, optional badges.
5. **Add a runnable quickstart** with copy-pastable commands: installation first, then usage.
6. **Add usage examples** for the 1–3 most common tasks.
7. **Add configuration/reference sections** only after core onboarding is complete.
8. **Add project-specific content** (features, API reference, examples, support, etc.).
9. **Add contributor guidance** or link to `CONTRIBUTING.md`, plus the license section.
10. **Revise** any dense prose for clarity and concision, then run the [checklist](references/checklist.md).

## Step: Analyze the project

Detect project type:

```bash
ls package.json && echo "Node.js project" || \
ls setup.py pyproject.toml && echo "Python project" || \
ls go.mod && echo "Go project"
```

Gather:
- Project name (from `package.json`, `pyproject.toml`, etc.)
- Description (from manifest or git)
- Main language and framework
- Key features (scan source files)
- Dependencies (from manifest files)

## Step: Select a template

| Type | Template | Key Sections |
|------|----------|--------------|
| Library | library | Installation, API, Examples |
| CLI Tool | cli | Installation, Commands, Options |
| Web App | webapp | Features, Setup, Deployment |
| API | api | Endpoints, Authentication, Examples |

Templates for all four are in [references/templates.md](references/templates.md).

## Core sections

**Title and description:**
```markdown
# Project Name

Brief one-line description of what the project does.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](package.json)
```

**Installation:**
````markdown
## Installation

```bash
npm install project-name
# or
pip install project-name
```
````

**Usage:**
````markdown
## Usage

```javascript
const project = require('project-name');

// Basic example
project.doSomething();
```
````

**Common badges:**
```markdown
![Build Status](https://github.com/user/repo/workflows/CI/badge.svg)
![Coverage](https://codecov.io/gh/user/repo/branch/main/graph/badge.svg)
![npm version](https://badge.fury.io/js/package-name.svg)
```

## Section structure

Default order (adapt as needed):

1. Project name (one H1)
2. Short value proposition
3. Features / capabilities
4. Installation
5. Quickstart / usage
6. Configuration (if applicable)
7. Development / testing
8. Contributing
9. License

Classification of sections:

- **Essential (all projects):** Title and Description, Installation, Quick Start / Usage, License
- **Recommended:** Features (what makes it useful), Documentation (link to full docs), Examples (common use cases/real-world), Contributing (how to help), Support (where to get help)
- **Optional:** Requirements (system dependencies), Configuration (setup options), Troubleshooting (common issues), Changelog (recent changes), Acknowledgments (credits)
- **Project-specific:** API Reference (libraries), Configuration (configurable tools), Deployment (web apps), Endpoints/Authentication (APIs)

## Style constraints

- Prefer concrete examples over abstract claims.
- Keep setup commands in fenced code blocks, never embedded in paragraphs.
- Keep each section focused on one user question.
- Avoid burying setup steps deep in prose; keep installation steps together, not split across distant sections.
- Use relative links for in-repo docs.
- Link to full docs rather than duplicating them in the README.
- Be clear, concise, active voice; short paragraphs (2–4 sentences, one idea each); use lists, tables, and headings.
- Make code examples runnable, show expected output, include error handling.
- Use consistent terminology and inline code formatting.
- Use descriptive link text and alt text for images.

Detailed rules and good/bad examples: [references/best-practices.md](references/best-practices.md).

## Advanced GitHub Flavored Markdown

Use these where they add genuine value (never decoratively): `<kbd>` key caps, `<details>`/`<summary>` collapsibles, Mermaid diagrams, GeoJSON/TopoJSON maps, STL 3D models, SVG `<foreignObject>` CSS animations with `<picture>` dark/light variants, color swatches, and alerts (`> [!NOTE]`, one or two per README max). Syntax and caveats: [references/gfm-features.md](references/gfm-features.md).

## Output expectations

When using this skill for a user task:

1. Return the revised README content.
2. Summarize what changed in the onboarding flow.
3. Note any missing information that requires user input (for example, deployment steps or support policy).

## References

- [Templates](references/templates.md): README templates by project type (library, CLI, web app, API)
- [Best Practices](references/best-practices.md): writing style, structure, code examples, formatting, maintenance, accessibility
- [Checklist](references/checklist.md): must-have / should-have items, anti-patterns, automated-check guardrails
- [GFM Features](references/gfm-features.md): advanced GitHub Flavored Markdown
