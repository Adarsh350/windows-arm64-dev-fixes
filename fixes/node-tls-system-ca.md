# `UNABLE_TO_VERIFY_LEAF_SIGNATURE` on every Node network command

## Symptom

Everything that touches the network from Node fails:

```
Error: unable to verify the first certificate
code: 'UNABLE_TO_VERIFY_LEAF_SIGNATURE'
```

`npm install` may not even show that — it wraps the failure as:

```
npm ERR! Exit handler never called!
```

Browsers on the same machine reach the same URLs without complaint.

## Cause

Node ships its **own** bundled CA store and ignores the Windows certificate store. Behind a
TLS-inspecting corporate proxy — or with any locally-installed root CA — the chain terminates
in a root Windows trusts and Node has never heard of.

That's why the browser works and Node doesn't: they consult different trust stores.

## Fix

Tell Node to use the OS store (Node 22+):

```bash
NODE_OPTIONS=--use-system-ca npm install
```

Make it permanent by exporting it from your shell profiles:

```bash
# ~/.bash_profile
export NODE_OPTIONS=--use-system-ca
```

```powershell
# $PROFILE
$env:NODE_OPTIONS = "--use-system-ca"
```

Shell profiles are loaded by freshly spawned tool shells, so this reaches subprocesses started
by editors and CLI tools.

**`setx NODE_OPTIONS ...` is not equivalent.** User-scope environment variables set that way
were not picked up by editor-spawned shells in this environment, so the fix appears to do
nothing while looking correct in System Properties.

### Child processes that don't run through a shell

A process spawned directly (no shell in between) inherits neither profile. Those need the
variable set explicitly in whatever config launches them:

```json
{ "env": { "NODE_OPTIONS": "--use-system-ca" } }
```

This is the usual reason a tool works in your terminal and fails when launched by an
application.

### curl

Same root cause, different error:

```
curl: (35) schannel: CRYPT_E_NO_REVOCATION_CHECK
```

```bash
curl --ssl-no-revoke https://...
```

## What not to do

Do **not** reach for these:

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0     # disables certificate validation entirely
npm config set strict-ssl false    # same, scoped to npm
```

Both make the error disappear by turning off the check that catches real
machine-in-the-middle attacks. `--use-system-ca` keeps verification on and simply trusts the
store the OS already trusts.

## Diagnosis note

A TLS failure and a blocked socket are different problems with different fixes. TLS fails
**fast** with a certificate error. A sandboxed or firewalled socket **hangs**, then times out.
If a command hangs rather than erroring, the certificate is not your problem.
