# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based static site ("TechNotes") that hosts personal technical documentation, published to GitHub Pages at `https://roshan1895.github.io/tech_notes/`. Currently contains a 20-part Server Deployment guide with plans for Android Development, Web Technologies, and Cloud Infrastructure sections.

## Build & Development

```bash
# Serve locally (requires Jekyll installed)
bundle exec jekyll serve

# The site is deployed via GitHub Pages — push to main to publish
```

## Architecture

- **Static site generator**: Jekyll with kramdown markdown and rouge syntax highlighting
- **Layout**: Single layout (`_layouts/default.html`) with fixed top nav, left sidebar, and main content area
- **Styling**: One CSS file (`assets/css/style.css`) — custom design using CSS variables, Inter + JetBrains Mono fonts, no framework
- **Content structure**: Each topic area gets its own directory (e.g., `server_deployment/`) with numbered markdown files (`00-OVERVIEW.md` through `19-...`) and an `index.md` serving as the section table of contents
- **Config**: `_config.yml` sets `baseurl: "/tech_notes"` — all internal links must use `relative_url` filter

## Content Conventions

- Guide files are numbered with zero-padded prefixes (`01-`, `02-`, ...) for ordering
- Internal links omit `.md` extension (Jekyll generates extensionless URLs)
- The `server_deployment/index.md` organizes guides into four sections: Foundations, Stack-Specific, Deployment Progression, Operations & Reference
- Home page (`index.md`) uses card-based layout with inline SVG icons
