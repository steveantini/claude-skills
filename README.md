# Claude Code Skills

A portable library of Claude Code skills and project configuration templates. Start new projects with battle-tested conventions and Claude Code skill files instead of building project context from scratch every time.

## How this library works

**`invocable/` holds SKILL.md-format skills, symlinked into `~/.claude/skills/`.** These are the skills the Claude Code `Skill` tool discovers and runs (e.g. `codebase-review`). They are global to your account, not per-project, so linking them once makes them available in every repo on the machine. New machine (or after a disk loss): clone this repo, then symlink each invocable skill into your skills directory:

```
git clone https://github.com/steveantini/claude-skills.git ~/Projects/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/Projects/claude-skills/invocable/codebase-review ~/.claude/skills/codebase-review
```

That is the whole recovery: the skill's real bytes live in this repo, and `~/.claude/skills/` just points at them.

**The category folders (`backend/`, `security/`, `frontend/`, …) are the reference-doc master library.** These `.md` files are not invoked by the Skill tool; they are the routing-rule reference docs a project's `CLAUDE.md` points at. At the start of a new project, copy the relevant subset into that project's `.claude/skills/` (flat, no category nesting) so the project's routing table can cite them. A project takes only the docs it needs (a Python project skips the frontend set, etc.).

**The sync convention: this repo is always canonical.** When work in a project improves a reference doc or an invocable skill, copy the generalized improvement back here and push, so this library stays the most-evolved copy. Keep project-*specific* additions in the project (a version pin, a local decision reference, a path into `node_modules`) — only portable, project-agnostic lessons sync back. If a project's copy and this one disagree, this repo is the source of truth for the general guidance; the project's copy is the source of truth for its own local overrides.

**Total skills:** 25 across 7 categories (plus the invocable skills under `invocable/`)

## Skill Summary

| Category | Skills | Description |
|---|---|---|
| AI Integration | 4 | LLM APIs, MCP servers, prompt engineering, provider abstraction |
| Backend | 3 | APIs, databases, Supabase patterns |
| Design | 3 | UI/UX patterns, responsive design, microcopy |
| DevOps | 3 | CI/CD, deployment, environment management |
| Frontend | 4 | React, Next.js, Tailwind CSS, accessibility |
| Product Ops | 4 | Analytics, cost tracking, evals, observability |
| Security | 5 | API, backend, database, frontend, and infrastructure security |
| **Total** | **25** | |

---

## Skill Catalog

### AI Integration

**`anthropic-api.md`** — Comprehensive Claude API reference covering model selection, message construction, tool use, streaming, extended thinking, and structured output. Includes token counting, cost estimation formulas, rate limit handling, and the Batches API for bulk processing.

**`mcp-development.md`** — Guide to building MCP (Model Context Protocol) servers using FastMCP (Python) and the TypeScript SDK. Covers tool, resource, and prompt design patterns, testing with the MCP inspector, integration with Claude Code and Claude Desktop, and deployment strategies.

**`model-abstraction.md`** — Provider-agnostic LLM abstraction layer design. Defines a unified interface with Claude and OpenAI adapters, a provider registry, capability negotiation across models, fallback strategies for resilience, and cost normalization for cross-provider comparison.

**`prompt-engineering.md`** — Prompt engineering patterns including system prompt architecture, instruction hierarchy, structured output with XML and JSON, few-shot examples, chain-of-thought reasoning, prompt injection prevention, A/B testing methodology, template management, and evaluation frameworks.

### Backend

**`database-patterns.md`** — PostgreSQL patterns covering normalization vs. denormalization trade-offs, indexing strategy, migration management, query optimization with EXPLAIN ANALYZE, connection pooling, transaction isolation, full-text search, JSONB usage, and audit trail implementation.

**`python-api.md`** — FastAPI reference including project structure, application factory pattern, pydantic-settings configuration, request/response models, dependency injection, async patterns, structured error handling, CORS setup, health checks, API versioning, and testing with pytest.

