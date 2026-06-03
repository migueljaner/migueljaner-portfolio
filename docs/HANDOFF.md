# Portfolio Handoff

> Fresh-agent brief. All decisions made, patterns established, gotchas documented.

---

## What This Is

Minimal B&W developer portfolio for **Miguel Janer** (Full-Stack Developer). Dark-default theme with light toggle. All content is JSON-driven. Built from scratch in one session.

**Run it:** `npm run dev` → `http://localhost:4321`
**Build:** `npm run build`

---

## Stack

| Concern | Choice | Why |
|---|---|---|
| Framework | Astro 6 | Static-first, MDX native, fast |
| Styling | Tailwind v4 | Already in user's stack |
| Animations | GSAP + ScrollTrigger | User requested |
| Blog | MDX files | Astro native, no external dep |
| Fonts | Geist (sans) + JetBrains Mono | Contrast pair, dev aesthetic |
| Theme | Dark default + toggle | User preference |
| Deploy | Out of scope | Deferred |

---

## File Structure

```
src/
├── content.config.ts          ← Astro v6 content collection config (glob loader)
├── content/
│   ├── blog/
│   │   └── *.mdx              ← Blog posts
│   └── data/
│       ├── site.json          ← Name, bio, avatar, contact, meta
│       ├── projects.json      ← All projects (personal + work)
│       ├── uses.json          ← Editor, languages, skills, techs, tools
│       └── courses.json       ← Courses + education/studies
├── layouts/
│   └── BaseLayout.astro       ← HTML shell, Nav, footer, theme script
├── components/
│   ├── Nav.astro              ← Sticky top nav, hamburger on mobile
│   ├── ThemeToggle.astro      ← Sun/moon icon button, localStorage
│   ├── ProjectCard.astro      ← Project card (image, title, desc, tags, links)
│   ├── ProjectFilter.astro    ← Client-side filter by type + tags
│   └── BlogCard.astro         ← Blog post card
├── pages/
│   ├── index.astro            ← Home: hero + featured projects + latest posts
│   ├── projects.astro         ← Projects grid + filter
│   ├── uses.astro             ← Editor / Languages / Skills / Techs / Tools
│   ├── courses.astro          ← Courses + Education timeline
│   └── blog/
│       ├── index.astro        ← Blog post list
│       └── [slug].astro       ← Individual post (MDX render)
└── styles/
    └── global.css             ← CSS vars, font imports, base styles, .prose, .tag

public/
├── images/
│   ├── profile.jpg            ← REPLACE with actual photo (CV placeholder)
│   └── placeholder-project.svg
```

---

## Theme System

**How it works:**
- CSS vars on `:root` (light) and `:root.dark` (dark)
- `BaseLayout.astro` has an inline `<script is:inline>` that runs before render, reads `localStorage.getItem('theme') ?? 'dark'` and adds the class to `<html>` → no FOUC
- `ThemeToggle.astro` toggles `.dark`/`.light` class and saves to localStorage

**CSS vars (all colors must use these — no Tailwind color classes):**
```css
--bg          /* page background */
--fg          /* primary text */
--muted       /* secondary text */
--border      /* borders, dividers */
--card-bg     /* card background */
--tag-bg      /* tag/badge background */
--tag-fg      /* tag/badge text */
--nav-bg      /* nav backdrop-blur bg */
```

**Rule:** Never use Tailwind color utilities like `text-gray-500`. Always `style="color: var(--muted);"` etc.

---

## Content System

All editable content lives in `src/content/data/`. No rebuild needed to update text — just edit JSON.

### site.json
```json
{
  "name": "Miguel Janer",
  "title": "Full-Stack Developer",
  "bio": "...",
  "avatar": "/images/profile.jpg",
  "contact": { "email": "", "github": "", "linkedin": "" },
  "meta": { "description": "", "url": "" }
}
```

### projects.json
Array of:
```json
{
  "title": "", "description": "", "tags": [],
  "url": "", "github": "",
  "image": "/images/placeholder-project.svg",
  "featured": false,
  "type": "personal|work"
}
```
`featured: true` → shown on Home page (max 3 recommended).

