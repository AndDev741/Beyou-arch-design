---
title: "Language and Theme"
summary: "How the bilingual system (EN/PT) and the 9-theme system work — from user selection to CSS variables to backend persistence."
---

This document covers the two personalization systems in the Beyou frontend: language (i18n with English and Portuguese) and visual theme (9 color themes with CSS variable injection).

## Language System (i18n)

### Architecture

```mermaid
flowchart TD
  DET["🌐 Browser Language Detection<br/>i18next-browser-languagedetector"]
  DET --> I18N["i18next<br/>fallback: English"]
  I18N --> EN["🇺🇸 en/translation.json<br/>500+ keys"]
  I18N --> PT["🇧🇷 pt/translation.json<br/>500+ keys"]

  USER["👤 User changes language"] --> HOOK["useChangeLanguage hook"]
  HOOK --> I18N
  HOOK --> RDX["Redux<br/>perfil.languageInUse"]
  HOOK --> API["Backend API<br/>persist preference"]
```

### Configuration

- **Library:** i18next + react-i18next
- **Detection:** i18next-browser-languagedetector (auto-detects from browser settings)
- **Fallback:** English if detection fails or language unsupported
- **Languages:** English (en) and Portuguese (pt, pt-BR)
- **Interpolation:** escapeValue disabled (React handles XSS)

### Translation file structure

Both en/translation.json and pt/translation.json use a flat key structure with 500+ keys:

| Category | Example Keys |
|----------|-------------|
| Auth | Login, Register, ForgotPasswordTitle, PasswordMismatch |
| Validation | YupNameRequired, YupMinimumName, YupMaxName |
| Pages | YourCategories, YourHabits, Your Goals |
| Actions | created successfully, edited successfully, Logout |
| Errors | WrongPassOrEmailError, GoogleLoginError, UnexpectedError |
| Themes | beYou, beYouDark, Sunset, Amethyst, Midnight, Cyberpunk |
| Greetings | GoodMorning, GoodAfternoon, GoodEvening |

Theme names in translation keys must match the theme.mode values so the theme selector displays the correct localized name.

### Language change flow

```mermaid
sequenceDiagram
  participant U as User
  participant BTN as TranslationButton
  participant HOOK as useChangeLanguage
  participant I18N as i18next
  participant RDX as Redux
  participant BE as Backend

  U->>BTN: Click PT button
  BTN->>HOOK: setLng("pt")
  HOOK->>I18N: i18n.changeLanguage("pt")
  Note right of I18N: All useTranslation() hooks<br/>re-render with new language
  HOOK->>BE: editUser({ language: "pt" })
  HOOK->>RDX: dispatch(languageInUserEnter("pt"))
```

Three systems are updated in parallel:

1. **i18next** — immediate UI update, all translated strings re-render
2. **Backend** — persists preference so it survives across devices
3. **Redux** — persists locally via redux-persist so it survives page refreshes

### Language restoration on login

When a user logs in, the backend returns their saved languageInUse. The frontend dispatches it to Redux, and the dashboard's useChangeLanguage hook applies it to i18next. This ensures the app immediately switches to the user's preferred language.

## Theme System

### Architecture

```mermaid
flowchart TD
  USER["👤 User selects theme"]
  USER --> SEL["🎨 ThemeSelector"]
  SEL --> API["Backend API<br/>editUser({ theme: mode })"]
  SEL --> RDX["Redux<br/>dispatch(themeInUseEnter(theme))"]
  SEL --> CTX["ThemeContext<br/>setTheme(theme)"]

  CTX --> EFFECT["useEffect"]
  EFFECT --> CSS["Update :root CSS variables"]
  CSS --> BG["--background"]
  CSS --> PRI["--primary"]
  CSS --> SEC["--secondary"]
  CSS --> DESC["--description"]
  CSS --> ICO["--icon"]
  CSS --> PH["--placeholder"]
  CSS --> SUC["--success"]
  CSS --> ERR["--error"]
```

