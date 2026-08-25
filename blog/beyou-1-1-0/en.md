---
title: "Beyou 1.1: The Release My Test Users Wrote"
summary: "I tagged 1.0 on 17 August and handed the app to people who had never seen it. Eight days later, 1.1 closes thirty tickets they found for me: an onboarding that created 58 habits when someone asked for 3, a timezone every account was born with and nobody ever chose, and an account you could lock yourself out of forever by losing one email."
---

I tagged 1.0 on 17 August. Then I gave the app to a handful of people who had never seen it.

Eight days later I am tagging 1.1. It closes thirty tickets, and almost none of them are features. What they found described my blind spots better than I could have, so this post is mostly their list.

## The onboarding that got carried away

One of my test users picked three habits in the AI onboarding. They ended up with fifty-eight.

The wizard creates real habits through the ordinary REST endpoints, one call per habit, and it puts a Retry button in front of you when a call fails. Retry re-ran the whole batch. Every press added another full set of everything that had already worked, and the failures kept the button on screen. Press after press, their account turned into a wall of duplicates.

The creates now read the account first and skip anything already there by name, so Retry is idempotent. I also went into the database and cleaned their account up by hand, which is the sort of chore that makes you fix a bug properly rather than quickly.

## A day that turned over an hour late

I found this one by reading my own code, and it turned out smaller than my first write-up of it claimed.

Every account was created with its timezone set to `UTC`, and nothing ever changed it unless you walked into Settings and clicked the suggestion yourself. `UTC` was also indistinguishable from a deliberate choice, because there was no null state and no flag, so nothing could safely backfill it either.

My original note on this card said a user in Brazil was losing habits every night. That did not survive checking. Beyou has no production users yet, and the only reachable accounts are mine. The honest version is seasonal and small: Lisbon is UTC+0 in winter, where the stored zone is simply correct, and UTC+1 in summer, where the day turns over an hour late. A check made between midnight and 1am local lands on the previous day, and if it was that calendar day's only check, the real day closes as missed at 3am. One hour, seven months a year, on test accounts.

The large-offset numbers are still worth writing down, because they are what this does the moment somebody signs up several hours from UTC. At UTC-3 the day rolls at 9pm and closes at 11pm, so an evening routine gets filed under tomorrow while today is stamped missed, and `DayCloseService` is insert-only so that stamp never heals. Fixing this before anyone is in that position was the cheapest the repair will ever be.

The browser and the phone now send the zone the device is really in, on all four signup paths. A `timezone_source` column separates never-set from chosen, so the one-shot adoption for existing accounts only ever writes over the default. That policy lives in the backend rather than in each client, because a laptop opened in another country should not move the day boundary for someone who is travelling.

A second bug came out of the same audit and survives the timezone fix on its own: `checkTime` was stamped from the server's clock while `checkDate` used the owner's zone. Both read the owner's zone now.

I skipped the `X-Timezone` header my own scope had called for. It is a second adoption path that has to stay consistent with the first, it makes a GET write to the database, and the only accounts it reaches are ones that never open the app again, which no client-side mechanism can help with anyway.

## Locked out by one lost email

If your verification email never arrived, you had no way to ask for another. The account existed, refused to log you in, and offered no route forward. Registering again with the same address was blocked, correctly, because the address was already taken.

There is a resend now, and it sits on the screen that tells you the email is unverified, which is where someone in that state is actually standing. The endpoint answers identically for an address that exists and one that does not, so nobody can use it to find out who has an account. Google sign-in used to walk straight past the verification gate as well. That is closed.

## The agent learned to count

The AI agent took more fixes than anything else in this release.

It could not move goal progress at all. Increase and decrease both threw a lazy-initialization error, because the tool ran outside a session and the goal's collections were never loaded. It also picked goals by an id it had guessed instead of by name, so now and then it would create a second goal rather than edit the one you meant.

The schedule tool advertised MONDAY through SUNDAY while the enum was Monday. Every schedule the agent tried to build failed on the first attempt, came back as an error, and cost a full extra round trip to the model before it landed. Adding a habit to a routine section returned the new group with a null id, because the mapper ran before the flush. Editing a routine that already had group ids threw a detached-entity error.

None of these are interesting bugs on their own. Together they made the agent feel unreliable, which is worse than an agent that plainly refuses.

The chat also stopped shrugging. When a provider fails, or when you reach the hourly budget, it now says which of those happened. The rate limit reaches the streaming endpoint, which it did not before, and the provider call is no longer cut off at two minutes on a long answer.

## I finally know whether anyone is using it

For the whole of 1.0 I had no idea how many people opened the app on a given day.

Beyou now sends product events to PostHog from the web app, the mobile app, the docs site and the landing page, routed through a first-party proxy so an ad blocker does not quietly delete half the picture. The backend reports who is active and when people last logged in, and Grafana has a board for it. Signup day travels with the identify call, so a cohort can be aged.

I stripped OAuth codes and password reset tokens out of the capture before any of this shipped. Analytics libraries take the whole URL by default, and both of those live in query strings.

## Backups, at last

In the self-hosting post I wrote that backups were the honest gap and that they sat above everything else on my infrastructure list. They are done. Restic pushes to Cloudflare R2 nightly, there is a weekly restore drill, and the repository size is capped, because R2 has no hard spend limit and I would rather hear about it from a guard than from a bill.

## Getting ready for the Play Store

A good slice of this release started as Play Store paperwork and turned into real fixes. The app was declaring permissions it never used, which Android shows to people as though it does. The privacy policy the listing points at now exists, says what the analytics actually send, and gives the routine emails a legal basis before those emails are switched on.

Profile photos can be removed now, and they appear in your data export, which they did not. The export had an N+1 and got its own rate limit bucket, since it is comfortably the heaviest thing one account can ask for.

## What is still open

Engagement emails are built and switched off. The sender, the consent, the preference toggle on both platforms and the unsubscribe are all in, sitting behind a flag I have not turned on, because I want to read a couple of weeks of analytics before I start emailing anyone.

Push notifications remain the biggest thing missing. A streak about to break is exactly the moment a reminder would earn its keep, and Beyou has nothing there yet.

The Play Store listing is not live either. The signed bundle builds on demand. The rest of it is paperwork.
