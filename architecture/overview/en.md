---
title: "Architecture Overview"
summary: "High-level view of Beyou's architecture — frontend, backend, database, and external integrations."
---

This document describes the overall architecture of the Beyou application, covering the main layers, data flow, domain model, authentication, and gamification system.

## System Architecture

```mermaid
flowchart LR
  FE["⚛️ Frontend<br/>React · TypeScript · Vite"]
  BE["🍃 Backend<br/>Spring Boot · Java 21"]
  DB[("🐘 Database<br/>PostgreSQL 15")]
  GO["🔐 Google OAuth"]
  GH["📦 GitHub API"]
  ML["✉️ SMTP Server"]

  FE -->|"REST API (JWT)"| BE
  BE -->|JPA / Hibernate| DB
  BE <-->|OAuth 2.0| GO
  BE <-->|Docs Import| GH
  BE -->|Password Reset| ML
```

## Tech Stack

| Layer | Technologies |
|-------|-------------|
| **Frontend** | React 18, TypeScript, Vite, Redux Toolkit + redux-persist, Axios, react-hook-form + Zod, i18next (en/pt), Tailwind CSS 3 |
| **Backend** | Spring Boot 3.3, Java 21 (virtual threads), Spring Security, JWT (auth0 java-jwt), Undertow, Spring AOP, Lombok |
| **Database** | PostgreSQL 15, Hibernate JPA, UUID primary keys, ddl-auto: update |
| **DevOps** | Docker Compose, nginx (prod), hot-reload (dev) |

## Domain Model

```mermaid
erDiagram
  USER ||--o{ CATEGORY : owns
  USER ||--o{ HABIT : owns
  USER ||--o{ TASK : owns
  USER ||--o{ GOAL : owns
  USER ||--o{ ROUTINE : owns

  CATEGORY }o--o{ HABIT : tagged
  CATEGORY }o--o{ TASK : tagged
  CATEGORY }o--o{ GOAL : tagged

  ROUTINE ||--o{ ROUTINE_SECTION : contains
  ROUTINE ||--|| SCHEDULE : "scheduled by"

  ROUTINE_SECTION ||--o{ HABIT_GROUP : groups
  ROUTINE_SECTION ||--o{ TASK_GROUP : groups

  HABIT_GROUP }o--|| HABIT : references
  TASK_GROUP }o--|| TASK : references

  HABIT_GROUP ||--o{ HABIT_GROUP_CHECK : tracks
  TASK_GROUP ||--o{ TASK_GROUP_CHECK : tracks
```

### Entity highlights

- **User** — profile, preferences (theme, language, widgets), and embedded XP progression (level, xp, constance streak).
- **Category** — groups habits, tasks, and goals via ManyToMany. Has its own XP/level.
- **Habit** — trackable behavior with importance, difficulty, motivational phrase, and XP/level progression.
- **Task** — similar to habit but can be one-time (oneTimeTask) with soft-delete via markedToDelete.
- **Goal** — target-based with currentValue / targetValue, status (active/completed/failed), and term (short/long/life).
- **Routine** — abstract base with DiaryRoutine concrete type. Contains sections with habit/task groups.
- **Schedule** — days of the week (Monday–Sunday) linked to a routine.
- **Checks** — daily check/skip records for habit and task groups inside routines, with XP generation tracking.

## Authentication Flow

```mermaid
sequenceDiagram
  participant U as User
  participant FE as Frontend
  participant BE as Backend
  participant GO as Google

  rect rgba(59, 130, 246, 0.25)
  U->>FE: 🔑 Email + Password Login
  FE->>BE: POST /auth/login
  BE-->>FE: JWT (header) + Refresh Token (HttpOnly cookie)
  FE->>FE: Store JWT in memory
  end

  rect rgba(234, 88, 12, 0.25)
  U->>FE: 🔐 Google OAuth Login
  FE->>GO: Authorization redirect
  GO-->>FE: Authorization code
  FE->>BE: GET /auth/google?code=...
  BE->>GO: Exchange code for access token
  GO-->>BE: User profile
  BE-->>FE: JWT + Refresh Token
  end

  rect rgba(16, 185, 129, 0.25)
  FE->>BE: 🔄 Token Refresh — request with expired JWT
  BE-->>FE: 401 Unauthorized
  FE->>BE: POST /auth/refresh (cookie)
  BE-->>FE: New JWT + new Refresh Token
  FE->>BE: Retry original request
  end
```

