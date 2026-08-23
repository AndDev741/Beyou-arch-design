---
title: "Beyou Dev Env"
summary: "The Compose layer that assembles the other repositories into something you can run: dev with hot reload, production from published images, e2e isolation, and the monitoring overlay."
---
# Beyou Dev Env

This repository contains no application code. It is the orchestration layer: a base Compose file with Postgres and the shared network, plus four overlays that stack on top of it.

## The overlays

- **dev** builds the backend and frontend from sibling checkouts with bind-mounted source and hot reload; build artifacts live in named volumes so host and container tooling never fight.
- **prod** pulls the published GHCR images, serves the web build through nginx, and lets Watchtower redeploy on every new image. All ports bind to loopback; the Cloudflare Tunnel in front does the exposing.
- **e2e** boots an isolated stack under its own project name and database, which the backend's safety check enforces.
- **monitoring** carries Prometheus, Grafana, Loki with Alloy, and GlitchTip: the same file in development and production, so what you debug locally is what runs deployed.

## The scripts

`up-dev.sh` and `up-prod.sh` (both accepting `--monitoring`), `down.sh`, the loud-failure `reset-db.sh`, and `bootstrap-glitchtip.sh`, which creates and reconciles the error tracker's organization, projects, twelve uptime monitors, three heartbeats, and the alert rules from code.

`backup.sh` and `restore-check.sh` are the pair that matter when the disk does. The first dumps the database, the uploads volume, and `.env` to an encrypted restic repository offsite plus a plain local copy; the second restores the newest snapshot into a scratch database and compares row counts against the live one, because an untested backup is a directory that makes you feel better. Both run from systemd timers, nightly and weekly.

## Deep dives

The infrastructure and monitoring architecture topics document this repository end to end, from the laptop it all runs on to the alert that fires when the snapshot scheduler goes quiet.
