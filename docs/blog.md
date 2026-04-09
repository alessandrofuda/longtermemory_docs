# Blog — Astro Content Site

The [LongTermMemory](https://longtermemory.com) blog is a static content marketing site built with Astro v5. It serves content about spaced repetition, learning science, AI-powered education, and memory techniques.

It is deployed at [longtermemory.com/b/](https://longtermemory.com/b/) as a subdirectory of the main domain.

## Tech Stack

| Tool | Purpose |
|------|---------|
| Astro v5 | Static site generation |
| Tailwind CSS v3 | Utility-first styling (purple/gradient theme) |
| Content Collections + Zod | Type-safe content with schema validation |
| Fuse.js | Client-side full-text search |
| Shiki | Syntax highlighting (GitHub Dark theme) |

---

## Development Commands

```bash
cd blog

npm install          # Install dependencies
npm run dev         # Dev server → http://localhost:4321/b/
npm run build       # Production build → dist/
npm run preview     # Preview production build locally
```

> **Note**: No test suite is configured for this project.

---

## Configuration

Key settings in `astro.config.mjs`:

- **Site URL**: `https://longtermemory.com`
- **Base path**: `/b` — all routes and internal links are prefixed with `/b`
- **Syntax highlighting**: GitHub Dark via Shiki
- **Prefetch**: enabled for faster navigation

---

## Routing

File-based routing under `src/pages/`:

| Route | File | Purpose |
|-------|------|---------|
| `/b/` | `index.astro` | Homepage (featured + all posts) |
| `/b/{slug}/` | `[...slug].astro` | Individual blog post |
| `/b/categories/` | `categories/index.astro` | All categories |
| `/b/categories/{category}/` | `categories/[category].astro` | Posts by category |
| `/b/about/` | `about.astro` | About page |
| `/b/rss.xml` | `rss.xml.js` | RSS feed |
| `/b/sitemap-index.xml` | (auto-generated) | Sitemap — build only |

---

## Content Management

Content is managed via Astro Content Collections with strict Zod schemas defined in `src/content.config.ts`.

### Adding a Blog Post

1. Create `src/content/blog/your-slug.md`
2. Add required frontmatter:

```yaml
---
title: "Your Post Title"           # max 100 chars
description: "Post description"    # max 160 chars
pubDate: 2026-01-15
author: "Author Name"
authorSlug: "author-slug"          # must match a file in src/content/authors/
category: "Spaced Repetition"
tags: ["memory", "studying", "ai"]
# Optional fields:
updatedDate: 2026-02-01
featured: false
draft: false
---
```

3. Write content in Markdown below the frontmatter
4. The filename (without `.md`) becomes the URL slug automatically

**Draft and scheduled posts**: Posts with `draft: true` are never published. Posts with a future `pubDate` are excluded from the build until the server rebuilds on or after that date.

### Adding an Author

1. Create `src/content/authors/author-slug.json`:

```json
{
  "name": "Author Name",
  "slug": "author-slug",
  "bio": "Short author bio.",
  "avatar": "/images/authors/author-slug.jpg",
  "role": "Content Writer",
  "twitter": "https://twitter.com/...",
  "linkedin": "https://linkedin.com/in/..."
}
```

2. Reference the author in posts via `authorSlug: "author-slug"`

Author bios and social links are displayed inline at the bottom of each blog post. There are no dedicated author pages.

---

## Import Aliases

```typescript
import BlogCard from '@components/BlogCard.astro';
import Layout from '@layouts/BaseLayout.astro';
import { formatDate } from '@utils';         // bare import
import type { Post } from '@types';          // bare import
```

| Alias | Resolves to |
|-------|-------------|
| `@components` | `src/components/` |
| `@layouts` | `src/layouts/` |
| `@utils` | `src/utils/index.ts` |
| `@types` | `src/types/index.ts` |
| `@content` | `src/content/` |
| `@styles` | `src/styles/` |

---

## Components

All components are in `src/components/` using `.astro` format:

| Component | Purpose |
|-----------|---------|
| `SEO.astro` | Meta tags, Open Graph, Twitter Card, structured data |
| `Header.astro` | Navigation with search trigger (Cmd/Ctrl + K) |
| `Footer.astro` | Site footer |
| `BlogCard.astro` | Post preview card |
| `SearchModal.astro` | Full-text search via Fuse.js |
| `TableOfContents.astro` | Auto-generated from H2–H4 headings |
| `SocialShare.astro` | Share buttons (Twitter, LinkedIn, Facebook, Reddit, Email) |
| `AuthorCard.astro` | Author bio with social links |
| `Newsletter.astro` | Email subscription form |

---

## SEO

The blog has comprehensive SEO built in:

- **Sitemap**: auto-generated at build time (`sitemap-index.xml` + `sitemap-0.xml`)
  - Priorities: Homepage (1.0) > Posts (0.9) > Categories index (0.7) > Category pages (0.6) > Static pages (0.5)
- **Structured data**: `BlogPosting` schema on post pages, `BreadcrumbList` on all pages, `Organization` schema in `SEO.astro`
- **Meta tags**: Open Graph, Twitter Card, canonical URL, language/geo, author, category
- **RSS feed**: auto-generated at `/b/rss.xml`

---

## Utility Functions (`src/utils/index.ts`)

| Function | Description |
|----------|-------------|
| `calculateReadingTime(content)` | Word-count based (200 WPM) |
| `extractTableOfContents(content)` | Parses H2–H4 headings |
| `getRelatedPosts(posts, currentPost, limit)` | Tag-based similarity |
| `generateBlogPostStructuredData(post, authorName, siteUrl)` | JSON-LD |
| `generateShareUrls(url, title)` | Social sharing URLs |

---

## Deployment

### Build and Deploy

```bash
# Copy environment template
cp .env.example .env
# Edit .env with deployment credentials (DEPLOY_USER, DEPLOY_HOST, DEPLOY_BLOG_PATH)

# Deploy (commits changes, SSHes into server, builds remotely)
./deploy.sh
```

The deploy script:
1. Commits and pushes local changes to git
2. SSHes into the server, pulls the latest code
3. Runs `npm install` and `npm run build`
4. The `dist/` directory is served directly from the configured server path

### Scheduled Publishing

Since the build happens server-side, a daily cron job can trigger a rebuild to publish date-scheduled posts automatically — no code changes required. Posts whose `pubDate` is today will appear in the next build.

A dedicated `cron-build.sh` script (on the server) handles this:
- Sources `nvm` so `npm` is available in the cron environment
- Skips the build if no post is scheduled for today
- Logs output to a rotating log file

Example crontab entry (on the server):
```bash
MAILTO=your@email.com
# Rebuild blog daily at 14:00 UTC
0 14 * * * /path/to/cron-build.sh
```
