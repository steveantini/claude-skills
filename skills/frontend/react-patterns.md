# React Patterns & Component Architecture

---
version: 1.1.0
last_updated: 2026-09-22
applicability: React 18+, React 19, RSC-compatible
dependencies: React 18+, TypeScript 5+
---

## Server vs. Client Components

### Server Components (Default in App Router)

- Execute on the server only, never ship JS to the client
- Can `await` async operations directly in the component body
- Can access backend resources (DB, filesystem, secrets)
- Cannot use state, effects, or event handlers

```tsx
// No directive needed - server by default
export default async function UserProfile({ userId }: { userId: string }) {
  const user = await db.user.findUnique({ where: { id: userId } });
  return (
    <section>
      <h1>{user.name}</h1>
      <p>{user.bio}</p>
    </section>
  );
}
```

### Client Components

- Add `"use client"` at the file top
- Required for: state, effects, event handlers, browser APIs, third-party libs using these

**Decision rule**: Start as a Server Component. Add `"use client"` only when you hit a concrete need for interactivity or browser APIs. Push the boundary as far down the tree as possible.

### Sharing Data Between Server and Client

Server Components pass data to Client Components via props. Data must be serializable (no functions, classes, Dates serialize as strings).

```tsx
// Server Component
import { LikeButton } from "./LikeButton"; // client component

export default async function Post({ id }: { id: string }) {
  const post = await getPost(id);
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={post.id} initialCount={post.likes} />
    </article>
  );
}
```

## State Management

### Local State: `useState`

For simple, component-scoped state.

```tsx
const [isOpen, setIsOpen] = useState(false);
```

### Complex Local State: `useReducer`

When state transitions have defined logic or multiple related values.

```tsx
type State = { count: number; step: number };
type Action = { type: "increment" } | { type: "decrement" } | { type: "setStep"; step: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + state.step };
    case "decrement":
      return { ...state, count: state.count - state.step };
    case "setStep":
      return { ...state, step: action.step };
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
```

### Shared State: Context + useReducer

For state shared across a subtree without prop drilling. Avoid for high-frequency updates (every consumer re-renders on context change).

```tsx
// context/cart.tsx
"use client";

import { createContext, useContext, useReducer, type ReactNode } from "react";

type CartItem = { id: string; name: string; quantity: number };
type State = { items: CartItem[] };
type Action =
  | { type: "add"; item: CartItem }
  | { type: "remove"; id: string }
  | { type: "clear" };

const CartContext = createContext<State | null>(null);
const CartDispatchContext = createContext<React.Dispatch<Action> | null>(null);

function cartReducer(state: State, action: Action): State {
  switch (action.type) {
    case "add":
      return { items: [...state.items, action.item] };
    case "remove":
      return { items: state.items.filter((i) => i.id !== action.id) };
    case "clear":
      return { items: [] };
  }
}

export function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });
  return (
    <CartContext value={state}>
      <CartDispatchContext value={dispatch}>
        {children}
      </CartDispatchContext>
    </CartContext>
  );
}

// Typed hooks with safety checks
export function useCart() {
  const context = useContext(CartContext);
  if (!context) throw new Error("useCart must be used within CartProvider");
  return context;
}

export function useCartDispatch() {
  const context = useContext(CartDispatchContext);
  if (!context) throw new Error("useCartDispatch must be used within CartProvider");
  return context;
}
```

**Split context pattern**: Separate state and dispatch into two contexts so components that only dispatch don't re-render on state changes.

### When to Reach for External State Libraries

- **High-frequency updates** across many consumers (consider Zustand, Jotai)
- **Server state synchronization** (TanStack Query / SWR)
- **Complex cross-cutting state** with middleware needs (Zustand)
- **URL state** (use `nuqs` or `useSearchParams`)

Avoid Redux for new projects unless there is a specific need for its middleware ecosystem.

## Custom Hooks

### Principles

- Name with `use` prefix
- Extract reusable logic, not just to reduce file size
- Keep hooks focused on a single concern
- Return only what consumers need

### Common Patterns

