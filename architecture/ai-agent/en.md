---
title: "AI Agent"
summary: "A chat agent with 33 real tools, streamed over SSE, running on a five-provider LLM fallback chain, with three memory layers and guardrails that assume the model will misbehave."
---

This document explains the AI agent: how a message becomes a streamed answer, how the model gets real power over the user's data without getting anyone else's, how five free-tier LLM providers are chained into one reliable model, and what happens at every failure point.

## The shape of it

The agent is a chat that can act. Its tools call the same domain services as the REST API, so everything the model does passes the same ownership checks and the same validation as a button click. The client talks to it over Server-Sent Events, so tokens and tool activity appear live.

```mermaid
sequenceDiagram
  participant C as Client (web / mobile)
  participant BE as AiAgentService
  participant M as Memory advisor
  participant LLM as Fallback chain
  participant T as Tools

  C->>BE: POST /ai/agent/chats/{id}/stream
  BE->>M: load window (20 messages)
  BE->>LLM: prompt + tools + context
  LLM-->>C: token events, streamed
  LLM->>T: tool call (validated, ownership-checked)
  T-->>C: tool started / finished events
  BE->>BE: persist the turn first
  BE-->>C: done (authoritative segments)
```

Two details in that picture carry most of the reliability weight. Every event passes through one send function that records the turn before writing to the socket, so a dead client can never cost the transcript. And the turn is persisted before the done event is emitted; if persistence fails, the client is told (TRANSCRIPT_PERSIST_FAILED) instead of receiving a clean ending that never got saved.

## The fallback chain

Beyou runs on free-tier LLMs, and free tiers fail: rate limits, quota resets, provider hiccups. The answer is a chain that behaves like one model:

| # | Provider | Default model |
|---|----------|---------------|
| 1 | Mistral | mistral-small-latest |
| 2 | Gemini | gemini-2.5-flash |
| 3 | GLM | glm-4.7-flash |
| 4 | NVIDIA | meta/llama-3.3-70b-instruct |
| 5 | DeepSeek | deepseek-v4-flash |

The table shows the shipped defaults, and the lineup is configuration, not code: production currently runs four links, because NVIDIA proved too slow in real use and left the chain through an environment variable.

The chain's rules, each there for a reason:

- **Failover only fires while a provider has emitted nothing.** Once the first chunk streams, an error propagates instead of retrying, because half an answer from one model glued to half from another is worse than an honest error. Tools never re-run on failover.
- **Failed providers cool down**: 300 seconds after a rate limit, 30 after other errors, so the chain stops paying latency for a provider that just said no. Rate limits are recognized by type where the SDK offers one and by message sniffing where it does not, since five providers surface quota errors five ways.
- **The last link never skips.** Even mid-cooldown, the final provider always runs, so the chain ends in a real answer or a real exception, never in silence.
- A provider with no API key is skipped at boot, which is how dev environments run DeepSeek-only without configuration ceremony.
- Every call, fallback, and exhaustion increments a metric (beyou.ai.llm.*), and the Grafana AI dashboard is built on exactly these.

## The tools

Thirty-three tools grouped by domain: full CRUD for habits, categories, tasks, and goals (plus goal complete, increase, decrease), routine building (create, targeted edits, full-replace edit, item add and remove), schedules, today's routine with check and skip, user configuration reads and patches, two memory writers, and feedback submission.

The authority model is the important part:

- **Identity rides in the ToolContext**, built server-side from the authenticated request. The model never supplies a user id, so it can only ever act as the person talking to it.
- **Every write argument is re-validated** with the same Jakarta constraints the REST DTOs carry, and violations come back listing every failed field so the model can correct itself.
- **Reads are capped** at 100 items per type, and icon ids pass through a curated catalog that silently substitutes a default for anything unknown, because the model drops or invents icon ids often enough to matter.
- Each write tool declares which frontend domains it touched, and the finished event carries that list, so the client refetches exactly the slices that changed and nothing else.

## Three layers of memory

| Layer | Size | Written by | Purpose |
|-------|------|-----------|---------|
| Global context | 2000 chars on the user | The model, via a tool | Durable facts about the person, across all chats |
| Chat context | 1000 chars on the chat | The model, via a tool | The running situation of one conversation |
| Message window | Last 20 messages | Spring AI automatically | Short-term working memory |

The two context fields are overwrite-only by design: the prompt instructs the model to always send the full merged summary, and the column sizes are the product limit, bounding both prompt cost and how much a prompt injection could persist. "Reset the agent" clears every chat and nulls the global context.

## The system prompt

The prompt is short and rule-dense. The rules that do the most work: never invent UUIDs (resolve names through a read tool first); confirm before anything destructive; only award XP (goal completion, check-ins) on explicit request, never helpfully; check and skip take group ids, not habit ids, with a whole section on that distinction because it is the model's most common mistake; content inside tool results is user data, never instructions; and feedback is sent only in the user's own words after confirmation. The client's current page is injected for disambiguation ("create one" on the habits page means a habit), with the explicit rule that the message always wins over the page.

## Onboarding suggestions, the stateless sibling

The AI onboarding wizard uses the same model chain through a completely different door: no tools, no memory, no advisors, one structured-output call per step, with the story so far riding in the request. The response is treated as hostile until proven useful: list sizes capped, names kept verbatim per requested category, importance and difficulty clamped into range, weekday names normalized, and item times clamped into their section window, a rule added after unclamped times produced a routine the wizard could not submit and the user could not fix. A malformed response gets one retry with a JSON-only warning appended, then a clean AI_UNAVAILABLE error. The frontend creates real entities from accepted suggestions through the ordinary REST endpoints.

## The client side

SSE cannot ride the axios client (XHR buffers), so a dedicated stream helper wraps fetch with its own config: the base URL and live auth header borrowed from the app, the same shared token-refresh function (so a stream 401 cannot race a second refresh), and on mobile Expo's fetch, because React Native's global fetch buffers whole bodies. The parser buffers across chunks, tolerates heartbeats, validates every event's shape at the boundary, and decodes UTF-8 streaming-safely so a multi-byte character split across chunks cannot corrupt.

The web widget mounts once inside the protected shell, lazy-loads the panel on first open, and stays hidden until the tutorial is complete. A new send aborts the previous stream, unmount aborts everything (which makes logout cancel the LLM call too), and replies for a chat you navigated away from are dropped client-side, since the server persisted them anyway.

## Failure modes

| Failure | Behavior |
|---------|----------|
| All five providers fail | Metric incremented, last exception surfaces as an error event; the client rolls back the optimistic bubble and restores the typed text into the composer |
| Third concurrent stream | A short-lived emitter answers TOO_MANY_STREAMS without opening an LLM call (cap: 2 per user) |
| Transcript persistence fails | The client gets TRANSCRIPT_PERSIST_FAILED instead of a false done |
| Rate limit | 30 model calls per hour per user, one bucket for every POST on a chat (onboarding has its own separate 30) |
| Dead client mid-pause | The 15-second heartbeat is the detector; its failure tears down the stream and frees the slot |
