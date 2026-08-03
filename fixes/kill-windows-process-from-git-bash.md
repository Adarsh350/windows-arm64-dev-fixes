# Killing a Windows process by PID from Git Bash

## Symptom

A dev server is holding port 3000. You have its PID. Nothing kills it:

```bash
kill 25800
# bash: kill: (25800) - No such process

taskkill /PID 25800 /F
# bash: taskkill: command not found

/mnt/c/Windows/System32/taskkill.exe /PID 25800 /F
# bash: /mnt/c/...: No such file or directory
```

## Cause

Each failure has a different reason, which is why this wastes time:

- `kill` is Git Bash's builtin operating on **MSYS2 PIDs**. A Windows process started outside
  that shell has no MSYS PID, so the Windows PID you're holding matches nothing.
- `taskkill` is a real Windows executable but isn't on Git Bash's `PATH` by default.
- `/mnt/c/...` is the **WSL** mount convention. Git Bash mounts drives at `/c/...`. That path
  simply doesn't exist here.

The third one is the trap: WSL and Git Bash look similar and are not.

## Fix

Use Node as the killer. Node runs as a native Windows process, and `process.kill` maps to the
Win32 API, so it accepts real Windows PIDs:

```bash
node -e "process.kill(25800)"
```

Silent on success. If the process is already gone, Node throws `ESRCH` — harmless.

Two alternatives that also work:

```bash
# taskkill via its absolute Git Bash path
/c/Windows/System32/taskkill.exe //PID 25800 //F
```

Note the doubled slashes: MSYS rewrites `/PID` into a Windows path before `taskkill` sees it.
`//PID` escapes that translation.

```powershell
Stop-Process -Id 25800 -Force
```

## Finding the PID

Next.js prints it on a port conflict:

```
⨯ Another next dev server is already running.
- Local:  http://localhost:3001
- PID:    25800
```

Otherwise, ask Windows what owns the port:

```bash
netstat -ano | grep :3000     # last column is the PID
```

## Verify

Restart the server. It should bind to 3000 rather than incrementing to 3001. Give it a second
or two — Windows does not release the socket instantly.