**Encapsulating side effects**:

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    const media = window.matchMedia(query);
    setMatches(media.matches);

    const listener = (e: MediaQueryListEvent) => setMatches(e.matches);
    media.addEventListener("change", listener);
    return () => media.removeEventListener("change", listener);
  }, [query]);

  return matches;
}
```

**Wrapping async operations with status tracking**:

```tsx
function useAsync<T>(asyncFn: () => Promise<T>, deps: unknown[]) {
  const [state, setState] = useState<{
    data: T | null;
    error: Error | null;
    status: "idle" | "pending" | "success" | "error";
  }>({ data: null, error: null, status: "idle" });

  useEffect(() => {
    let cancelled = false;
    setState((prev) => ({ ...prev, status: "pending" }));

    asyncFn()
      .then((data) => {
        if (!cancelled) setState({ data, error: null, status: "success" });
      })
      .catch((error) => {
        if (!cancelled) setState({ data: null, error, status: "error" });
      });

    return () => { cancelled = true; };
  }, deps); // eslint-disable-line react-hooks/exhaustive-deps

  return state;
}
```

**Debounced value**:

```tsx
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}
```

## Error Boundaries

Error boundaries catch rendering errors in their subtree. They must be class components (or use `react-error-boundary` library).

### Using `react-error-boundary` (Recommended)

```tsx
"use client";

import { ErrorBoundary } from "react-error-boundary";

function ErrorFallback({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

export function ProtectedSection({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      onError={(error, info) => {
        // Log to error reporting service
        console.error("Caught by boundary:", error, info);
      }}
      onReset={() => {
        // Reset application state if needed
      }}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### Next.js `error.tsx` Convention

```tsx
// app/dashboard/error.tsx
"use client"; // error boundaries must be client components

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

**Note**: `error.tsx` does NOT catch errors in the `layout.tsx` of the same segment. For that, place `error.tsx` in the parent segment or use `global-error.tsx` at the app root.

## Suspense

Suspense lets you declaratively handle async loading states.

```tsx
import { Suspense } from "react";

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<SkeletonChart />}>
        <AsyncChart />     {/* server component that awaits data */}
      </Suspense>
      <Suspense fallback={<SkeletonTable />}>
        <AsyncTable />
      </Suspense>
    </div>
  );
}
```

**Streaming**: Wrapping Server Components in Suspense enables streaming. The shell renders immediately; suspended content streams in as it resolves.

**Nested Suspense**: Boundaries can nest. The closest ancestor Suspense catches the suspension. Use this for granular loading states.

**Client-side Suspense with `use`** (React 19):

```tsx
"use client";
import { use } from "react";

function UserName({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // suspends until resolved
  return <span>{user.name}</span>;
}
```

## Composition Patterns

### Composition Over Inheritance

React does not use class inheritance for component reuse. Use composition.

**Slot pattern with children**:

```tsx
function Card({ children }: { children: React.ReactNode }) {
  return <div className="rounded-lg border p-4 shadow-sm">{children}</div>;
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="mb-4 border-b pb-2">{children}</div>;
}

// Usage
<Card>
  <CardHeader><h2>Title</h2></CardHeader>
  <p>Content</p>
</Card>
```

**Named slots via props**:

```tsx
type DialogProps = {
  trigger: React.ReactNode;
  title: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;
};

function Dialog({ trigger, title, children, footer }: DialogProps) {
  return (
    <>
      {trigger}
      <div role="dialog">
        <header>{title}</header>
        <main>{children}</main>
        {footer && <footer>{footer}</footer>}
      </div>
    </>
  );
}
```

**Render prop / function-as-child** (use sparingly, hooks are usually better):

```tsx
type ListProps<T> = {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
};

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item, i)}</li>)}</ul>;
}
```

**Compound components** (for tightly coupled component sets):

```tsx
const Tabs = ({ children }: { children: React.ReactNode }) => {
  const [activeIndex, setActiveIndex] = useState(0);
  return (
    <TabsContext value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext>
  );
};

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;
```

## Avoiding Prop Drilling

Ranked by preference:

1. **Component composition** -- restructure so parent passes data directly via children/slots
2. **Context** -- for truly shared state across a subtree
3. **URL state** -- for state that should survive refresh (`searchParams`)
4. **Custom hooks abstracting context** -- cleaner API for consumers
5. **External state library** -- only when above options are insufficient

### Composition Fix for Prop Drilling

```tsx
// BEFORE: prop drilling through intermediate components
function Page() {
  const user = useUser();
  return <Layout user={user} />;
}
function Layout({ user }: { user: User }) {
  return <Nav user={user} />;
}
function Nav({ user }: { user: User }) {
  return <Avatar name={user.name} />;
}

// AFTER: composition, parent owns the rendering
function Page() {
  const user = useUser();
  return (
    <Layout nav={<Nav avatar={<Avatar name={user.name} />} />}>
      <MainContent />
    </Layout>
  );
}
```

## Component Design Rules

### Props

- Use TypeScript interfaces or types, not `PropTypes`
- Prefer specific types over `any` or broad unions
- Destructure props in the function signature
- Provide sensible defaults with default parameter values

```tsx
type ButtonProps = {
  variant?: "primary" | "secondary" | "ghost";
  size?: "sm" | "md" | "lg";
  isLoading?: boolean;
  children: React.ReactNode;
} & Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, "children">;

