# Vercel Deployment

| Field | Value |
|---|---|
| **Version** | 1.2.0 |
| **Last Updated** | 2026-09-07 |
| **Applicability** | Next.js, SvelteKit, Nuxt, Astro, Remix, static sites deployed to Vercel |
| **Dependencies** | Vercel CLI (`vercel`), GitHub integration, Node.js 18+ |

---

## Project Configuration

### `vercel.json` — Core Settings

```jsonc
{
  "framework": "nextjs",           // auto-detected; override if needed
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "installCommand": "npm ci",
  "regions": ["iad1"],             // deployment region(s)
  "cleanUrls": true,
  "trailingSlash": false,
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" }
      ]
    }
  ],
  "rewrites": [
    { "source": "/app/:path*", "destination": "/app" }
  ],
  "redirects": [
    { "source": "/old-path", "destination": "/new-path", "permanent": true }
  ]
}
```

### Key Config Decisions

- **Framework preset**: Let Vercel auto-detect unless overriding build behavior.
- **Root directory**: Set in project settings for monorepos (not `vercel.json`).
- **Node.js version**: Set via `engines.node` in `package.json` or project settings. Default is 20.x.
- **Package manager**: Auto-detected from lockfile. Include exactly one lockfile.

---

## Environment Variable Management

### Scoping

| Scope | When Applied |
|---|---|
| `production` | Production deployments only |
| `preview` | All preview deployments (PR branches) |
| `development` | `vercel dev` local development |

### Best Practices

- Store secrets in Vercel dashboard, never in code.
- Use `NEXT_PUBLIC_` prefix only for values safe to expose client-side.
- Sensitive keys: use Vercel's encrypted environment variables (marked "Sensitive").
- Per-branch overrides: use "Git Branch" targeting for preview env vars.
- Pull env to local: `vercel env pull .env.local`.

### Common Variables

```
DATABASE_URL          — production → connection pooler URL
NEXT_PUBLIC_API_URL   — production → https://api.example.com
NEXTAUTH_SECRET       — production → generated secret (sensitive)
NEXTAUTH_URL          — production → https://example.com
```

---

## Preview Deployments

Every push to a non-production branch creates a preview deployment.

### URL Patterns

- `https://<project>-<hash>-<team>.vercel.app` — unique per commit
- `https://<project>-git-<branch>-<team>.vercel.app` — branch alias

### Preview Configuration

- **Protected**: Enable Vercel Authentication to restrict access.
- **Comments**: Enable deployment comments on PRs for team review.
- **Preview env vars**: Separate database/API URLs pointing to staging resources.
- **Ignored builds**: Skip builds for non-code changes via `vercel.json`:

```jsonc
{
  "ignoreCommand": "git diff --quiet HEAD^ HEAD -- . ':!docs' ':!*.md'"
}
```

### PR Workflow

1. Push branch triggers preview build.
2. Vercel bot comments on PR with preview URL.
3. Team reviews at preview URL.
4. Merge triggers production deployment.

---

## Custom Domains

### Setup

1. Add domain in project settings.
2. Configure DNS: CNAME to `cname.vercel-dns.com` (subdomains) or A record to `76.76.21.21` (apex).
3. SSL auto-provisioned via Let's Encrypt.

### Domain Configuration

- **Production branch**: Assign domains to the production branch (usually `main`).
- **Redirect www**: Configure www-to-apex or apex-to-www redirect.
- **Multiple domains**: All resolve to same deployment; set one as primary.
- **Wildcard**: `*.example.com` supported on Pro/Enterprise plans.

### DNS Propagation

- CNAME changes: typically minutes.
- Nameserver delegation: up to 48 hours.
- Verify: `dig +short CNAME your-domain.com` or check Vercel dashboard.

### Redirecting www from the command line

`vercel domains add` has no redirect option. The redirect is a property of the project-domain
record, set through the API (the dashboard does the same call):

```
PATCH https://api.vercel.com/v9/projects/<project>/domains/www.example.com?teamId=<team>
{"redirect": "example.com", "redirectStatusCode": 308}
```

Add both hostnames to the project first, then patch the one that should redirect. Verify with
`curl -sI https://www.example.com/path?x=1` (expect 308 with the path and query preserved).

---

## Setting Up a Project from the CLI (gotchas)

Creating a project with `vercel project add` instead of importing it in the dashboard skips
framework detection and a few defaults. Learned the hard way; check all four before the first push.

- **Framework preset is "Other".** The build command falls back to `npm run build` and the output
  directory to `public`, which does not deploy a Next.js app correctly. Set it explicitly before
  `vercel git connect` or the first push: `vercel project update <name> --framework nextjs --yes`.
- **`vercel link` appends `.env*` to `.gitignore`.** If `.gitignore` already has `.env.*` followed by
  `!.env.example`, the appended line comes last and re-ignores `.env.example`, so the next
  `git add` of it fails silently. Diff `.gitignore` after linking and drop the added line.
- **`vercel git connect` deploys immediately** from the current default branch, and Dependabot (if
  configured) opens PRs and preview deployments within a minute of the config landing. Set the
  environment variables the app needs (`vercel env add NAME production,preview --sensitive`,
  value on stdin) and confirm any access gate is in the code before connecting, so the first
  deployment is never ungated.
- **`vercel env add` reads the value from stdin.** `printf '%s' "$VALUE" | vercel env add NAME
  production,preview --sensitive` avoids a trailing newline in the secret and keeps it out of the
  shell history.

---

## Edge Functions & Middleware

### Middleware (`middleware.ts`)

Runs before every request at the edge. Use for:

- Authentication checks / redirects
- Geo-based routing
- A/B testing via cookies
- Request header manipulation

