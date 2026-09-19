# Little Vitruvius (Le Petit Vitruve)

A bilingual (**French by default**, English) architecture wiki to learn vocabulary through structured pages and simple vector diagrams, and to find each element in real buildings.

Example page: `/fenetre-serlienne` (FR) and `/en/serlian-window` (EN) show the description, aliases (*Venetian window*), an annotated SVG, tags (`style/palladian`, `region/europe`), how to spot the element, and example buildings.

The site is **fully static**: content is validated and compiled at build time, pages are pre-rendered, and the result is deployed to a CDN. There is no backend and no database.

---

## 1. Architecture at a glance

```
content/ (YAML + MDX + SVG + images, versioned in Git)      <- source of truth
        |  scripts/build-content.ts  (Zod validation, generation)
        v
src/generated/  (JSON indexes, per-element JSON)  +  public/_redirects, sitemap, optimized images
        |
        v
Vite + React + TS + MUI  --vite-react-ssg-->  dist/ (pre-rendered HTML per page, FR + EN)
        |
        v
Pagefind indexes dist/  -->  client-side search
        |
        v
Cloudflare Pages (or Netlify / GitHub Pages)
```

**Key decisions**

| Topic | Decision |
|---|---|
| Content | Authored as files in `content/`, reviewed through PRs. |
| Type safety | Zod schemas in `src/content-schema/` are the contract. They validate content at build time and give TS types via `z.infer`. |
| Rendering | Static site generation with `vite-react-ssg`, so every page ships as real HTML (good for SEO and first paint). |
| Search | Pagefind, built from the generated HTML. It is scoped per language through `<html lang>`. |
| Stable IDs | Every element/building has a language-neutral `id` (`serlian-window`). It never changes; relations, study paths, quiz and redirects reference it. |
| i18n | Content translations live in the content files; UI strings in `src/locales/{fr,en}.json`. FR is un-prefixed, EN lives under `/en`. Slugs are localized. |
| Anonymous state | Quiz progress and preferences use `localStorage`. No accounts. |
| Backend | None. See section 10 for when to add one. |

---

## 2. Repository layout

```
.
├── AGENTS.md
├── README.md
├── package.json
├── vite.config.ts                  # react-swc, mdx, ssg options
├── tsconfig.json
├── .github/workflows/deploy.yml
├── content/
│   ├── elements/
│   │   └── serlian-window/
│   │       ├── element.yaml        # id, slugs, aliases, tags, relations, parts, examples
│   │       ├── fr.mdx              # default language
│   │       ├── en.mdx
│   │       └── diagram.svg         # named <g id="..."> parts, no text labels
│   ├── buildings/
│   │   └── basilica-palladiana/
│   │       ├── building.yaml
│   │       └── images/             # source images (optimized at build)
│   ├── tags/
│   │   └── style/palladian.yaml    # labels + descriptions per locale
│   └── paths/
│       └── renaissance-facades.yaml  # study path: ordered element ids
├── scripts/
│   ├── build-content.ts            # validate + generate (run with tsx)
│   └── optimize-images.ts          # sharp: resize + WebP
├── public/                         # static assets; _redirects and media/ are generated
└── src/
    ├── main.tsx                    # ViteReactSSG entry
    ├── routes.tsx                  # FR (un-prefixed) + /en routes, getStaticPaths
    ├── content-schema/             # Zod schemas (shared by scripts and app)
    ├── generated/                  # GENERATED and git-ignored
    │   ├── elements.index.json     # light index for glossary, search fallback, slug map
    │   ├── elements/<id>.json
    │   ├── buildings.json
    │   ├── tags.json
    │   └── paths.json
    ├── locales/{fr,en}.json
    ├── features/
    │   ├── elements/               # ElementPage, AnnotatedDiagram, TagChips
    │   ├── glossary/
    │   ├── buildings/              # building pages + map
    │   ├── paths/
    │   ├── quiz/
    │   └── search/                 # Pagefind wrapper
    ├── components/                 # shared MUI-based components
    └── lib/                        # content loaders, slug helpers, i18n, seo
```

---

## 3. Content model

`content/elements/serlian-window/element.yaml`:

```yaml
id: serlian-window
category: opening            # opening | support | roof | ornament | structure
slug:
  fr: fenetre-serlienne
  en: serlian-window
name:
  fr: Fenêtre serlienne
  en: Serlian window
aliases:
  fr: [fenêtre vénitienne, fenêtre palladienne]
  en: [Venetian window, Palladian window]
tags: [style/palladian, region/europe, period/renaissance, part/opening]
relations:
  - { type: often-with, target: pediment }   # variant-of | part-of | see-also | often-with
parts:                        # SVG legend; ids must match <g id> in diagram.svg
  - id: central-arch
    label: { fr: Arc central, en: Central arch }
    definition: { fr: "…", en: "…" }
examples: [basilica-palladiana]              # building ids
```

- `fr.mdx` / `en.mdx` hold the prose: definition, characteristics, history, and a **"How to spot it"** section (recognition tips, common confusions).
- French is mandatory; English is optional. A missing English page is generated from the French text with a "not yet translated" banner, `noindex`, and a canonical link to the FR page.
- Building images carry `credit`, `licence` and `source_url` in `building.yaml`; the build fails if one is missing.

