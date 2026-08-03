# `npm install` exits 0 and installs nothing (bun-managed repo)

## Symptom

Add a dependency to `package.json`, run `npm install`, get exit code 0 and no errors. The
package is not in `node_modules`. The build then fails:

```
Module not found: Can't resolve 'react-markdown'
```

Sometimes worse: the package lands in a *parent* directory's `node_modules`, so it resolves
locally and breaks in CI.

## Cause

The repo is managed by bun. A `bun.lock` is present — often alongside a stale
`package-lock.json` from earlier in the project's life, which is what makes it look like an
npm repo.

Two behaviours combine:

1. With bun's lockfile authoritative, npm can no-op rather than reconcile a tree it doesn't own.
2. npm walks *up* looking for a package root. If a `package.json` exists higher in the tree, it
   can install there instead of in the project directory.

Exit code 0 in both cases, which is what makes this expensive to debug — you go looking at
bundler config instead of at the installer.

## Diagnosis

```bash
ls bun.lock                       # present -> bun-managed
ls node_modules/<package>         # missing
ls ../node_modules/<package>      # found here -> npm installed to the parent
npm root -g && which npm && node --version
```

## Fix

Use the package manager that owns the lockfile:

```bash
bun install
```

If bun can't reach the registry (restricted network, sandboxed shell), npm works as a fallback
— but install the packages explicitly rather than relying on a bare `npm install`, and check
where they landed:

```bash
npm install <package> [...]
ls node_modules/<package>
```

On a machine behind a corporate TLS-inspecting proxy, both installers additionally need Node
pointed at the system certificate store — see [Node TLS / system CA](node-tls-system-ca.md):

```bash
NODE_OPTIONS=--use-system-ca bun install
```

## Prevention

Delete the stale `package-lock.json` and commit that deletion. One lockfile per repo. If both
are tracked, every contributor gets to rediscover this independently.

## Verify

```bash
ls node_modules/<package>   # in the project, not the parent
npm run build               # or: bun run build
```
