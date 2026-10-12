---
title: "Beyou Infrastructure"
summary: "Production is a 2012-era laptop in a bedroom: Debian 13, a single-node k3s cluster run by Flux from Git, a Cloudflare Tunnel dialing out, and zero ports open to the internet."
---

The [Architecture Overview](/architecture/overview) shows what runs. This topic is about what it runs on, and it is my favorite part of the whole project: Beyou's production environment is my first self-hosted infrastructure, built on the first computer I ever owned.

## The machine

Production is an LG S460 laptop. My mom bought it for me in 2020, used, so I could study during high school, and it is the machine I learned to program on. Today it serves every request Beyou gets, from a bedroom in my house.

| | |
|---|---|
| **Model** | LG S460 (bought used in 2020) |
| **CPU** | Intel Core i3-3120M, 2 cores / 4 threads @ 2.5 GHz (2012-era Ivy Bridge) |
| **RAM** | 8 GB DDR3 |
| **Storage** | 120 GB SATA SSD |
| **Network** | A regular LAN cable into the home router |

Two cores and 8 GB is not much, and it turns out to be plenty. The whole stack fits: the API, the database, both nginx sites, the monitoring stack, and Kubernetes itself on top.

## The operating system

Debian 13 (trixie), headless, no desktop environment. The setup was deliberate and manual: format the disk, install Debian, apply the security patches, then build the stack piece by piece. Doing it by hand first is part of the point. Every chore I repeat becomes motivation to automate it, and that loop has taught me more about low-level infrastructure than any managed platform ever did.

