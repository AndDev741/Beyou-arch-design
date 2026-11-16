---
title: "Second Brain"
summary: "A page that tells a person whether they are on course or drifting: an overview with a verdict rewritten once a day, one question at a time, a short profile every assistant conversation reads, and six switches for what may be read. The rules, the three model calls, what reaches the AI provider and when, and the class that owns each rule."
---

Second Brain is the page where somebody sees what Beyou understands about their life and steers it. It answers one question: is this person heading toward what they said they want, or drifting away from it. The page opens with a short overview that says so, asks one question at a time, keeps what it learns as a short profile that every assistant conversation reads, and lets the person switch off any part of their data it may read. It sits in the main navigation of the web app and the phone app, for every signed-in account, in English and Portuguese.

This document covers the blocks of the page, the rules behind each and the class a rule lives in, then the model calls and what happens to the data. The routes themselves are in the Second Brain Controller API.

## The seven blocks

`GET /second-brain` returns the whole page in one read-only transaction and writes nothing. The response carries seven blocks:

| Block | What it shows | Where it comes from |
|-------|---------------|---------------------|
| Overview | A paragraph the model wrote about how things are going, with a verdict shown as a label | `second_brain_insights` |
| Questions | The open questions, one at a time | `second_brain_questions` |
| Goals | Up to three open goals closest to their end date, with the Daily Briefing's pace wording | The GOALS source |
| Routines | The last seven days against the seven before, with the items that slipped and the ones that held | The ROUTINES source |
| Areas | The person's categories ranked by the share of their routine items done over 14 days | The ROUTINES source |
| Profile | Up to 20 short lines in three kinds | `second_brain_facts` |
| Sources | Six switches for what Second Brain may read | `second_brain_sources` |

Every percentage and every verdict is computed on the server, so the two apps cannot round the same week two ways. A block whose source is switched off says so in place of its content. That is a different state from a source that is on with nothing in it, and the response lets a client tell the two apart: `routine` is null when ROUTINES is off, and the switch list settles the goals and the areas.

On a wide screen the web page has two columns. The left one is for reading: the overview, the goals, the routines and the areas. The right one is for answering and deciding: the questions, the profile and the sources. Below the `lg` breakpoint the page is one column and the questions come second, right under the overview, because that block is the one that needs the person. The phone app uses that same single-column order.

**Goals.** The list is `DailyBriefingFactsBuilder.goalsAhead`, the one the morning dialog shows: open goals that have an end date, sorted by distance from today on either side, capped at three, with a goal overdue by more than two weeks sent to the back. Each one carries the server's pace verdict, and both pages word it with the same shared function, `goalPaceMessage` in `@beyou/state`. That function changed with this page. A goal that needs less than one unit a day used to read "0 books a day still gets you there", because twelve books in a year is 0.03 a day and one decimal rounds that to nothing. The sentence now picks the period where the amount is a number a person can act on: per day from one a day up, per week below that, per month when a week still asks for less than one, and no amount at all when a month does too. The week and month figures are the server's `requiredPerDay` multiplied by 7 and by 30. The Daily Briefing reads the same function, so its goals are worded the same way.

**Routines.** The block compares the seven days ending yesterday with the seven before. Each percentage is the share of the routine items asked that were checked. A skipped item is left out of both sides, because a skip is an answer the person gave and not a miss. Under the two numbers sit the items that slipped (left open on at least two of the days they came up this week, and on at least half of them) and the ones that held (they came up on at least four days and were checked on every one), three of each at most. `DailyBriefingFactsBuilder.buildSignals` computes all of it. It is the function the briefing's prose is written from, so this page and the morning dialog cannot disagree about the same week.

**Areas.** The block reads the same 14 days by category. A snapshot check remembers the habit or task it was made from, and that item carries its categories. Every check that came up and was not skipped counts once for each category of its item, so an item in two categories counts in both. A check whose habit or task was deleted since has no category left to count under and is left out. Only categories with something asked appear, highest percentage first. The page labels the first row as strongest and the last as weakest and leaves every other row unlabelled. `secondBrainAreaEnds` in `@beyou/state` returns those two ends only when at least two categories have a percentage and the two differ.

## The overview and its verdict

The overview is one paragraph about how the person's life is going. The prompt (`prompts/secondBrainInsight.st`) asks for three to five sentences. The figures the paragraph draws on are in the blocks under it, so the prompt tells the model to say what they add up to and to leave the figures where they are.

