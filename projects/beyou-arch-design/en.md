---
title: "Beyou Arch Design"
summary: "The source of truth for everything on this site: architecture topics, blog posts, the OpenAPI controller specs, and this projects catalog, published by a workflow on every merge."
---
# Beyou Arch Design

The content repository. Everything rendered on this documentation site lives here as files: sixteen architecture topics (English, Portuguese, and a Mermaid diagram each), the engineering blog, the API reference as one OpenAPI spec per backend controller, and this projects catalog.

## How publishing works

A merge to main is a deployment. The publish workflow signs in to the backend, triggers the four import endpoints behind the JWT-plus-secret double gate, and then dispatches the docs site's CI, which prerenders every route against the fresh content and ships a new image. Topics removed from the repository are archived in the database rather than deleted, so unpublishing is reversible.

## Conventions

One directory per topic, a YAML descriptor, one markdown file per locale, and a diagram where the area requires it. The API specs are generated from the backend's live springdoc output, split per controller with each spec carrying only the schemas it references.
