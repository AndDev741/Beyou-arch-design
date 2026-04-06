---
title: "How I Used Claude Code to Find 26 Security Vulnerabilities Before Deploying to Production"
summary: "Before pushing Beyou to production, I ran an adversarial security audit using Claude Code as my AI-assisted pen tester. It found 26 vulnerabilities across the full stack — from business logic flaws in the gamification system to infrastructure misconfigurations. Here's the full process: the prompt, the methodology, what it found, and how I fixed everything in 3 structured groups."
---

Beyou was feature-complete. The habits worked, the routines tracked XP, the gamification loop felt right, and the 9 themes looked clean. I was ready to deploy. But before putting real users on it, I had one question I couldn't answer confidently: **is this app actually secure?**

I'm a full-stack developer, not a security engineer. I know the basics — hash passwords, use HTTPS, don't put secrets in the frontend — but I also know there's a gap between "I think it's fine" and "I've actually tested it." Security audits from professional firms cost thousands and take weeks. I didn't have that budget or timeline.

So I tried something different: I used **Claude Code** as an adversarial security auditor for the entire codebase.

## Why an AI-Assisted Audit?

The idea wasn't to replace a professional penetration test. It was to get a thorough first pass — the kind of review that catches the obvious (and not-so-obvious) mistakes before a human specialist would even start.

What makes Claude Code interesting for this is that it can read your entire codebase, understand the architecture, follow data flows across layers, and think like an attacker. It's not just pattern matching for `eval()` or hardcoded secrets. It can reason about business logic — things like "what happens if a user sends this ID that belongs to someone else?" or "what if this integer field has no upper bound?"

I gave it full access to the Beyou monorepo: the Spring Boot backend, the React/Redux frontend, the Docker Compose infrastructure, and the API specs.

## The Prompt

This is where the magic happens. The quality of the audit depends entirely on how well you brief the AI. I didn't just say "find security bugs." I wrote a detailed prompt that frames Claude Code as a specific role, gives it a structured analysis framework, and tells it exactly what output I expect.

Here's the full prompt (it's long — that's intentional):

> You are a Senior Application Security Engineer and Penetration Tester with 20+ years of experience conducting security audits for production web applications, with deep expertise in OWASP standards, threat modeling, and secure software development lifecycle (SSDLC). You have a track record of finding critical vulnerabilities before malicious actors do.
>
> Your task is to conduct a comprehensive, adversarial security review of a full-stack habit manager application called "BeYou". You are not a developer checking boxes — you are a security specialist who thinks like an attacker and protects like a guardian.

Then I defined 9 analysis areas for it to cover systematically:

1. **Authentication & Session Management** — JWT implementation, refresh token strategy, brute force protection, password policy, account enumeration
2. **Authorization & Access Control** — BOLA/IDOR, broken function-level auth, mass assignment, privilege escalation
3. **Input Validation & Injection** — SQL injection, XSS, SSTI, path traversal, ReDoS
4. **API Security** — CORS policy, rate limiting, mass assignment, HTTP method enforcement
5. **Data Security & Privacy** — PII handling, encryption at rest/transit, sensitive data in logs, GDPR compliance
6. **Frontend Security** — CSP headers, token storage, Redux state exposure, source maps
7. **Infrastructure & Configuration** — Secrets management, Docker security, exposed admin interfaces, dependency CVEs
8. **Business Logic Security** — XP/gamification manipulation, race conditions, negative value attacks, workflow bypass
9. **Incident Response Readiness** — Security logging, intrusion detection, session revocation

For each area, I gave specific things to check. For example, under Business Logic:

> XP/gamification manipulation: can users forge XP gains, bypass habit checks, or manipulate level-up logic from the client? Race conditions in critical operations (double-submitting habit completions for double XP). Negative value attacks: what happens with negative habit counts, XP values, or progress?

I also specified the exact output format — CVE-style finding IDs, CVSS scores, proof-of-concept attack scenarios, a remediation roadmap with time-boxed priorities, and a final deploy/no-deploy verdict.

Key constraints I included:

> Think like an attacker first, then a defender — what would you target in the first 10 minutes? Never mark something as "low risk" just because it requires authentication — authenticated attacks are the most common in real breaches. Flag any finding that could violate GDPR given the app handles EU user personal data.