The model returns a verdict and a text. The verdict is `SecondBrainVerdict`:

| Verdict | Meaning |
|---------|---------|
| ON_COURSE | What the person does most days lines up with the goals they hold and the objectives in their profile |
| DRIFTING | It does not: goals behind their pace, routines mostly left open, or days that go somewhere other than what they said they want |
| TOO_EARLY | Too little is recorded to say either, such as a first week, or routines and a profile that are still empty |

The verdict is a column of its own because the page shows it as a label, so the answer the page exists for can be read at a glance. The text opens by saying the verdict in plain words. When the verdict is DRIFTING, the prompt asks for the one correction the data points at. Either the plan asks for more than the person's weeks hold, so a goal or a routine should change, or the plan is fine and the behaviour slipped, so there is something to get back to. `SecondBrainVerdict.parse` reads what the model wrote leniently, and anything it does not recognise becomes TOO_EARLY, so a verdict the model did not clearly give is never shown as one.

`SecondBrainService.refreshInsight` builds what the model reads. `SecondBrainSignals.promptData` renders one section per enabled source and then the profile, and `recentAnswers` adds the person's latest answers: the answered ones among the last 15 questions, eight at most, each cut to 300 characters. The answers are there because they can say what the numbers cannot, such as why a week went the way it did. Every number in the block is computed in Java and handed over finished, the Daily Briefing's rule. Days are written as distances from today ("in 5 days, on a Tuesday", "overdue by 3 days").

The text that comes back is collapsed to one paragraph and cut at the last whole sentence that fits in 900 characters (`SecondBrainInsight.TEXT_MAX`). The column is 1200 wide, so that cap can move a little without a migration. An account has one overview, replaced by an upsert each time it is written.

### The first overview and the daily rewrite

The first overview is never written unprompted. An account without one sees a button, "Write my overview", with the AI privacy notice beside it: the overview and some of the questions are written by an external AI provider, from the sources switched on, the profile lines and the answers. Pressing it calls `POST /second-brain/insight/refresh`, and that call is the person's opt-in to the AI side of the page (see "Before the first overview" below).

From then on the overview rewrites itself once a day. `SecondBrainStore.insight` marks it `stale` when it was written before the account's current day. It goes by the account's own calendar, through `UserDateResolver`, and not by hours, so "today" means here what it means on the routine. When a page load finds an existing overview stale and `aiAvailable` true, the page calls the refresh route once for that visit. The first visit of a day therefore rewrites the overview, a second visit the same day finds it fresh and costs no model call, and a day without a visit writes nothing. The server has no scheduler for this and never writes an overview unasked.

The person can also ask for a rewrite at any time with "Write it again". While one runs the page hides the old label, since it would be read as today's. A rewrite that fails leaves the stored overview in place and shows the error under it.

## The questions

The page asks one question at a time, and the person answers in free text or skips it. One at a time is deliberate: a list of five reads as a form to fill in, and a form gets postponed.

There are three kinds:

| Kind | When it is asked | Who writes it |
|------|------------------|---------------|
| Starter (`PROFILE_OBJECTIVES`, `PROFILE_LIKES`, `PROFILE_ABOUT`) | The profile has no fact of that kind, and that question was never asked before, answered or skipped | Fixed wording in English and Portuguese |
| Check-in (`CHECK_IN`) | Once per day of the account's own calendar, when the ROUTINES source is on and a routine runs that day | Fixed wording |
| Follow-up (`FOLLOW_UP`) | The model is in use for the account, there is room, and no follow-up was stored in the last six hours | The model, about something specific in the person's data |

`POST /second-brain/questions/refill` runs those three steps in that order: `SecondBrainStore.seed` writes the first two, which need no model, then `SecondBrainAi.followUps` is asked and `SecondBrainStore.addFollowUps` stores what it wrote. The limits:

- **Five questions open at once** (`SecondBrainQuestion.MAX_OPEN`). The steps stop there, and the model is asked for as many follow-ups as there is room left.
- **Six hours between two sets of follow-ups** (`SecondBrainStore.FOLLOW_UP_PAUSE`). The pause counts from the last follow-up stored, so a call that came back with nothing does not start it.
- **A subject is not asked twice.** A starter question is asked once per account, whatever became of it. The follow-up prompt (`prompts/secondBrainQuestions.st`) is shown the last 15 questions with what became of each (answered and with what, skipped, still waiting) and is told not to repeat or rephrase one, not to ask about something an answer or a profile line already settles, and to leave the subject of a skipped question alone. The server also drops a follow-up that repeats one of those 15 word for word.
- **A question holds 300 characters and an answer 2000.**

