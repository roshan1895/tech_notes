# CLAUDE.md

Guidance for Claude Code (and humans) working in this repository.

## Project Overview

**StackNotes** is a Jekyll static-site **documentation portal** published to GitHub Pages at
`https://roshan1895.github.io/tech_notes/` (project page — note the `/tech_notes` baseurl).
It hosts structured engineering docs, learning roadmaps, real production case studies, cheat
sheets, and interview prep. The design targets a Kubernetes / Google Cloud / MS-Learn feel:
data-driven navigation, reusable doc components, dark/light theme, client-side search, and mermaid diagrams.

Everything is **framework-first**: navigation, landing pages, search, "latest notes", TOC, and
prev/next all **generate themselves** from collections + data files. Adding content is almost
always just dropping a Markdown file in a folder or editing a YAML data file — never editing HTML.

## Build & Local Development

Local Ruby here is system Ruby 2.6 with a user-installed bundler; the gem bin dir must be on PATH:

```bash
export PATH="$HOME/.gem/ruby/2.6.0/bin:$PATH"
bundle install                 # first time only
bundle exec jekyll serve       # http://127.0.0.1:4000/tech_notes/  (live-reload)
bundle exec jekyll build       # outputs to _site/
```

Deployment is automatic: **push to `main`** and GitHub Pages rebuilds. Only GitHub-Pages-whitelisted
plugins are used (`jekyll-seo-tag`, `jekyll-sitemap`, `jekyll-feed`), so the hosted build matches local.

## Directory Structure

```
_config.yml            Site config: collections, defaults, plugins, SEO fields
Gemfile                jekyll + the 3 whitelisted plugins (jekyll_plugins group)

_data/                 ← DATA-DRIVEN CONTENT (edit YAML, no code)
  navigation.yml         Top nav + sidebar (groups: Learn, Topics, Resources)
  roadmaps.yml           The 10 learning roadmaps + their beginner/intermediate/advanced steps
  certifications.yml     Certification cards

_layouts/
  default.html           Page shell: head + topnav + search modal + sidebar + main + footer + scripts
  doc.html               Technical page: breadcrumb, title, meta, content, related/refs/edit, TOC rail
  section.html           Collection landing: auto-lists that collection's docs (+ empty state)
  roadmap.html           Roadmaps landing grid AND individual roadmap detail (from _data/roadmaps.yml)

_includes/
  head.html              <head>: SEO tag, favicons, fonts, theme bootstrap
  topnav.html            Brand/logo, primary nav (from nav data), hamburger, search btn, theme toggle
  sidebar.html           Multi-level collapsible nav (from nav data + each collection's docs)
  search-modal.html      Search dialog markup
  scripts.html           Theme, mobile drawer, lunr search (⌘K), TOC, mermaid, copy-code, lazy images
  doc-pager.html         Prev/next (collection-scoped)
  doc/                   Reusable doc components: breadcrumb, meta, toc, footer, callout
  home/                  The 10 homepage sections (hero, search, roadmaps, featured, latest,
                         certifications, incidents, interview, cheatsheets, updates)

_templates/              Copyable page templates (EXCLUDED from build)
  technical-page.md      Full technical-doc skeleton with every front-matter field
  case-study.md          12-section production case-study skeleton

_<category>/             ← DOC CONTENT lives here, one folder per category (see Collections below)
  index.md               Collection landing (layout: section, nav_order: 0)
  <slug>.md              A doc (layout: doc via defaults)

index.md                 Homepage (composes _includes/home/*)
docs/index.md            Docs hub (auto grid from nav data)
certifications.md        Certifications page (from _data/certifications.yml)
search.json              Client-side search index (pages + all collection docs)
about.md notes/ glossary/ cheatsheets/index.md   Existing pages (do not rewrite)
server_deployment/       Existing 20-part guide (layout: default) — DO NOT MODIFY

assets/css/style.css     All styling (CSS variables, dark/light, all components)
assets/favicon.*         favicon.svg + favicon.ico + favicon-32/192.png + apple-touch-icon.png
```

