# CLAUDE.md — chaching-docs

Product documentation for the ChaChing platform. Built with Redocly.

## Structure

**Top-level directories:**

- `Developer Guide/` — API documentation and webhook guides
- `Using Chaching/` — User-facing feature documentation (Hosted Page, etc.)
- `Users/` — User management and account documentation
- `@theme/` — Redocly theme customizations
- `_partials/` — Reusable markdown fragments
- `images/` — Screenshots, diagrams, and visual assets
- `settings/` — Configuration files for documentation rendering

**Top-level files:**

- `index.md` — Documentation homepage
- `onboarding.md` — Getting started guide
- `createAccount.md` — Account creation guide
- `dashboardOverview.md` — Dashboard UI walkthrough
- `entities.md` — Core entity definitions
- `migrateSubscriptions.md` — Subscription migration guide
- `support&Communication.md` — Support and communication channels
- `redocly.yaml` — Redocly build configuration
- `sidebars.yaml` — Navigation sidebar configuration

## Conventions

- Markdown content (.md files) for all documentation
- Keep developer docs in sync with API changes in cc-backend
- Add or update docs for every user-facing feature before it ships
- Images go in `images/` with descriptive filenames
- Use `Developer Guide/` for API and integration documentation
- Use `Using Chaching/` for end-user feature guides
- Update `redocly.yaml` navbar when adding new top-level documentation pages
- Use `_partials/` for shared content across multiple pages
