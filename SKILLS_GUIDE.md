# Skills Guide

This is the canonical guide to how skills work in this setup: what the two kinds
are, what is in the library right now, how to wire them into a new project, and
how they stay backed up. It lives in `claude-templates` and is the source of
truth. Reread the Quick Reference whenever you start a project; read the rest
when something feels unclear.

---

## 1. Quick reference: starting a new project

The compressed checklist. The full version with the "why" is section 4.

1. Make the new local repo and its GitHub remote as you normally would.
2. Invocable skills: nothing to do. `codebase-review` is already global on this
   machine (symlinked into `~/.claude/skills/`), so it works in every repo.
3. Create `.claude/skills/` in the new project.
4. Copy the RELEVANT reference docs from this library into it, flat (no category
   folders). Pick only what the project uses: a Next.js app takes `nextjs.md`,
   `react-patterns.md`, etc.; a Python API skips those and takes `python-api.md`.
5. Add a "Skill Routing Rules" table to the project's `CLAUDE.md` that maps task
   types to the docs you copied (shape in section 4c; legalOS is the worked
   example).
6. Any project-specific pin or override (a version note, a local decision
   reference) stays in the project's copy, not here.

---

## 2. The two kinds of skills

There are two separate systems. They are both called "skills," which is a little
confusing, so here is the plain distinction.

**Invocable skills** are folders under `invocable/`, each with a `SKILL.md`
inside, symlinked into `~/.claude/skills/` so they are global to this machine
(available in every repo, not per-project). You run one by asking for it in
plain language, for example "run a codebase review." The skill carries its own
full instructions inside `SKILL.md`; you do not have to explain the procedure,
you just name it and it takes over.

**Reference docs** are the category `.md` files (`backend/`, `security/`,
`frontend/`, and so on). These are not run. They are knowledge Claude Code
consults while it works: a project's `CLAUDE.md` has a routing table that says
"before touching the database, read `supabase.md` and `database-patterns.md`,"
and Claude reads them as needed. They are wired per-project by copying the
relevant subset into that project's `.claude/skills/`.

Why two systems: a procedure you actively run (an audit, a migration ritual) is
different from background knowledge you want consulted automatically while doing
ordinary work. Invocable skills are the first; reference docs are the second.

---

## 3. The current library

Kept accurate by convention (update this section in the same commit whenever
skills are added or removed).

### Invocable skills (`invocable/`, symlinked global)

- **codebase-review** - a structured four-pass read-only audit (backend,
  frontend, architecture, tests and docs) that finds real bugs, security holes,
  architecture drift, and test/doc gaps, then fixes worst-first. Run it by
  asking for a codebase review, audit, or pre-launch health check.

### Reference docs (26, across 7 categories)

AI Integration (4)
- **anthropic-api.md** - Claude API: model choice, tool use, streaming, prompt caching, token and cost math.
- **mcp-development.md** - Building MCP servers (FastMCP and the TypeScript SDK): tools, resources, testing, deployment.
- **model-abstraction.md** - Provider-agnostic LLM layer: unified interface, a DB-backed model registry with code fallback, cost normalization.
- **prompt-engineering.md** - Prompt patterns: system-prompt architecture, structured output, injection defense, evaluation.

Backend (3)
- **database-patterns.md** - Postgres schema and query reference: indexing, migrations, JSONB, batched rollups (avoiding N+1), audit trails.
- **python-api.md** - FastAPI reference: project structure, settings, dependency injection, async, error handling, testing.
- **supabase.md** - Supabase patterns: RLS, auth across client/SSR/JWT, realtime, edge functions, storage, CLI workflows, gotchas.

Design (3)
- **responsive-design.md** - Responsive layout: breakpoints, fluid type, mobile-first, container queries.
- **ui-patterns.md** - Reusable UI component and interaction patterns.
- **ux-writing.md** - Microcopy: error states, empty states, honest product voice.