export function Button({
  variant = "primary",
  size = "md",
  isLoading = false,
  children,
  ...props
}: ButtonProps) {
  return (
    <button disabled={isLoading || props.disabled} {...props}>
      {isLoading ? <Spinner /> : children}
    </button>
  );
}
```

### Component File Structure

```tsx
// 1. Imports
import { useState, type ReactNode } from "react";

// 2. Types
type Props = { /* ... */ };

// 3. Constants (outside component)
const MAX_ITEMS = 10;

// 4. Component
export function MyComponent({ items }: Props) {
  // hooks first
  const [selected, setSelected] = useState<string | null>(null);

  // derived values
  const visibleItems = items.slice(0, MAX_ITEMS);

  // handlers
  const handleSelect = (id: string) => setSelected(id);

  // early returns
  if (!items.length) return <EmptyState />;

  // render
  return (/* ... */);
}

// 5. Sub-components (if small and tightly coupled)
function EmptyState() {
  return <p>No items found.</p>;
}
```

## Performance Patterns

### `React.memo`

Wrap components that receive the same props frequently but have expensive renders.

```tsx
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return <>{items.map((item) => <ExpensiveRow key={item.id} item={item} />)}</>
});
```

**Do NOT** wrap everything in `memo`. Measure first. Overuse adds overhead.

### `useMemo` and `useCallback`

```tsx
// useMemo: cache expensive derived values
const sorted = useMemo(() => items.toSorted((a, b) => a.name.localeCompare(b.name)), [items]);

// useCallback: stable function reference for memoized children
const handleClick = useCallback((id: string) => setSelected(id), []);
```

**Use when**: Passing callbacks to memoized children, computing expensive derived data, or stabilizing dependency arrays.

**Skip when**: The computation is trivial, or the component re-renders regardless.

### `useTransition` (React 18+)

Mark state updates as non-urgent to keep the UI responsive.

```tsx
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
  setInputValue(query);           // urgent: update input immediately
  startTransition(() => {
    setSearchResults(query);      // non-urgent: can be interrupted
  });
}
```

### Key Prop for Resetting Components

Change the `key` to force React to unmount and remount a component (reset its state).

```tsx
<UserProfile key={userId} userId={userId} />
```

## Refs

### DOM Refs

```tsx
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus();
}, []);

return <input ref={inputRef} />;
```

### Forwarding Refs (React 19+)

In React 19, `ref` is a regular prop. No need for `forwardRef`.

```tsx
function Input({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> } & InputHTMLAttributes<HTMLInputElement>) {
  return <input ref={ref} {...props} />;
}
```

For React 18 and below, use `forwardRef`:

```tsx
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />;
});
Input.displayName = "Input";
```

## JSX Whitespace: An Interpolated Value Beside an Entity

**The rule.** When an interpolated value shares a JSX text run with an HTML
entity, write the space after the value explicitly: `{value}{" "}word`. Never
rely on a literal space there.

**Why.** JSX trims and collapses whitespace in text runs. In a production
build, a text run that contains an entity (`&rsquo;`, `&ldquo;`, `&amp;`) is
compiled differently from a plain one, and the leading space of the run that
follows an expression is dropped. The development server and a static render
in tests both keep the space, so the bug shows only in the built page: the
product name or a label runs into the next word ("nameword"). It appeared in
three places in one project, one of which had shipped that way for weeks.

**Example.**

```tsx
// Renders "Ask about NAMEand I’ll help" in the production build.
<p>Ask about {PRODUCT_NAME} and I&rsquo;ll help.</p>

// Renders correctly everywhere.
<p>Ask about {PRODUCT_NAME}{" "}and I&rsquo;ll help.</p>
```

The same idiom covers a value that ends a source line (the newline-and-indent
that follows is dropped entirely), so `{value}{" "}` at a line end, and
`{" "}{value}` at a line start, are the safe forms.

**How to test for it.** A source scan catches the pattern before it ships:

```ts
// A value followed by a plain space and a text run that holds an entity.
const risky = /\{(?:PRODUCT_NAME|LABEL)\}( [^<{]*)/g;
for (const file of tsxFiles) {
  for (const match of file.source.matchAll(risky)) {
    expect(match[1], `${file.path}: use {" "} after the value`).not.toContain("&");
  }
}
```

Pair it with a check on the built HTML when a page prerenders: grep the output
for the value immediately followed by a letter. A static-render test alone
does not reproduce the drop.
