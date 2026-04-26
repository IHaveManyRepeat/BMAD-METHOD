# Bug Analysis Registry

This directory stores archived bug analysis documents from BMAD-METHOD projects.

## Structure

```
bug-analysis-registry/
├── index.md          ← Auto-generated index of all bug analyses
├── README.md         ← This file
├── workflow-flow/    ← Workflow logic bugs
├── missing-validation/ ← Missing validation/gate bugs
├── state-management/ ← State management bugs
├── api-contract/     ← API contract violation bugs
├── performance/      ← Performance bugs
├── security/         ← Security bugs
├── ui-ux/            ← UI/UX bugs
├── integration/      ← Integration bugs
└── other/            ← Uncategorized bugs
```

## Usage

Bug analyses are archived automatically by `bmad-analyze-bug` workflow and `tools/bug-archive.js`.

Each entry follows the naming convention: `YYYYMMDD-NNN-slug-title.md`

- `YYYYMMDD` — Date of archival
- `NNN` — Auto-incremented version number (001, 002, ...)
- `slug-title` — URL-safe short title

## Unified Archive

This directory is part of the unified BMAD-METHOD archive. It lives inside the `_bmad/experience/` submodule alongside `experience/` and `verification-reports/`.