I also gave it the test credentials and the live API endpoint so it could make actual HTTP requests and confirm findings against the running application — not just guess from source code.

The key was the framing: **"You are not a developer checking boxes — you are a security specialist who thinks like an attacker."** That shifted the entire analysis from "does this code follow best practices?" to "how would I break this?" — a much more valuable perspective.

**The lesson: the more structured and specific your prompt, the more structured and actionable the output.** Vague prompts get vague results. A detailed analysis framework gets a detailed report.

## The Methodology

Claude Code didn't just grep for keywords. It performed a layered analysis:

1. **Source code review** — reading every controller, service, filter, and configuration file. Following data from the HTTP request all the way to the database query and back.

2. **Configuration analysis** — checking `application.yaml`, Docker Compose files, Spring Security config, CORS settings, and cookie policies.

3. **Live API testing** — making real HTTP requests to confirm vulnerabilities. For example, it tested CORS by sending a request with `Origin: https://evil-attacker.com` and checking if the server accepted it with credentials.

4. **Business logic reasoning** — understanding the gamification model (XP formula, levels, streaks) and asking "how could someone exploit this?"

5. **Cross-layer analysis** — finding issues that only appear when you look at how the frontend and backend interact. Things like a field the frontend never sends but the backend accepts, or a response header that only works because of a specific library behavior.

The output was a structured report with each finding categorized by severity (Critical, High, Medium, Low), assigned a CVSS score, and paired with a specific fix recommendation.

## What It Found

Out of the 26 findings, I'll highlight the ones that taught me the most — the kind of issues you don't find in a typical code review.

### Business Logic Flaws in Gamification

This was the most eye-opening category. Beyou's XP system works like this: when you check a habit in your daily routine, you earn XP based on a formula (`10 × difficulty × importance`). Sounds simple. But Claude Code found two ways to game it:

**Unbounded difficulty and importance** — The create habit endpoint accepted any integer for difficulty and importance. The frontend showed a 1-5 dropdown, but an attacker could call the API directly with `difficulty: 1000, importance: 1000` and earn 10,000,000 XP per check. The fix was straightforward: `@Min(1) @Max(5)` annotations on the DTO with `@Valid` on the controller.

**Experience level as raw integers** — When creating a habit, users can select their experience level (Beginner, Intermediary, Advanced). The frontend mapped these to fixed XP/level pairs, but the backend accepted raw integers. Someone could start a habit at level 100. We replaced the integers with a server-side `ExperienceLevel` enum that maps each tier to its correct values.

```java
public enum ExperienceLevel {
    BEGINNER(0, 0),
    INTERMEDIARY(5, 750),
    ADVANCED(8, 1800);
    // server resolves the actual values
}
```

### Authorization and Ownership Gaps

Claude Code traced the data flow through the routine check endpoint and found that while the request included a `userId` parameter, the service **never actually used it** to verify ownership. Any authenticated user could check habits in any other user's routine — and the XP would be awarded to the attacker, not the routine owner.

This was a classic IDOR (Insecure Direct Object Reference), but it was subtle because the `userId` was in the method signature — it looked like it was being used. You had to read the implementation carefully to notice it was silently ignored.

The fix was a `verifyRoutineOwnership()` method that loads the routine and compares the owner to the authenticated user before any operation proceeds.

### Infrastructure Hardening

Several findings were about production-readiness rather than active exploits:

- **Spring Actuator** was exposing all endpoints without authentication — health, metrics, environment variables, even heap dumps. We restricted it to `health,metrics,prometheus` and bound it to `localhost`.

- **Monitoring dashboards** were accessible without authentication. We added proper credentials.

- **Hardcoded database passwords** in the main configuration file were replaced with environment variables, with a startup validator that rejects insecure configurations when running in production mode.

### The Discussions That Changed Findings

Not everything Claude Code flagged was actually a vulnerability. This is an important part of the process — **reviewing the findings critically**.

For example, it flagged the CORS configuration as critical because the wildcard pattern (`*`) was being used. But after discussing it, we realized the CORS origin is externalized via environment variable — `*` is the dev default, and production uses the actual domain via AWS secrets. The architecture was correct; we just added a startup guard that rejects wildcard in the `prod` Spring profile.

