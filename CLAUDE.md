# Brazil Visible — Project Instructions

## Project Overview
Documentation site (Next.js 15, App Router, standalone) cataloging 92 Brazilian public data sources for government oversight. PT-BR primary language.

- **URL**: https://brazilvisible.org
- **License**: MIT
- **Branch de produção**: `main`
- **Branch de desenvolvimento**: `develop`

## Key Commands
- `npm install` — install dependencies
- `npm run dev` — local dev server (http://localhost:3000)
- `npm run build` — production build (standalone output to `.next/`)
- `npm run lint` — ESLint
- `node scripts/validate-frontmatter.mjs` — validate API page frontmatter
- `node scripts/health-check.mjs` — check API availability (generates `public/health.json`)

## Tech Stack
- **Framework**: Next.js 15 (App Router, `output: 'standalone'`)
- **Styling**: Tailwind CSS 3.4 with Brazilian flag color palette
- **MDX**: `next-mdx-remote/rsc` with remark/rehype plugins
- **Themes**: `next-themes` (class-based dark mode)
- **Deploy**: Docker (node:22-alpine, standalone) via Coolify

## Architecture

### Principles
1. **Standalone pre-rendered** — `next build` pre-renders all pages via `generateStaticParams`. Node server (`server.js`) serves pre-generated content.
2. **Content as data** — All content in `.md` files with structured YAML frontmatter. TypeScript reads filesystem at build time.
3. **Zero database** — Content from `docs/`, health check generates `public/health.json`.
4. **SEO-first** — Metadata, canonical URLs, JSON-LD, dynamic sitemap on every page.

### Build Flow
```
docs/*.md (Markdown + YAML frontmatter)
    -> gray-matter (parse frontmatter)
    -> lib/content.ts (sidebar, tags, docs)
    -> next-mdx-remote/rsc (render MDX)
    -> next build (standalone)
    -> .next/standalone/ (server.js + static)
    -> Docker (node:22-alpine)
    -> Coolify (production)
```

### Health Check Flow
```
docs/apis/**/*.md (extract url_base from frontmatter)
    -> scripts/health-check.mjs (normalize URLs, deduplicate)
    -> HTTP GET with browser headers (3-tier retry)
    -> public/health.json (result)
    -> GitHub Actions (every 6h, auto-commit)
    -> components/status-badge.tsx (client-side fetch + badge)
```

## Project Structure
- `app/` — Next.js App Router pages and layouts
- `components/` — React components (navbar, sidebar, search, status-badge, scroll-reveal, etc.)
- `lib/content.ts` — filesystem-based content utilities (memoized)
- `lib/mdx.tsx` — MDX rendering pipeline with remark/rehype plugins
- `lib/remark-admonitions.ts` — custom remark plugin for :::warning/:::tip/:::note
- `docs/apis/<category>/<source>.md` — API documentation pages (92 sources across 22 categories)
- `docs/cruzamentos/<recipe>.md` — cross-referencing recipes (5 recipes)
- `recipes/<name>/` — Jupyter notebooks (3 notebooks)
- `scripts/health-check.mjs` — API availability checker (WAF-bypass, 3-tier strategy)
- `scripts/validate-frontmatter.mjs` — frontmatter validation
- `public/health.json` — health check results (generated, committed by CI)
- `Dockerfile` — multi-stage standalone build (node:22-alpine)

## Color Palette (tailwind.config.js)

| Token | Hex | Usage |
|-------|-----|-------|
| `brazil-green` | `#009C3B` | Primary, links, buttons |
| `brazil-green-light` | `#00B847` | Light variant for dark mode |
| `brazil-green-dark` | `#007A2E` | Button hover |
| `brazil-yellow` | `#FFDF00` | Accents |
| `brazil-blue` | `#002776` | Secondary accents |
| `dark-bg` | `#0c0c18` | Dark mode background |
| `dark-surface` | `#14142a` | Dark mode surfaces |

## Conventions

### API Pages
- Every API page MUST have required frontmatter: title, slug, orgao, url_base, tipo_acesso, status
- Optional frontmatter: autenticacao, formato_dados, frequencia_atualizacao, campos_chave, tags, cruzamento_com
- Body sections: O que e, Como acessar, Endpoints/recursos principais, Exemplo de uso, Campos disponiveis, Cruzamentos possiveis, Limitacoes conhecidas
- Content in PT-BR
- Status values: documentado (fully documented), parcial (partial info), stub (metadata only)
- url_base MUST be a URL that returns HTTP 200 (for health check accuracy)

### Cross-Reference Recipes
- Frontmatter: title, dificuldade, fontes_utilizadas, campos_ponte, tags
- Body sections: Objetivo, Fluxo de dados, Passo a passo, Exemplo de codigo, Resultado esperado, Limitacoes

### General
- Commit messages follow conventional commits (feat:, docs:, ci:, fix:, refactor:, style:)
- Branch from `develop`, PR to `develop`, merge `develop` to `main` for releases
- `main` = production, `develop` = development branch
- Always run `npm run build` before committing to verify no build errors
- Run `node scripts/validate-frontmatter.mjs` when modifying API pages
- Run `node scripts/health-check.mjs` when modifying url_base values

## Deploy

```bash
docker build -t brazil-visible .
docker run -p 3000:3000 brazil-visible
```

Dockerfile multi-stage (3 stages):
1. **deps** (node:22-alpine): `npm ci` — isolated dependency install
2. **builder** (node:22-alpine): copy modules + code, `npm run build` -> `.next/standalone/`
3. **runner** (node:22-alpine): copies only `public/`, `.next/standalone/`, `.next/static/`. Runs as `nextjs` user (non-root) on port 3000

Security headers in `next.config.ts` (X-Content-Type-Options, X-Frame-Options, Referrer-Policy).

Deploy via **Coolify**: push to `main` -> Coolify detects via GitHub App -> Docker build -> auto deploy.

## Adding a New API Source
1. Create `docs/apis/<category>/<slug>.md`
2. Fill frontmatter with all required fields
3. Write content following the template sections
4. Run `node scripts/validate-frontmatter.mjs` to validate
5. Run `npm run build` to verify
6. Run `node scripts/health-check.mjs` to verify url_base
7. Commit with `docs: add <source name> to <category>`

## Adding a Cross-Reference Recipe
1. Create `docs/cruzamentos/<slug>.md`
2. Fill frontmatter (title, dificuldade, fontes_utilizadas, campos_ponte, tags)
3. Write content following template sections
4. Run `npm run build` to verify
5. Commit with `docs: add <recipe name> cross-reference recipe`

## Accessibility
- `focus-visible:ring` on all interactive elements
- `aria-expanded` on collapsible menus/sidebar
- `aria-label` on sections and icon-only buttons
- `role="combobox"` with `aria-autocomplete` on search
- `role="dialog"` on mobile menu with focus trap
- `prefers-reduced-motion` respected in hero and scroll-reveal
- Touch targets >= 44px (TOC, mobile buttons)

## SEO
- `<title>` and `<meta description>` on all pages
- Canonical URLs on all pages
- `hreflang` pt-BR on root layout
- JSON-LD: WebSite (landing), Article (API pages), BreadcrumbList (breadcrumbs)
- Open Graph and Twitter Card metadata
- Dynamic `sitemap.xml` generated at build time
- Permissive `robots.txt`