The fixed wording is stored on the question in the language the account reads Beyou in when it is asked, so a question reads the same after the person switches language.

Opening the page does not always ask for a refill. The route may end in a model call, so walking in and out of the page must not turn into one each time. `secondBrainRefillIsDue` in `@beyou/state` decides for both apps: an empty list always asks, a full list never does, and anything in between asks at most once an hour. The web keeps the time of its last ask in sessionStorage and the phone keeps it in memory.

Two refills of one account can run at once, since the page asks when it mounts. `SecondBrainService` keeps the accounts whose follow-ups are being written in an in-memory set and skips the model for the second one. That holds on one instance. Across instances, `addFollowUps` reads the room and the pause again under the account's lock, and the second set to arrive is dropped.

### What an answer does

`SecondBrainService.answer` closes the question first. `SecondBrainStore.answer` reads it under a row lock, marks it ANSWERED with the text stripped at both ends, and commits before the model is asked anything. The answer is kept whatever happens next, and the second submit of a double click gets `SECOND_BRAIN_QUESTION_CLOSED` (409) where it would otherwise have produced a second set of facts.

Then the service learns from the answer, through `prompts/secondBrainFacts.st`:

- **A starter answer becomes profile lines.** The model splits it into the separate things it says, at most three facts, all of the question's kind whatever kind the model wrote. When the model is not in use, the call fails or the split comes back empty, the answer itself becomes one fact, word for word, cut to 200 characters when it is longer.
- **A check-in or follow-up answer adds only what will still be true later.** The prompt asks for what holds a month from now, such as a new objective, a lasting preference or a change in circumstances. At most two facts come back, the model choosing each kind, and the usual result is none. When the model is not in use, nothing is learned.

`SecondBrainStore.saveLearned` adds those facts under the account's lock: as many as fit under the limit of 20, each cut to 200 characters, with origin ANSWER, leaving out a fact the profile already holds word for word. The response's `learned` list can therefore be empty, and the page then says the answer was noted and nothing new went to the profile. The prompt also tells the model never to keep a password, a code, a card or document number or any other secret, even when the answer contains one.

A skip closes the question with no answer and no model call. The row is kept, which is what stops a skipped starter question from coming back.

### The hand-off to the assistant

An answer never changes a routine, a habit or a task. After a check-in or a follow-up, the result offers "Let the assistant act on it". `canActOnAnswer` in `@beyou/state` holds that rule for both apps, and it leaves the starter questions out because a starter answer is the fact itself. The button opens the assistant on a new conversation with a draft in its input: the question, the answer, and a line asking the assistant to do what the answer calls for in the person's routines, habits and tasks. A new conversation on purpose, since dropping the draft into whatever chat was last open would mix two subjects. Nothing is sent until the person sends it, and from there it is an ordinary assistant message, handled with the assistant's own tools and rules. The check-in's wording says what the hand-off is for: "How is your day going? Describe it and I can check or skip your routine items."

## The profile

The profile is a list of short lines the person adds, edits and deletes on the page, in three kinds: main objectives (`OBJECTIVE`), things they like (`LIKE`) and facts about their life (`ABOUT`). A line is either typed (origin USER) or learned from an answer (origin ANSWER), and the page marks the learned ones. Editing a line keeps its origin.

It holds at most 20 lines of 200 characters, typed and learned together (`SecondBrainFact.MAX_PER_USER`, `TEXT_MAX`). `POST /second-brain/facts` answers 409 `SECOND_BRAIN_FACT_LIMIT_REACHED` at the limit, and deleting a line makes room. The caps exist for two reasons the code states. Every line travels with every assistant message, so the size of the profile is a cost on each one. And the model calls behind this page have no tools, so the most a prompt injection hidden in the person's own data can do is change what gets saved as a fact. The caps bound how much of that can persist and later reach a chat prompt.