Same with Docker port bindings — the database and management ports were exposed on all interfaces, but only in Docker Compose files used for local development. Production uses entirely different AWS infrastructure. We downgraded these from "High" to "Informational."

These discussions improved the report's accuracy and taught me to distinguish between **code vulnerabilities** and **deployment-context configurations**.

## The Remediation Strategy

With 26 findings to fix, I needed a plan. Fixing things randomly would be chaotic and risk regressions. So I organized the remediation into 3 groups:

### Group 1: Foundation & Critical (7 items)
The things that must be fixed before any deployment. This included the IDOR fix, the AOP logging issue, actuator lockdown, and — most importantly — **Spring Profiles**. We created `application-dev.yaml` and `application-prod.yaml` to cleanly separate dev and production configurations, with a `SecurityConfigValidator` that fails fast on startup if production settings are insecure.

This foundation made everything else easier because we could now say "in dev this is fine, in prod this is enforced."

### Group 2: Business Logic & Auth Hardening (9 items)
Rate limiting, the date bypass fix, the ExperienceLevel enum, difficulty bounds, OAuth CSRF protection, email verification on registration, stronger password policy (12 characters, 2 character classes), and cookie security externalization.

The email verification flow was the biggest addition — using Spring Events with `@Async` to send bilingual verification emails without blocking the registration response.

### Group 3: Compliance & Hardening (10 items)
Security headers (CSP, Referrer-Policy, Permissions-Policy), GDPR data export endpoint, redux-persist cleanup, constant-time secret comparison, source map removal, dependency vulnerability scanning (OWASP plugin + npm audit), and the unique constraint on habit checks to prevent race conditions.

Each fix was committed individually with a clear message referencing the finding ID (e.g., `fix(security): add ownership verification to check/skip endpoints — CVE-BYU-003`). This made the git history a traceable audit trail.

## Key Architecture Decisions

Some fixes were simple one-liners. Others required architectural decisions:

**Tiered rate limiting with Bucket4j** — Instead of a single global rate limit, we implemented 4 tiers: auth endpoints (5 requests per 15 minutes per IP), domain writes (30/min per user), domain reads (60/min per user), and docs (30/min per IP). The filter runs at `@Order(1)` and is conditionally disabled in test environments.

**Spring Profiles as the security backbone** — The `SecurityConfigValidator` runs at `@PostConstruct` and rejects insecure configurations in production: wildcard CORS, short JWT secrets, disabled cookie security. This means even if someone forgets to set an environment variable, the app won't start instead of running insecurely.

**GDPR self-service export** — Rather than building a complex admin panel, we added a simple `GET /user/export` endpoint that returns all of a user's data (profile, categories, habits, goals, tasks) as structured JSON. Authenticated, read-only, and fast thanks to the existing Caffeine cache layer.

## The Numbers

| Metric | Value |
|--------|-------|
| Total findings | 26 |
| Critical | 4 |
| High | 8 |
| Medium | 10 |
| Low / Informational | 4 |
| Time to audit | ~2 hours (with Claude Code) |
| Time to remediate | ~1 day (3 groups) |
| New test files added | 12 |
| Lines changed | ~2,500 across 3 repos |

## What I Learned

**Security is a different mindset.** Writing features is about "how does this work?" Security is about "how does this break?" Asking Claude Code to think like an attacker shifted my perspective in ways that reviewing my own code never would.

**Most vulnerabilities are logic bugs, not technical exploits.** Nobody was going to SQL-inject Beyou — JPA parameterizes everything. But the business logic around XP, ownership, and input bounds? That's where the real risks were.

**Dev/prod separation is foundational.** Half the "critical" findings turned out to be dev-only configurations. Once we had proper Spring Profiles, the real production risk surface was much smaller than the initial report suggested.

**AI-assisted auditing isn't a replacement — it's an accelerator.** Claude Code found things I wouldn't have thought to check, and it found them fast. But the discussion phase — where I explained my architecture and we refined the findings together — was equally valuable. The tool is most powerful when you treat it as a collaborator, not an oracle.

If you're building an app and thinking about deploying it, I'd highly recommend running this kind of audit before you do. The cost of finding a vulnerability before launch is a few hours of work. The cost of finding it after is a lot more.