**`supabase.md`** — Supabase-specific patterns for row-level security policies, auth integration across client-side, SSR, and JWT contexts, real-time subscriptions, edge functions, storage buckets, migration workflows, local development with the Supabase CLI, TypeScript type generation, and common gotchas.

### Design

**`responsive-design.md`** — Mobile-first responsive design covering breakpoint strategy, touch target sizing, viewport considerations, responsive image techniques, navigation patterns for different screen sizes, cross-device testing approaches, and PWA considerations.

**`ui-patterns.md`** — UI component architecture following a five-layer system: tokens, primitives, components, compositions, and templates. Covers design tokens as CSS variables, component API conventions, data tables, forms, modals, toasts, loading states, dark mode, animation, icons, and typography scale.

**`ux-writing.md`** — UX microcopy guidelines for error messages (what happened, why, what to do next), loading and empty states, confirmation dialogs, tooltips, onboarding flows, tone and voice consistency, action-oriented button labels, and effective placeholder text.

### DevOps

**`ci-cd.md`** — GitHub Actions CI/CD design covering workflow structure, composite actions, PR checks (lint, typecheck, test, build, security scan), test strategy by layer, dependency and secrets scanning, build verification, automated deployment pipelines, staging promotion gates, and workflow optimization.

**`environment-management.md`** — Multi-environment strategy across local, preview, staging, and production. Covers configuration hierarchy, environment variable management with Zod validation, database migration promotion, feature flag patterns, rollback procedures, and data seeding for non-production environments.

**`vercel-deployment.md`** — Vercel deployment reference including vercel.json configuration, environment variable scoping by deployment context, preview deployments, custom domain setup, edge functions and middleware, build optimization, caching strategy, monorepo support, GitHub integration, rollback, and CLI usage.

### Frontend

**`nextjs.md`** — Next.js App Router patterns covering file conventions, route organization, server vs. client component decision framework, data fetching strategies, server actions, route handlers, middleware, image and font optimization, metadata and SEO, environment variable exposure rules, and caching behavior summary.

**`react-patterns.md`** — React component patterns including server vs. client component boundaries, state management with useState, useReducer, and Context, custom hook extraction, error boundaries, Suspense for async UI, composition over inheritance, prop drilling avoidance, and performance optimization with memo, useMemo, useCallback, and useTransition.

**`tailwind.md`** — Tailwind CSS conventions covering utility class ordering, mobile-first responsive breakpoints, dark mode implementation, custom theme extension, component-level class organization using cva (class variance authority), the cn() merge utility, animation patterns, common anti-patterns to avoid, and differences between Tailwind v3 and v4.

**`web-accessibility.md`** — WCAG 2.1 AA compliance guide covering semantic HTML, ARIA roles and patterns, keyboard navigation, focus management, screen reader support, color contrast requirements, accessible form design, skip navigation links, live regions for dynamic content, and a testing checklist.

### Product Ops

**`analytics.md`** — Privacy-preserving analytics design covering event taxonomy, PII handling rules, self-hosted tool options (Plausible, PostHog, Umami), server-side event tracking, funnel and cohort analysis, A/B testing infrastructure, and dashboard design principles.

**`cost-tracking.md`** — AI and infrastructure cost management including per-request cost calculation, cost attribution by user, feature, model, plan, and endpoint, budget alert thresholds, optimization strategies (semantic caching, prompt compression, model routing), infrastructure cost monitoring, and unit economics tracking.

**`eval-framework.md`** — AI output quality evaluation covering eval suite design, automated evaluation methods, LLM-as-judge patterns, human evaluation workflows, bias detection, factual accuracy scoring, consistency testing, adversarial test cases, CI/CD integration for regression detection, and reporting dashboards.

**`observability.md`** — System observability across the three pillars: structured JSON logging, distributed tracing with OpenTelemetry, and metrics collection. Covers error tracking with Sentry, performance monitoring using RED and USE methods with percentile-based alerting, uptime monitoring, and log retention policies.

### Security

