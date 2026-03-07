# ichbinkalt-specs

Specification repository for the ichbinkalt project.

This repository manages specs across all services (frontend, backend, infra, etc.).

## Prerequisites

[Spec-kit](https://github.com/speckit) must be installed in the **root of your workspace directory** (i.e. the directory containing this repo). The `.github/prompts/` speckit prompt workflows and the `.specify/` tooling directory live there, not inside this repository.

## Structure

- `<workspace-root>/.github/prompts/` — Speckit AI prompt workflows for authoring and refining specs
- `<workspace-root>/.specify/` — Speckit tooling: scripts, templates, and memory
- `ichbinkalt-specs/` — this repository; contains per-feature spec branches/worktrees
