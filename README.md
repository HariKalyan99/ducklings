# @singing-duck/capture-duck

<p align="center">
  <img src="./assets/duck-logo.png" alt="Singing Duck logo" width="140" />
</p>

<p align="center">
  <strong>Version:</strong> 0.1.6 &nbsp;|&nbsp; <strong>License:</strong> MIT
</p>

<p align="center">
  <strong>Production-friendly error capture for browser and Node.js applications.</strong><br />
  Capture structured errors, enrich context, and forward to your observability pipeline.<br />
  Supports server dry-run validation before side effects and transaction-based DB rollbacks.
</p>

---

> **Reliability note:** Supports server dry-run validation before side effects, plus transaction-based DB rollbacks so failed paths do not leave partial writes.

## Why use capture-duck?

- Unified error capture API for both frontend and backend.
- Optional PostHog integration for browser telemetry.
- Pluggable backend reporter (`DB`, queue, HTTP, and more).
- Structured stack parsing with safe client-facing error responses.
- Lightweight setup with Node.js `>=18`.

## Installation

```bash
npm install @singing-duck/capture-duck
```

Optional dependency (browser PostHog integration):

```bash
npm install posthog-js
```

## Package exports

| Export | Purpose |
| --- | --- |
| `@singing-duck/capture-duck` | Stack parsing helpers |
| `@singing-duck/capture-duck/browser` | Browser error capture + global handlers |
| `@singing-duck/capture-duck/node` | Backend capture factory + safe client response helper |

## Quick start: Browser

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

### Browser behavior

- Sends ingest payload with `fetch` JSON when `ingestUrl` is set.
- Installs global handlers for `window.onerror` and `window.onunhandledrejection`.
- Global handler capture is fire-and-forget.

## Quick start: Node / Backend

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

## API overview

### Browser API

- `initErrorTracking(options)`
  - `ingestUrl?: string | null`
  - `getIngestHeaders?: () => Record<string, string> | Promise<Record<string, string>>`
  - `beforeSend?: (payload) => payload | false`
  - `timeoutMs?: number`
  - `posthogClient?: object` (already initialized)
  - `posthog?: { apiKey: string; host?: string; ... }` (lazy init)
- `captureDuck(error, extra?)`
  - Returns `{ posthog, ingest }`

### Node API

- `createCaptureDuck(options)`
  - Required: `report(payload)`
  - Optional: `environment`, `readSnippet`, `defaultUserAgent`
  - Returns `captureDuck(error, extra?)`
- `buildClientErrorResponse(error, options?)`
  - Safe payload for frontend clients (no stack in production by default)

### Core API

- `parseStackTrace(stack)` for structured stack frames

## Security

- Use only public PostHog keys (`phc_...`) in browser code.
- Keep secrets and tokens on server side.
- Use `beforeSend` to strip sensitive fields before ingest.

## Maintainer commands

```bash
npm test
npm pack --dry-run
```

For release workflow details, see `PUBLISHING.md`.

## License

MIT
