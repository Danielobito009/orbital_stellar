# StellarEventBoundary

A client-side boundary wrapper component that ensures EventSource-dependent components only render after hydration in SSR environments.

## The Problem

When using `useStellarEvent`, `useStellarPayment`, or `useStellarActivity` hooks in a Next.js or other SSR framework, you may encounter runtime errors like:

```
ReferenceError: EventSource is not defined
```

This happens because:

1. **EventSource is browser-only**: The EventSource API only exists in browsers, not in Node.js
2. **SSR renders on the server**: During server-side rendering, React components execute in a Node.js environment
3. **Hooks run immediately**: React hooks like `useStellarEvent` run during render, before hydration
4. **Mismatch causes errors**: When the server tries to use EventSource (which doesn't exist), it crashes

## The Solution

`StellarEventBoundary` wraps your event-consuming components and ensures they only render **after hydration** when the browser APIs are available.

### How It Works

```
SSR Phase (Server)
├─ StellarEventBoundary renders fallback
└─ Children are NOT rendered

Hydration Phase (Browser)
├─ useEffect runs
├─ isMounted state updates to true
└─ Children are rendered

Live Phase (Browser)
└─ Children render with live EventSource data
```

## Installation

`StellarEventBoundary` is exported from `@orbital/pulse-notify`:

```bash
npm install @orbital/pulse-notify
```

## Basic Usage

Wrap any component that uses pulse-notify hooks:

```tsx
import { StellarEventBoundary, useStellarEvent } from "@orbital/pulse-notify";

function EventConsumer() {
  const { event, connected } = useStellarEvent(
    "https://api.example.com",
    "GABC123"
  );

  return (
    <div>
      <p>Status: {connected ? "Connected" : "Disconnected"}</p>
      <p>Latest event: {event?.type}</p>
    </div>
  );
}

export default function Page() {
  return (
    <StellarEventBoundary>
      <EventConsumer />
    </StellarEventBoundary>
  );
}
```

## With Custom Fallback

Show a loading skeleton or placeholder while the component hydrates:

```tsx
import { StellarEventBoundary } from "@orbital/pulse-notify";

function LoadingSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-3/4"></div>
      <div className="h-4 bg-gray-200 rounded w-1/2 mt-2"></div>
    </div>
  );
}

export default function Page() {
  return (
    <StellarEventBoundary fallback={<LoadingSkeleton />}>
      <EventConsumer />
    </StellarEventBoundary>
  );
}
```

## Advanced: Multiple Event Consumers

You can nest multiple event consumers inside a single boundary:

```tsx
export default function Dashboard() {
  return (
    <StellarEventBoundary fallback={<DashboardSkeleton />}>
      <PaymentWidget />
      <ActivityFeed />
      <ConnectionStatus />
    </StellarEventBoundary>
  );
}
```

## Common Mistakes

### ❌ Don't: Use hooks directly in SSR components

```tsx
// This will crash during SSR
export default function Page() {
  const { event } = useStellarEvent("https://api.example.com", "GABC");
  return <div>{event?.type}</div>;
}
```

### ✅ Do: Wrap with StellarEventBoundary

```tsx
// This works in SSR
function EventDisplay() {
  const { event } = useStellarEvent("https://api.example.com", "GABC");
  return <div>{event?.type}</div>;
}

export default function Page() {
  return (
    <StellarEventBoundary>
      <EventDisplay />
    </StellarEventBoundary>
  );
}
```

### ❌ Don't: Use "use client" on the page component

```tsx
// This disables SSR for the entire page
"use client";

export default function Page() {
  return <EventConsumer />;
}
```

### ✅ Do: Use "use client" only on the boundary

```tsx
// The boundary has "use client" built-in
export default function Page() {
  return (
    <StellarEventBoundary>
      <EventConsumer />
    </StellarEventBoundary>
  );
}
```

## Props

### `children` (required)

The component(s) to render after hydration. These components can safely use pulse-notify hooks.

```tsx
<StellarEventBoundary>
  <MyEventComponent />
</StellarEventBoundary>
```

### `fallback` (optional)

Content to display during SSR and before hydration. Defaults to `null`.

```tsx
<StellarEventBoundary fallback={<Skeleton />}>
  <MyEventComponent />
</StellarEventBoundary>
```

## TypeScript

The component is fully typed:

```tsx
import {
  StellarEventBoundary,
  type StellarEventBoundaryProps,
} from "@orbital/pulse-notify";

const props: StellarEventBoundaryProps = {
  children: <div>Content</div>,
  fallback: <div>Loading...</div>,
};
```

## Framework Integration

### Next.js App Router

```tsx
// app/events/page.tsx
import { StellarEventBoundary } from "@orbital/pulse-notify";

export default function EventsPage() {
  return (
    <StellarEventBoundary fallback={<EventsSkeleton />}>
      <EventsClient />
    </StellarEventBoundary>
  );
}
```

### Next.js Pages Router

```tsx
// pages/events.tsx
import { StellarEventBoundary } from "@orbital/pulse-notify";

export default function EventsPage() {
  return (
    <StellarEventBoundary fallback={<EventsSkeleton />}>
      <EventsClient />
    </StellarEventBoundary>
  );
}
```

### Remix

```tsx
// routes/events.tsx
import { StellarEventBoundary } from "@orbital/pulse-notify";

export default function EventsRoute() {
  return (
    <StellarEventBoundary fallback={<EventsSkeleton />}>
      <EventsClient />
    </StellarEventBoundary>
  );
}
```

## FAQ

### Q: Do I need to wrap every component that uses pulse-notify hooks?

**A:** No, only the top-level component that uses the hooks needs to be wrapped. Child components can use the hooks freely once inside the boundary.

```tsx
// ✅ Correct: wrap once at the top
<StellarEventBoundary>
  <Parent>
    <Child1 /> {/* Can use hooks */}
    <Child2 /> {/* Can use hooks */}
  </Parent>
</StellarEventBoundary>

// ❌ Unnecessary: wrapping multiple times
<StellarEventBoundary>
  <StellarEventBoundary>
    <Child />
  </StellarEventBoundary>
</StellarEventBoundary>
```

### Q: Can I use StellarEventBoundary in a client-only app?

**A:** Yes, it's safe to use even in client-only apps. It will render the fallback briefly during initial render, then switch to children after hydration. This is harmless in non-SSR environments.

### Q: What if I need different fallbacks for different sections?

**A:** Use multiple boundaries:

```tsx
<div>
  <StellarEventBoundary fallback={<PaymentSkeleton />}>
    <PaymentWidget />
  </StellarEventBoundary>

  <StellarEventBoundary fallback={<ActivitySkeleton />}>
    <ActivityFeed />
  </StellarEventBoundary>
</div>
```

### Q: Does StellarEventBoundary affect performance?

**A:** Minimal impact. It adds a single `useState` and `useEffect` hook. The fallback is only rendered during SSR and the brief hydration phase, then discarded.

### Q: Can I use StellarEventBoundary with Suspense?

**A:** Yes, they work together:

```tsx
<Suspense fallback={<Skeleton />}>
  <StellarEventBoundary>
    <EventComponent />
  </StellarEventBoundary>
</Suspense>
```

### Q: What about error boundaries?

**A:** Error boundaries work normally with StellarEventBoundary:

```tsx
<ErrorBoundary>
  <StellarEventBoundary>
    <EventComponent />
  </StellarEventBoundary>
</ErrorBoundary>
```

## Related

- [`useStellarEvent`](./pulse-notify.md) - Hook for subscribing to Stellar events
- [`useStellarPayment`](./pulse-notify.md) - Hook for payment events
- [`useStellarActivity`](./pulse-notify.md) - Hook for all activity events
- [`StellarConnectionStatus`](./pulse-notify.md) - Component showing connection status
