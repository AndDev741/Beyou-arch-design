---
title: "Redux e Arquitetura de Dados"
summary: "Como o gerenciamento de estado funciona no frontend — configuração do store, todos os 16 slices, persistência, fluxo de dados e padrões de dispatch."
---

Este documento explica a arquitetura Redux completa no frontend Beyou: como o store é configurado, o que cada slice armazena, como os dados fluem entre componentes e a API, e os padrões usados para atualizações de estado.

## Arquitetura do Store

```mermaid
flowchart TD
  subgraph "Redux Store"
    PERFIL["👤 perfil<br/>perfil do usuário + configurações"]
    ENTITIES["📦 Slices de Entidades<br/>habits, tasks, goals,<br/>categories, routines"]
    EDIT["✏️ Slices de Edição<br/>editHabit, editTask,<br/>editGoal, editCategory,<br/>editRoutine"]
    TODAY["📋 todayRoutine"]
    FILTERS["🔽 viewFilters"]
    AUTH["🔐 register"]
    ERR["⚠️ errorHandler"]
  end

  PERSIST["💾 redux-persist<br/>localStorage key: 'root'"]
  STORE --> PERSIST
  PERSIST --> STORE

  subgraph "Redux Store"
    STORE[" "]
  end
```

### Configuração

- **Store:** configureStore do @reduxjs/toolkit
- **Persistência:** redux-persist com localStorage (key: "root")
- **Middleware:** serializableCheck ignora ações REGISTER, REHYDRATE, PERSIST (necessário para redux-persist)
- **Reidratação:** PersistGate envolve o app, loading null até o estado ser restaurado do localStorage

Isso significa que todo o estado Redux sobrevive a refreshes de página. Quando o usuário fecha e reabre o app, perfil, tema, idioma e dados de entidades estão imediatamente disponíveis.

## Todos os 16 Slices

### perfil — Perfil e Configurações do Usuário

O slice mais importante. Armazena tudo sobre o usuário logado.

| Campo | Tipo | Propósito |
|-------|------|-----------|
| username | string | Nome de exibição |
| email | string | Email do usuário |
| phrase / phrase_author | string | Citação motivacional |
| photo | string | URL da foto de perfil |
| isGoogleAccount | boolean | Flag OAuth |
| themeInUse | ThemeType | Objeto do tema atual (9 temas disponíveis) |
| languageInUse | string | Código do idioma atual (en/pt) |
| xp, level, nextLevelXp, actualLevelXp | number | Estado de gamificação |
| constance, maxConstance | number | Rastreamento de streak |
| widgetsIdsInUse | string[] | Widgets ativos no dashboard |
| isTutorialCompleted | boolean | Flag de onboarding |
| checkedItemsInScheduledRoutine | number | Numerador do progresso de hoje |
| totalItemsInScheduledRoutine | number | Denominador do progresso de hoje |

**18 actions** — uma action Enter por campo (ex: nameEnter, themeInUseEnter, languageInUserEnter).

Populado após login com todos os dados do usuário da resposta do backend.

### Slices de Coleção de Entidades

Cinco slices que mantêm as listas de entidades do usuário:

| Slice | Estado | Actions |
|-------|--------|---------|
| **categories** | { categories: category[] } | enterCategories, updateCategorie, refreshCategorie |
| **habits** | { habits: habit[] } | enterHabits |
| **tasks** | { tasks: task[] } | enterTasks |
| **goals** | { goals: goal[] } | enterGoals, updateGoal |
| **routines** | { routines: Routine[] } | enterRoutines |

Categories e goals têm actions extras de update para mudanças parciais de estado (refresh de XP, progresso de meta).

### todayRoutine — Rotina do Dashboard

| Campo | Tipo | Propósito |
|-------|------|-----------|
| routine | Routine ou null | Rotina agendada para hoje |

**Actions:**

- enterTodayRoutine — define a rotina de hoje da API
- refreshItemGroup({groupItemId, check}) — atualiza um único status de check sem refetch

### Slices de Modo Edição

Cinco slices que gerenciam o estado de "editando uma entidade". Todos seguem o mesmo padrão:

```mermaid
flowchart LR
  CLICK["Clique no card"] -->|"dispatch editModeEnter(true)"| EDIT["Edit Slice<br/>editMode: true<br/>id: entity.id<br/>...campos populados"]
  EDIT -->|"formulário renderiza"| FORM["Formulário de Edição"]
  FORM -->|"submit + dispatch editModeEnter(false)"| DONE["Edit Slice<br/>editMode: false"]
```

Quando o usuário clica no botão editar de um card, o componente dispara múltiplas actions para popular cada campo do edit slice. O formulário de edição lê esses valores como padrões. No submit ou cancelamento, editModeEnter(false) reseta o modo.

