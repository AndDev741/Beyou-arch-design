---
title: User Flow Journey
summary: From onboarding to daily habit check-ins.
---

The design below shows the full experience for a new user, mixing narrative and diagrams.

## Onboarding sequence

```mermaid
sequenceDiagram
  participant U as User
  participant App as Beyou App
  participant API as Backend
  U->>App: Opens Beyou
  App->>API: Checks session
  API-->>App: Session invalid
  App-->>U: Shows onboarding
  U->>App: Creates first habit
  App->>API: Saves habit
  API-->>App: Habit saved
```

## Daily flow

```mermaid
flowchart LR
  Start([Open app]) --> Dash[Home dashboard]
  Dash --> Check{Check-in?}
  Check -->|Yes| Habit[Log habit]
  Check -->|No| Explore[Explore tips]
  Habit --> Reward[XP + streak]
  Reward --> Dash
  Explore --> Dash
```

## Notes

- Keep the check-in call-to-action visible after onboarding.
- Reward feedback should be immediate to reinforce the habit loop.
