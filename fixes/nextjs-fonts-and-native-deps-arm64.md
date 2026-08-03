# Next.js on Windows ARM64: fonts and native dependencies

Two independent failures that both trace back to Windows ARM64 being a second-class build
target.

---

## `next/font/google` fails at build and dev

### Symptom

A project using `next/font/google` fails to compile. Setting `NODE_OPTIONS=--use-system-ca`
does not help, which rules out the usual certificate explanation.

### Cause

`next/font/google` fetches font metadata from Google over **HTTP/2** at build and dev time.
Node's http2 client was not functional under Turbopack on Windows ARM64, so the fetch fails
before TLS is even a consideration.

This is not a certificate problem, and treating it as one costs an afternoon.

### Fix

Self-host the fonts and remove the network fetch entirely:

```bash
npm install @fontsource-variable/inter
```

```css
/* globals.css */
@import "@fontsource-variable/inter";

:root {
  --font-body: "Inter Variable", system-ui, sans-serif;
}
```

Check the actual filenames inside the package before importing a specific weight or style —
they aren't uniform across families. The italic variant is typically `wght-italic.css`, not
`index-italic.css`.

Self-hosting is worth doing regardless: it removes a build-time network dependency and a
third-party request at runtime.

### Note on Turbopack

Next.js 16 uses Turbopack for both `dev` and `build`. There is no `--no-turbopack` escape
hatch for either command. If the dev server serves stale content after an edit, delete `.next/`
— the cache can hold bad state after a failed compile.

---

## `@sentry/profiling-node` crashes on startup

### Symptom

The app dies during instrumentation:

```
Cannot find module './sentry_cpu_profiler-win32-arm64-<abi>.node'
```

### Cause

`@sentry/profiling-node` ships prebuilt native binaries and publishes none for win32-arm64.
Because the import sits at the top level of the Sentry server config, it executes as the
instrumentation hook loads — before any runtime environment check you might have written.

### Fix

Make the import dynamic and let it fail quietly on platforms with no binary:

```ts
// sentry.server.config.ts
let nodeProfilingIntegration;
try {
  ({ nodeProfilingIntegration } = await import("@sentry/profiling-node"));
} catch {
  // no prebuilt binary for this platform (win32-arm64); skip profiling locally
}

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  integrations: nodeProfilingIntegration ? [nodeProfilingIntegration()] : [],
});
```

A static top-level `import` cannot be guarded this way — module resolution happens before any
statement in the file runs, so the `try` never gets a chance.

This is a local-development-only problem when deploying to x64 Linux: the deployment platform
pulls the correct binary and profiling works normally in production.
