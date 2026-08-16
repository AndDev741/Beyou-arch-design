---
title: "Beyou E2E Tests"
summary: "A suíte Playwright que dirige a stack real de ponta a ponta: registrar, logar, montar rotinas, marcar hábitos e ver o XP se mover."
---
# Beyou E2E Tests

Specs Playwright que exercitam o produto inteiro do jeito que um usuário faz: frontend, backend e Postgres juntos, sem mocks. A suíte tranca os fluxos que nunca podem quebrar em silêncio: registro e login (incluindo o comportamento anti-enumeração nas falhas), persistência de sessão entre reloads, desmontagem no logout, CRUD de hábitos, check-ins de rotina com seus efeitos de XP e streak, a assimetria de conclusão de metas, o feedback de gamificação e o tutorial de onboarding completo.

## Como roda

Contra uma stack isolada: o backend sobe com o perfil e2e no banco dedicado `beyou_e2e`, e uma checagem de segurança se recusa a iniciar contra qualquer coisa que não pareça um banco de teste. Fixtures fornecem páginas autenticadas com o tutorial pulado ou intacto, e helpers de seed constroem categorias e estados completos de onboarding pela API real.

A suíte também é portão de merge: o CI do repositório do frontend sobe esta stack e roda estes specs em cada pull request, com um resolvedor de branch irmã para uma mudança de frontend que precisa de specs novos apontar para uma branch correspondente aqui antes de qualquer merge.

## Mergulhos profundos

O tópico de segurança documenta os comportamentos de auth que estes specs fixam, e o de infraestrutura cobre o overlay e2e isolado em que rodam.