---

## 4. Build pipeline

`npm run content:build` (runs automatically before `dev` and `build`):

1. Reads and validates everything in `content/` with Zod. Any error stops the build with a file path and message.
2. Checks cross-references: tags exist, relation and example targets exist, slugs and aliases are unique per locale, every SVG part id in `element.yaml` exists in `diagram.svg` and vice versa.
3. Generates `src/generated/*.json` (index plus one JSON per element).
4. Generates `public/_redirects` (alias slugs -> canonical slugs, other-locale slugs -> the right locale) and `sitemap.xml` with `hreflang` alternates.
5. Optimizes building images to WebP into `public/media/`.

MDX bodies and SVGs are loaded lazily with `import.meta.glob` (MDX compiled by `@mdx-js/rollup`, SVG imported as raw text and inlined by `AnnotatedDiagram`).

`npm run build` then runs `vite-react-ssg build` (pre-renders every route from `getStaticPaths`) and `pagefind --site dist`.

---

## 5. Routing and i18n

| Page | FR (default) | EN |
|---|---|---|
| Home | `/` | `/en` |
| Element | `/fenetre-serlienne` | `/en/serlian-window` |
| Alias redirect | | `/en/venetian-window` -> `/en/serlian-window` |
| Tag | `/etiquette/style/palladien` | `/en/tag/style/palladian` |
| Glossary | `/glossaire` | `/en/glossary` |
| Map | `/carte` | `/en/map` |

- Routes are generated from the content: `getStaticPaths` lists every slug per locale.
- Aliases are redirected by the host through `_redirects` (Cloudflare Pages and Netlify support it). GitHub Pages does not, so the 404 page falls back to a client-side lookup in the slug map.
- The language switcher links to the equivalent page in the other locale, via the element `id`.
- Each page sets `<html lang>`, `hreflang` alternates and a canonical URL.

---

## 6. Frontend stack

- Vite + React + TypeScript, SWC plugin, `vite-react-ssg`
- MUI + Emotion, `next-themes` for light/dark
- `react-router-dom` with locale-aware routes
- `react-i18next` for UI strings (FR default, EN fallback to FR)
- `react-error-boundary` around each route
- MapLibre GL for the map of example buildings
- Pagefind for search
- `AnnotatedDiagram`: inlines the SVG, highlights `<g id>` parts on hover or tap, shows translated labels, and renders an accessible legend list
- No Axios, no React Query and no OpenAPI codegen: there is no server state. Data comes from typed loaders over the generated JSON.

**Known risks**

- **MUI/Emotion with static rendering.** Emotion styles must be extracted at render time to avoid a flash of unstyled content. Set this up in the first sprint with a spike page, and check the built HTML before adding more features.
- **Map tiles.** Do not rely on the public OpenStreetMap tile servers in production. Use a tile provider with a free tier or self-hosted PMTiles.
- **Search in dev.** Pagefind indexes built HTML, so use `npm run build && npm run preview` to test search.

---

## 7. Getting started

```bash
npm install
npm run dev
```

### Commands

```bash
npm run dev              # content:build, then Vite dev server
npm run build            # content:build, SSG build, Pagefind index
npm run preview          # serve dist/ (needed to test search)
npm run content:build    # validate + generate only
npm run content:validate # validate only, no output written
npm run lint
npm run typecheck
npm run test
```

### Adding a new element

1. Create `content/elements/<id>/` with `element.yaml`, `fr.mdx`, optional `en.mdx`, and `diagram.svg`.
2. Run `npm run content:validate`.
3. Open a PR. CI validates content, builds the site and deploys a preview.

Docker is optional; it is not needed in production. If you want dev parity, a single Node service is enough:

```yaml
services:
  web:
    image: node:lts
    working_dir: /app
    volumes: [".:/app"]
    command: sh -c "npm ci && npm run dev -- --host"
    ports: ["5173:5173"]
```

---

## 8. Deployment

Cloudflare Pages (recommended, supports `_redirects` and custom headers):

- Build command: `npm run build`
- Output directory: `dist`

`.github/workflows/deploy.yml` (outline):

```yaml
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: lts/*, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck && npm run test
      - run: npm run build
      # deploy dist/ with cloudflare/wrangler-action (production on main, preview on PRs)
```

---

## 9. Roadmap

1. **MVP**: Zod schemas, build pipeline, element/tag/glossary pages, FR/EN, 10-15 elements with SVG diagrams, pre-rendering working with MUI.
2. **Learning layer**: Pagefind search, building pages and map, sitemap, redirects, SEO checks.
3. **Quiz and study paths**, with progress kept in `localStorage`.
4. **Polish and contributions**: accessibility audit, performance, and optionally a Git-based CMS (Keystatic/Decap) for non-developer authors.

**Media licensing:** each building image needs `credit`, `licence` and `source_url` (prefer Wikimedia Commons). Diagrams are original work. Optimize images before committing and keep the repo small; move to object storage only if it grows past a few hundred MB.