### Available themes

Beyou has 9 themes, each defining 8 color variables:

| Theme | Mode | Background | Primary | Style |
|-------|------|-----------|---------|-------|
| **beYou** | beYou | #FFFFFF | #0082E1 | Light blue on white |
| **beYou Dark** | beYouDark | #18181B | #0082E1 | Blue on dark gray |
| **Sunset** | Sunset | #FFF3E0 | #FB923C | Orange warm light |
| **Amethyst** | Amethyst | #F5F3FF | #8B5CF6 | Purple light |
| **Midnight** | Midnight | #0F172A | #60A5FA | Blue on navy |
| **Cyberpunk** | Cyberpunk | #0D0C1D | #D946EF | Pink on dark |
| **Mocha** | Mocha | #FAF3E0 | #B45309 | Brown warm light |
| **Polar** | Polar | #1E293B | #0EA5E9 | Cyan on slate |
| **Late Latte** | Late Latte | #2C1E1E | #947347 | Gold on dark brown |

### ThemeContext

The ThemeContext is a React context that wraps the entire app via ThemeProvider:

1. Reads the user's saved theme from Redux (perfil.themeInUse)
2. If no saved theme, checks OS preference via matchMedia("(prefers-color-scheme: dark)")
3. Falls back to defaultLight
4. On every theme change, updates CSS custom properties on :root

**Priority:** User preference > OS dark mode > defaultLight

### CSS variable integration

All components use Tailwind CSS classes that reference CSS variables:

| Variable | Used by | Tailwind Class |
|----------|---------|---------------|
| --background | Page backgrounds, cards | bg-background |
| --primary | Buttons, links, accents | bg-primary, text-primary |
| --secondary | Text, headings | text-secondary |
| --description | Muted text | text-description |
| --icon | Icon colors | text-icon |
| --success | Success states | text-success |
| --error | Error states, validation | text-error, border-error |

Tailwind is configured with these CSS variables in tailwind.config.js, so every color-related class automatically adapts to the active theme.

### Theme change flow

```mermaid
sequenceDiagram
  participant U as User
  participant SEL as ThemeSelector
  participant BE as Backend
  participant RDX as Redux
  participant CTX as ThemeContext
  participant DOM as :root CSS

  U->>SEL: Click Midnight theme
  SEL->>BE: editUser({ theme: "Midnight" })
  SEL->>RDX: dispatch(themeInUseEnter(midnightTheme))
  SEL->>CTX: setTheme(midnightTheme)
  CTX->>DOM: --background: #0F172A
  CTX->>DOM: --primary: #60A5FA
  CTX->>DOM: --secondary: #E2E8F0
  Note right of DOM: All components instantly<br/>reflect new colors
```

### Theme selector UI

The ThemeSelector renders a grid of theme previews. Each preview is a split rectangle showing the theme's background (left half) and primary color (right half), with a border in the primary color. Clicking a preview triggers the three-way sync (API + Redux + Context).

### Theme restoration on login

Same as language: the backend returns themeInUse as a mode string. The frontend looks up the matching theme object from the themes array and dispatches it. ThemeContext reacts and updates CSS variables immediately.

## How They Work Together

Both systems follow the same three-way sync pattern:

```mermaid
flowchart LR
  CHANGE["User changes<br/>language or theme"]
  CHANGE --> I["1. UI Framework<br/>i18next or ThemeContext<br/>(immediate visual update)"]
  CHANGE --> R["2. Redux<br/>(local persistence<br/>via redux-persist)"]
  CHANGE --> A["3. Backend API<br/>(server persistence<br/>across devices)"]
```

This ensures:

- **Instant UI response** — no loading state when switching
- **Survives page refresh** — redux-persist restores from localStorage
- **Survives device change** — backend stores the preference
- **Works offline** — redux-persist applies even without API connection
