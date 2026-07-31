# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```
npm run dev       # dev server at localhost:4321
npm run build     # production build to ./dist/
npm run preview   # preview the build locally
npm run astro ... # run any Astro CLI command (e.g. astro check)
```

No test framework, linter, or formatter is configured in this repo — don't invent one.

**Package manager**: both `package-lock.json` and `pnpm-lock.yaml` exist at the repo
root, but `.github/workflows/deploy.yml` runs `npm ci`. Treat **npm** as authoritative;
the `pnpm-lock.yaml` is stray.

## Architecture

Astro 6, static output (no SSR adapter). Every page shares one layout
(`src/layouts/BaseLayout.astro`); the only dynamic route is `src/pages/blog/[slug].astro`.

**Content is two distinct halves, don't confuse them:**
- One real Astro content collection, `blog` (glob loader over `src/content/blog/**/*.{md,mdx}`),
  defined in `src/content.config.ts`. It **must** live at that path, not
  `src/content/config.ts` — the old location throws `LegacyContentConfigError` in
  Astro v6. Entries have `.id`, not `.slug`; render via
  `import { render } from 'astro:content'`, not the old `entry.render()`.
- Four plain JSON files under `src/content/data/` (`site.json`, `projects.json`,
  `uses.json`, `courses.json`), imported directly with **no schema validation**.
  Editing site content (bio, projects, tools, courses) is a JSON edit, not a code
  change. See `.claude/projects-structure.md` for how `projects.json` entries map to
  where each project actually lives.

**Styling**: Tailwind v4, CSS-first config (no `tailwind.config.js`). Theme tokens and
`@import "tailwindcss"` live in `src/styles/global.css`. **Never use Tailwind color
utilities** (`text-gray-500` etc.) — always the CSS custom properties
(`style="color: var(--muted)"`), so the light/dark theme system stays authoritative.
Dark/light is toggled by adding/removing `.dark` on `<html>`
(`ThemeToggle.astro` + an inline pre-hydration script in `BaseLayout.astro` that reads
`localStorage` before paint, to avoid a flash of the wrong theme).

**Animation**: GSAP has no shared wrapper — the same pattern
(`gsap.registerPlugin(ScrollTrigger)`, `clearProps: 'all'`,
`scrollTrigger: { once: true }`) is repeated inline in every page that animates
(`index`, `projects`, `courses`, `uses`, `blog/index`). Keep repeating it rather than
extracting an abstraction. Separately, `ParticlesBackground.astro` is a non-GSAP
animated background (vendored `public/vendor/particles.js` + a hand-rolled canvas
cursor-trail effect, theme-aware via a `MutationObserver`, using `transition:persist`
to survive Astro View Transitions) — easy to mistake for GSAP, it isn't.

## Deployment

Deployed as a prebuilt `dist/` via GitHub Actions + rsync to Hostinger — not
Vercel/Netlify. Full details: `.claude/github-actions-deploy.md`. The wider Hostinger
account (subdomains, other Node.js apps, hard-won platform gotchas) is documented in
`.claude/hostinger-deployment.md`.

`public/.htaccess` is version-controlled and must not be blown away — it rewrites bare
paths like `/tictactoe` to `/projects/tictactoe` so the standalone mini-project demos
resolve correctly in production. It ships as part of `dist/` automatically (Astro
copies everything under `public/` verbatim, dotfiles included).

## Known issues

- `BaseLayout.astro`'s default `image` prop fallback still points at
  `/images/profile.jpg`, but the real file on disk is `profile.png` — currently a
  broken reference.
- `public/.htaccess`'s rewrite slug list is manually maintained and only covers the
  Hostinger-hosted static demo projects — it intentionally excludes projects that are
  just external links (`fita-a-fita`, `kredit-de`, etc.). If you add a *new* standalone
  demo under `miguel.janermudoy.com/projects/<slug>/`, you must add `<slug>` to this
  file's rewrite rule yourself — it's not automatic.

## See also

- `docs/HANDOFF.md` — original onboarding brief (JSON schemas, file structure, GSAP
  snippets, typography/responsive rules). Partially stale on deployment status but
  still accurate on content/styling conventions.
- `.claude/hostinger-deployment.md` — Hostinger Node.js App gotchas, applies to any
  future Node app on this account.
- `.claude/github-actions-deploy.md` — this repo's own auto-deploy pipeline.
- `.claude/projects-structure.md` — how to add a new project entry, and which of the
  three hosting shapes it should use.
- `.claude/mcp-tools-reference.md` — which MCP servers/tools were used this session
  and their sharp edges.
