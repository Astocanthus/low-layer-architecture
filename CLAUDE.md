# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

---

## Project Overview

Architecture documentation repository for the LOW-LAYER platform. This repo contains no executable code — only Markdown documentation describing platform design, repository structure, and infrastructure architecture.

### License

This project is licensed under the **LOW-LAYER Source Available License** (proprietary). See [LICENSE](LICENSE).

---

## Repository Contents

| Document | Purpose |
|----------|---------|
| `docs/platform.md` | Platform infrastructure architecture: tenant hierarchy, organisations, VPC environments, service bastion, resource modes, service catalog |
| `docs/infrastructure.md` | Low-level infrastructure: bare metal bootstrap, Fedora CoreOS images, OpenBao PKI, operator/agent model, recursive deployment |
| `docs/repositories.md` | Multi-repository structure: all LOW-LAYER repos, their tech stacks, inter-dependencies, versioning, CI/CD |
| `docs/console/architecture.md` | Console core module (auth, HTTP, ApiService, BaseRepository), shared Zod schemas, features (dashboard, organizations, catalog), MSW mock system |
| `docs/console/graph-editor.md` | Rete.js v2 graph editor: topology model, typed socket/pin system, connection validation, relation editor, auto-chain, sub-graph editor |

---

## Documentation Organization

### Directory Structure

```
docs/
├── platform.md              # Platform-wide: tenants, orgs, environments, service catalog
├── infrastructure.md        # Low-level: bare metal, CoreOS, operator, agent
├── repositories.md          # Multi-repo overview: all repos, CI/CD, versioning, domains
└── console/                 # Console application (Angular)
    ├── architecture.md      # Core modules, auth, features, MSW
    └── graph-editor.md      # Graph editor subsystem
```

### Organization Principles

1. **Top-level docs** (`docs/*.md`) cover platform-wide concerns that span multiple repositories or describe the overall system design.
2. **Application directories** (`docs/<app-name>/`) group documentation specific to a single application or repository. Each application gets its own subdirectory.
3. **Within an application directory**, split documents by subsystem when the content would exceed ~500 lines in a single file. One document per major subsystem or architectural concern.

### Adding Documentation for a New Application

When a new application joins the LOW-LAYER platform (e.g., `low-layer-monitoring`):

1. **Create a subdirectory**: `docs/<app-name>/` (e.g., `docs/monitoring/`)
2. **Create `architecture.md`** as the main entry point: application purpose, tech stack, project structure (tree), core module design, feature overview
3. **Split large subsystems** into separate files (e.g., `docs/monitoring/alerting.md`). A subsystem warrants its own file when it has 3+ major sections and would add >200 lines to the main architecture doc
4. **Update `docs/repositories.md`**: add a short summary entry (5-10 lines) under the numbered repository list, with links to the detailed docs in the subdirectory
5. **Update this file** (`CLAUDE.md`): add entries to the Repository Contents table
6. **Update `README.md`**: add the new files to the Project Structure tree and a bullet in the Features section

### Writing Conventions

- **Title**: Each file starts with a `# Title` (H1). Only one H1 per file.
- **Sections**: Use `## Section` (H2) for major sections, `### Subsection` (H3) for subsections. Keep heading depth at H4 max.
- **Horizontal rules**: Use `---` between major H2 sections for visual separation.
- **Code blocks**: Use fenced code blocks with language hints (` ```typescript `, ` ```bash `, ` ```hcl `, etc.). Use plain ` ``` ` for ASCII diagrams and tree structures.
- **Tables**: Prefer tables for structured comparisons (tech stacks, API endpoints, config values). Keep columns aligned.
- **Cross-references**: Use relative links between docs (e.g., `[Graph Editor](graph-editor.md)`, `[Platform](../platform.md)`). Always link from summary sections to detailed docs.
- **Diagrams**: Use ASCII art or Mermaid syntax. Prefer ASCII for simple trees and flows; use Mermaid for dependency graphs.
- **Tone**: Technical, concise, present tense. Describe what the system **does**, not what it **will do** (unless explicitly marked as draft/future).

---

## Conventions

- All documentation is written in **English**
- Architecture diagrams use ASCII art or Mermaid syntax
- Changes to architecture documents should be reflected in `CHANGELOG.md`
- Follow [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format for version history
