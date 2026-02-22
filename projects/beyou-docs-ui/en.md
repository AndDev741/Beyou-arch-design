---
title: "Beyou Docs UI"
summary: "Documentation UI for the Beyou project, providing a user-friendly interface to browse API, architecture, and design documentation."
---
# Beyou Docs UI

This is the documentation UI for the Beyou project, providing a user-friendly interface to browse API, architecture, and design documentation.

## Overview

Built with React, TypeScript, and Tailwind CSS, the docs UI renders markdown content and OpenAPI specifications fetched from the backend. It supports multi-language documentation (English and Portuguese) and offers a clean, searchable interface.

## Features

- Real-time rendering of API documentation using OpenAPI 3.0
- Interactive architecture and design diagrams
- Multi‑language support (en/pt)
- Full‑text search across documentation
- Responsive design for mobile and desktop
- Integration with the Beyou backend documentation import system

## Architecture

The UI is structured as a Single Page Application (SPA) with:

- `src/pages` – top‑level pages (Projects, API, Architecture, Design)
- `src/components` – reusable components (cards, modals, search bars)
- `src/lib` – utilities for parsing OpenAPI and markdown
- `src/hooks` – custom hooks for data fetching and state management

## Development

To start the development server:

```bash
npm install
npm run dev
```

The app will be available at `http://localhost:3000`.

## Deployment

The docs UI is deployed on **Vercel** and automatically updated when changes are pushed to the `main` branch.