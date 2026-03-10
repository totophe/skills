---
name: astro-static-site
description: "Scaffold and configure static websites with Astro, React, TypeScript, and TailwindCSS, deployed to GitHub Pages via GitHub Actions. Use when creating a new static site, initializing an Astro project, setting up GitHub Pages deployment, adding SEO/social meta tags, configuring dev containers with dc-toolbelt, or when the user mentions building a static website, landing page, or marketing site."
license: MIT
metadata:
  version: "1.0.0"
  author: totophe
---

# Astro Static Site

Build static websites with Astro + React + TypeScript + TailwindCSS, deployed to GitHub Pages.

## Initialization Workflow

Astro must be created in an empty directory. For existing repos (the usual case), use `/tmp`:

```bash
cd /tmp
npm create astro@latest my-site -- --template minimal --typescript strictest
cp -a /tmp/my-site/. /path/to/repo/
cd /path/to/repo
npx astro add react
npx astro add tailwind
npx astro add sitemap
```

After initialization, set up the project structure:

1. **Create required directories**: `docs/`, `mockups/` (for documentation and Google Stitch mockups)
2. **Copy asset templates** from this skill's `assets/` directory:
   - `assets/deploy.yml` → `.github/workflows/deploy.yml`
   - `assets/devcontainer.json` → `.devcontainer/devcontainer.json`
   - `assets/SEO.astro` → `src/components/SEO.astro`
   - `assets/Layout.astro` → `src/layouts/Layout.astro`
   - `assets/robots.txt` → `public/robots.txt` (replace `__SITE_URL__` with actual site URL)
3. **Configure `astro.config.mjs`** with `site` and `base` for GitHub Pages
4. **Add a default OG image** at `public/og-default.png` (1200x630px)

## Configuration

### astro.config.mjs

```js
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://<username>.github.io',
  base: '/<repo-name>',  // omit for user sites or custom domains
  output: 'static',
  integrations: [react(), sitemap()],
});
```

### Dev Container

The project runs from `totophe/dc-toolbelt`. Available images:
- `ghcr.io/totophe/dc-toolbelt:node24-astro` — lightweight, Astro-focused
- `ghcr.io/totophe/dc-toolbelt:node24-toolbox` — full toolbox with cloud CLIs

Port 4321 is forwarded for the Astro dev server.

## Key Patterns

### Pages use the Layout component with SEO props
```astro
---
import Layout from '../layouts/Layout.astro';
---
<Layout title="Page Title" description="Page description">
  <main><!-- content --></main>
</Layout>
```

### React components need client directives
```astro
<MyComponent client:load />      <!-- immediate -->
<MyComponent client:visible />   <!-- when scrolled into view (preferred) -->
```

### Tailwind v4 — no config file needed
Configuration is done in CSS via `@import "tailwindcss"` in `src/styles/global.css`.

### GitHub Pages deployment
- Workflow at `.github/workflows/deploy.yml` uses `withastro/action@v5`
- Set repo Settings > Pages > Source to **GitHub Actions**
- For custom domains: add `public/CNAME`, update `site`, remove `base`

## SEO Checklist

- [ ] `site` set in `astro.config.mjs`
- [ ] SEO component used in all layouts with title + description
- [ ] Default OG image at `public/og-default.png` (1200x630)
- [ ] `public/robots.txt` with sitemap URL
- [ ] Sitemap integration enabled
- [ ] Structured data (JSON-LD) for key pages
- [ ] `<Image />` component for optimized images
- [ ] `client:visible` preferred over `client:load` for non-critical components

## References

- `references/astro-setup.md` — Detailed Astro project setup, integrations, content collections, view transitions
- `references/github-pages.md` — GitHub Pages deployment, GitHub Actions workflow, dev container configuration
- `references/seo.md` — SEO meta tags, OG images, structured data, performance checklist

## Mockups

When mockups exist in the `mockups/` folder (typically Google Stitch exports), read them to understand the target design before building pages. Use them as the source of truth for layout, colors, and component structure.
