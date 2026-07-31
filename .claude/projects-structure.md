# Projects structure

`src/content/data/projects.json` drives the `/projects` page and the featured section
on `/`. Flat array, no schema validation (plain JSON import). Entry shape and the
`featured`/`type` conventions are documented in `docs/HANDOFF.md` — not repeated here.

Every entry needs a real screenshot at `public/images/projects/<slug>.png`. The
established pattern for adding a new project: navigate the live URL with the
Playwright MCP, take an actual screenshot (not a placeholder svg), save it under that
path, then add the JSON entry pointing at it.

## Three hosting shapes — know which one before adding a project

A project entry's `url` can point at one of three genuinely different setups. They
are not interchangeable, and each has its own way of getting updated:

### 1. External sites
`kredit-de`, `privacy-synatix`, `saltytours`, `safetycraft`, `hotel-cas-contador`.
Just a URL to somewhere else entirely. Nothing on this Hostinger account to maintain.
Not part of `public/.htaccess`'s rewrite list (correctly — that list is only for
locally-hosted demo paths).

### 2. Static mini-demos hosted on this Hostinger account
`tictactoe`, `flip7`, `mousefollow`, `fitnessweb`, `githubusers`, `moviesearch`,
`randomusers`, `shopcart`, `spaceinvaders`, `lunarlander`.

These live physically under `public_html/miguel/projects/<slug>/` on the server, and
are exposed at the bare path `miguel.janermudoy.com/<slug>/` via the rewrite rule in
`public/.htaccess`:
```apache
RewriteRule ^(tictactoe|flip7|...|lunarlander)(/.*)?$ projects/$1$2 [L]
```

To add a new one: build the static site locally, `rsync` the built files over SSH
into `public_html/miguel/projects/<slug>/`, then add `<slug>` to the alternation in
`public/.htaccess` and commit that change (it's version-controlled — see
`.claude/github-actions-deploy.md`). Forgetting the `.htaccess` update means the demo
is reachable at `/projects/<slug>/` but not at the shorter `/​<slug>/` path everything
else uses.

### 3. Standalone full-stack apps on their own subdomains
Currently just `fita-a-fita`: frontend at `appfitaafita.janermudoy.com`, backend API
at `apifitaafita.janermudoy.com`. These are entirely separate Hostinger Node.js Apps,
living in a separate repo (`proyecto-dam`, a sibling directory to this one), with their
own deploy process (manual archive upload through hPanel — see
`.claude/hostinger-deployment.md` for the full set of gotchas involved). Nothing about
these apps is automated through this repo's GitHub Actions workflow.
