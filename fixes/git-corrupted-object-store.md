# Refs exist but git can't read them: corrupted object store recovery

## Symptom

Branch files are sitting in `.git/refs/heads/`, but git behaves as though they aren't:

```
error: pathspec 'main' did not match any file(s) known to git
fatal: unable to read tree (a1b2c3d...)
warning: ignoring ref with broken name refs/heads/main
error: refs/heads/main: invalid sha1 pointer 0000000000000000000000000000000000000000
```

`git branch` lists unrelated branches, or nothing you recognise, while `ls .git/refs/heads/`
clearly shows the branch you want.

## Cause

Refs and objects are separate things. A ref is a file containing a commit SHA; the commit,
tree, and blob objects live in `.git/objects` (loose or packed). If the objects are missing,
the refs point into nothing and every operation that needs to read history fails — while the
ref files themselves still look perfectly healthy on disk.

Common ways to get here:

- A `.git` directory copied without its pack files
- An interrupted clone or fetch
- A wrong remote configured, so the objects were never fetched
- A ref file with a broken name, which can poison branch enumeration

## Diagnosis

```bash
ls .git/refs/heads/          # ref files present
git fsck 2>&1 | head -20     # "bad sha1 file" / "unable to read"
git remote -v                # is this even the right repository?
git cat-file -t "$(cat .git/refs/heads/main)"   # fails if the object is gone
```

If `cat-file` can't type the SHA a ref points at, the object store is the problem — not the ref.

## Fix

When the remote holds the real history, the fastest correct move is to rebuild local git state
from it. **Your working files are not touched by this** — `.git` is metadata; the source on
disk survives.

```bash
# Preserve anything uncommitted FIRST — it is not recoverable afterwards
cp -r src ../src-backup    # or copy the whole working tree

rm -rf .git
git init
git remote add origin <correct-remote-url>
git fetch origin

# find the most recently updated branch
git for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin

git checkout -f -b <branch> origin/<branch>
```

`-f` is needed when untracked files in the working tree collide with files arriving from the
branch.

## After recovery

```bash
npm install        # package.json may differ from what your node_modules was built for
git branch -vv     # confirm upstream tracking is set
```

Restart any dev server so it picks up the new dependency tree.

## What doesn't work

- `git checkout -f <branch>` — `-f` forces working-tree overwrites, it cannot conjure missing
  objects.
- Deleting individual ref files — the refs aren't the problem.
- `git reflog` — the reflog also stores SHAs, and reading them needs the same absent objects.

## Prevention

If the objects are gone locally *and* the remote is behind, the missing commits are gone. Push
branches you care about. A branch that exists only on one machine is one `.git` corruption away
from not existing.
