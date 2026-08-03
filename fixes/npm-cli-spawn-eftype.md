# `spawn EFTYPE` from an npm-installed CLI on Windows ARM64

## Symptom

A globally installed CLI throws at startup:

```
Error: spawn EFTYPE
    at ChildProcess.spawn (node:internal/child_process:421:11)
  errno: -4028,
  code: 'EFTYPE',
  syscall: 'spawn'
```

The install itself looked fine — `npm install -g <tool>` printed `added 1 package in 2s`.

Hit with `agent-browser`, but the shape applies to any npm package that downloads a
platform binary in `postinstall`.

## Cause

Two things stack up.

Packages that ship native binaries install a small JS wrapper plus a `postinstall.js` that
downloads the correct binary for your platform. The wrapper then picks a binary by `os.arch()`
and spawns it.

1. **The postinstall never ran successfully.** Under nvm4w, `node` is often not on `PATH` for
   the `cmd.exe` subprocess npm uses to run lifecycle scripts. The script dies with
   `'node' is not recognized`, but npm reports the install as successful — postinstall failure
   is frequently non-fatal.
2. **The arm64 binary is left as a 0-byte stub.** `os.arch()` returns `arm64`, the wrapper
   spawns the empty file, and Windows rejects it as not a valid executable → `EFTYPE`.

`EFTYPE` means "exec format error": you tried to run something that isn't a runnable binary.
An empty file qualifies.

## Diagnosis

Confirm the binary is empty. arm64 should be several MB, not zero:

```bash
ls -la "$(npm root -g)/<package>/bin/"
```

```
-rw-r--r-- 1 user 197121 10485760 <pkg>-win32-x64.exe
-rw-r--r-- 1 user 197121        0 <pkg>-win32-arm64.exe   <-- 0 bytes
```

And confirm what Node reports:

```bash
node -e "console.log(process.platform, process.arch)"   # win32 arm64
```

Zero-byte arm64 binary → this is the bug.

## Fix

Copy the x64 binary into the arm64 slot. Windows 11 ARM64 emulates x64 natively, and for a
CLI process the emulation overhead is irrelevant:

```bash
BIN="$(npm root -g)/<package>/bin"
cp "$BIN/<pkg>-win32-x64.exe" "$BIN/<pkg>-win32-arm64.exe"
```

## Preventing it on reinstall

The real problem is postinstall not finding `node`. Put Node on `PATH` explicitly for the
install so lifecycle scripts inherit it:

```powershell
$env:PATH = "C:\nvm4w\nodejs;" + $env:PATH
C:\nvm4w\nodejs\npm.cmd install -g <package>
```

With `node` resolvable, postinstall downloads the real binary and no patching is needed —
though it will still download the x64 build, since most projects publish no win32-arm64
artifact at all.

## Verify

```bash
<package> --version
```

Prints a version instead of throwing.

## Note

If a package *does* publish a genuine win32-arm64 build, prefer it — reinstall with `node`
on `PATH` first and only fall back to copying the x64 binary if the arm64 slot is still empty.
