# GitHub Actions → Hostinger deploy (this repo)

`.github/workflows/deploy.yml` — on every push to `main`:

1. `npm ci && npm run build`
2. Loads an SSH key (`webfactory/ssh-agent`) and trusts the Hostinger host key.
3. `rsync`'s `dist/` to the Hostinger path over SSH.

Required repo secrets: `HOSTINGER_SSH_KEY`, `HOSTINGER_HOST`, `HOSTINGER_PORT`,
`HOSTINGER_USER`, `HOSTINGER_PATH`.

## Why the rsync has no `--delete`

Deliberate. Without `--delete`, the sync only adds/updates files — it never removes
anything that isn't part of this repo's own `dist/` output. That specifically
protects:

- The manually-deployed static demo project subfolders under
  `.../public_html/miguel/projects/` (`tictactoe`, `spaceinvaders`, etc.) — these are
  not tracked in this repo and would be silently deleted by a `--delete` sync.
- The separately-hosted fita-a-fita Node.js apps (different subdomains, different
  repo entirely — see `.claude/hostinger-deployment.md`).

Trade-off accepted: if a page is ever removed from this Astro site, its old built file
lingers on the server forever (harmless, just not tidy). Don't add `--delete` without
first solving how to protect the untracked directories some other way (e.g. an
explicit `--exclude=projects/`).

## Why `public/.htaccess` is committed

It's version-controlled specifically so the Hostinger rewrite rules aren't silently
lost or overwritten by a future deploy. Astro copies everything under `public/`
verbatim into `dist/` (dotfiles included), so it ships automatically with every build
— no special-casing needed in the workflow itself.

## GitHub MCP write-tool caveat

`push_files`/`create_or_update_file` (GitHub MCP) returned a 403 the first time they
were tried against this repo, right after it was created — likely a permissions-sync
delay on a brand-new repo, not a real access problem. Plain local `git push` worked
fine throughout and is the reliable fallback if the MCP write path acts up again.
