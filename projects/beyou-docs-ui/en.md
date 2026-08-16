---
title: "Beyou Docs UI"
summary: "The site you are reading: a React viewer for the imported docs, with an OpenAPI endpoint browser, theme-aware Mermaid, search, and prerendered SEO pages."
---
# Beyou Docs UI

The documentation site itself. A React and Vite SPA that reads the backend's public docs API and renders four collections: architecture topics with theme-aware Mermaid diagrams and a table-of-contents rail, the engineering blog, the API reference with a per-controller OpenAPI endpoint browser, and the projects catalog, plus cross-collection search with highlighted matches.

## The SEO half

Client-rendered content is invisible to most crawlers, so a prerender script fetches every topic in both locales from the same public API and writes one real HTML file per route, with titles, canonicals, hreflang pairs, structured data, and the article text itself. The SPA boots on top and takes over. Content changes rebuild the whole image: the content repository's workflow dispatches this repository's CI after each import, and Watchtower deploys the result.

## Details worth knowing

Detail pages are addressed by path under a locale prefix, and the architecture and API listings auto-open their first entry. The two-base, five-accent theme system matches the app's, with a pre-paint script preventing a flash of the wrong theme.
