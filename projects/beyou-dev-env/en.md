---
title: "Beyou Dev Env"
summary: "The Compose layer that assembles the other repositories into something you can run locally: dev with hot reload, e2e isolation, and the monitoring overlay. Production moved to Kubernetes in October 2026."
---
# Beyou Dev Env

This repository contains no application code. It is the local orchestration layer: a base Compose file with Postgres and the shared network, plus three overlays that stack on top of it.

## The overlays

- **dev** builds the backend and frontend from sibling checkouts with bind-mounted source and hot reload; build artifacts live in named volumes so host and container tooling never fight.
- **e2e** boots an isolated stack under its own project name and database, which the backend's safety check enforces.
- **monitoring** carries Prometheus, Grafana, Loki with Alloy, and GlitchTip, answering locally the same three questions the production stack answers in the cluster.

## The scripts

`up-dev.sh` (accepting `--monitoring`), `down.sh`, the loud-failure `reset-db.sh`, and `bootstrap-glitchtip.sh`, which creates and reconciles the error tracker's organization, projects, uptime monitors, heartbeats, and the alert rules from code.

## Production lives elsewhere now

Until October 2026 this repository also ran production: a `prod` overlay pulling the published images behind Watchtower, an `up-prod.sh`, and a backup and restore-drill pair on host systemd timers. All of that moved to a single-node k3s cluster managed by Flux from the public [be-infra](https://github.com/AndDev741/be-infra) repository, where deploys are commits, the backups and the weekly restore drill are CronJobs, and monitoring is a shared platform stack. What stays here is everything you need to run Beyou on your own machine.

## Deep dives

The infrastructure and monitoring architecture topics document both sides, this local stack and the production cluster, from the laptop it all runs on to the alert that fires when the snapshot scheduler goes quiet.