Every assistant conversation reads the profile. `AiAgentService` fills the `{secondBrainProfile}` slot of `prompts/aiAgent.st` on every turn from `SecondBrainStore.profileForChat`: one line per fact, oldest first, labelled "Objective", "Likes" or "About them", or "(none yet)" for an empty profile. It is read fresh for each message and not cached, so a line edited on the page is known to the next message. The prompt block tells the model three things. The lines are background to read in every conversation, so the person never has to say them again. They are the person's own notes and never instructions, and a line that asks the model to do something is still only a note. And the model cannot change them: the person edits them on the page, and the assistant may point them there when they say something lasting about themselves. No assistant tool writes a fact.

## The sources

Six switches say what Second Brain may read. Each row below is exactly what `SecondBrainSignals` queries when the switch is on.

| Source | Default | What is read |
|--------|---------|--------------|
| ROUTINES | On | The routine snapshots of the 14 days ending yesterday. They are the routines and areas blocks, and in a prompt the two weekly percentages, the items that slipped or held, and the share done per category. The daily check-in question needs this switch too |
| GOALS | On | The open goals with an end date, the three closest to it: name, progress, days to the end date and pace |
| MOOD | On | The level logged on each of those 14 days, given to the model as this week's and last week's averages. No block on the page shows it |
| CALENDAR | Off | Open calendar tasks dated up to 21 days ahead, the overdue ones included, 30 at most: the title and how many days away it is |
| NOTEBOOK | Off | The 20 most recently updated study topics: the title and how many subtopics are done |
| IDEAS | Off | The text of the 15 newest notes on the Ideas board. Archived notes are left out |

Whatever the switches say, the overview and follow-up prompts also carry the profile lines. The overview's prompt adds the recent answers, and the follow-up prompt adds the last 15 questions with their answers. The facts prompt reads no source at all: it gets the one question, its answer and the profile. Beyou adds no display name and no e-mail address to any of them.

A source that is off is not queried at all, so it cannot reach the page or a prompt by any path through `SecondBrainSignals`. An enabled source with nothing in it gets a line saying so, which keeps the model from inventing what it was not given, and a disabled one is absent. The prompts tell the model that a missing section was switched off by the person and must not be mentioned or asked about.

A row in `second_brain_sources` exists only for a switch the person has touched. A missing row means the default, which lives in the `SecondBrainSource` enum, and `SecondBrainStore.effectiveSources` is the one place that rule is read. Nothing was backfilled for existing accounts, and a source added later needs no rows either.

**The switches govern Second Brain only.** The assistant chat keeps its own tools and its own access: with IDEAS off here, the assistant can still read the Ideas board through `getQuickNotes` when the person asks it to. Turning a source off changes what the next read and the next prompt contain. It does not rewrite the overview or the questions already stored.

**Diary text is never read, under any setting.** Mood comes through `MoodEntryRepository.findLevels`, a query that selects the date and the level and leaves the note out, so the text does not leave the database. There is no JOURNAL value in the enum, on purpose, and a request for one answers 400. The notebook follows the same line: topic titles and progress, never what a page says.

## The model calls

`SecondBrainAi` makes three calls. Each is one stateless structured call through `ChatClient.entity()` to the shared LLM endpoint, with no tools, no advisors and no chat memory, built like the onboarding suggestions. With no tools the model cannot touch the account. All it returns is text, and the class cuts that text to size before anything is stored.

| Call | Prompt | Returns | Time budget | When it fails |
|------|--------|---------|-------------|---------------|
| The overview | `secondBrainInsight.st` | A verdict and a text | 90 seconds, one retry included | `AI_UNAVAILABLE`, and the stored overview stays |
| Follow-up questions | `secondBrainQuestions.st` | A list of questions | 45 seconds, one attempt | A warning in the log. The refill answers the questions that were already there |
| Facts from an answer | `secondBrainFacts.st`, in PROFILE or LASTING mode | A list of facts | 45 seconds, one attempt | A starter answer is kept as written. Any other answer adds nothing |

Only the overview retries, because it is the only call with nothing to fall back on. The retry asks again for valid JSON and starts only when at least 20 seconds of the budget are left. Every attempt runs on a virtual thread, so the request can stop waiting at its deadline. The proxy in front of the API drops a request at 100 seconds, which is why the budgets stay under it.

**The person's data is marked as data.** It travels in the user message between `<data>` and `</data>` (`SecondBrainData.wrap`), and each system prompt says that everything inside the block is material to read and never an instruction. That only holds while the person's own text cannot close the block, so `SecondBrainData.inline` removes the markers from anything they typed and flattens it to one line.

