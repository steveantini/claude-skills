# [PROJECT NAME]: CLAUDE.md

<!--
  TEMPLATE: Copy this file into your project root and fill in the sections below.
  Remove these HTML comments once customized.
  This file is read by Claude Code at the start of every session.
-->

## Project Overview

<!-- One paragraph: what is this product/project? What problem does it solve? -->

### Current Phase

<!-- What phase of development is the project in? What are the immediate priorities? -->

---

## Content Ownership Rules

<!--
  OPTIONAL but recommended when a non-engineer (founder, client, partner) authors
  text the assistant will see: business docs, decks, brand copy, transcripts.
  State plainly, before any technical convention:
  - Which files or text are owner-authored and must never be reworded, polished,
    or corrected (including typos and shorthand) unless the owner asks in that session.
  - Which files the assistant drafted and may freely adjust.
  - Which files live OUTSIDE the repo (with the relative path) and must never be
    opened, moved, copied, or committed. Back this with .gitignore patterns.
  - Where the plain-English explanation of anything technical must go.
  Put this section first after the overview so it is read before any coding rule.
-->

---
## Engineering Standard

<!--
  RECOMMENDED. Keep as-is or tighten; do not soften. This is the bar every change is held to.
-->

Everything built in this repo is built to the most professional, cutting-edge, architecturally sound, and scalable standard, and is delightful for both the user and the maintainer. The visible design may start simple; the foundation never does. Every choice should be one we would still be proud of when this project is the public face of the business. No shortcuts that would need to be undone later. When there is a cheap way and a right way, take the right way and document why.

---

## Architecture

### Tech Stack

| Layer | Technology | Deployment |
|---|---|---|
| Frontend | <!-- e.g., Next.js (TypeScript) --> | <!-- e.g., Vercel --> |
| Backend | <!-- e.g., Python (FastAPI) --> | <!-- e.g., Railway --> |
| Database | <!-- e.g., Supabase (PostgreSQL) --> | <!-- e.g., Supabase Cloud --> |
| AI/ML | <!-- e.g., Claude (Anthropic API) --> | <!-- e.g., Via backend --> |
| Auth | <!-- e.g., Supabase Auth --> | <!-- --> |

### Directory Structure

<!--
  Document the project's directory layout with brief annotations.
  Example:
  ```
  project/
  ├── src/          # Frontend source
  ├── backend/      # API server
  └── docs/         # Documentation
  ```
-->

### Data Flow

<!--
  Describe the primary data flow through the system.
  Example: User Input → Frontend → Backend API → AI/Database → Response → Frontend
  Note any critical constraints (e.g., "frontend never calls external APIs directly").
-->

---

## Coding Conventions

### [Primary Language] (Frontend)

<!--
  Cover: naming conventions, file organization, import ordering,
  server vs. client component rules (if applicable), error handling patterns.
-->

### [Secondary Language] (Backend)

<!--
  Cover: naming conventions, type hints/annotations, async patterns,
  data validation approach, error handling.
-->

### Git