**`api-security.md`** — API security patterns for JWT validation, API key management (generation, hashing, scoping, rotation), OAuth2 flows with Authorization Code + PKCE, CORS configuration, rate limiting, request size limits and timeouts, API versioning security considerations, and webhook signature verification.

**`backend-security.md`** — Backend security covering authentication and authorization middleware, password hashing with Argon2id and bcrypt, session management, rate limiting by endpoint type, request validation with Zod, SQL injection prevention, file upload security, HTTP security headers, and sensitive data logging rules.

**`database-security.md`** — Database security patterns including row-level security (RLS) policy design, column-level encryption with AES-256-GCM, connection security via SSL/TLS, backup encryption, access audit logging with pgaudit, least-privilege database roles, ORM-level SQL injection defense, and PII handling and retention policies.

**`frontend-security.md`** — Frontend security covering Content Security Policy (CSP), XSS prevention with DOMPurify and output encoding, CSRF protection using synchronizer tokens and SameSite cookies, secure cookie configuration, client-side storage security, clickjacking prevention, subresource integrity (SRI), and secure form handling.

**`infra-security.md`** — Infrastructure security covering environment separation, secrets management (Vault, AWS Secrets Manager, Vercel), CI/CD pipeline security with GitHub Actions and branch protection, supply chain security, dependency vulnerability scanning (Dependabot, Snyk), pre-commit hooks with gitleaks, container security with Trivy, and DNS/SSL certificate management.

---

## How to Use

### Project Configuration

1. Copy `CLAUDE.template.md` into your new project root as `CLAUDE.md`
2. Fill in the blanks — each section has inline comments explaining what to provide
3. Claude Code reads `CLAUDE.md` automatically at session start

### Skills

1. Browse the skill catalog above to find relevant skills for your project
2. Copy the skill files you need into your project's `.claude/skills/` directory
3. Customize any project-specific details (the templates are intentionally generic)

### Skill Template Sync Convention

These templates are living documents. At the end of every phase of every project:

1. **Review** project-specific skill files that were referenced or updated
2. **Extract** generalized lessons — strip project-specific details, keep the universal best practice
3. **Update** the corresponding portable template in this repo
4. **Create** new templates when a project produces a skill with no existing counterpart
5. **Version bump** — update the version and "Last updated" date on any modified skill file

---

## Folder Structure

```
claude-skills/
├── CLAUDE.template.md              # Project configuration template
├── skills/
│   ├── ai-integration/             # LLM APIs, MCP servers, prompt engineering
│   │   ├── anthropic-api.md
│   │   ├── mcp-development.md
│   │   ├── model-abstraction.md
│   │   └── prompt-engineering.md
│   ├── backend/                    # APIs, databases, server patterns
│   │   ├── database-patterns.md
│   │   ├── python-api.md
│   │   └── supabase.md
│   ├── design/                     # UI/UX, responsive design, writing
│   │   ├── responsive-design.md
│   │   ├── ui-patterns.md
│   │   └── ux-writing.md
│   ├── devops/                     # CI/CD, deployment, environments
│   │   ├── ci-cd.md
│   │   ├── environment-management.md
│   │   └── vercel-deployment.md
│   ├── frontend/                   # React, Next.js, Tailwind, a11y
│   │   ├── nextjs.md
│   │   ├── react-patterns.md
│   │   ├── tailwind.md
│   │   └── web-accessibility.md
│   ├── product-ops/                # Analytics, cost tracking, observability
│   │   ├── analytics.md
│   │   ├── cost-tracking.md
│   │   ├── eval-framework.md
│   │   └── observability.md
│   └── security/                   # API, backend, DB, frontend, infra security
│       ├── api-security.md
│       ├── backend-security.md
│       ├── database-security.md
│       ├── frontend-security.md
│       └── infra-security.md
└── README.md
```

---

## License

MIT License. See LICENSE file for details.

This is a personal template library shared publicly. It is not accepting external contributions at this time. Feel free to fork, adapt, and use it in your own projects.

---

**Built by:** Steven J. Antini

**Last updated:** 2026-04-05
