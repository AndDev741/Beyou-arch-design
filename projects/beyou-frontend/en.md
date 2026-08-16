---
title: "Beyou Frontend"
summary: "The monorepo holding both clients: the React web app, the native mobile app, eight shared source packages, and the marketing site."
---
# Beyou Frontend

One TypeScript codebase, two clients. npm workspaces with Turborepo hold the React 18 web app (`apps/web`), the React Native 0.85 + Expo mobile app (`apps/mobile`), and eight shared packages consumed as raw source, so a single edit hot-reloads both apps.

## What is shared, what is not

The state (17 Redux slices and the gamification logic), the API layer behind a narrow HttpClient interface, the theme tokens, the translations, the validation schemas, and the icon registry are all one implementation. Each platform keeps only what must differ: persistence, navigation, token storage, and the entire render layer. The marketing site lives at the repo root, deliberately outside the workspace, and ships to Cloudflare Pages on its own.

## How it ships

One CI graph typechecks, builds, and tests every workspace, then runs the Playwright e2e suite against the full stack. A push to main publishes the web image to GHCR (Watchtower deploys it) and, when the mobile app or any shared package changed, builds a signed Android APK published to a rolling GitHub release.

## Deep dives

The monorepo, UI components, Redux and data, language and theme, and UI security architecture topics all document this repository.
