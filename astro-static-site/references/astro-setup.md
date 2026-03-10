# Astro Project Setup Reference

## Creating the Project

Astro must be initialized in an empty directory. For existing repos, create in `/tmp` then migrate:

```bash
# Create in /tmp to avoid conflicts with existing repo files
cd /tmp
npm create astro@latest my-site -- --template minimal --typescript strictest
# Copy everything back to repo root
cp -a /tmp/my-site/. /path/to/repo/
```

**Node.js requirement:** v22.12.0+. Odd-numbered versions (v23) not supported.

## Adding Integrations

```bash
# React
npx astro add react

# Tailwind CSS v4 (uses Vite plugin, NOT the deprecated @astrojs/tailwind)
npx astro add tailwind

# Sitemap
npx astro add sitemap
```

### React Manual Setup (if CLI fails)

```bash
npm install @astrojs/react react react-dom @types/react @types/react-dom
```

Add to `astro.config.mjs`:
```js
import react from '@astrojs/react';
export default defineConfig({
  integrations: [react()],
});
```

Add to `tsconfig.json`:
```json
{ "compilerOptions": { "jsx": "react-jsx", "jsxImportSource": "react" } }
```

### Tailwind CSS v4

No `tailwind.config.js` needed. Configuration is done via CSS.

The CLI creates `src/styles/global.css`:
```css
@import "tailwindcss";
```

Import in layouts:
```astro
---
import "../styles/global.css";
---
```

## Project Structure

```
/
├── public/              # Static assets (favicon, CNAME, robots.txt)
├── src/
│   ├── components/      # Astro & React components
│   ├── content/         # Content collections (markdown, MDX)
│   ├── layouts/         # Page layouts
│   ├── pages/           # File-based routing
│   └── styles/          # Global CSS
│       └── global.css   # Tailwind entry point
├── docs/                # Project documentation
├── mockups/             # UI mockups (Google Stitch exports, etc.)
├── astro.config.mjs
├── tsconfig.json
└── package.json
```

## Content Collections

Define in `src/content.config.ts`:

```ts
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

const pages = defineCollection({
  loader: glob({ pattern: "**/*.md", base: "./src/content/pages" }),
  schema: z.object({
    title: z.string(),
    description: z.string().optional(),
  }),
});

export const collections = { pages };
```

Query:
```ts
import { getCollection, getEntry } from 'astro:content';
const allPages = await getCollection('pages');
```

## View Transitions

Add to layout `<head>`:
```astro
---
import { ClientRouter } from "astro:transitions";
---
<head>
  <ClientRouter />
</head>
```

Directives: `transition:animate="fade"`, `transition:persist`, `transition:name="hero"`.

## React Components in Astro

Use client directives to hydrate React components:
```astro
---
import Counter from '../components/Counter';
---
<Counter client:load />      <!-- Hydrate immediately -->
<Counter client:idle />      <!-- Hydrate when browser is idle -->
<Counter client:visible />   <!-- Hydrate when visible in viewport -->
```

Prefer `client:visible` for below-the-fold components (better performance).
