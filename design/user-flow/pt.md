---
title: Jornada do Fluxo do Usuário
summary: Do onboarding aos check-ins diários.
---

O design abaixo mostra a experiência completa para um novo usuário, combinando narrativa e diagramas.

## Sequência de onboarding

```mermaid
sequenceDiagram
  participant U as Usuário
  participant App as Beyou App
  participant API as Backend
  U->>App: Abre o Beyou
  App->>API: Verifica sessão
  API-->>App: Sessão inválida
  App-->>U: Mostra onboarding
  U->>App: Cria primeiro hábito
  App->>API: Salva hábito
  API-->>App: Hábito salvo
```

## Fluxo diário

```mermaid
flowchart LR
  Start([Abrir app]) --> Dash[Dashboard]
  Dash --> Check{Check-in?}
  Check -->|Sim| Habit[Registrar hábito]
  Check -->|Não| Explore[Explorar dicas]
  Habit --> Reward[XP + streak]
  Reward --> Dash
  Explore --> Dash
```

## Observações

- Mantenha o CTA de check-in visível após o onboarding.
- O feedback de recompensa deve ser imediato para reforçar o loop de hábito.
