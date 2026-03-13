---
title: "Visão Geral da Arquitetura"
summary: "Visão de alto nível da arquitetura do Beyou — frontend, backend, banco de dados e integrações externas."
---

Este documento descreve a arquitetura geral da aplicação Beyou, cobrindo as camadas principais, fluxo de dados, modelo de domínio, autenticação e sistema de gamificação.

## Arquitetura do Sistema

```mermaid
flowchart LR
  FE["⚛️ Frontend<br/>React · TypeScript · Vite"]
  BE["🍃 Backend<br/>Spring Boot · Java 21"]
  DB[("🐘 Banco de Dados<br/>PostgreSQL 15")]
  GO["🔐 Google OAuth"]
  GH["📦 GitHub API"]
  ML["✉️ Servidor SMTP"]

  FE -->|"REST API (JWT)"| BE
  BE -->|JPA / Hibernate| DB
  BE <-->|OAuth 2.0| GO
  BE <-->|Import de Docs| GH
  BE -->|Reset de Senha| ML
```

## Stack Tecnológica

| Camada | Tecnologias |
|--------|-------------|
| **Frontend** | React 18, TypeScript, Vite, Redux Toolkit + redux-persist, Axios, react-hook-form + Zod, i18next (en/pt), Tailwind CSS 3 |
| **Backend** | Spring Boot 3.3, Java 21 (virtual threads), Spring Security, JWT (auth0 java-jwt), Undertow, Spring AOP, Lombok |
| **Banco de Dados** | PostgreSQL 15, Hibernate JPA, chaves primárias UUID, ddl-auto: update |
| **DevOps** | Docker Compose, nginx (prod), hot-reload (dev) |

## Modelo de Domínio

```mermaid
erDiagram
  USER ||--o{ CATEGORY : possui
  USER ||--o{ HABIT : possui
  USER ||--o{ TASK : possui
  USER ||--o{ GOAL : possui
  USER ||--o{ ROUTINE : possui

  CATEGORY }o--o{ HABIT : categoriza
  CATEGORY }o--o{ TASK : categoriza
  CATEGORY }o--o{ GOAL : categoriza

  ROUTINE ||--o{ ROUTINE_SECTION : contém
  ROUTINE ||--|| SCHEDULE : "agendada por"

  ROUTINE_SECTION ||--o{ HABIT_GROUP : agrupa
  ROUTINE_SECTION ||--o{ TASK_GROUP : agrupa

  HABIT_GROUP }o--|| HABIT : referencia
  TASK_GROUP }o--|| TASK : referencia

  HABIT_GROUP ||--o{ HABIT_GROUP_CHECK : rastreia
  TASK_GROUP ||--o{ TASK_GROUP_CHECK : rastreia
```

### Destaques das entidades

- **User** — perfil, preferências (tema, idioma, widgets) e progressão de XP embutida (level, xp, constância).
- **Category** — agrupa hábitos, tarefas e metas via ManyToMany. Possui seu próprio XP/level.
- **Habit** — comportamento rastreável com importância, dificuldade, frase motivacional e progressão de XP/level.
- **Task** — similar ao hábito, mas pode ser única (oneTimeTask) com soft-delete via markedToDelete.
- **Goal** — baseada em meta com currentValue / targetValue, status (ativa/completa/falha) e prazo (curto/longo/vida).
- **Routine** — base abstrata com tipo concreto DiaryRoutine. Contém seções com grupos de hábitos/tarefas.
- **Schedule** — dias da semana (segunda a domingo) vinculados a uma rotina.
- **Checks** — registros diários de check/skip para grupos de hábitos e tarefas dentro das rotinas, com rastreamento de XP gerado.

## Fluxo de Autenticação

```mermaid
sequenceDiagram
  participant U as Usuário
  participant FE as Frontend
  participant BE as Backend
  participant GO as Google

  rect rgba(59, 130, 246, 0.25)
  U->>FE: 🔑 Login com Email + Senha
  FE->>BE: POST /auth/login
  BE-->>FE: JWT (header) + Refresh Token (cookie HttpOnly)
  FE->>FE: Armazena JWT em memória
  end

  rect rgba(234, 88, 12, 0.25)
  U->>FE: 🔐 Login com Google OAuth
  FE->>GO: Redirecionamento de autorização
  GO-->>FE: Código de autorização
  FE->>BE: GET /auth/google?code=...
  BE->>GO: Troca código por access token
  GO-->>BE: Perfil do usuário
  BE-->>FE: JWT + Refresh Token
  end

  rect rgba(16, 185, 129, 0.25)
  FE->>BE: 🔄 Refresh de Token — requisição com JWT expirado
  BE-->>FE: 401 Unauthorized
  FE->>BE: POST /auth/refresh (cookie)
  BE-->>FE: Novo JWT + novo Refresh Token
  FE->>BE: Reexecuta requisição original
  end
```

