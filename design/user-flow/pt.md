---
title: Jornada do Fluxo do Usuario
summary: Do onboarding aos check-ins diarios.
---

O design abaixo mostra a experiencia completa para um novo usuario, combinando narrativa e diagramas.

## Sequencia de onboarding

```mermaid
sequenceDiagram
  participant U as Usuario
  participant App as Beyou App
  participant API as Backend
  U->>App: Abre o Beyou
  App->>API: Verifica sessao
  API-->>App: Sessao invalida
  App-->>U: Mostra onboarding
  U->>App: Cria primeiro habito
  App->>API: Salva habito
  API-->>App: Habito salvo
```

## Fluxo diario

```mermaid
flowchart LR
  Start([Abrir app]) --> Dash[Dashboard]
  Dash --> Check{Check-in?}
  Check -->|Sim| Habit[Registrar habito]
  Check -->|Nao| Explore[Explorar dicas]
  Habit --> Reward[XP + streak]
  Reward --> Dash
  Explore --> Dash
```

## Observacoes

- Mantenha o CTA de check-in visivel apos o onboarding.
- O feedback de recompensa deve ser imediato para reforcar o loop de habito.
