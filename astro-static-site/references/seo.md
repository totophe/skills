# SEO & Social Media Optimization Reference

## Essential Meta Tags

Every page should include these in `<head>`:

```astro
---
interface Props {
  title: string;
  description: string;
  image?: string;
  canonicalURL?: string;
}

const {
  title,
  description,
  image = '/og-default.png',
  canonicalURL = Astro.url.href,
} = Astro.props;

const siteURL = Astro.site?.toString().replace(/\/$/, '');
const imageURL = image.startsWith('http') ? image : `${siteURL}${image}`;
---

<!-- Primary Meta Tags -->
<title>{title}</title>
<meta name="title" content={title} />
<meta name="description" content={description} />
<link rel="canonical" href={canonicalURL} />

<!-- Open Graph / Facebook -->
<meta property="og:type" content="website" />
<meta property="og:url" content={canonicalURL} />
<meta property="og:title" content={title} />
<meta property="og:description" content={description} />
<meta property="og:image" content={imageURL} />

<!-- Twitter -->
<meta property="twitter:card" content="summary_large_image" />
<meta property="twitter:url" content={canonicalURL} />
<meta property="twitter:title" content={title} />
<meta property="twitter:description" content={description} />
<meta property="twitter:image" content={imageURL} />
```

## SEO Component Pattern

Create `src/components/SEO.astro`:

```astro
---
interface Props {
  title: string;
  description: string;
  image?: string;
  canonicalURL?: string;
  type?: 'website' | 'article';
  publishedDate?: string;
  noindex?: boolean;
}

const {
  title,
  description,
  image = '/og-default.png',
  canonicalURL = Astro.url.href,
  type = 'website',
  publishedDate,
  noindex = false,
} = Astro.props;

const siteURL = Astro.site?.toString().replace(/\/$/, '');
const imageURL = image.startsWith('http') ? image : `${siteURL}${image}`;
---

<title>{title}</title>
<meta name="description" content={description} />
<link rel="canonical" href={canonicalURL} />
{noindex && <meta name="robots" content="noindex, nofollow" />}

<meta property="og:type" content={type} />
<meta property="og:url" content={canonicalURL} />
<meta property="og:title" content={title} />
<meta property="og:description" content={description} />
<meta property="og:image" content={imageURL} />
<meta property="og:site_name" content={title} />

<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:url" content={canonicalURL} />
<meta name="twitter:title" content={title} />
<meta name="twitter:description" content={description} />
<meta name="twitter:image" content={imageURL} />

{publishedDate && <meta property="article:published_time" content={publishedDate} />}
```

Usage in layouts:
```astro
---
import SEO from '../components/SEO.astro';
---
<html>
  <head>
    <SEO title="My Page" description="Page description" />
  </head>
</html>
```

## Sitemap Configuration

```js
// astro.config.mjs
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://example.com',  // REQUIRED for sitemap
  integrations: [
    sitemap({
      filter: (page) => !page.includes('/draft/'),
      changefreq: 'weekly',
      priority: 0.7,
    }),
  ],
});
```

## robots.txt

Create `public/robots.txt`:
```
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap-index.xml
```

## OG Image (Default)

Place a default OG image at `public/og-default.png`:
- Recommended size: **1200x630px**
- Format: PNG or JPG
- Keep text centered (safe zone: inner 80%)

## Structured Data (JSON-LD)

For enhanced search results, add structured data:

```astro
---
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Site Name",
  "url": Astro.site,
};
---
<script type="application/ld+json" set:html={JSON.stringify(jsonLd)} />
```

## Performance Checklist (SEO impact)

- Use Astro's `<Image />` component for automatic optimization
- Lazy-load below-the-fold images
- Use `client:visible` instead of `client:load` for non-critical React components
- Minimize JavaScript — prefer Astro components over React when interactivity isn't needed
- Set explicit `width` and `height` on images to prevent CLS
