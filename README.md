# Danilo Technical Portfolio

Technical portfolio and writing website built with [Astro](https://astro.build/)
and published with GitHub Pages at `https://ndanilo.github.io`.

## Stack

- Astro
- TypeScript
- Markdown content collections
- GitHub Pages
- GitHub Actions

## Requirements

- Node.js 24 or newer
- npm 11 or newer

Check local versions:

```powershell
node --version
npm --version
```

## Setup

Install dependencies:

```powershell
npm install
```

Start the local development server:

```powershell
npm run dev
```

Build the production site:

```powershell
npm run build
```

Preview the production build locally:

```powershell
npm run preview
```

## Scripts

- `npm run dev` starts Astro locally.
- `npm run check` checks Astro and TypeScript files.
- `npm run build` validates and builds the static site into `dist/`.
- `npm run preview` serves the built site locally.

## Project Structure

```text
.
├── .github/workflows/deploy.yml
├── public/
├── src/
│   ├── content/
│   │   ├── posts/
│   │   └── projects/
│   ├── layouts/
│   ├── pages/
│   └── styles/
├── astro.config.mjs
├── package.json
└── tsconfig.json
```

## Content

Posts live in `src/content/posts/`.

```md
---
title: "Post Title"
description: "Short summary used on listing pages and search previews."
publishDate: 2026-07-10
tags: ["astro", "portfolio"]
draft: false
---

Post content goes here.
```

Projects live in `src/content/projects/`.

```md
---
title: "Project Name"
description: "Short project summary."
role: "Your role"
stack: ["Astro", "TypeScript"]
link: "https://example.com"
featured: true
---

Project content goes here.
```

Set `draft: true` on a post to hide it from public pages. Set
`featured: true` on a project to show it on the homepage.

## Deployment

The GitHub Actions workflow in `.github/workflows/deploy.yml` builds and
publishes the site when changes are pushed to `main`.

Deployment steps:

1. Install dependencies with `npm ci`.
2. Run `npm run build`.
3. Publish the generated `dist/` folder to GitHub Pages.

GitHub Pages should be configured to use **GitHub Actions** as the source.