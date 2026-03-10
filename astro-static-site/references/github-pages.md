# GitHub Pages Deployment Reference

## astro.config.mjs for GitHub Pages

```js
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  // For project repos: https://<username>.github.io/<repo-name>
  site: 'https://<username>.github.io',
  base: '/<repo-name>',  // omit for <username>.github.io repos or custom domains

  output: 'static',
  integrations: [react(), sitemap()],
});
```

## GitHub Actions Workflow

File: `.github/workflows/deploy.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout your repository using git
        uses: actions/checkout@v5
      - name: Install, build, and upload your site
        uses: withastro/action@v5

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## GitHub Repository Settings

1. Go to **Settings > Pages**
2. Under "Build and deployment", select **GitHub Actions** as the source
3. The workflow will deploy automatically on push to `main`

## Custom Domain

1. Add a `public/CNAME` file with your domain:
   ```
   www.example.com
   ```
2. Configure DNS records at your domain registrar
3. Update `site` in `astro.config.mjs` to match the custom domain
4. Remove the `base` option (not needed with custom domains)

## Dev Container Setup

The project uses the `totophe/dc-toolbelt` dev container images. Create `.devcontainer/devcontainer.json`:

### Using the Astro-specific image

```json
{
  "name": "Astro Dev",
  "image": "ghcr.io/totophe/dc-toolbelt:node24-astro",
  "forwardPorts": [4321],
  "customizations": {
    "vscode": {
      "extensions": [
        "astro-build.astro-vscode",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "bradlc.vscode-tailwindcss"
      ]
    }
  }
}
```

### Using the full toolbox image

```json
{
  "name": "Astro Dev",
  "image": "ghcr.io/totophe/dc-toolbelt:node24-toolbox",
  "forwardPorts": [4321],
  "mounts": [
    "source=dc-toolbelt-config,target=/home/node/.dc-toolbelt,type=volume"
  ],
  "customizations": {
    "vscode": {
      "extensions": [
        "astro-build.astro-vscode",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "bradlc.vscode-tailwindcss"
      ]
    }
  }
}
```

Available images from `totophe/dc-toolbelt`:
- `node24-astro` — Lightweight, Astro-focused
- `node24-toolbox` — Full toolbox with cloud CLIs, IaC tools, AI tools
- `node24-toolbox-code` — Code/framework tools without cloud CLIs
- `node24` — Base Node.js image
