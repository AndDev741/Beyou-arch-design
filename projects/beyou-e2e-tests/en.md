---
title: "Beyou E2E Tests"
summary: "The Playwright suite that drives the real stack end to end: register, log in, build routines, check habits, and watch the XP move."
---
# Beyou E2E Tests

Playwright specs that exercise the whole product the way a user does: frontend, backend, and Postgres together, no mocks. The suite locks in the flows that must never silently break: registration and login (including the anti-enumeration behavior on failures), session persistence across reloads, logout teardown, habit CRUD, routine check-ins with their XP and streak effects, goal completion asymmetry, the gamification feedback, and the full onboarding tutorial.

## How it runs

Against an isolated stack: the backend boots with the e2e profile on the dedicated `beyou_e2e` database, and a safety check refuses to start against anything that does not look like a test database. Fixtures provide authenticated pages with the tutorial bypassed or intact, and seeding helpers build categories and full onboarding states through the real API.

The suite is also a merge gate: the frontend repository's CI boots this stack and runs these specs on every pull request, with a sibling-branch resolver so a frontend change that needs new specs can point at a matching branch here before anything merges.

## Deep dives

The security topic documents the auth behaviors these specs pin, and the infrastructure topic covers the isolated e2e overlay they run on.
