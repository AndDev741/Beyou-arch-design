---
title: "Beyou Backend"
summary: "The Spring Boot API behind everything: domain services with per-user ownership checks, the XP engine, the AI agent, and the docs pipeline."
---
# Beyou Backend

The single API serving the web app, the native mobile app, and this documentation site. Spring Boot 4.1 on Java 25 with virtual threads, PostgreSQL 15 behind a Flyway-owned schema, and 24 REST controllers under `/api/v1`.

## What lives here

- **The domain**: categories, habits, tasks, goals, and routines, with the gamification engine on top: the check-in XP formula, the quadratic level curve, derived streaks, daily routine snapshots with XP decay, and the signed per-day XP ledger.
- **Security**: stateless JWT auth with rotating refresh tokens, e-mail verification, two-factor account deletion, eleven rate-limit tiers plus a login lockout, and boot-time validators that refuse a misconfigured production.
- **The AI agent**: a chat with 33 tools over the same domain services, streamed by SSE, running on a five-provider free-tier LLM fallback chain, plus the stateless onboarding suggestion endpoint.
- **The docs system**: imports this documentation from GitHub into Postgres and serves the public read and search API the docs site runs on.

## How it ships

Merging to main is the deployment: CI runs the tests, builds the image, publishes it to GHCR, and Watchtower on the production host pulls and restarts. Caffeine caches sit in front of the hot reads, and Micrometer metrics feed the Grafana dashboards, with errors reporting to the self-hosted GlitchTip.

## Deep dives

The architecture topics cover this repo in detail: the domain model, security, gamification, the AI agent, caching, AOP logging, the email service, and the docs system all have their own pages.
