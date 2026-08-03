# Windows ARM64 dev environment fixes

Working notes from running a full JavaScript/TypeScript toolchain on a Windows 11 ARM64 laptop
(Samsung Galaxy Book, Snapdragon). Most of these bugs share one root cause: **a package publishes
no `win32-arm64` build**, and the failure surfaces somewhere far away from that fact — a silent
`npm install`, a zero-byte binary, a `spawn EFTYPE`, an install script that never ran.

Windows 11 on ARM runs x64 binaries through emulation, so the fix is usually "register the x64
build in the arm64 slot" rather than "wait for upstream support". Each note records the symptom
verbatim (so it's searchable), the actual cause, the fix, and how to verify it worked.

Every fix here was hit and resolved on a real machine, not reproduced from an issue tracker.

## Index

| Symptom | Cause | Fix |
|---|---|---|
| `Error: spawn EFTYPE` from a CLI installed via npm | postinstall never ran; arm64 binary is 0 bytes | [agent-browser EFTYPE](fixes/npm-cli-spawn-eftype.md) |
| `Unsupported platform: win32 arm64 LE` from `workerd` | Cloudflare ships no win32-arm64 workerd | [wrangler / workerd](fixes/wrangler-workerd-arm64.md) |
| `npm install` exits 0 but installs nothing | repo is bun-managed; npm defers to `bun.lock` | [bun vs npm silent no-op](fixes/bun-npm-silent-noop.md) |
| `kill <PID>` says "No such process" in Git Bash | Git Bash PIDs are not Windows PIDs | [killing processes from Git Bash](fixes/kill-windows-process-from-git-bash.md) |
| `unable to read tree`, refs exist but won't check out | object store missing; refs point nowhere | [corrupted object store recovery](fixes/git-corrupted-object-store.md) |
| MCP server configured but its tools never appear | `.cmd` shims don't spawn as MCP child processes | [MCP servers on Windows](fixes/mcp-servers-on-windows.md) |
| TLS failures on every network command | corporate root CA not in Node's bundled store | [Node TLS / system CA](fixes/node-tls-system-ca.md) |
| `next/font/google` build failure; Sentry profiler module not found | http2 unavailable; no win32-arm64 native build | [Next.js fonts + native deps](fixes/nextjs-fonts-and-native-deps-arm64.md) |

## Conventions used in these notes

Paths are written generically:

- `<user>` — your Windows username
- `C:\nvm4w\nodejs\` — [nvm4w](https://github.com/coreybutler/nvm-windows) install root; substitute
  your Node install path (`npm root -g` prints the real one)
- Shell examples are Git Bash unless the block says `powershell`

## License

MIT — see [LICENSE](LICENSE).
