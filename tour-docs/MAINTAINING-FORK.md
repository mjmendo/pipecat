# Maintaining This Fork

This fork (`mjmendo/pipecat`) tracks `pipecat-ai/pipecat` upstream.
The only divergence is the `tour-docs/` directory added as a single commit on top of `main`.

## Remote setup

| Remote   | URL                                        | Purpose          |
|----------|--------------------------------------------|------------------|
| `origin` | `https://github.com/mjmendo/pipecat.git`   | Personal fork    |
| `upstream` | `https://github.com/pipecat-ai/pipecat.git` | Official repo |

Verify with:
```bash
git remote -v
```

If starting from a fresh clone of the fork, add upstream:
```bash
git remote add upstream https://github.com/pipecat-ai/pipecat.git
```

## Syncing with upstream

```bash
git fetch upstream
git rebase upstream/main   # tour-docs/ stays as top commit
git push origin main
```

`tour-docs/` is a novel directory not present in upstream, so it never conflicts.
The rebase simply replays the single `docs: add pipecat architecture tour documentation` commit
on top of whatever new commits upstream has.

## Updating tour-docs content

Edit files under `tour-docs/`, then:
```bash
git add tour-docs/
git commit -m "docs: <describe what you added or changed>"
git push origin main
```

If you have multiple tour-docs commits and want to keep history tidy before a sync:
```bash
git fetch upstream
git rebase -i upstream/main   # squash or reorder tour-docs commits as needed
git push origin main
```

## What NOT to commit

- `.mcp.json` — references a local Docker container name, not portable.

## One-liner sync alias (optional)

Add to `~/.zshrc` or `~/.bashrc`:
```bash
alias pipecat-sync='git fetch upstream && git rebase upstream/main && git push origin main'
```

Then just run `pipecat-sync` from the repo directory.
