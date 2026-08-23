---
title: "Self-Hosting Beyou: The Challenges and the Nice Things About Trying It"
summary: "Beyou's production is a 2012-era laptop in my bedroom, reached through a tunnel, with zero ports open on the router. This is why I chose self-hosting over free tiers, what actually breaks, and why opening my own app every day still feels this good."
---

Every day I open Beyou on the web, then the mobile app, sometimes the docs page, then Grafana and GlitchTip. And every time, some part of my brain remembers: all of this is running on my laptop. Every request from every one of those screens travels to a machine sitting in my bedroom. All the control, all the architecture I built, working smoothly. It's one of the best feelings this project has given me.

This post is about how that setup came to be, what it actually costs to run production at home, and why I'd do it again.

## Why not just use a cloud?

Beyou is a 100% free app, and I decided early that I didn't want to spend a coin hosting it. So I did what everyone does first: cloud free tiers and cheap VPS plans. They technically worked. They also bored me out of the project every single time. Managing quota limits and dashboards built to upsell me was the opposite of the energy I wanted to put into Beyou.

The turning point was discovering Cloudflare Tunnels, and realizing how easy it had become to put a locally running app on the real internet. No port forwarding, no fixed IP, no exposing my home network. A daemon on the machine dials out to Cloudflare, and Cloudflare sends visitors down that tunnel. The moment I understood that, the plan wrote itself.

## The machine

Here's my favorite part of the whole story. Production runs on an LG S460 laptop: two cores, 8 GB of DDR3, a 120 GB SSD, hardware from around 2012. My mom bought it for me, used, in 2020, so I could study during high school. It's the machine I learned to program on.

I formatted it, installed Debian 13 without a desktop, applied the security patches, and built the stack piece by piece: the containers, the tunnel, Tailscale so SSH never touches the public internet, fail2ban underneath. The router forwards nothing. Not one port. And the classic home-server problem, the dynamic IP, dissolves by design: the tunnel is an outbound connection, so when my ISP hands out a new address, the daemon reconnects and DNS never points at my house anyway.

The computer I learned to code on now serves every request my product gets. There's a symmetry in that I refuse to be cynical about.

## The challenges, honestly

Self-hosting on a home machine means owning a failure model most tutorials skip:

- **Power and internet are my SLA.** If either blinks long enough, the app is down until someone (me) presses the power button. After boot, Docker's restart policies bring every container back on their own, so recovery is exactly one button, but the button is physical.
- **My only production incident so far**: a family member unplugging the internet cable. No postmortem template covers this. I checked.
- **Discipline replaces the platform.** Nobody else rotates my secrets or hardens my defaults, so the rules are strict: everything binds to loopback, the tunnel and Tailscale are the only ways in, and the server boots refusing insecure production configuration.
- **Backups are the honest gap.** The database has no backup routine yet, and it sits above everything else on the infrastructure list. Self-hosting makes gaps like this personal: there is no support ticket to hide behind.

There's also a pattern I've come to like: things start manual, and the annoyance of repeating them becomes the motivation to automate. Deploys went from manual to Watchtower pulling images automatically. The error tracker's configuration went from clicking to a bootstrap script. Every chore that survived long enough got automated, and I learned something each time.

## The nice things

The cost is zero, and for a free app that matters, but the money is honestly the smallest part.

Everything is mine. When something breaks, the entire path from the browser to the database is inspectable by me, on hardware I can physically touch. When I added the monitoring stack, I wasn't reading a pricing page, I was learning how scrape targets and log labels actually work. Self-hosting turned infrastructure from a bill into a curriculum.

And it changed my relationship with the product itself. When Beyou only ran on localhost, using it felt like testing. A real production, where everything persists and I can open my routines from anywhere, made me want to use my own app every day. That daily use is where half the product ideas come from now.

## If you're thinking about trying it

My honest advice, based on nothing but this experience: the entry cost is much lower than it looks. An old machine, a Linux install, and a tunnel get you a real production URL without opening a single port. Be upfront with yourself about the failure model (yours will also be a power cable and a family member), keep every admin surface off the public internet, and put backups higher on your list than I did.

The reward is that feeling I started this post with. Opening your own app, on your own domain, served by your own machine, and watching every layer you built do its job. I don't think a managed platform can sell that.

## What's next

The setup stays as it is for now. The curiosity list for going deeper: backend replication, load balancing, and figuring out what a real failover story looks like on hardware like this. Backups come first, though. I wrote that in the documentation, so now it's a promise.

## Update, August 2026

The promise is kept. Backups run nightly now: the database, the uploads volume and the `.env` file go to Cloudflare R2 through restic, encrypted before they leave the machine, and a weekly job restores the newest snapshot into a throwaway database and compares its row counts against the live one. That second job is the one I would argue for hardest. A backup nobody has restored is a guess, and I would rather find out on a Monday morning than on the worst day.

Two things surprised me while building it. The first is that the most valuable thing on the box was never the database — it was the 5 KB `.env` file, the only place the OAuth secret, the mail password and every API key existed. Losing the database costs data; losing that file means reissuing every credential I own while the app is down. The second is that my first version quietly never deleted anything: restic groups snapshots by path when it applies a retention policy, and because I staged each run in a fresh temporary directory, every snapshot landed in a group of its own and "keep 7 daily" kept all of them forever. I only caught it because I ran the thing three times in a row and counted.

What this does not fix is uptime. It is still one disk in one laptop with no failover, so this closes the data-loss half of the failure model and leaves the harder half exactly where it was.
