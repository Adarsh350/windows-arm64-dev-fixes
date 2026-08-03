# wrangler / workerd on Windows ARM64

## Symptom

In any Cloudflare Workers project:

```
Error: Unsupported platform: win32 arm64 LE
    at .../node_modules/workerd/install.js
```

`npm install` hangs or fails outright. Afterwards, *every* `npx wrangler` command — including
`npx wrangler --version` — crashes at module load with the same error, this time from
`workerd/lib/main.js`.

## Cause

`workerd` is wrangler's runtime and is required at import time, so a missing binary breaks the
CLI entirely rather than only at `dev`/`deploy`.

Cloudflare publishes workerd for darwin-arm64, darwin-x64, linux-arm64, linux-x64 and
windows-x64. There is no windows-arm64 package. Both the postinstall download and the runtime
platform lookup throw on the unknown `win32 arm64 LE` key.

Windows 11 ARM64 runs the x64 build fine under emulation — it just has to be registered.

## Fix

Install without scripts, then add the x64 package explicitly:

```sh
npm install --ignore-scripts
npm install --save-optional --os win32 --cpu x64 @cloudflare/workerd-windows-64 --ignore-scripts
```

The `--os`/`--cpu` flags are not optional. The package declares `cpu: ["x64"]`, so npm skips it
on an arm64 host unless you override the platform check.

Then teach workerd's platform map about the arm64 key. In
`node_modules/workerd/lib/main.js`, inside `knownPackages`, add:

```js
"win32 arm64 LE": "@cloudflare/workerd-windows-64",
```

**Map values are plain package-name strings, not `[pkg, subpath]` tuples.** Inserting a tuple
produces:

```
TypeError: pkg.replace is not a function
```

from `downloadedBinPath()`, which calls `.replace()` on the value directly. The
`bin/workerd.exe` subpath is computed separately.

## Surviving reinstalls

`node_modules` patches evaporate on the next clean install. Make it a postinstall step that
no-ops off win32-arm64:

```json
{
  "scripts": { "postinstall": "node scripts/patch-workerd-arm64.mjs" },
  "optionalDependencies": { "@cloudflare/workerd-windows-64": "*" }
}
```

The script should exit early unless `process.platform === 'win32' && process.arch === 'arm64'`,
then apply the `knownPackages` edit idempotently (check whether the key is already present
before writing).

The `optionalDependencies` entry alone does **not** solve it — npm still skips the package on
arm64 because of the `cpu` mismatch. It exists so the package is in the lockfile; the explicit
`--os/--cpu` install is what actually places it on disk.

## Verify

```sh
npx wrangler --version          # prints a version, no crash
npx wrangler deploy --dry-run   # bundles successfully
```

`deploy --dry-run` exercises the bundler as well. esbuild publishes a real win32-arm64 build,
so nothing extra is needed there.