## Core Architecture

- **Collections** — each documentation category is a Jekyll collection declared in `_config.yml`
  (`collections:` + a matching `defaults:` scope). Collections give scoped iteration (`site.cloud`),
  clean `/<category>/:path/` URLs, and auto-applied layout/metadata. Current collections:
  `roadmaps, cloud, containers, linux, networking, devops, android, architecture, databases,
  real-projects, interview-questions, cheatsheets, leadership`.
- **Data-driven navigation** — `_data/navigation.yml` is the single source of truth for the top nav
  and sidebar. The sidebar auto-lists each collection's docs (sorted by `nav_order`) as collapsible children.
- **Reusable doc components** — `_layouts/doc.html` + `_includes/doc/*` render difficulty badge,
  auto reading-time, last-updated, breadcrumb, right-rail TOC, prerequisites, related, references,
  and an "Edit this page" link — all from front matter, so page bodies stay pure Markdown.
- **Ordering** — `nav_order` (integer) in front matter controls order everywhere. `nav_order: 0` is
  reserved for the collection landing (`index.md`) and is excluded from doc lists/pagers.

---

## How to Make Changes (cookbook)

> Rule of thumb: **content = a Markdown file in `_<category>/`; lists/roadmaps/certs = a `_data/*.yml` file.**
> You never edit HTML/CSS to add content.

### Add a documentation page
1. Pick the folder = category: `_cloud/ _containers/ _linux/ _networking/ _devops/ _android/
   _architecture/ _databases/ _real-projects/ _interview-questions/ _cheatsheets/ _leadership/`.
2. Create `_<category>/<slug>.md`. Fastest start: copy `_templates/technical-page.md`.
3. Minimum front matter: `title` + `nav_order`. Save. It auto-appears in the sidebar (collapsible child),
   the category landing page, search, and the homepage "Latest Notes" / "Recent Updates".

### Add a case study / production incident
Copy `_templates/case-study.md` into `_real-projects/`. It has the 12 sections (Problem → Environment →
Symptoms → Investigation → Root Cause → Fix → Commands → Architecture Diagram → Lessons → Prevention →
Related → References). Auto-lists under Real Projects and the homepage "Real Production Incidents" block.

### Add / edit a learning roadmap
Edit `_data/roadmaps.yml`. Each roadmap has `beginner` / `intermediate` / `advanced` step lists:
```yaml
    beginner:
      - title: "IAM basics"
        url: /cloud/iam/                 # optional: internal doc path
      - title: "Regions & zones"
        url: "https://cloud.google.com/…" # optional: external link (opens new tab)
```
Progress bars/rings and the `/roadmaps/<key>/` page update automatically. To add a whole new roadmap,
also add a stub file `_roadmaps/<key>.md` (front matter: `title`, `roadmap: <key>`, `permalink: /roadmaps/<key>/`).

### Add / edit certifications
Edit `_data/certifications.yml` (`status: planned | in-progress | earned`, `roadmap:` links to a roadmap).

### Add a brand-new category
1. `_config.yml` → add a line under `collections:` and a matching scope under `defaults:`.
2. `_data/navigation.yml` → add one item under the right sidebar `group`.
3. Create the folder `_<name>/` with an `index.md` (`layout: section`, `permalink: /<name>/`, `nav_order: 0`).

### Edit navigation (top nav or sidebar)
Only `_data/navigation.yml`. `primary:` = top nav; `sidebar:` = grouped left nav. Item fields:
`title, icon (emoji), collection` OR `url`, `status: available|soon`, `external: true`.

### Edit the homepage
Sections are `_includes/home/*.html`, composed in `index.md`. Reorder/remove by editing the include list in `index.md`.

### Styling
All CSS is `assets/css/style.css`. It uses CSS variables (`:root` for dark, `[data-theme="light"]` for light) —
change a variable to re-theme globally. Add new component styles under the "PORTAL v2" section.

