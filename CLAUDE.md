# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start local dev server
npm run build        # Production build
npm run build:cached # Build with cached Notion content (faster)
npm run preview      # Preview production build

# Code quality
npm run lint         # ESLint for .js/.ts/.astro files
npm run format       # Prettier formatting

# Notion data caching
npm run cache:fetch  # Fetch and cache Notion blog content to tmp/
npm run cache:purge  # Clear all caches (nx + tmp/)
```

There are no tests defined in this project.

## Architecture

This is a **Notion-powered Astro blog** that fetches content from a Notion database and generates a static site.

### Data Flow

1. Build fetches posts from Notion database via `@notionhq/client` (filtering by `Published` checkbox and `Date <= now`)
2. Block content is fetched recursively and cached to `tmp/{blockId}.json`
3. Images are downloaded locally, EXIF-stripped, and resized via `sharp`
4. Astro generates static HTML; interactive components use React (client-side hydration)
5. API routes (`/api/likes.json`) run server-side on Vercel

### Key Files

- `src/server-constants.ts` — All env vars and site-wide constants (posts per page, timeouts)
- `src/lib/notion/client.ts` — Entire Notion API integration: fetching posts, blocks, incrementing likes
- `src/lib/notion/interfaces.ts` — TypeScript types for all Notion data structures
- `src/layouts/Layout.astro` — Root HTML layout with OG meta, sidebar slot, GA
- `src/pages/posts/[slug].astro` — Post detail page; uses `getStaticPaths()` for SSG
- `src/pages/api/likes.json.ts` — GET/POST API for like counts (server-rendered, `prerender = false`)

### Rendering Strategy

- `output: 'server'` with Vercel adapter — most pages are SSR by default
- Post pages use `export const prerender = true` (or `getStaticPaths`) for static generation
- API routes (`/api/`) always use server-side rendering

### Notion Block Rendering

Each Notion block type has its own component in `src/components/notion-blocks/`. Text annotations (bold, italic, code, etc.) are in `src/components/notion-blocks/annotations/`. The top-level dispatcher is `src/components/NotionBlocks.astro`.

### LikeButton

`src/components/LikeButton.tsx` is a React component that:

- Checks `localStorage` for `liked:{slug}` to prevent double-liking
- Makes GET to `/api/likes.json?slug=` on mount
- Makes POST to `/api/likes.json?slug=` to increment
- Uses optimistic UI (increments immediately, disables button)

In `[slug].astro` it is rendered with `client:visible` for lazy hydration.

### Environment Variables

| Variable             | Required | Description              |
| -------------------- | -------- | ------------------------ |
| `NOTION_API_SECRET`  | Yes      | Notion integration token |
| `DATABASE_ID`        | Yes      | Notion database ID       |
| `CUSTOM_DOMAIN`      | No       | e.g. `example.com`       |
| `BASE_PATH`          | No       | Sub-directory path       |
| `REQUEST_TIMEOUT_MS` | No       | Default: 10000ms         |

### Custom Astro Build Integrations (astro.config.mjs)

Four custom integrations run at build time:

- `CoverImageDownloader` — Downloads database cover image
- `FeaturedImageDownloader` — Downloads all post featured images
- `CustomIconDownloader` — Downloads custom post icons
- `PublicNotionCopier` — Copies downloaded assets to the output directory