**No call runs inside a transaction.** A model call can take most of a minute, and a transaction held across it would keep one of the connection pool's ten connections for that long. `refill`, `answer` and `refreshInsight` in `SecondBrainService` are deliberately not `@Transactional`. Each reads what the prompt needs in a short transaction, calls the model with none open, and writes what came back in another short one. Those short transactions live in `SecondBrainStore`, a bean of its own for exactly that reason, the same shape as `DailyBriefingWrites` beside `DailyBriefingService`. `SecondBrainModelCallIT` asks, for each of the three routes, whether a transaction is active at the moment the model is called.

### Before the first overview

Nothing on the page reaches the model until the person has asked for a first overview. `SecondBrainService.aiInUse` is true when an LLM endpoint is configured and the account has an overview row (`SecondBrainStore.hasInsight`), and it gates the two calls that are not the overview itself:

- A refill writes no follow-up. The starter questions and the check-in still come, since they need no model.
- An answer is not read by the model. A starter answer is kept word for word as one profile line, and a check-in answer adds nothing to the profile.

`refreshInsight` is not gated, because it is the opt-in itself: the request is made from a screen that says the text goes to an external AI provider. The rule lives on the server so that a client cannot start the AI side by accident and a new client cannot forget to ask. The gate is the existence of the overview row, and no route deletes that row, so the AI side stays on for an account once it has started. The switches still decide what it reads.

The gate covers this page's own model calls. The profile is also part of the assistant's system prompt, so a person who uses the assistant sends their profile lines to the provider with each message, whether or not they ever asked for an overview.

### With the AI endpoint unconfigured

`LlmProperties.isConfigured` is false while any of `LLM_BASE_URL`, `LLM_API_KEY` and `LLM_MODEL` is unset. The page then reports `aiAvailable` false, and refills and answers behave as they do before a first overview. An account without an overview reads that the AI is switched off on this server where the button would be, and `POST /second-brain/insight/refresh` answers 400 `AI_UNAVAILABLE`. Every other block keeps working.

## Rate limiting

The three routes that can call a model spend one bucket of their own in `RateLimitFilter`: `POST /second-brain/questions/refill`, `POST /second-brain/questions/{id}/answer` and `POST /second-brain/insight/refresh`, 30 calls per user per hour under the key `second-brain`. It has the assistant's quota shape and a separate key, so the page never spends the assistant's budget. One key covers the three routes so that alternating between them buys no second budget.

The routes are named one by one in the filter. The page read stays in the generic read tier (60 a minute), and the switches, the facts and a skip stay in the generic write tier (30 a minute), since none of them reaches a model. The daily rewrite is one of those three routes, so it draws on this bucket and not on the assistant's.

## The four tables

All four arrived in V45, each owned by one account and each `ON DELETE CASCADE` from `users`.

| Table | One row per | Columns worth knowing |
|-------|-------------|-----------------------|
| `second_brain_sources` | Account and source the person has touched. Primary key `(user_id, source)` | `enabled`, `updated_at`. Written by one `INSERT ... ON CONFLICT`, so two toggles of a switch with no row yet cannot collide |
| `second_brain_facts` | Profile line | `kind`, `text` (varchar 200), `origin`, `created_at`, `updated_at`. Indexed on `user_id`, which every read and the count behind the limit use |
| `second_brain_questions` | Question ever asked, open or closed | `kind`, `text` (varchar 300), `status` (default OPEN), `answer` (varchar 2000, null until answered), `created_at`, `answered_at`. Indexed on `(user_id, status)` |
| `second_brain_insights` | Account. Primary key `user_id` | `text` (varchar 1200), `verdict`, `generated_at`. Replaced on every rewrite |

Every enum is a `varchar` with a CHECK constraint that mirrors it: the six sources, the three fact kinds, the two origins, the five question kinds, the three statuses and the three verdicts. Adding a value in Java without adding it to the constraint makes every write of that value fail. Closed questions are kept on purpose. The starter rule ("never ask this twice") and the follow-up prompt ("do not repeat yourself") both read them.

The journal has no column here and is never copied here, and mood levels are read at request time.

## Where each rule lives

