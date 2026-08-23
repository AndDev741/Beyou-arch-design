---
title: "Beyou Infrastructure"
summary: "Production is a 2012-era laptop in a bedroom: Debian 13, Docker, a Cloudflare Tunnel dialing out, and zero ports open to the internet."
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

Two cores and 8 GB is not much, and it turns out to be plenty. The whole stack fits: the API, the database, both nginx sites, and the entire monitoring overlay.

## The operating system

Debian 13 (trixie), headless, no desktop environment. The setup was deliberate and manual: format the disk, install Debian, apply the security patches, then build the stack piece by piece. Doing it by hand first is part of the point. Every chore I repeat becomes motivation to automate it, and that loop has taught me more about low-level infrastructure than any managed platform ever did.

## The way in: Cloudflare Tunnel

The router forwards nothing. Not one port. Instead, `cloudflared` runs on the laptop and dials out to Cloudflare's edge, holding a persistent connection. Cloudflare terminates TLS and pushes requests for the public hostnames down that tunnel to loopback services:

| Hostname | Service on the laptop |
|----------|----------------------|
| **app.beyouweb.com** | web nginx on 127.0.0.1:3000 (proxies /api/v1 to the backend) |
| **docs.beyouweb.com** | docs nginx on 127.0.0.1:3002 (same /api/v1 proxy) |
| **obs.beyouweb.com** | Grafana on 127.0.0.1:3001 |
| **mnt.beyouweb.com** | GlitchTip on 127.0.0.1:8000 |

This design quietly solves the classic home-server problem: the dynamic IP. Since the tunnel is an outbound connection, the house's public IP is irrelevant. DNS points at Cloudflare, never at my router, and when the ISP hands out a new address cloudflared just reconnects. I never had to think about it, which I only fully appreciated when I sat down to write this doc.

The marketing site at **beyouweb.com** is the one public surface that skips the laptop entirely: static HTML on Cloudflare Pages.

## Remote hands: Tailscale

SSH is not reachable from the internet at all. The laptop joins a Tailscale network, and administration happens only from devices inside it. fail2ban runs as an extra layer underneath.

The nicest consequence: deploying needs no SSH session. CI publishes images to GHCR, Watchtower on the laptop polls for them and restarts what changed. Merging to main is the deployment; the laptop pulls, nothing pushes in.

## The failure model, honestly

This is a home machine, and the doc would be dishonest if it pretended otherwise.

- **Power or internet outage**: the app goes down and stays down until someone boots the laptop. There is no auto-recovery for the machine itself.
- **After boot**: Docker's restart policies bring every container back on their own. Recovery is one power button.
- **Backups**: nightly, offsite, encrypted. The database, the uploads volume and the `.env` file go to Cloudflare R2 through restic, and a weekly job restores the newest snapshot into a scratch database and compares row counts against the live one, because a backup nobody has restored is a guess. This was the biggest gap in the setup for a long time. It is closed for data loss and it does nothing for uptime: still one disk, one machine, no failover.
- **Incidents so far**: exactly one category, and it is not in any postmortem template: family members unplugging the internet cable.

## Why a laptop in a bedroom

Beyou is a 100% free app, and I did not want to spend a single coin hosting it. I tried the usual road first: cloud free tiers and cheap VPS plans. They technically worked, and managing them bored me out of the project every time.

The turning point was discovering Cloudflare Tunnels and realizing how easy it had become to put a locally running app on the real internet. From there it snowballed: get the old machine, format it, install Debian 13, patch it, build the infrastructure, buy the domain, and put the web app, the backend, and the database online. Running the app only on localhost had been demotivating; a real production, where everything persists and I can open Beyou from my phone anywhere, made me actually want to use my own product. The monitoring stack came next, then the native mobile app, and each layer has made it better.

## What's next

For now it stays exactly as it is. The curiosity list for going deeper: backend replication, load balancing, and working out what a real failover story looks like on hardware like this. Backups used to head that list. Now that they exist, what is left is the harder half — staying up, rather than merely not losing anything.