### Regenerate the favicon
Source glyph is the stacked-layers mark (`assets/favicon.svg`). The `.ico`/PNG sizes were generated with a
Pillow script (gradient rounded square + layers glyph). To change it: edit the SVG, then re-run a Pillow
script to emit `favicon.ico`, `favicon-32.png`, `favicon-192.png`, `apple-touch-icon.png`.

---

## Conventions & Rules

- **Do not modify** existing content: `server_deployment/*`, `notes/`, `glossary/`, `cheatsheets/index.md`, `about.md`.
- **Preserve routing** — the `permalink` in each collection keeps `/<category>/…` URLs stable.
- **GitHub Pages safe** — only add plugins from the GitHub Pages whitelist; keep everything else client-side
  (search = lunr, diagrams = mermaid, both lazy-loaded from CDN).
- **YAML front matter with URLs must use block style**, and quote external URLs:
  ```yaml
  references:
    - title: Docs
      url: "https://example.com"     # flow style { url: https://… } BREAKS the YAML parser
  ```
- **Front-matter reference (docs):** `title`, `nav_order`, `description`, `difficulty` (beginner|intermediate|advanced),
  `last_updated` (YYYY-MM-DD), `tags: []`, `prerequisites: [{title,url}]`, `related: [{title,url}]`, `references: [{title,url}]`.
- Internal links use the `relative_url` filter (respects the `/tech_notes` baseurl); never hardcode `/tech_notes/…`.

---

## SEO & Getting Indexed on Google

### What is already configured
- **`jekyll-seo-tag`** (`{% seo %}` in `_includes/head.html`) → per-page `<title>`, meta description,
  canonical URL, Open Graph + Twitter cards, and JSON-LD. Uses `title`, `description`, `author`, `logo`,
  `twitter`, `social` from `_config.yml` and per-page `title`/`description`/`image` front matter.
- **`jekyll-sitemap`** → auto-generates `/tech_notes/sitemap.xml` covering all pages and collection docs.
- **`jekyll-feed`** → RSS at `/tech_notes/feed.xml`.
- **Favicons** and `theme-color` for browser/tab/social previews.

### To rank well, per page
- Always set a unique `title` and a concise `description` in front matter (description feeds the SERP snippet).
- Optionally set `image: /assets/…` in front matter for a custom social-share preview.
- Use one `#`/`h1`-level title (the `doc` layout renders it) and meaningful `##` headings.
- Add descriptive `alt` text to images (`![alt text](path)`).

### To get indexed on Google (one-time setup)
1. Confirm `_config.yml` has the correct `url:` (`https://roshan1895.github.io`) and `baseurl:` (`/tech_notes`) —
   `jekyll-seo-tag`/sitemap build absolute URLs from these. Fill in `author.name`, `author.twitter`, `social.links`.
2. Go to **Google Search Console** → add the property. For a project page, use the URL-prefix property
   `https://roshan1895.github.io/tech_notes/`.
3. **Verify ownership** — easiest is the HTML-meta-tag method: paste Google's `<meta name="google-site-verification" …>`
   into `_includes/head.html` (inside `<head>`), push, then click Verify.
4. **Submit the sitemap** in Search Console → Sitemaps → enter `sitemap.xml`
   (full URL `https://roshan1895.github.io/tech_notes/sitemap.xml`).
5. Use **URL Inspection → Request indexing** for the homepage and any key pages to speed up first crawl.
6. Indexing takes days to weeks; check Search Console → Pages for coverage/errors.

### robots.txt note (project-page nuance)
Google reads `robots.txt` only from the **domain root** (`https://roshan1895.github.io/robots.txt`), which
belongs to the root GitHub Pages repo (`roshan1895.github.io`), **not** this `/tech_notes` project repo.
So a `robots.txt` committed here would live at `/tech_notes/robots.txt` and be ignored for crawl directives.
Crawling of this project site is controlled by that root `robots.txt` (default = allow) plus the sitemap you
submit in Search Console. If you later move this site to a custom domain or the root repo, add a `robots.txt`
with `Sitemap: https://<domain>/sitemap.xml`.
