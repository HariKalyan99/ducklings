# @singing-duck/capture-duck

Lightweight error capture for browser and Node.js apps.

## Install

```bash
npm install @singing-duck/capture-duck
```

Optional (browser PostHog integration only):

```bash
npm install posthog-js
```

## Exports

- `@singing-duck/capture-duck`: stack parsing helpers
- `@singing-duck/capture-duck/browser`: browser error capture + global handlers
- `@singing-duck/capture-duck/node`: backend capture factory + safe client response helper

## Quick Start (Browser)

```javascript
import posthog from "posthog-js";
import {
  buildPosthogInitOptions,
  initErrorTracking,
  captureDuck,
} from "@singing-duck/capture-duck/browser";

posthog.init(
  import.meta.env.VITE_PUBLIC_POSTHOG_KEY,
  buildPosthogInitOptions({
    apiKey: import.meta.env.VITE_PUBLIC_POSTHOG_KEY,
    host: import.meta.env.VITE_PUBLIC_POSTHOG_HOST, // optional
  }),
);

await initErrorTracking({
  ingestUrl: "https://api.example.com/errors",
  getIngestHeaders: () => ({
    Authorization: `Bearer ${token}`,
  }),
  posthogClient: posthog,
  timeoutMs: 8000,
  beforeSend: (payload) => payload, // sanitize or return false to skip
});

const out = await captureDuck(new Error("Checkout failed"), {
  context: "checkout",
});

if (out.ingest?.ok) {
  console.log("stored", out.ingest.data);
}
```

Browser behavior:
- Sends ingest payload with `fetch` JSON when `ingestUrl` is set
- Installs global handlers for `window.onerror` and `window.onunhandledrejection`
- Global handler capture is fire-and-forget

## Quick Start (Node / Backend)

```javascript
import {
  createCaptureDuck,
  buildClientErrorResponse,
} from "@singing-duck/capture-duck/node";

const captureDuck = createCaptureDuck({
  report: async (payload) => {
    await db.errors.insert(payload); // DB, queue, HTTP, etc.
  },
  environment: process.env.NODE_ENV,
  // readSnippet: null, // disable snippet reads if needed
});

try {
  throw new Error("Order service failure");
} catch (err) {
  const result = await captureDuck(err, {
    url: "/api/orders",
    type: "backend",
    serviceContext: { service: "orders.create" },
  });
  console.log(result.ok ? result.payload.fingerPrint : result.error);
}
```

Safe response helper for API clients:

```javascript
app.post("/orders", async (req, res) => {
  try {
    // ...
  } catch (err) {
    await captureDuck(err, { url: "/orders" });
    return res.status(500).json(
      buildClientErrorResponse(err, {
        code: "ORDERS_CREATE_FAILED",
        requestId: req.id,
      }),
    );
  }
});
```

## API Overview

Browser:
- `initErrorTracking(options)`
  - `ingestUrl?: string | null`
  - `getIngestHeaders?: () => Record<string, string> | Promise<Record<string, string>>`
  - `beforeSend?: (payload) => payload | false`
  - `timeoutMs?: number`
  - `posthogClient?: object` (already initialized)
  - `posthog?: { apiKey: string; host?: string; ... }` (lazy init)
- `captureDuck(error, extra?)`
  - returns `{ posthog, ingest }`

Node:
- `createCaptureDuck(options)`
  - required: `report(payload)`
  - optional: `environment`, `readSnippet`, `defaultUserAgent`
  - returns `captureDuck(error, extra?)`
- `buildClientErrorResponse(error, options?)`
  - safe error payload for frontend clients (no stack in production by default)

Core:
- `parseStackTrace(stack)` for structured stack frames

## Security Notes

- Use only public PostHog keys (`phc_...`) in browser code
- Keep secrets/tokens on server side
- Use `beforeSend` to strip sensitive fields before ingest

## Maintainer Notes

```bash
npm test
npm pack --dry-run
```

For release workflow details, see `PUBLISHING.md`.

## License

MIT