### uses.json
```json
{
  "editor": [{ "name": "", "description": "" }],
  "languages": [{ "name": "", "icon": "" }],
  "skills": [{ "name": "", "description": "" }],
  "techs": [{ "name": "", "category": "Frontend|Full-Stack|Backend|Styling" }],
  "tools": [{ "name": "", "description": "" }]
}
```

### courses.json
```json
{
  "courses": [{ "title": "", "author": "", "platform": "", "url": "", "completed": true, "year": "" }],
  "studies": [{ "institution": "", "location": "", "degree": "", "field": "", "start": "", "end": "", "modules": [] }]
}
```

### Blog posts (MDX)
Create `src/content/blog/my-post.mdx`:
```mdx
---
title: "Post Title"
description: "One line summary"
date: 2024-06-01
tags: ["astro", "typescript"]
draft: false
---

Post content here...
```

---

## GSAP Patterns

All animations use the same pattern — import in `<script>` tag:

```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Page load: hero entrance
gsap.from('.hero-content > *', {
  opacity: 0, y: 20, duration: 0.6, stagger: 0.1, ease: 'power2.out'
});

// Scroll reveal on cards
gsap.from(cards, {
  opacity: 0, y: 30, duration: 0.5, stagger: 0.08,
  ease: 'power2.out', clearProps: 'all',
  scrollTrigger: { trigger: container, start: 'top 85%', once: true }
});
```

**Rule:** Always `clearProps: 'all'` and `once: true` on ScrollTrigger to prevent style leaks.

---

## Astro v6 Gotchas

1. **Content config location:** Must be `src/content.config.ts` (NOT `src/content/config.ts`). Old location causes `LegacyContentConfigError`.

2. **Content Layer API (new in v5/v6):**
   - Use `glob` loader: `loader: glob({ pattern: '**/*.{md,mdx}', base: './src/content/blog' })`
   - Entries have `.id` not `.slug`
   - Render: `import { render } from 'astro:content'; const { Content } = await render(entry);`
   - Old: `entry.render()` — BROKEN in v6

3. **Tailwind v4:** Config is CSS-first (in `global.css` via `@theme {}`), no `tailwind.config.js` needed. Use `@import "tailwindcss"` not `@tailwind base/components/utilities`.

---

## Responsive Rules

- Mobile-first breakpoints: `sm:` / `md:` / `lg:`
- Nav: desktop links hidden on mobile (`hidden md:flex`), hamburger shows on mobile
- Projects grid: `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`
- Hero: stacked on mobile, `md:flex-row` for side-by-side
- All tap targets: min `h-9 w-9` (≥36px)
- Max content width: `max-w-4xl mx-auto px-6`
- Blog post content: `max-w-2xl mx-auto px-6`

---

## Typography Rules

- Body / UI text: Geist (`font-sans` in Tailwind, or default body)
- Code / labels / tags / mono accents: JetBrains Mono — use class `font-mono` or inline `font-family: var(--font-mono)`
- Blog prose: `.prose` class (defined in `global.css`) — wraps MDX `<Content />`
- Tags/badges: `.tag` class (defined in `global.css`) — uses mono font, border, bg-tag vars

---

## Things NOT Done Yet

- [ ] Real profile photo (replace `public/images/profile.jpg`)
- [ ] Real project screenshots (replace `placeholder-project.svg` per project)
- [ ] GitHub links for personal projects
- [ ] More blog posts
- [ ] Deployment config
- [ ] OG image generation
- [ ] RSS feed
- [ ] Reading time on blog posts
- [ ] Search / tag filter on blog
- [ ] `sitemap.xml` (Astro has `@astrojs/sitemap` integration)
- [ ] Analytics

---

## Suggested Skills for Next Session

- `/run` — start dev server and visually test the site before making changes
- `/verify` — confirm a feature works after implementing it
- `/grill-me` — if requirements are unclear before building a new feature
- `/code-review` — after adding new components
