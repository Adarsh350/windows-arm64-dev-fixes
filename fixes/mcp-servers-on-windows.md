# MCP servers that never load on Windows

## Symptom

An MCP server is configured with `command` + `args`, the config is valid JSON, the client
restarts cleanly — and the server's tools never appear in the session. No error surfaces. Run
the same command by hand in a terminal and the server starts fine.

## Cause

Three separate problems produce the identical "nothing happened" outcome.

**1. `.cmd` shims don't spawn reliably as child processes.**
`npx`, `npm`, and most global bins on Windows are `.cmd` batch files. Spawning a batch file
requires going through the shell, and MCP runners typically spawn the executable directly.
The spawn fails, and because the server never completes a handshake there's nothing to report.

**2. `npx` doesn't resolve in a minimal environment.**
MCP servers launch with a stripped environment, not your interactive shell's. A bare `npx` that
works in your terminal isn't guaranteed to resolve when launched by the client. The `env` block
in the config sets variables *for the server*; it does not fix how the command itself is
located.

**3. Node version managers move the global root.**
With nvm4w, globals go to `C:\nvm4w\nodejs\node_modules\`, not
`C:\Users\<user>\AppData\Roaming\npm\node_modules\`. Config copied from any generic tutorial
points at the wrong path. `npm root -g` prints the real one.

## Fix

Point the config at `node.exe` and the server's entry script, with absolute paths and no shims
anywhere.

```bash
npm install -g @scope/package@latest
npm root -g                                    # -> C:\nvm4w\nodejs\node_modules
node -p "require('@scope/package/package.json').main"   # -> dist/index.js
```

```json
{
  "mcpServers": {
    "server-name": {
      "command": "C:\\nvm4w\\nodejs\\node.exe",
      "args": ["C:\\nvm4w\\nodejs\\node_modules\\@scope\\package\\dist\\index.js"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

Notes that matter:

- Backslashes must be escaped in JSON (`\\`).
- Resolve `main` from the package's own `package.json` rather than guessing `dist/index.js` —
  plenty of packages use `bin/cli.js`, `build/index.js`, or a `bin` map instead.
- Pass credentials via `env`, not as CLI arguments. Argument-passing is inconsistent across
  servers and leaks the key into process listings.

## Prefer HTTP servers where available

If the server offers an HTTP transport, use it. There's no process spawning, so none of the
above applies:

```json
{ "mcpServers": { "server-name": { "type": "http", "url": "https://..." } } }
```

## Verify

Restart the client fully and check whether the server's tools are listed. If a stdio server
still won't load after using absolute `node.exe` paths, run its exact command line manually and
watch stderr — a server that crashes on startup (bad key, missing dependency) looks identical
from the client side to one that never spawned.
