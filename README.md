# BMad Experience Library

Cross-project experience library for the BMad Method framework. Accumulates lessons learned, best practices, patterns, and reference cases across all BMad-managed projects.

## Structure

```
experience/
├── openapi/                    # OpenAPI generation experiences
│   ├── _index.md
│   └── *.md                    # Individual experience entries
├── architecture/               # Architecture experiences
│   ├── patterns/               # Architecture pattern cases
│   ├── decisions/              # Architecture Decision Records (ADR)
│   └── pitfalls/               # Architecture pitfall records
└── ux-design/                  # UX design experiences
    ├── tokens/                 # Design tokens
    ├── style-guides/           # Design style guides
    ├── reference-cases/        # Design reference cases
    └── pitfalls/               # UX design pitfall records
```

## How It Works

### Automatic Loading

When running BMad workflows (OpenAPI, Architecture, UX Design), relevant experiences are automatically discovered and loaded during initialization steps. The discovery algorithm:

1. Scans experience files in the relevant category
2. Scores entries by matching tags, tech stack, applicability, and recency
3. Presents top matches to inform current work

### Manual Saving

After completing a workflow, you'll be prompted to save lessons learned. You can also manually save experiences anytime using:

- **`bmad-save-experience`** — Save a specific lesson learned or experience entry
- **`bmad-experience-library`** — Browse, search, and manage the experience library

## Integration with Projects

Other projects reference this library via git submodule:

```bash
git submodule add -b main https://github.com/IHaveManyRepeat/BMAD-METHOD.git _bmad/experience
```

The experience library will be available at `{project-root}/_bmad/experience/`.

## Experience Entry Format

Each experience entry is a Markdown file with YAML frontmatter:

```yaml
---
type: {type}              # Experience type (see schemas below)
title: "Descriptive Title"
date: YYYY-MM-DD
tags: [relevant, tags]
category: pitfall | best-practice | pattern | tool-tip | decision | tokens | style-guide | reference-case
---
```

Detailed frontmatter schemas vary by category. See individual `_index.md` files for category-specific schemas.

## Contributing

1. Create a new `.md` file with proper frontmatter in the appropriate category
2. Follow the naming convention: `YYYY-MM-DD-{short-slug}.md`
3. Commit and push to the BMAD-METHOD repository
4. Projects can update via `git submodule update --remote`
