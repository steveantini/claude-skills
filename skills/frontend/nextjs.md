# Next.js Patterns & Conventions

---
version: 1.1.0
last_updated: 2026-09-21
applicability: Next.js 14+, App Router
dependencies: React 18+, Node.js 18+
---

## App Router File Conventions

### Reserved Files

| File | Purpose |
|------|---------|
| `layout.tsx` | Shared UI wrapper, persists across navigations, does NOT re-render |
| `page.tsx` | Unique route UI, required to make a route publicly accessible |
| `loading.tsx` | Instant loading state via Suspense boundary |
| `error.tsx` | Error boundary for the segment (must be `"use client"`) |
| `not-found.tsx` | UI for `notFound()` calls or unmatched routes |
| `template.tsx` | Like layout but re-mounts on navigation (new instance each time) |
| `default.tsx` | Fallback for parallel routes when no match exists |
| `route.ts` | API endpoint (cannot coexist with `page.tsx` in same directory) |
| `middleware.ts` | Runs before every matched request (project root or `src/`) |

### Route Organization

```
app/
  (marketing)/          # Route group - no URL impact
    about/page.tsx      # /about
  (dashboard)/
    settings/page.tsx   # /settings
  @modal/              # Parallel route (named slot)
    login/page.tsx
  blog/
    [slug]/page.tsx    # Dynamic segment
    [...slug]/page.tsx # Catch-all segment
    [[...slug]]/page.tsx # Optional catch-all
  api/
    route.ts           # /api
```

**Private folders**: Prefix with `_` (e.g., `_components/`) to exclude from routing.

## Server vs. Client Components

### Default: Server Components

All components are Server Components by default. They:
- Run only on the server
- Can directly `await` async operations (DB queries, file reads, fetch)
- Cannot use hooks (`useState`, `useEffect`, etc.)
- Cannot use browser APIs (`window`, `document`, etc.)
- Cannot use event handlers (`onClick`, `onChange`, etc.)
- Have zero JS bundle cost

### Client Components (`"use client"`)

Add directive at the **top of the file** (before imports):

```tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Rules**:
- The `"use client"` directive defines the boundary; all imports below it become client code
- Push `"use client"` as far down the tree as possible
- Server Components CAN be passed as `children` to Client Components (composition pattern)

### Composition Pattern (Interleaving)

```tsx
// ServerWrapper.tsx (server component - no directive)
import { ClientInteractive } from "./ClientInteractive";
import { ServerData } from "./ServerData";

export function ServerWrapper() {
  return (
    <ClientInteractive>
      <ServerData /> {/* Stays a server component */}
    </ClientInteractive>
  );
}
```

## Data Fetching

### Server Components (Preferred)

```tsx
// Direct async/await in server components
export default async function Page() {
  const data = await fetch("https://api.example.com/data", {
    cache: "force-cache",       // default: cached (equivalent to SSG)
    // cache: "no-store",       // no caching (equivalent to SSR)
    // next: { revalidate: 60 } // ISR: revalidate every 60s
    // next: { tags: ["posts"] } // on-demand revalidation via tag
  });
  const posts = await data.json();
  return <PostList posts={posts} />;
}
```

**Fetch deduplication**: Next.js automatically deduplicates identical `fetch` requests in the same render pass. No need for prop drilling to avoid duplicate calls.

### Server Actions

Define with `"use server"` directive. Can be called from Client or Server Components.

```tsx
// app/actions.ts
"use server";

import { revalidatePath, revalidateTag } from "next/cache";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  // DB operation
  await db.post.create({ data: { title } });

  revalidatePath("/posts");   // purge cached page
  // revalidateTag("posts");  // purge by cache tag
  redirect("/posts");         // redirect after mutation
}
```

```tsx
// In a client component form
<form action={createPost}>
  <input name="title" />
  <button type="submit">Create</button>
