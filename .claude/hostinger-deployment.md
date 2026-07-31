# Hostinger deployment gotchas

Hard-won knowledge from getting Node.js apps running on this Hostinger account. This
applies to **any** future Node app deployed here, not just the specific project
(`proyecto-dam`'s fita-a-fita) that surfaced it.

## Account facts

- SSH alias `hostinger-hosting` is configured in `~/.ssh/config`.
- Business Hosting plan, LiteSpeed + Passenger.
- **MongoDB is not available at all on this plan** — only MySQL. If a project needs
  MongoDB, use an external service (MongoDB Atlas free tier was used for fita-a-fita).
- No Node.js app can be provisioned over plain SSH (no `nodevenv`, CageFS blocks manual
  Node installation) — it must go through hPanel's Node.js App wizard.
- The Hostinger MCP's `hosting_createNodeJSBuildFromArchiveV1` **cannot practically be
  called directly** — its `archive` parameter rejects both local file paths and
  realistic base64 payloads ("The archive field must be a file"). In practice: have
  the user upload the zip through hPanel manually, and use the MCP's read-only tools
  (`hosting_listNodeJSBuildsV1`, `hosting_getNodeJSBuildLogsV1`) plus SSH to diagnose.

## pnpm gotchas (Hostinger's build pipeline always uses pnpm via Corepack,
## regardless of the "package manager" dropdown shown in hPanel)

Hit in this order — check for all of them up front on a new project rather than
discovering them one deploy attempt at a time:

1. **Blocked build scripts**: newer pnpm blocks dependency postinstall/build scripts
   by default (`ERR_PNPM_IGNORED_BUILDS`). Fix in `pnpm-workspace.yaml`:
   ```yaml
   allowBuilds:
     sharp: true
   onlyBuiltDependencies:
     - sharp
   ```
   The old `package.json` `"pnpm"` field (e.g. `"pnpm": {"onlyBuiltDependencies": [...]}`)
   is **no longer read at all** — silently ignored with just a warning.

2. **`minimumReleaseAge`**: a newer pnpm supply-chain-safety default that can crash
   the install pipeline outright (not just warn) when it tries to gate a recently
   published package version. Set explicitly to disable:
   ```yaml
   minimumReleaseAge: 0
   ```

3. **Executable-bit stripping** (`EACCES`/`spawnSync ... EACCES` on a package's own
   downloaded binary, e.g. `esbuild`'s postinstall): binaries get extracted/hard-linked
   from pnpm's content-addressable store *without* the execute bit. Manually
   `chmod +x`-ing the store's `*-exec` files (`~/.local/share/pnpm/store/v11/files/**/*-exec`)
   fixes it *once*, but gets silently undone before the next build attempt — this
   looks like an active security/malware-scanner process on the account
   re-stripping `+x` from files it doesn't recognize (consistent with a separate,
   unrelated observation: a file literally named `seed.xss` — a fuzz-test fixture
   containing XSS payload strings — got silently blocked from being written at all,
   suggesting a content-aware security scanner is active on the account). **Don't
   fight this with chmod — it doesn't stick.** Real fix: eliminate the native binary
   that needs to `spawnSync` itself:
   - `esbuild` → override to `npm:esbuild-wasm@<exact-version>` in
     `pnpm-workspace.yaml`'s `overrides:` (WASM, no subprocess spawn, immune to the
     exec-bit issue entirely).
   - Rollup's native bindings can separately fail with a **GLIBC version mismatch**
     (`` GLIBC_2.29' not found ``, a genuine ABI incompatibility with the server's
     Node build, unrelated to the exec-bit issue) → override `rollup` to
     `npm:@rollup/wasm-node@<exact-version>`.
   - `sharp` has no WASM escape hatch, but its postinstall has worked fine so far —
     it uses `require()`/dlopen to load a native `.node` addon, not a subprocess
     spawn, so it isn't exposed to the exec-bit problem the same way `esbuild` is.
     Before assuming you need `sharp` at runtime, check whether the project actually
     uses `astro:assets`/`<Image>` (build-time image optimization) — if not, `sharp`
     is only needed at build time and its presence/absence at runtime doesn't matter.
   - Example working override block:
     ```yaml
     overrides:
       esbuild: npm:esbuild-wasm@0.25.12
       rollup: npm:@rollup/wasm-node@4.62.3
     ```