DevOps (3)
- **ci-cd.md** - CI/CD pipelines: PR checks, deploy gates, GitHub Actions.
- **environment-management.md** - Env vars and secrets across dev/preview/prod, with fail-closed defaults.
- **vercel-deployment.md** - Vercel build config, preview deploys, env vars, domains.

Frontend (4)
- **nextjs.md** - Next.js App Router: server and client components, server actions, routing, rendering strategies.
- **react-patterns.md** - React component architecture: hooks, effect cleanup, state, composition.
- **tailwind.md** - Tailwind conventions: design tokens, utility patterns, theming.
- **web-accessibility.md** - WCAG 2.1 AA: forms, keyboard navigation, modals, semantic structure.

Product Ops (4)
- **analytics.md** - Privacy-preserving analytics: event design, no-PII capture, dashboards.
- **cost-tracking.md** - AI and infrastructure cost management: per-call token logging, budgets, ROI framing.
- **eval-framework.md** - AI quality evals: golden datasets, LLM-as-judge, regression gates, DB-backed editable cases.
- **observability.md** - Logging, tracing, monitoring, and audit surfaces designed for redaction from day one.

Security (5)
- **api-security.md** - Endpoint security: authorization per route, input validation, rate limits, CORS.
- **backend-security.md** - Server-side security: secret handling, service-role usage, no-PII logging.
- **database-security.md** - RLS design and testing, least-privilege roles, migration safety.
- **frontend-security.md** - CSP, XSS prevention, cookie handling, client-storage hygiene.
- **infra-security.md** - Deploy and infrastructure hardening: security headers, container config, least privilege.

### In the library but not yet used in a project (8)

These exist here but have not been copied into any project's `.claude/skills/`
yet. They are ready when a fitting project comes along: **mcp-development.md**,
**model-abstraction.md**, **python-api.md**, **ci-cd.md**, **analytics.md**,
**cost-tracking.md**, **eval-framework.md**, **observability.md**. (legalOS, a
Next.js and Supabase app, simply had no use for the Python API doc, the
MCP-server-building doc, and so on, so they stayed in the library.)

---

## 4. Starting a new project (the full ritual)

### a. New repo, new remote (tight refresher)

You know this part. Make the local repo, make its GitHub remote, connect them:

```
git init
git remote add origin https://github.com/<you>/<project>.git
git push -u origin main
```

### b. Invocable vs reference docs: what you actually do

The explicit answer to the recurring question:

- **Invocable skills: nothing to do.** They are already global on this machine.
  `codebase-review` works in the new repo the moment it exists, because
  `~/.claude/skills/` is symlinked to this library. You never copy an invocable
  skill into a project.
- **Reference docs: yes, still a manual copy.** Copy the RELEVANT subset from
  this library into the project's `.claude/skills/` (flat, no category folders).
  This copy is deliberate, not laziness we never automated: a Python API does not
  want `nextjs.md` in its head, and a marketing site does not want
  `database-security.md`. The manual pick IS the curation. You are choosing the
  knowledge this specific project should carry, and a smaller, relevant set makes
  the routing table honest and keeps Claude focused.

### c. Wire the project's CLAUDE.md routing table

Copying the docs is half of it; the project's `CLAUDE.md` has to point at them so
they get consulted. Add a "Skill Routing Rules" table mapping task types to the
docs you copied. The shape (from legalOS, the worked example):

```
| Task Type | Read First | Examples |
|---|---|---|
| Any frontend work | nextjs.md + react-patterns.md + tailwind.md | Components, pages, layouts, styling |
| Any backend/API work | api-security.md + backend-security.md | Route handlers, server actions |
| Any database work | supabase.md + database-patterns.md + database-security.md | Schema, migrations, RLS |
```

The left column is the trigger, the middle column names the copied docs to read
first, the right column is examples so the match is obvious. legalOS's full table
covers frontend, backend, database, auth, AI/prompt, deployment, analytics, and
more. Cover the task types the project actually has; skip the rest.

### d. Project-local additions stay local

