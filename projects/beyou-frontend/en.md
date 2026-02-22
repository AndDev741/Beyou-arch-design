---
title: "Beyou Frontend"
summary: "React-based frontend for the Beyou application, providing a user interface for habit tracking and productivity."
---
# Beyou Frontend

This is the frontend application for the Beyou productivity platform.

## Overview

Built with modern web technologies:

- **React** for UI components
- **TypeScript** for type safety
- **Vite** for fast builds and development
- **Tailwind CSS** for styling
- **shadcn/ui** for component library

## Features

- User authentication and session management
- Goal tracking with visual progress
- Habit formation with streaks
- Task management with drag‑and‑drop
- Routine scheduling and calendar integration
- Real‑time notifications

## Architecture

The frontend follows a feature‑first folder structure, separating concerns into:

- `src/components` – reusable UI components
- `src/pages` – top‑level page components
- `src/lib` – utilities, API clients, and shared logic
- `src/context` – React context providers (theme, auth, etc.)
- `src/hooks` – custom React hooks

## Development

To start the development server:

```bash
npm install
npm run dev
```

The app will be available at `http://localhost:5173`.

## Testing

We use **Vitest** for unit tests and **Playwright** for end‑to‑end tests.

Run tests with:

```bash
npm run test
```

## Deployment

The frontend is automatically deployed to **Vercel** on every push to the `main` branch.