4. **Archive size limit**: the Node build tool caps uploads at 50MB. Shipping your own
   pre-built `node_modules` to dodge a build step isn't viable for anything beyond a
   trivial app — a real frontend's `node_modules` was 215MB here.

## The entry-file gotcha that cost the most time

When "Directorio de salida" (output directory) in hPanel is set to e.g. `dist`, the
"Archivo de entrada" (entry file) must be given **relative to that output directory**
— e.g. `server/entry.mjs` — **not** relative to the project root
(`dist/server/entry.mjs`). Using the project-root-relative path silently resolves
wrong: the build itself succeeds, but the app is never promoted/activated (see next
section). This is a documented pattern for deploying Astro on Hostinger specifically,
not something reverse-engineered blind.

## "Build succeeded but state stays `failed`" symptom

Check `hosting_listNodeJSBuildsV1` — a build can have a **100% clean log** (every
dependency installed, `astro build` completes, "Complete!") and still be marked
`"state": "failed"` at the API level, with zero informative error anywhere: not in the
build log, not in any SSH-visible file, and no runtime log ever gets created. This
means the *promotion* step — copying the successful build into
`.builds/versions/<id>/` and writing the live `public_html/.htaccess` Passenger config
— silently never ran.

**How to diagnose**: compare `~/domains/<domain>/.builds/versions/` over SSH between
a known-working app and the failing one. An empty `versions/` directory on the failing
app (while the working one has a populated `<version-id>/public_html/.htaccess` +
`<version-id>/nodejs/` folder) confirms promotion never fired — no point debugging the
build itself further.

One concrete cause found this way: Astro's default `output` mode is `"static"` **even
with an adapter configured**, unless you explicitly set `output: 'server'` in
`astro.config.mjs`. Set it explicitly, and confirm the build log actually prints
`[build] output: "server"` (not `"static"`) before assuming the app is correctly SSR.

## App type behavior (hPanel's "tipo de aplicación")

- **Express** app type has **no build-command field at all** in the UI — it assumes
  pre-built/plain Node code with no separate build step. If a framework needs a real
  build step (Astro does), don't switch to Express purely to dodge an app-type quirk
  without first checking Express actually supports what you need — it likely doesn't.
- `package.json`'s own `"start"` script matters for the promotion step to work:
  ```json
  "scripts": { "start": "node ./dist/server/entry.mjs" }
  ```
  Mirror whatever convention the working app on the account already uses (a plain
  Express backend here uses `"start": "node dist/server.js"`).

## Backend static assets get silently dropped by a `tsc`-only build

If a backend's build script is just `"build": "tsc"`, static assets under `src/`
(e.g. `src/public/img/`) are **never copied to `dist/`** — `tsc` only touches `.ts`
files. Symptom: `express.static(path.join(__dirname, "public"))` 500s in production
(not a graceful 404) because `dist/public` doesn't exist at all. Fix:
```json
"build": "tsc && cp -r src/public dist/public"
```

## Version-pinning discipline

Caret ranges (`^1.2.3`) in `package.json` let Hostinger's fresh pnpm install resolve
**newer** versions than whatever was tested locally — different `@types/express`,
different `stripe`, even a different **TypeScript compiler** version (this one
changed real type-narrowing behavior for `req.params`, causing build failures that
only reproduced on Hostinger, never locally). Once a local build is verified working,
pin every dependency to the exact resolved version (no `^`/`~`) to eliminate this
whole class of environment drift between "works on my machine" and Hostinger's fresh
install.

## No env-var-setting API

No tool anywhere in the Hostinger MCP surface sets environment variables for a Node
app (confirmed by searching for it twice). Workaround used here: bundle a production
`.env` file directly into the deploy archive. `dotenv` reads it from the app's own
root at runtime; it's excluded from git via `.gitignore`; it's not web-exposed since
Express only serves the `public/` subfolder, not its own root directory.

## Static (non-Node) demos are a different, simpler path

Plain static mini-project demos don't need any of the above — they're just `rsync`'d
over SSH straight into `public_html/miguel/projects/<slug>/`. No Hostinger Node App,
no pnpm, no build pipeline involved at all. See `.claude/projects-structure.md`.