</form>
```

**Key rules for server actions**:
- Must be async functions
- Can be defined inline in server components or in separate `"use server"` files
- Arguments and return values must be serializable
- Use `useActionState` (React 19) for pending/error states
- Use `useOptimistic` for optimistic UI updates

### Route Handlers

```tsx
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const query = searchParams.get("q");
  const data = await db.post.findMany({ where: { title: { contains: query } } });
  return NextResponse.json(data);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  // ...
  return NextResponse.json({ success: true }, { status: 201 });
}
```

**Supported methods**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.

`GET` route handlers are cached by default when they don't read the `Request` object. Opt out with `export const dynamic = "force-dynamic"`.

## Middleware

File: `middleware.ts` at project root (or `src/`).

```tsx
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  // Redirect
  if (request.nextUrl.pathname === "/old") {
    return NextResponse.redirect(new URL("/new", request.url));
  }

  // Rewrite
  if (request.nextUrl.pathname.startsWith("/proxy")) {
    return NextResponse.rewrite(new URL("/api/proxy", request.url));
  }

  // Set headers
  const response = NextResponse.next();
  response.headers.set("x-custom-header", "value");
  return response;
}

export const config = {
  matcher: [
    // Match all paths except static files and _next
    "/((?!_next/static|_next/image|favicon.ico).*)",
  ],
};
```

**Middleware limitations**: Runs on the Edge runtime. Cannot use Node.js APIs, heavy computation, or direct DB access. Keep it lightweight (auth checks, redirects, header manipulation).

## Image Optimization

```tsx
import Image from "next/image";

// Static import (automatically provides width/height)
import heroImage from "@/public/hero.jpg";

export function Hero() {
  return (
    <Image
      src={heroImage}
      alt="Descriptive alt text"     // required
      placeholder="blur"             // blur placeholder from static import
      priority                       // preload for LCP images
      sizes="(max-width: 768px) 100vw, 50vw"
    />
  );
}

// Remote image (must configure domains in next.config)
<Image
  src="https://example.com/photo.jpg"
  alt="Description"
  width={800}
  height={600}
  // OR use fill for unknown dimensions:
  // fill
  // className="object-cover"
/>
```

**next.config.js for remote images**:
```js
module.exports = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "example.com", pathname: "/images/**" },
    ],
  },
};
```

## Font Loading

```tsx
// app/layout.tsx
import { Inter, Roboto_Mono } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-inter",       // CSS variable for Tailwind
});