That loop is also what moved production off Docker Compose. Until October 2026 the stack was a Compose project with a monitoring overlay, Watchtower for updates and systemd timers for backups. Today the laptop runs **k3s**, a single-node Kubernetes cluster, and everything on it is declared in a public Git repository, [be-infra](https://github.com/AndDev741/be-infra), that [Flux](https://fluxcd.io) applies. Compose still lives on as the local development and e2e environment (the [Beyou Dev Env](/projects/beyou-dev-env) project); it no longer runs production.

## The way in: Cloudflare Tunnel

The router forwards nothing. Not one port. Instead, `cloudflared` runs on the laptop and dials out to Cloudflare's edge, holding a persistent connection. Cloudflare terminates TLS and pushes requests for every public hostname down that tunnel to a single place: Traefik, the cluster's only ingress controller, on a node port. Traefik routes on the Host header to one Ingress per hostname:

| Hostname | Where Traefik sends it |
|----------|------------------------|
| **app.beyouweb.com** | the web app's nginx (proxies /api/v1 to the backend) |
| **api.beyouweb.com** | the backend, directly (/api/v1) |
| **docs.beyouweb.com** | the docs site's nginx (same /api/v1 proxy) |
| **obs.beyouweb.com** | Grafana |
| **mnt.beyouweb.com** | GlitchTip |

No app has a NodePort or a host port of its own, so adding a hostname is an Ingress in Git, not a new hole in anything.

This design quietly solves the classic home-server problem: the dynamic IP. Since the tunnel is an outbound connection, the house's public IP is irrelevant. DNS points at Cloudflare, never at my router, and when the ISP hands out a new address cloudflared just reconnects. I never had to think about it, which I only fully appreciated when I sat down to write this doc.

The marketing site at **beyouweb.com** is the one public surface that skips the laptop entirely: static HTML on Cloudflare Pages.

## The cluster

Each app gets its own namespace, and each namespace gets the same three fences: a **ResourceQuota** (a memory budget, so one app can never starve the node), a **LimitRange** for defaults, and a **default-deny NetworkPolicy** that is then opened one hop at a time. For Beyou that chain is Traefik to the two nginx pods, nginx and Traefik to the backend, the backend to the database, and Prometheus and GlitchTip's uptime probes to the ports they check. Everything else in the cluster is refused.

| Namespace | What runs there |
|-----------|-----------------|
| **beyou** | web (nginx), docs (nginx), the Spring Boot backend, and the backup CronJobs |
| **data** | the shared PostgreSQL 17 cluster, run by the CloudNativePG operator |
| **monitoring** | Prometheus, Grafana, Loki and Alloy (the [monitoring topic](/architecture/monitoring)) |
| **glitchtip** | the error tracker, with its Valkey |
| **ingress** | Traefik |

The backend runs as **one replica with the Recreate strategy**, on purpose. The uploads live on a local-path volume that only one pod can mount, and the three `@Scheduled` jobs must not run twice. Both have to change before a second replica makes sense.

The database is a CloudNativePG cluster named `pg`: one PostgreSQL 17 instance holding one database and one role per app, with the roles declared in Git and reconciled by the operator. Beyou's database is `beyou`, owned by its own role; a second, read-only role exists only so Grafana's product dashboard can query it without being able to change anything.

Secrets are in Git too, encrypted with SOPS and age, and decrypted by Flux inside the cluster. The repository is public, so this is the line that matters: the age key lives on the server (with a copy inside the cluster, for Flux) and in a password manager, and nowhere else.

## Remote hands: Tailscale, and Git

SSH is not reachable from the internet at all. The laptop joins a Tailscale network, and administration happens only from devices inside it. fail2ban runs as an extra layer underneath.

The nicest consequence: deploying needs no SSH session. CI builds each image and pushes it to GHCR as `:latest`. Flux's image automation notices the new digest, commits it to be-infra, and rolls the workload out from that commit. Merging to main is still the deployment; the laptop pulls, nothing pushes in. What changed from the Watchtower days is the record: Git always says exactly which build is running, and rolling back is a `git revert`, not a guess at an old tag. Anything changed by hand with `kubectl` on something Flux manages is put back on the next reconcile.

## The failure model, honestly

This is a home machine, and the doc would be dishonest if it pretended otherwise.

- **Power or internet outage**: the app goes down and stays down until someone boots the laptop. There is no auto-recovery for the machine itself.
- **After boot**: k3s starts on its own, and the cluster brings every workload back to the state Git describes. Recovery is one power button.
- **Backups**: nightly, offsite, encrypted, and run by the cluster itself. A CronJob dumps the `beyou` database with `pg_dump`, tars the uploads volume, and hands both to restic, which encrypts them in the pod and sends them to Cloudflare R2 with a 7 daily / 4 weekly / 6 monthly retention. A size guard fails the run if the repository outgrows its budget, and a heartbeat goes to GlitchTip only on success. Once a week a second CronJob restores the newest snapshot into a throwaway Postgres inside its own pod (never the shared instance) and compares row counts against the live database; it sends its own heartbeat only on PASS, because a backup nobody has restored is a guess. The configuration no longer needs backing up separately: it is in Git, encrypted. This was the biggest gap in the setup for a long time. It is closed for data loss and it does nothing for uptime: still one disk, one machine, no failover.
- **Incidents so far**: exactly one category, and it is not in any postmortem template: family members unplugging the internet cable.

## Why a laptop in a bedroom

Beyou is a 100% free app, and I did not want to spend a single coin hosting it. I tried the usual road first: cloud free tiers and cheap VPS plans. They technically worked, and managing them bored me out of the project every time.

The turning point was discovering Cloudflare Tunnels and realizing how easy it had become to put a locally running app on the real internet. From there it snowballed: get the old machine, format it, install Debian 13, patch it, build the infrastructure, buy the domain, and put the web app, the backend, and the database online. Running the app only on localhost had been demotivating; a real production, where everything persists and I can open Beyou from my phone anywhere, made me actually want to use my own product. The monitoring stack came next, then the native mobile app, and each layer has made it better.

Kubernetes came last, for the same reason as everything else: to learn it on something real. One node does not need an orchestrator. It does benefit from what came with it: every app isolated behind its own quota and network policy, one monitoring stack watching the host, the cluster and every pod, and a production I can rebuild from a Git repository.

## What's next

The next step is two backend replicas behind the Service, which means the scheduled jobs need a lock so only one replica runs each of them, and the uploads need to move off the local volume into object storage. Past that, the curiosity list is still what it was: working out what a real failover story looks like on hardware like this. Backups used to head that list. Now that they exist, what is left is the harder half — staying up, rather than merely not losing anything.
