---
title: "Email Service"
summary: "Seven e-mails, one service class: six transactional and one that nobody asked for, how each decouples from its database transaction, the three limits standing between a nudge trigger and an inbox, and why the sender ships switched off."
---

This document covers the e-mail subsystem: which mails exist, how each one is kept transaction-safe, how failures are handled per flow, and how the templates are built and localized.

## What gets sent

The system sends seven e-mails, all owned by one service class in the notification package. Six are transactional — somebody asked for each by doing something — and that is why none of those consults a preference. The seventh is the engagement nudge, the first mail here that nobody asked for, which is why it is the only one that checks a switch, counts against a budget, and ships disabled. See [Engagement mail is opt-out](#engagement-mail-is-opt-out) and [The nudges, and when they do not go out](#the-nudges-and-when-they-do-not-go-out).

| # | Mail | Trigger | Contains |
|---|------|---------|----------|
| 1 | Registration verification | New account created, or a resend requested | Button link to the verify page, 24-hour expiry notice |
| 2 | Password reset | Forgot-password request | Button link to the reset page, TTL in minutes |
| 3 | Account deletion code | Deletion requested | Six digits, deliberately no link and no button: a deletion must not be one click away from an inbox |
| 4 | Feedback acknowledgement | User submits feedback | Echo of the category and the submitted text |
| 5 | Feedback reply | Admin replies | The reply plus the original quoted back |
| 6 | Feedback inbox alert | User submits feedback | A link to the admin console, and nothing else |
| 7 | Engagement nudge | A scheduled pass, at a civil hour where the reader lives | One thing at stake and the number behind it, a link into the app, and an unsubscribe line. The only mail that consults a preference, and the only one that can be switched off entirely |

Just as deliberate is what never sends: a feedback status change mails nobody (there is no listener, so no future endpoint can e-mail by accident), Google sign-ups get nothing (Google already verified the address), and nothing announces password-reset completion or the account's actual deletion.

The inbox alert is the only mail addressed to the operator rather than to a user, and its emptiness is the design. It carries a link and no category, no submitter and none of the submitted text: feedback can be personal, and copying it into a mail provider to save one click is a bad trade. Who receives it is whoever holds ROLE_ADMIN at the moment of the submission, minus the submitter, so an admin writing feedback gets the ordinary receipt and no second mail about themselves. One submission therefore produces one receipt plus one message per admin, each addressed individually — a shared To: line would show every admin the others' addresses.

```mermaid
flowchart LR
  REG["📝 Registration"] --> ES["✉️ EmailService"]
  PR["🔑 Password reset"] --> ES
  DEL["🗑️ Deletion code"] --> ES
  FB["💬 Feedback ack + reply"] --> ES
  FBA["🔔 Feedback inbox alert"] --> ES
  ES --> SMTP["📤 SMTP (StartTLS)"] --> INBOX["📬 Inbox"]
```

## Three ways to decouple from the transaction

Every mail must wait for its database transaction to commit; a reset link pointing at a token that rolled back would be worse than no mail. The codebase reaches that goal three different ways, and the differences matter:

| Flow | Mechanism | Thread |
|------|-----------|--------|
| Feedback ack, reply and inbox alert | @Async listener on an AFTER_COMMIT transactional event | Background |
| Password reset, deletion code | Hand-registered afterCommit synchronization | The request thread: the HTTP response waits on SMTP |
| Registration verification | Plain @Async event listener, no transaction phase | Background |
| Verification resend | Hand-registered afterCommit synchronization | The request thread: the HTTP response waits on SMTP |

The registration path deserves its own warning label: it is only correct because `registerUser` carries no `@Transactional`, so the save has already committed when the event publishes. Adding `@Transactional` to that method would silently reintroduce the send-before-commit race the other flows guard against.

No executor is configured; `@Async` rides the virtual-thread default. Unbounded, no queue, no retry anywhere.

## When SMTP fails

Each flow answers the failure differently, and the differences are the design:

- **Password reset**: the error is logged and the token row is deleted, so the 5-minute cooldown cannot strand a user waiting on a link that never left the building.
- **Deletion code**: same idea, sharper edge. The code row is discarded through a REQUIRES_NEW helper, because the discard runs after commit where a plain repository call would join a dead transaction. Nobody got the code, so nobody should sit out the cooldown.
- **Feedback mails**: logged and swallowed. The submission or reply survives; the receipt is best-effort. The receipt and the inbox alert sit in separate try blocks on purpose, and the alert catches again per recipient, so one dead mailbox costs one message rather than hiding a submission from the console.
- **Registration verification**: nothing catches it. The exception dies in the async handler's default logging, and the user row and its token stay. That used to be unrecoverable — login refuses unverified accounts with EMAIL_NOT_VERIFIED, the email column is unique so registering again is refused too, and after 24 hours the token expired with nothing able to issue another. The only repair was an UPDATE by hand.
- **Verification resend**: the cure for the line above, and it answers failure the way the deletion code does. The send runs in `afterCommit`, and a throw clears the account's `verificationTokenSentAt` through a REQUIRES_NEW helper, so nobody sits out a cooldown for a mail that never left. Worth saying plainly: this only catches SMTP refusing the handoff. A message the provider accepts and then bounces, or files as spam, throws nothing anywhere — which is why a second attempt the user can ask for matters more than any amount of failure logging.

There is no retry, no outbox, and no delivery tracking anywhere. What makes the swallowed failures visible is the logging pipeline: every failure logs at ERROR, and ERROR log lines become GlitchTip events. The error tracker is the retry bell.

Two operational notes, both the result of a fix rather than a design choice. Spring's mail health indicator is off: it opened a real authenticated SMTP session on every hit of `/actuator/health`, uncached, which handed the uptime monitor's verdict to the mail provider. It was also redundant, since a failed send already logs at ERROR and every ERROR is already a GlitchTip event. And all three JavaMail timeouts are pinned at 5s, because the default is to wait forever: the reset and deletion sends run synchronously inside `afterCommit`, before the JDBC connection returns to the pool, so an unanswered SMTP session used to hold both the user's request and one of Hikari's ten connections indefinitely.

## SMTP configuration

| Variable | Purpose |
|----------|---------|
| MAIL_HOST / MAIL_PORT | SMTP server, StartTLS enabled, auth on |
| MAIL_USERNAME / MAIL_PASSWORD | Credentials |
| MAIL_FROM | Sender address, defaults to MAIL_USERNAME |

Mail is not optional. The four core values ship without defaults, and the from-address resolution fails bean creation when the variables are missing entirely, so the app does not boot without a mail environment. Empty values boot and then fail per-send. The only sanctioned no-SMTP mode is the e2e profile, which sidesteps mail instead of disabling it: registration auto-verifies and skips the event, and the deletion code is returned in the response body. Both escape hatches are blocked in production by the boot validator.

## Templates and languages

No template engine and no template files: each mail body is an inline Java text block, HTML only, formatted with String.formatted. With two languages per mail that makes thirteen hardcoded templates sharing the same header, the brand blue, and the year-stamped footer — an odd number rather than an even one because the inbox alert is English only. The nudge's is the only body assembled from parts: its headline and text are chosen per trigger and escaped before they reach the frame, because they carry numbers read off the account. Every other template branches on the reader's language because a user reads it; that one is addressed to whoever operates the product, runs two sentences, and its payload is a URL.

Language selection is a two-branch decision per mail: anything starting with "pt" gets Portuguese, everything else (including null) gets English. The interesting part is where each flow reads the language from:

- The feedback acknowledgement prefers the language captured in the submission's UI context over the profile preference, because the receipt lands immediately and the profile field stays null until the user ever opens settings. Reading the profile first used to send every new account an English receipt.
- The feedback reply prefers the current profile preference, because a reply can land days later, when the captured context is stale.

User-authored and admin-authored text is HTML-escaped before interpolation, so a feedback body cannot inject markup into its own receipt.

## Cooldowns and rate limits

Two independent layers throttle the mail-producing endpoints:

| Flow | Service-level cooldown | Rate-limit bucket |
|------|------------------------|-------------------|
| Password reset | 5 min between requests per account; each new token invalidates the previous | 5 / 15 min per IP (auth tier) |
| Deletion code | 60 seconds, in seconds on purpose: the user is waiting on-screen and resend must work in the same sitting | 10 / hour per user |
| Registration | None | 5 / 15 min per IP (auth tier) |
| Verification resend | 60 seconds per account, in seconds for the same reason as the deletion code: the user is on the login screen reading "email not verified" and the button beside it has to work now. Each new token invalidates the previous, so one inbox never holds two live links | 5 / 15 min per IP (auth tier) |
| Feedback | None | 10 / hour per user |

One configuration nit carried here for honesty: the yaml default for the reset TTL is 15 minutes, while the env template still ships 30. The yaml wins unless the operator copies the template value.

## Test coverage

No test ever speaks SMTP. The feedback flows mock the JavaMailSender itself and assert on captured messages (recipients, subject, body, and the load-bearing case: a submission survives a throwing send). The deletion flow mocks the EmailService and pins the six-digit format, the hash-only storage, and the discard-on-failure behavior. The inbox alert has its own suite, asserting per recipient rather than by counting sends: every admin alerted exactly once, the submitter never alerted about their own message, no feedback text anywhere in the body, and a throwing receipt still leaving the console alerted. The resend flow arrived with its own suite, and one detail there is worth copying rather than rediscovering: it is the only mail test that is NOT `@Transactional`, because the send hangs off `afterCommit` and a test transaction that rolls back never commits, so every assertion about a mail going out would have passed against a service that sends nothing. Its negative assertions read the stored row rather than counting invocations, since the registration mail is async and racing it makes a coin toss of `verifyNoInteractions`. The one gap left: the password-reset flow has no dedicated test of its own, so its token-cleanup-on-failure behavior rides on code review alone.

## Engagement mail is opt-out

The switch was built before the sender, deliberately: a nudge is the first mail in this product that nobody asked for, and building consent after the thing that needs it is how a product ends up mailing people who cannot make it stop.

The state is one boolean and one token in a `notification_preferences` table, keyed by user. A table rather than two columns on `users`, because `users` is loaded in full by the security filter on every authenticated request and those columns would be read thousands of times a day to answer a question a nightly job asks.

The default is on. These are messages about the reader's own routine — a streak about to break, a goal running out of days — rather than offers, and they carry one-click opt-out; a default of off would mean the feature only ever reaches people who go looking for it in settings. Absence of a row means the default, so nothing was backfilled and no token exists until the first mail needs one.

The token is where this departs from every other token in the codebase, and deliberately. Password-reset tokens and deletion codes are stored as BCrypt hashes because they are one-shot secrets. This one is *stable* — every nudge for the rest of the account's life links to it — and a hash cannot be un-hashed to build that link, so hashing would force a fresh token per send and silently kill the unsubscribe link in every message already delivered. It is 256 random bits, stored raw, and the worst a leak of that column allows is unsubscribing somebody from mail they can re-enable in settings.

Two things about the endpoint are worth keeping. It is a POST, and the mail links to a page in the app which posts it, rather than to the API directly: mail clients prefetch links to build previews and scan for malware, so a state-changing GET gets clicked by a robot and unsubscribes people who only opened the message. And it is public — listed in **both** the security config's permitAll set and the security filter's own bypass list, which are two separate lists of the same thing with nothing checking that they agree. The filter runs first, so a path permitted in one and missing from the other is not public: it answers 401 before authorization is ever consulted, and the endpoint looks broken rather than protected.

## The nudges, and when they do not go out

Two triggers, and both report a cost that exists whether or not anybody is told about it. That is the bar a nudge has to clear here: if the mail has to manufacture the urgency, it is an advertisement.

| Nudge | Fires when | What it says |
|---|---|---|
| Recovery window closing | The oldest day still open for a retroactive check falls out of the backfill window after today, and it was missed | Which day expires tonight, and what a check on it still earns |
| Streak record at risk | The run is at or near the account's own record, today is scheduled, and nothing is checked yet | The run, the record, and that today is still open |

The first one invents nothing. `XpDecayCalculator` already reduces what a late check earns and `MAX_BACKFILL_DAYS` already closes the day for good; the mail reports a deadline the product already enforces. It quotes the percentage for *that account's* decay strategy, because `GRADUAL`, `FLAT` and `TIME_WINDOW` pay differently and a mail quoting the wrong number is worse than one quoting none.

The second is tied to the record rather than to the day, and the difference is the whole point. "You have not checked anything today" is the same query with none of the meaning: for somebody with no run going it is a reprimand for a day still in progress. Tied to the record it is a warning about losing something they actually built.

Most of the logic is exclusions, which is why most of its tests assert that nothing is sent. The one worth stating: a streak counts **scheduled** days, so on an unscheduled day it cannot break. Telling a Mon/Wed/Fri user on a Tuesday that their streak ends today is false, and one mail like that spends the credibility of every later one.

### Three limits, because they are three different limits

| Limit | Mechanism | Why it exists |
|---|---|---|
| One account, one kind, one day | A UNIQUE constraint on `notification_sends` | The pass runs hourly and comes back round; a check-then-insert cannot be trusted when what it protects is somebody's inbox |
| One account, across kinds | A minimum gap in days | Two triggers can each be individually justified on the same morning. Their sum is a sender that writes every day |
| Everybody, per day | A global cap, set to a third of the provider's allowance | That allowance also carries password resets. A reset that does not arrive because a nudge spent the budget is a far worse failure than a nudge that never goes out |

The dates are the **reader's**, not the server's. "Already sent today" has to mean their today, or an account far enough east receives the same nudge twice inside one of its own days. The consequence is that the global cap spans two dates at the boundary and is approximate at the edges — accepted, because the alternative makes the first limit wrong, and being a few mails out on a budget of hundreds is a much smaller error than mailing somebody twice.

### Send first, record second

The row that suppresses tomorrow's duplicate is written only after the mail is handed to the mail layer. The other order is safer against double sends and much worse in practice: a failed send would leave a row claiming the mail went out, and the nudge would be silently suppressed for the rest of the day with nothing to notice. The reverse costs one duplicate at worst, and the constraint refuses the third.

### Off by default, and that is the ordering mechanism

`engagement.enabled` defaults to false, so merging the sender sends nothing. This is not caution for its own sake. The privacy policy has to describe a new use of somebody's data *before* it starts — its own "Changes to this policy" section promises exactly that — so the sequence is: policy merges, policy deploys, then the flag flips. A flag makes that a property of the deployment rather than something an operator has to remember on the right day.

Every threshold beside it is configuration rather than a constant, because every one was chosen without a baseline: the product analytics that would justify a number only began collecting the day the sender was written.

### Who is excluded, and one reversal

Unverified addresses get no engagement mail. This reverses an earlier reading of the same data, which had the never-activated cohort as the biggest opportunity and e-mail as the only channel that could reach it. Both of those are still true — but the address was never confirmed, so it may not belong to the person who typed it, and mail to unconfirmed addresses is how a sending domain's reputation goes bad. That cohort already has an honest repair path: the verification resend, which is transactional, needs no preference, and is the right thing to send somebody whose address is still unproven.

The activation sequence therefore lives with the verification flow rather than here. It is onboarding, not engagement.

### Monitoring

The pass checks in with the collector after each cycle, through the same inverted heartbeat the snapshot job uses — the monitor alerts on the *absence* of a check-in, because a health endpoint returning 200 says nothing about whether a scheduled pass still runs. It matters more here than for snapshots: a snapshot job that stops eventually shows up as missing history, whereas a nudge job that stops looks exactly like a quiet week. Nobody complains about mail they did not receive.

## What could be improved

| Area | Current state | Note |
|------|--------------|------|
| Retry | Single attempt everywhere | An outbox table or provider-side retry would remove the GlitchTip-as-retry-bell pattern |
| Reset/deletion sending thread | Request thread waits on SMTP | Moving to the async listener pattern the feedback flows use would cut response latency |
| Templates | Thirteen inline HTML blocks | The nudge added its own, and its chrome is a copy of the others'. Extracting the shared frame would shrink the duplication; a template engine is still overkill |
| Reset flow tests | None | The only mail flow without direct coverage |
| Verification token at rest | Stored in plaintext on the users row | The reset token is a BCrypt hash in `{UUID}.{raw}` form; a database read hands out email verification for free. Aligning the two breaks every link already sitting in an inbox, so it wants its own change |