### Token details

- **Access token (JWT)** — 15 minutes, HMAC256, sent in the Authorization: Bearer header.
- **Refresh token** — 15 days, opaque hashed token, HttpOnly cookie. Old token revoked on refresh.
- **Password reset** — secure token via email, 30 min TTL, 5 min cooldown between requests. All refresh tokens revoked on reset.

## API Layer

14 REST controllers organized by domain:

| Group | Controllers | Base paths |
|-------|-----------|------------|
| **Auth** | AuthenticationController | /auth/* |
| **Core entities** | CategoryController, HabitController, TaskController, GoalController | /category, /habit, /task, /goal |
| **Routines** | RoutineController, ScheduleController | /routine, /schedule |
| **User** | UserController | /user |
| **Docs** | ArchitectureDocsController, DesignDocsController, ApiDocsController, ProjectDocsController, SearchDocsController, DocsImportController | /docs/* |

### Request/response pattern

```mermaid
flowchart LR
  REQ["📥 Request"] --> FILT["🛡️ Security Filter<br/>JWT validation"]
  FILT --> CTRL["🎯 Controller<br/>DTO validation"]
  CTRL --> SVC["⚙️ Service<br/>Business logic"]
  SVC --> REPO["💾 Repository<br/>JPA queries"]
  REPO --> DB[("🐘 PostgreSQL")]
  DB --> REPO
  REPO --> SVC
  SVC --> MAP["🔄 Mapper<br/>Entity → DTO"]
  MAP --> CTRL
  CTRL --> RES["📤 Response"]
```

- Request DTOs validated with Jakarta Bean Validation (@NotBlank, @Size, @Email).
- Responses mapped through dedicated Mapper classes (entity → response DTO).
- Global exception handler translates errors into standardized ApiErrorResponse with error keys for frontend i18n.

## State Management (Frontend)

```mermaid
flowchart TD
  AX["Axios + Interceptor"]
  ST["Redux Store<br/>16 slices"]
  PS["redux-persist<br/>localStorage"]
  UI["React Components"]

  UI -->|"dispatch actions"| ST
  ST -->|"selectors"| UI
  ST <-->|"hydrate / persist"| PS
  UI -->|"API calls"| AX
  AX -->|"dispatch on success"| ST
  AX -->|"auto 401 → refresh"| AX
```

### Key slices

| Slice | Purpose |
|-------|---------|
| perfil | User profile, XP, level, theme, language, constance |
| habits, tasks, goals, routines, categories | Entity lists |
| editHabit, editTask, editGoal, editRoutine, editCategory | Edit mode state |
| todayRoutine | Today's scheduled routine for dashboard |
| viewFilters | Sort/filter preferences per page |
| register, errorHandler | Auth and error state |

## Gamification System

```mermaid
flowchart LR
  ACT["✅ Check habit/task<br/>in routine"] --> XP["🎮 XP Calculator"]
  XP --> UXP["👤 User XP<br/>+ level up"]
  XP --> HXP["💪 Habit XP<br/>+ level up"]
  XP --> CXP["📂 Category XP<br/>+ level up"]
  XP --> RXP["📋 Routine XP<br/>+ level up"]
  ACT --> STR["🔥 Constance<br/>streak tracking"]
```

- **XpProgress** is an embeddable component shared by User, Category, Habit, and Routine.
- XP is generated when a habit or task is checked inside a routine.
- Level progression follows a seeded XP-per-level table (XpByLevelSeeder).
- Constance (streak) tracks consecutive completed days on the User entity.
- Goals award a fixed xpReward on completion.

## Infrastructure

```mermaid
flowchart TB
  subgraph Docker Compose
    FE["⚛️ Frontend<br/>:3000"]
    BE["🍃 Backend<br/>:8099"]
    DB[("🐘 PostgreSQL<br/>:5490")]
    NG["🌐 nginx<br/>(prod only)"]
  end

  NG --> FE
  NG --> BE
  FE --> BE
  BE --> DB
```

- **Dev mode** — up-dev.sh: hot-reload for frontend and backend, direct port access.
- **Prod mode** — up-prod.sh: nginx reverse proxy routing /api → backend, / → frontend.
- **Reset** — reset-db.sh: wipes PostgreSQL data volume.
- Environment configured via .env file with secrets for JWT, Google OAuth, SMTP, CORS, and docs import.