If a project needs a note that is true only for it, keep it in the project's copy,
not here. The example: legalOS pins its `nextjs.md` copy with a Next.js 16 header
(a version pin plus a reference to its own DECISION_LOG) that the portable
`nextjs.md` deliberately does not carry, because this library stays version-general
(it targets Next.js 14+). Project-specific truth lives in the project; portable
truth lives here.

---

## 5. Using skills day to day

Very little manual effort once things are wired.

- **Invocable skills: just ask by name.** "Run a codebase review." The skill
  carries its own instructions, so you do not brief it; you name it and it runs.
- **Reference docs: near-zero effort once wired.** With the routing table in
  `CLAUDE.md`, Claude Code consults the relevant doc as it works. You do not paste
  them or remind it each time. The only time you intervene is if a doc is clearly
  being ignored on a task it should cover, in which case you point at it directly
  ("check `supabase.md` before writing this policy"). That is the exception, not
  the routine.

---

## 6. What syncs automatically vs manually

The honest version, because "it is in the repo folder" and "it is safely backed
up" are not the same thing.

| Change | Lands on disk | Backed up (GitHub) |
|---|---|---|
| Edit an invocable skill (SKILL.md) | Instantly, via the symlink | Only after you commit and push in claude-templates |
| Improve a reference doc inside a project | In the project's copy only | Only after you copy the portable part back here and push |
| Anything | Local disk | Nothing reaches any cloud except by `git push` |

Two things to internalize:

- **Automatic-on-disk is not automatic-on-GitHub.** Editing an invocable skill
  updates the repo folder immediately (the symlink points straight at it), but it
  is not backed up until you commit and push in `claude-templates`. Check
  `git status` in this repo occasionally. The cautionary example: four reference
  docs sat improved-but-uncommitted in this repo for weeks before this guide was
  written. On disk, invisible to git backup.
- **Reference-doc improvements need a deliberate copy-back.** When project work
  makes a general reference doc better, copy the PORTABLE part back into this
  library and push (the sync convention). The project-specific bits stay in the
  project. Nothing about this is automatic; it is a habit.

Nothing syncs to any cloud except via `git push` in this repo.

---

## 7. Growing the library

When a lesson is worth carrying into the NEXT project, add it here. First decide
which kind it is:

- **A procedure you run** (an audit, a migration ritual, a release checklist you
  invoke by name) becomes an **invocable skill**. Create
  `invocable/<name>/SKILL.md`, write its instructions so it is self-contained
  (it must work with only "run <name>" as the prompt), then symlink it into your
  skills directory: `ln -s ~/Projects/claude-templates/invocable/<name> ~/.claude/skills/<name>`.
  Commit and push. Add it to section 3.
- **Knowledge to consult** (patterns, gotchas, conventions for a technology)
  becomes a **reference doc**. Add or extend the `.md` in the fitting category
  folder, bump its version and date, commit and push. Add it to section 3, and to
  a project's routing table when a project uses it.

Rule of thumb: if you would want it read automatically while working, it is a
reference doc. If you would want to trigger it deliberately, it is invocable.

---

## 8. Disaster recovery / new machine

Everything is git-backed in this repo, so recovery is two commands.

```
git clone https://github.com/steveantini/claude-templates.git ~/Projects/claude-templates
ln -s ~/Projects/claude-templates/invocable/codebase-review ~/.claude/skills/codebase-review
```

The first clones the whole library (reference docs and invocable skills) from
GitHub. The second re-links the invocable skill into your global skills directory
so the Skill tool finds it. Repeat the `ln -s` line for each invocable skill you
have (today there is one). Reference docs need nothing extra here; you copy them
into projects as you start them (section 4).

---

## 9. Keeping this document current

This markdown, in `claude-templates`, is the canonical version. Whenever skills
are added or removed, or a convention changes, ask Claude Code to update this
file in the SAME commit as the change, so the guide never drifts from reality.

If you want a Word copy to read or share, ask Claude (in chat) to generate a
`.docx` from the current repo version of this file. That Word file is always a
snapshot of a moment; this file is the living source. When they disagree, this
one is right, and the `.docx` is stale.