```typescript
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const country = request.geo?.country ?? 'US';
  if (country === 'DE') {
    return NextResponse.redirect(new URL('/de', request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!api|_next/static|favicon.ico).*)'],
};
```

### Edge Functions (API Routes)

```typescript
// app/api/fast/route.ts
export const runtime = 'edge';

export async function GET(request: Request) {
  return Response.json({ fast: true });
}
```

### Edge vs. Serverless Decision

| Use Edge When | Use Serverless When |
|---|---|
| Low latency critical | Node.js APIs needed (fs, native modules) |
| Simple transformations | Database drivers requiring TCP |
| Auth/redirect logic | Heavy computation |
| Geo-routing | Large dependencies |

---

## Build Optimization

### Speed

- **Dependency caching**: Automatic. Ensure lockfile is committed.
- **Remote caching** (Turborepo): `npx turbo login && npx turbo link`.
- **Selective builds**: Use `ignoreCommand` to skip unnecessary rebuilds.
- **Output tracing**: Next.js auto-traces; avoid importing server modules in client code.

### Bundle Size

- Analyze: `ANALYZE=true next build` with `@next/bundle-analyzer`.
- Dynamic imports for heavy components: `dynamic(() => import('./HeavyChart'))`.
- Tree-shake: use named exports, avoid barrel files re-exporting everything.
- Image optimization: use `next/image` with Vercel's built-in optimizer.

### Build Output

- **ISR pages**: Pre-rendered at build, revalidated on demand.
- **Static pages**: Served from edge CDN, no function invocation.
- **Server components**: Rendered per-request or cached.

---

## Caching Strategy

### Vercel Edge Cache

```typescript
// API route caching
export async function GET() {
  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  });
}
```

### Cache Control Headers

| Directive | Purpose |
|---|---|
| `s-maxage=N` | CDN cache duration in seconds |
| `stale-while-revalidate=N` | Serve stale while refreshing in background |
| `no-store` | Never cache (auth endpoints, user data) |
| `private` | Browser cache only, not CDN |

### ISR (Incremental Static Regeneration)

```typescript
// app/posts/[id]/page.tsx
export const revalidate = 60; // seconds

// On-demand revalidation
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
export async function POST(request: Request) {
  revalidatePath('/posts');
  return Response.json({ revalidated: true });
}
```

### Data Cache

- `fetch()` responses cached by default in App Router.
- Opt out: `fetch(url, { cache: 'no-store' })`.
- Tag-based: `fetch(url, { next: { tags: ['posts'] } })`.

---

## Monorepo Support

### Turborepo Setup

```
apps/
  web/          ← Vercel project, root: apps/web
  docs/         ← Separate Vercel project, root: apps/docs
packages/
  ui/
  config/
turbo.json
```

### Configuration

- **Root directory**: Set per-project in Vercel dashboard.
- **Include files outside root**: Vercel auto-detects workspace dependencies.
- **Build command**: `cd ../.. && npx turbo build --filter=web`.
- **Remote caching**: Link Turborepo to Vercel for shared build cache.

### `turbo.json`

```jsonc
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "lint": {},
    "test": { "dependsOn": ["build"] }
  }
}
```

---

## GitHub Integration

### Automatic Behavior

- **Production**: Push to `main` (configurable) triggers production deploy.
- **Preview**: Push to any other branch triggers preview deploy.
- **PR comments**: Bot posts deploy URL and status.
- **Commit status**: Build pass/fail shown in PR checks.

### Configuration

- **Ignored branches**: Configure in project settings to skip specific branches.
- **Auto-cancel**: Previous in-progress builds on same branch auto-cancel.
- **Deploy hooks**: Trigger builds from external services via webhook URL.

### Deployment Protection

- **Vercel Authentication**: Require Vercel login. Included on Pro for preview and deployment URLs only ("Standard Protection"); covering production domains needs the Advanced Deployment Protection add-on.
- **Password protection**: NOT included on Pro. Requires the paid Advanced Deployment Protection add-on (the API answers `invalid_password_protection: Advanced Deployment Protection is not enabled on your team`). Enterprise includes it.
- **Trusted IPs**: Restrict by IP range on Enterprise.
- **Gotcha**: a new project's default `<project>.vercel.app` production domain is public under Standard Protection. If content must stay private and the add-on is not wanted, gate it in the app (middleware/proxy with a signed cookie) before the first production deploy.

---

## Deployment Rollback

### Instant Rollback

1. Dashboard: Deployments tab, find target deployment, click "..." menu, select "Promote to Production".
2. CLI: `vercel promote <deployment-url>`.
3. Effect: Instant, no rebuild. Points production to previous deployment's immutable artifacts.

### Rollback Considerations

- **Environment variables**: Rollback uses *current* env vars, not those at original deploy time.
- **Database migrations**: Rollback does NOT reverse database changes. Plan migrations to be backward-compatible.
- **Edge config**: Current edge config applies, not historical.
- **External services**: API contracts must remain compatible.

### Recovery Checklist

1. Identify the last known good deployment (dashboard or `vercel ls`).
2. Promote it: `vercel promote <url>`.
3. Verify production is restored.
4. Investigate the broken deployment's build logs.
5. Fix forward on a new branch; do not force-push over the broken commit.

---

## CLI Quick Reference

| Command | Purpose |
|---|---|
| `vercel` | Deploy from local (preview) |
| `vercel --prod` | Deploy to production |
| `vercel env pull .env.local` | Pull env vars locally |
| `vercel dev` | Local dev with Vercel features |
| `vercel ls` | List recent deployments |
| `vercel promote <url>` | Instant rollback |
| `vercel logs <url>` | Stream deployment logs |
| `vercel inspect <url>` | Deployment details |
| `vercel domains ls` | List configured domains |