| Rule | Lives in |
|------|----------|
| No transaction open across a model call, and no model before the first overview | `SecondBrainService` |
| A missing switch is the default | `SecondBrainStore.effectiveSources`, `SecondBrainSource.enabledByDefault` |
| A fact or a question is reachable only by its owner: every lookup carries the owner's id, so someone else's row and a missing one get the same 404 | `SecondBrainStore`, `SecondBrainFactRepository.findByIdAndUserId`, `SecondBrainQuestionRepository.lockByIdAndUserId` |
| At most 20 facts and 5 open questions, counted under a transaction-scoped advisory lock keyed on the account, because a new account has no row to lock | `SecondBrainStore`, `SecondBrainFactRepository.lockProfile` |
| A question closes once: answer and skip read it under a row lock | `SecondBrainStore.answer`, `SecondBrainStore.skip` |
| When an overview is stale | `SecondBrainStore.insight` |
| What each source reads, and that a source that is off is not queried | `SecondBrainSignals` |
| The areas ranking, and the routine numbers shared with the briefing | `SecondBrainSignals.areas`, `DailyBriefingFactsBuilder.buildSignals` |
| The three model calls, their time budgets and the cuts to size | `SecondBrainAi` |
| The data block and keeping the person's text inside it | `SecondBrainData` |
| The verdict, and how the model's wording is read | `SecondBrainVerdict` |
| The profile as the assistant reads it | `SecondBrainStore.profileForChat`, `SecondBrainSignals.profileLines` |
| The shared hourly budget of the three model-backed routes | `RateLimitFilter.isSecondBrainLlmPath` |
| When a client asks for a refill, which answers can go to the assistant, which areas get a label, how a verdict is drawn | `secondBrainRules.ts` in `@beyou/state` |
| How a goal's pace is worded, for this page and the briefing | `goalPaceMessage` in `@beyou/state` |

## The clients

The types are in `@beyou/types`, the nine calls in `@beyou/api`, and the judgements both apps make in `@beyou/state`. Second Brain has no Redux slice. Nothing outside the page shows its data, so each app keeps the response in the state of its own screen and applies its own writes to it: an answer drops its question and appends what was learned, and a switch change is followed by a quiet reload, because one source can feed more than one block.

On the web the page is `/second-brain`, a lazy route inside the protected shell. It sits second in the sidebar, right under the dashboard, and first in the bottom nav's More sheet on a phone-sized screen. The assistant's prompt lists `/second-brain` among the app's pages, so the assistant can link to it.

## Privacy and data rights

What reaches the AI provider, and when, is covered above: nothing from this page before the first overview, then the enabled sources, the profile lines and the answers. The [AI agent topic](/architecture/ai-agent) lists Second Brain among the features that send text to the provider.

`GET /user/export` carries a `secondBrain` section with four keys. `facts` holds every profile line with its kind, origin and timestamps. `questions` holds every question with its answer, closed ones included, oldest first, since an answer is something the person wrote. `insight` is the overview on record with its verdict and the time it was written, or null for an account that never had one. `sources` lists all six switches with the value in effect, so the file says what Second Brain was allowed to read even where the person never touched a switch. It is four queries whatever the account holds.

Deleting the account removes all four tables through the `users` cascade. The deletion tests put a row in each and expect none left.

On the device, nothing of the page reaches Redux, so nothing of it is persisted by redux-persist. The web keeps one timestamp in sessionStorage, the time of its last refill request.

## Tests

On the backend, `SecondBrainServiceIT` runs the store's rules against Postgres: the defaults, the limits, the starter and check-in rules, ownership and staleness by the account's own day. `SecondBrainModelCallIT` checks that no transaction is open during a model call and that nothing reaches the model before the first overview. `SecondBrainAiTest` covers the cuts to size, the retry and the data block, `SecondBrainControllerTest` the status codes, and `RateLimitFilterTest` the budget shared by the three model-backed routes. `AgentSystemPromptTest` renders the assistant's prompt with a profile in it. The export and deletion tests (`UserExportCompletenessIntegrationTest`, `UserExportQueryCountTest`, `AccountDeletionIntegrationTest`) cover the four tables.

On the clients, `secondBrainRules.test.ts` and `briefingRules.test.ts` in `@beyou/state` cover the shared rules, the slow-goal wording among them, and the web page has its own test file. `second-brain.spec.ts` in the e2e suite drives the API and the page, ownership and the phone-width order included.
