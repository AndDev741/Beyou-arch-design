---
title: "UI Components and Page Structure"
summary: "How the web app is organized inside the monorepo: the shared shell, the entity component quartet, the widget system, the two-part tutorial, and the code-splitting that keeps first load small."
---

This document maps the web frontend: how pages and components are organized, the patterns every feature follows, how the dashboard widgets and the tutorial work, and how the bundle is split. The web app lives in `apps/web` of the frontend monorepo and consumes the shared packages (state, theme, i18n, validation, icons, api) as source through Vite aliases.

## Routing and the shell

```mermaid
flowchart TD
  APP["⚛️ App.tsx<br/>ThemeProvider + Router + ErrorBoundary"]
  APP --> PUB["Public routes<br/>/ · /register · /forgot-password<br/>/reset-password · /auth/verify"]
  APP --> PROT["ProtectedRoute (layout route)"]
  PROT --> SHELL["Shell, mounted once:<br/>Sidebar · BottomNav · AgentWidget"]
  SHELL --> PAGES["/dashboard · /categories · /habits · /goals<br/>/tasks · /routines · /configuration · /feedback"]
  PROT --> ADMIN["AdminRoute → /admin/feedback"]
```

Every route component is lazy, including AdminRoute itself, so ordinary users never download the admin gate or its API calls. One Suspense boundary with a full-screen spinner wraps the whole route table. On boot, `useSilentRefresh` holds the app in a "checking" state until the refresh cookie has been traded for a token, which is what prevents a flash of 401s or a bounce to login on reload.

The shell mounts once inside the protected layout route: the collapsible desktop Sidebar (order: Today, Categories, Habits, Tasks, Routines, Goals, with Feedback and Config in the footer), the phone BottomNav (Today, Routines, the Assistant in the center slot as the only entry to the agent, Habits, and a More sheet), and the floating AgentWidget. Pages render no header of their own; a shared PageHeader component is the in-page title block. Nav items carry `data-tutorial-id` anchors for the tutorial spotlight.

Auth pages deliberately avoid the icon registry, keeping the icon and emoji chunks out of the unauthenticated first load.

## Component organization

```
src/
  pages/        one folder per route, tests colocated
  components/   per-domain folders (agent, categories, dashboard, goals,
                habits, routines, tasks, tutorial, widgets, ...)
  ui/           design-system primitives with no domain knowledge
  hooks/  context/  lib/  services/  redux/  utils/
```

The `ui/` layer holds the primitives: Card, Chip, Ring, XpBar, XpSparkline, StatTile, SegmentedControl, IconButton, IconTile, CheckStrip, GhostAdd, BeyouIcon, PageHeader. Domain components compose these. The shared Modal is a portal with a real focus trap: Tab cycling, focus restore on close, Escape, and aria-labelledby, and every dialog in the app renders through it.

### The entity quartet

Every domain entity (category, habit, task, goal, routine) follows the same four-part pattern:

| Part | Role |
|------|------|
| createX / editX | Thin modal wrappers choosing the form's mode |
| XForm | The shared react-hook-form component, one per entity |
| xBox | The expandable card showing one item, with edit and delete actions |
| renderXs | The responsive grid that maps the list |

Forms resolve through zod schemas that live in the shared validation package, written as factories taking the translation function, so every validation message is bilingual by construction. Cross-field routine rules (overlapping section times, overnight ranges) live beside them as plain functions the form and the routine builder both call. A list routine is validated by its own schema rather than a branch inside the daily one: the two forms hold genuinely different state, an array of timed sections against a flat array of picked habits and tasks, and one schema accepting both would check neither properly.

## Dashboard and widgets

The dashboard composes a profile card, shortcut links, today's routine with its check-in flow, a goals rail, and the configurable widget area.

Widget identity is shared state: the list of ids lives in the state package (worstArea, constance, constanceHeatmap, betterArea, dailyProgress, fastTips, levelProgress, categoryBalance), and both apps read it. Four render full-width. A fabric component maps id to component, so adding a widget is one entry in the map plus one entry in the shared list.

Selection lives in Configuration: a drag-to-reorder list that autosaves on every change, pushing the new order to the backend and Redux together, and rolling nothing into Redux when the server rejects. On phones the dashboard renders widgets in a snap-scroll carousel, one per screen, so adding widgets never pushes today's routine below the fold.

## The tutorial, in two systems

Onboarding is a phase machine persisted in localStorage, with values whitelist-validated on read:

```
intro → ai-onboarding → dashboard → categories → habits-dashboard → habits
→ routines-dashboard → routines → routines-summary → config-dashboard → config → done
```

Two distinct systems ride that machine:

- **The intro modal**: four concept cards (categories, habits, tasks, routines), and then a fork: take the AI onboarding wizard or the manual tour.
- **The spotlight tour**: each step names a CSS selector, a position, and an action (click or observe). The finder picks the first visible match, which is what lets the desktop sidebar and the phone More-sheet share the same step definitions, and the tooltip clamps itself to the viewport. Each page has a hook owning its steps and phase transitions.

The AI onboarding wizard walks five steps (categories, habits and tasks, routine, goals, summary), fetching typed suggestions from the backend and creating real entities step by step through the ordinary REST endpoints. Progress persists to localStorage as step-plus-created-references only; the suggestions themselves are deliberately not persisted, so a reload re-fetches instead of double-creating. The creates check the account first and skip a name that is already there, which is what keeps the error banner's Try again from adding a second copy of everything a failed pass had got through before it failed. A failure now says which kind it is: a suggestion call that failed keeps the AI-unavailable screen, while a rejected entity write names what fell over, shows the server's reason and lists what the wizard already saved.

## Gamification feedback

Three pieces turn a routine check into visible progress:

- **XpFloat**: a "+N XP" chip that rises over the checked item for a second, driven by the real xpGenerated value in the check response. Under reduced motion it fades in place.
- **CelebrationOverlay**: a global overlay draining a FIFO queue of celebrations (level-ups and streak milestones), auto-dismissing after four seconds, dismissable by click or Escape.
- **The RefreshUI flow**: check responses carry a RefreshUI payload, and a shared apply function updates the profile, categories, habit, and routine slices in one pass, deciding on the way whether a celebration belongs in the queue. The details live in the [Redux and data topic](/architecture/redux-data).

Data freshness is handled by a shared auto-refresh policy with three prompts: returning to the tab, the local day rolling over, and a five-minute interval while visible. Single-flight, silent on failure, and paused entirely while the tab is hidden or a check animation is mid-flight.

## Code splitting

Route-level laziness plus five manual chunks, in an order that matters:

| Chunk | Contents | Why |
|-------|----------|-----|
| icons-base | react-icons | The heaviest optional weight |
| telemetry | Sentry SDK | Must be matched before the forms rule: the SDK ships a file with "zod" in its path. Tree-shaken away entirely in builds without a DSN |
| motion | framer-motion | Only needed after login |
| forms | react-hook-form, resolvers, zod | Form-heavy pages only |
| vendor | react, router, redux family | The stable base |

The dev server pre-bundles the lazy-route dependencies, because discovering them mid-session used to trigger a re-optimization and a full reload halfway through using the app.

## Conventions worth keeping

- Every interactive element is a real button or link; icon tiles included. The icon picker's tiles were the last `<span onClick>` holdouts and are buttons with aria-labels now.
- Toasts are capped at three, positioned per device class, with custom close and icon components.
- One-shot invitations (like the empty-widgets nudge) dismiss through a shared localStorage-flag hook rather than ad-hoc state.
- Drag-and-drop uses react-beautiful-dnd behind a StrictMode shim.
