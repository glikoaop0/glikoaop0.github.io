# Personal Site

Built with [Astro](https://astro.build). Dark editorial aesthetic — IBM Plex Mono + Lora.

## Setup

```bash
npm install
npm run dev       # localhost:4321
npm run build     # production build → dist/
npm run preview   # preview build locally
```

## Writing a new article

Create a file in `src/pages/articles/your-slug.md`:

```markdown
---
layout: ../../layouts/Article.astro
title: "Your Article Title"
date: 2026-06-01
tags: [Tag1, Tag2]
excerpt: "One sentence summary shown on the home page."
---

Your content here in Markdown...
```

Push to `main` → GitHub Actions builds and deploys automatically (~30s).

## Deploy (first time)

1. Push repo to GitHub
2. Go to Settings → Pages → Source: **GitHub Actions**
3. Push any commit to `main` to trigger the first deploy

## Custom domain

1. Add a `CNAME` file to `/public/` containing your domain: `yourdomain.com`
2. In your domain registrar, add a CNAME record pointing to `yourusername.github.io`
3. In GitHub Pages settings, set the custom domain and enable HTTPS

## Config

Edit `astro.config.mjs`:
- Set `site` to your URL
- If deploying to a sub-path (e.g. `github.io/repo`), add `base: '/repo'`

Edit `src/layouts/Base.astro` to update your name and links.
Edit `src/pages/about.astro` to update your bio.