### Detalhes dos tokens

- **Access token (JWT)** — 15 minutos, HMAC256, enviado no header Authorization: Bearer.
- **Refresh token** — 15 dias, token opaco com hash, cookie HttpOnly. Token antigo revogado no refresh.
- **Reset de senha** — token seguro via email, TTL de 30 min, cooldown de 5 min entre requisições. Todos os refresh tokens revogados após reset.

## Camada de API

14 controllers REST organizados por domínio:

| Grupo | Controllers | Caminhos base |
|-------|-----------|--------------|
| **Auth** | AuthenticationController | /auth/* |
| **Entidades principais** | CategoryController, HabitController, TaskController, GoalController | /category, /habit, /task, /goal |
| **Rotinas** | RoutineController, ScheduleController | /routine, /schedule |
| **Usuário** | UserController | /user |
| **Docs** | ArchitectureDocsController, DesignDocsController, ApiDocsController, ProjectDocsController, SearchDocsController, DocsImportController | /docs/* |

### Padrão de requisição/resposta

```mermaid
flowchart LR
  REQ["📥 Requisição"] --> FILT["🛡️ Filtro de Segurança<br/>Validação JWT"]
  FILT --> CTRL["🎯 Controller<br/>Validação do DTO"]
  CTRL --> SVC["⚙️ Service<br/>Lógica de negócio"]
  SVC --> REPO["💾 Repository<br/>Consultas JPA"]
  REPO --> DB[("🐘 PostgreSQL")]
  DB --> REPO
  REPO --> SVC
  SVC --> MAP["🔄 Mapper<br/>Entidade → DTO"]
  MAP --> CTRL
  CTRL --> RES["📤 Resposta"]
```

- DTOs de requisição validados com Jakarta Bean Validation (@NotBlank, @Size, @Email).
- Respostas mapeadas através de classes Mapper dedicadas (entidade → DTO de resposta).
- Handler global de exceções traduz erros em ApiErrorResponse padronizado com chaves de erro para i18n no frontend.

## Gerenciamento de Estado (Frontend)

```mermaid
flowchart TD
  AX["Axios + Interceptor"]
  ST["Redux Store<br/>16 slices"]
  PS["redux-persist<br/>localStorage"]
  UI["Componentes React"]

  UI -->|"dispatch de actions"| ST
  ST -->|"selectors"| UI
  ST <-->|"hidratar / persistir"| PS
  UI -->|"chamadas à API"| AX
  AX -->|"dispatch no sucesso"| ST
  AX -->|"auto 401 → refresh"| AX
```

### Slices principais

| Slice | Finalidade |
|-------|-----------|
| perfil | Perfil do usuário, XP, level, tema, idioma, constância |
| habits, tasks, goals, routines, categories | Listas de entidades |
| editHabit, editTask, editGoal, editRoutine, editCategory | Estado do modo edição |
| todayRoutine | Rotina agendada do dia para o dashboard |
| viewFilters | Preferências de ordenação/filtro por página |
| register, errorHandler | Estado de autenticação e erros |

## Sistema de Gamificação

```mermaid
flowchart LR
  ACT["✅ Check no hábito/tarefa<br/>na rotina"] --> XP["🎮 Calculador de XP"]
  XP --> UXP["👤 XP do Usuário<br/>+ level up"]
  XP --> HXP["💪 XP do Hábito<br/>+ level up"]
  XP --> CXP["📂 XP da Categoria<br/>+ level up"]
  XP --> RXP["📋 XP da Rotina<br/>+ level up"]
  ACT --> STR["🔥 Constância<br/>rastreamento de streak"]
```

- **XpProgress** é um componente embutido compartilhado por User, Category, Habit e Routine.
- XP é gerado quando um hábito ou tarefa é marcado como concluído dentro de uma rotina.
- A progressão de level segue uma tabela semeada de XP por level (XpByLevelSeeder).
- Constância (streak) rastreia dias consecutivos completados na entidade User.
- Goals concedem um xpReward fixo ao serem completadas.

## Infraestrutura

```mermaid
flowchart TB
  subgraph Docker Compose
    FE["⚛️ Frontend<br/>:3000"]
    BE["🍃 Backend<br/>:8099"]
    DB[("🐘 PostgreSQL<br/>:5490")]
    NG["🌐 nginx<br/>(somente prod)"]
  end

  NG --> FE
  NG --> BE
  FE --> BE
  BE --> DB
```

- **Modo dev** — up-dev.sh: hot-reload para frontend e backend, acesso direto às portas.
- **Modo prod** — up-prod.sh: proxy reverso nginx roteando /api → backend, / → frontend.
- **Reset** — reset-db.sh: limpa o volume de dados do PostgreSQL.
- Ambiente configurado via arquivo .env com secrets para JWT, Google OAuth, SMTP, CORS e import de docs.