### viewFilters — Preferências de Ordenação

Armazena a opção de ordenação selecionada para cada página de feature:

| Key | Padrão | Exemplos de Opções |
|-----|--------|-------------------|
| categories | "default" | name-asc, name-desc, level-desc, xp-desc |
| habits | "default" | name-asc, importance-desc, difficulty-desc, xp-desc |
| tasks | "default" | name-asc, name-desc, created-desc |
| goals | "default" | name-asc, progress-desc, xp-desc |
| routines | "default" | name-asc, name-desc |

**Action:** setViewSort({ view, sortBy }) — persistido entre navegações via redux-persist.

### register

| Campo | Tipo | Propósito |
|-------|------|-----------|
| successRegister | boolean | Sinaliza registro bem-sucedido |

Usado para mostrar mensagem de sucesso na página de login após registro.

### errorHandler

| Campo | Tipo | Propósito |
|-------|------|-----------|
| defaultError | string | Mensagem de erro global |

Exibição de erro fallback para erros inesperados.

## Padrões de Fluxo de Dados

### Padrão 1: Carregamento de Página (Fetch + Dispatch)

```mermaid
sequenceDiagram
  participant PG as Componente da Página
  participant API as Serviço API
  participant BE as Backend
  participant RDX as Redux Store

  PG->>PG: useAuthGuard()
  PG->>API: getHabits(t)
  API->>BE: GET /habit
  BE-->>API: habit[]
  API-->>PG: habit[]
  PG->>RDX: dispatch(enterHabits(habits))
  RDX-->>PG: useSelector → hábitos ordenados renderizados
```

### Padrão 2: Criar Entidade

```mermaid
sequenceDiagram
  participant FM as Formulário
  participant API as Serviço API
  participant BE as Backend
  participant RDX as Redux Store
  participant UI as Página

  FM->>FM: Validação Zod
  FM->>API: createHabit(data)
  API->>BE: POST /habit
  BE-->>API: Hábito criado
  FM->>API: getHabits(t)
  API->>BE: GET /habit
  BE-->>API: habit[] atualizado
  FM->>RDX: dispatch(enterHabits(habits))
  FM->>FM: Reset form + toast sucesso
  RDX-->>UI: Grid re-renderiza com novo hábito
```

### Padrão 3: Editar Entidade

```mermaid
sequenceDiagram
  participant BOX as Card da Entidade
  participant RDX as Redux Store
  participant FM as Formulário de Edição
  participant API as Serviço API

  BOX->>RDX: dispatch(editModeEnter(true))
  BOX->>RDX: dispatch(idEnter(entity.id))
  BOX->>RDX: dispatch(nameEnter(entity.name))
  Note right of BOX: ...dispatch de todos os campos
  RDX-->>FM: Form de edição renderiza com valores
  FM->>API: editHabit(id, data)
  FM->>RDX: dispatch(editModeEnter(false))
  FM->>API: getHabits(t) → refetch
  FM->>RDX: dispatch(enterHabits(habits))
```

### Padrão 4: Check na Rotina (Update Otimista)

```mermaid
sequenceDiagram
  participant UI as Seção da Rotina
  participant RDX as Redux Store
  participant API as Serviço API

  UI->>API: checkItem(routineId, groupId)
  UI->>RDX: dispatch(refreshItemGroup({groupId, check: true}))
  Note right of RDX: Update imediato na UI
  API-->>UI: Dados de XP atualizados
  UI->>RDX: dispatch(xpEnter(newXp))
  UI->>RDX: dispatch(refreshCategorie({id, xp, level}))
```

## Estratégia de Persistência

```mermaid
flowchart LR
  STORE["🗄️ Redux Store"] -->|"auto-persist"| LS["💾 localStorage<br/>key: 'root'"]
  LS -->|"PersistGate reidratar"| STORE
  CLOSE["🚪 Usuário fecha aba"] -.-> LS
  OPEN["🔄 Usuário reabre app"] -.-> LS
```

**O que é persistido:** Tudo — todos os 16 slices, incluindo listas de entidades, perfil de usuário, estado de modo edição, preferências de ordenação e progresso do tutorial.

**O que isso significa:**

- Refresh de página não perde estado
- Usuário vê seus dados imediatamente ao reabrir (antes de qualquer chamada API)
- Tema e idioma aplicam instantaneamente (sem flash do tema padrão)
- Preferências de ordenação sobrevivem entre sessões

**Trade-off:** Dados obsoletos são possíveis se o usuário tem múltiplos dispositivos. Cada página faz refetch da API no mount, então os dados persistidos são rapidamente substituídos por dados frescos.