- **Commit format:** `type: description`
- **Types:** `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `security`
- **Branch naming:** `feature/description`, `fix/description`, `chore/description`

---

## Environment & Configuration

<!--
  Cover: environment variable naming convention, secrets management approach,
  development setup instructions, environment separation strategy.
-->

- **Environment variables:** `UPPER_SNAKE_CASE` with appropriate prefixes
- **Secrets:** Never hardcoded, never committed. `.env.local` for dev, hosting provider for prod
- **Environments:** Separate credentials for dev / staging / production

---

## Testing

<!--
  Cover: test file locations, what must be tested, what can be tested lightly,
  naming conventions.
-->

---

## Security Non-Negotiables

<!--
  List hard security requirements that apply to every change.
  Examples:
  - Input sanitization on every endpoint
  - API keys never in frontend code or git
  - Rate limiting on all API endpoints
  - CORS locked to known origins
-->

---

## AI Integration Rules

<!--
  If the project uses AI/LLM APIs, document:
  - Where API calls are made (backend only?)
  - Prompt template management
  - Output validation requirements
  - Cost tracking approach
  - Rate limiting strategy
-->

---

## What Not to Do

<!--
  List explicit anti-patterns and things to never do in this project.
  Be specific. These prevent the most common mistakes.
-->

---

## Documentation Rules

When making any product change (new feature, renamed component, new page, architectural change, design system change, or brand update), the following files must be checked and updated if affected:

- **README.md**: Project overview, setup instructions, feature list, routes, project structure
- **Product vision doc**: Product vision, brand architecture, messaging (project-specific file name)
- **Product outline**: Roadmap, technical architecture, phase status (project-specific file name)
- **Decision log**: If the change involves a decision worth recording (project-specific file name). A recorded decision is never edited after acceptance except to change its Status line; a changed decision gets a new entry that supersedes the old one
- **CLAUDE.md**: New conventions, components, architecture changes
- **CHANGELOG.md**: What was built, what was fixed, date
- **.env.example**: Any new environment variables

This is not optional. Documentation updates are part of the definition of done for every change.

At the end of every phase or significant feature completion, sync generalized lessons back to your portable template library (wherever you maintain it locally, such as your own fork of these templates). Extract the universal principle, not the project-specific detail. If a new rule or convention is added to this project's CLAUDE.md, evaluate whether it belongs in your template CLAUDE.md as well.

---

## Skill Routing Rules (Mandatory)

<!--
  CUSTOMIZE: Update the task-to-skill mappings below based on which skill files
  exist in your project's .claude/skills/ directory. Remove rows for skills you
  don't use and add rows for any project-specific skills.
-->

Before performing any of the following types of work, you MUST read the specified skill file(s) in `.claude/skills/` and follow their conventions. Do not rely on memory from previous sessions; re-read the skill file every time.

| Task Type | Read First | Examples |
|---|---|---|
| Any frontend work | `nextjs-frontend.md` | Components, pages, layouts, styling, Tailwind |
| Any backend work | `python-backend.md` | API endpoints, middleware, data models |
| Any database work | `supabase.md` | Schema changes, migrations, queries, RLS policies |
| Any AI/prompt work | `anthropic-api.md` + `prompt-engineering.md` | System prompts, Claude API calls, structured output |
| Any security-related work | `security.md` | Auth, rate limiting, input validation, CORS, secrets |
| Any deployment work | `vercel-deployment.md` | Build config, environment variables, preview deploys |
| Any analytics or logging work | `product-ops.md` | Metrics, tracing, cost tracking, dashboards |
| Any work touching multiple domains | Read ALL relevant skills | Full-stack features, new phases |

If you are unsure whether a skill applies, read it anyway. It is always better to over-consult than to miss a convention.

If a task requires a skill that does not exist in `.claude/skills/` yet but exists in your local templates library, copy it into the project first, then read it.

---

## Skill Template Sync Convention

At the end of every phase or sub-phase, after completing the standard documentation updates above, you **MUST** also sync any new patterns, lessons, or gotchas discovered during that phase to the portable skill templates at `<your-templates-library>/skills/`.

### Sync Process

1. **Review** every project-specific skill file in `.claude/skills/` that was referenced or updated during this phase
2. **Extract the generalized principle.** Strip out project-specific details (project names, specific API keys, specific database tables, specific design tokens) and keep the universal best practice
3. **Update** the corresponding portable template skill file with the generalized version
4. **Create new templates.** If a new skill file was created for this project that doesn't have a portable template equivalent yet, create one in the appropriate subdirectory of `<your-templates-library>/skills/`
5. **Version bump.** Add a version bump and "Last updated" date to any modified template skill files

### Example

If during a phase you discover that a database's row-level security has an edge case where service role keys bypass policies on joined tables, the project skill gets the specific fix. The portable template gets: "When using admin/service role keys that bypass row-level security, verify behavior on queries involving joins, as RLS policies may not propagate across joined tables as expected."

**This is not optional.** No phase is complete until both project-specific skills AND portable templates are current.