const robotoMono = Roboto_Mono({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-roboto-mono",
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

**Local fonts**:
```tsx
import localFont from "next/font/local";

const myFont = localFont({
  src: "./fonts/MyFont.woff2",
  display: "swap",
  variable: "--font-my-font",
});
```

## Metadata & SEO

### Static Metadata

```tsx
// app/page.tsx or layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Page Title",
  description: "Page description",
  openGraph: {
    title: "OG Title",
    description: "OG Description",
    images: ["/og-image.jpg"],
  },
  robots: { index: true, follow: true },
};
```

### Dynamic Metadata

```tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

### Title Template (in layout)

```tsx
export const metadata: Metadata = {
  title: {
    template: "%s | My Site",   // pages fill in %s
    default: "My Site",
  },
};
```

## Environment Variables

| Prefix | Available in | Exposed to browser |
|--------|-------------|-------------------|
| (none) | Server only | No |
| `NEXT_PUBLIC_` | Server + Client | Yes |

```env
# .env.local (gitignored, highest priority)
DATABASE_URL=postgres://...
NEXT_PUBLIC_API_URL=https://api.example.com
```

**Load order** (later overrides earlier):
1. `.env` (all environments)
2. `.env.local` (all environments, gitignored)
3. `.env.development` / `.env.production`
4. `.env.development.local` / `.env.production.local`

Access: `process.env.DATABASE_URL` (server), `process.env.NEXT_PUBLIC_API_URL` (anywhere).

**Do NOT destructure** `process.env` -- values are inlined at build time via static analysis:
```tsx
// WRONG
const { DATABASE_URL } = process.env;

// CORRECT
const dbUrl = process.env.DATABASE_URL;
```

The same rule reaches config helpers. A `NEXT_PUBLIC_` value is inlined into the
browser bundle only where the literal expression `process.env.NEXT_PUBLIC_X`
appears in source. Passing that expression as an argument is fine
(`resolve(process.env.NEXT_PUBLIC_X)`); looking it up by a computed key is not
(`process.env[name]` is `undefined` in the browser). Because the value is baked in
at build time, changing it on the host does nothing until the next build.

## Route Segment Config

Export these from `page.tsx`, `layout.tsx`, or `route.ts`:

```tsx
export const dynamic = "auto" | "force-dynamic" | "error" | "force-static";
export const revalidate = false | 0 | number;  // seconds
export const runtime = "nodejs" | "edge";
export const preferredRegion = "auto" | "global" | "home" | string[];
export const maxDuration = 5; // seconds (serverless function timeout)
```

## Common Pitfalls

### Hydration Mismatches

**Cause**: Server HTML differs from client render.

Common triggers:
- Browser extensions injecting elements
- Using `Date.now()`, `Math.random()` in render
- Accessing `window`/`localStorage` during initial render
- Conditional rendering based on client-only state

**Fix**: Use `useEffect` for client-only values, or suppress with `suppressHydrationWarning` for intentional mismatches (e.g., timestamps).

```tsx
"use client";
import { useState, useEffect } from "react";

function ClientOnlyValue() {
  const [value, setValue] = useState<string>("");

  useEffect(() => {
    setValue(localStorage.getItem("key") ?? "default");
  }, []);

  return <span>{value}</span>;
}
```

### Build-Time vs. Runtime

- **Build-time**: Static pages generated at `next build`. `process.env` values are inlined.
- **Runtime**: Dynamic pages rendered per-request. Use `dynamic = "force-dynamic"` or `cookies()`/`headers()` to opt in.
- Environment variables without `NEXT_PUBLIC_` prefix are only available at runtime on the server.

### Other Common Issues

- **Importing server-only code in client components**: Use the `server-only` package to get a build error instead of a runtime leak.
- **Large client bundles**: Audit with `@next/bundle-analyzer`. Move logic server-side.
- **Unnecessary `"use client"`**: Only add when you need hooks, event handlers, or browser APIs. Aim for leaf components.
- **`redirect()` in try/catch**: `redirect()` throws internally. Do not wrap in try/catch; call it outside.
- **Parallel data fetching**: Use `Promise.all()` to avoid serial waterfalls when fetching independent data.
- **`cookies()` and `headers()` in layouts**: Calling these opts the entire layout into dynamic rendering. Be intentional.

### Stale Generated Types After Deleting a Route

Next.js writes route type validators under `.next/types/` (from `next build` or
`next typegen`) and `.next/dev/types/` (from `next dev` only). They import every
page, layout, and route handler by path. Delete a route and those generated files
still import it, so `tsc --noEmit` fails with `TS2307: Cannot find module
'../../app/.../page.js'`, pointing into `.next/`, not into your source.

That is not a source error. Before judging a type failure, read the path:

- An error inside `.next/` means the generated types are stale. Run
  `next build` (or `next typegen`) and type-check again.
- `.next/dev/types/` is only rewritten by a running dev server. If none is
  running, delete that one stale directory; the next `next dev` recreates it.
- An error in your own files is real. A rebuild will not make it go away, so
  never reach for "it is probably stale" without reading the path first.

In CI this never appears (a clean checkout has no `.next/`); it is a local-only
trap that shows up exactly when you are deleting things.

## Caching Summary

| Mechanism | What | Where | Purpose |
|-----------|------|-------|---------|
| Request Memoization | `fetch` return values | Server | Dedupe identical requests in render |
| Data Cache | `fetch` return values | Server | Persist across requests/deployments |
| Full Route Cache | HTML + RSC payload | Server | Cache static routes |
| Router Cache | RSC payload | Client | Reduce navigation requests |

**Invalidation**: `revalidatePath()`, `revalidateTag()`, or time-based `revalidate` option.

## Project Structure Recommendation

```
src/
  app/                    # Routes and layouts
    (marketing)/
    (dashboard)/
    api/
    layout.tsx
    page.tsx
  components/
    ui/                   # Generic reusable (Button, Card, etc.)
    forms/                # Form-specific components
    layout/               # Header, Footer, Sidebar
  lib/                    # Utilities, helpers, constants
    db.ts
    utils.ts
    validations.ts
  actions/                # Server actions
  hooks/                  # Custom React hooks
  types/                  # TypeScript type definitions
  styles/                 # Global styles
```
