---
title: "Beyou Backend"
summary: "A API Spring Boot por trás de tudo: services de domínio com checagem de posse por usuário, o motor de XP, o agente de IA, o caderno de estudos e o pipeline de documentação."
---
# Beyou Backend

A única API servindo o web app, o app mobile nativo e este site de documentação. Spring Boot 4.1 sobre Java 25 com virtual threads, PostgreSQL 15 atrás de um schema controlado pelo Flyway e 34 controllers REST sob `/api/v1`.

## O que vive aqui

- **O domínio**: categorias, hábitos, tarefas, metas e rotinas, com o motor de gamificação por cima: a fórmula de XP do check-in, a curva quadrática de levels, streaks derivados, snapshots diários de rotina com decaimento de XP e o razão diário assinado de XP.
- **O caderno de estudos**: tópicos organizados em quadros de roadmap feitos de páginas, com flashcards agendados pelo SM-2. As fontes (PDF, link, texto colado) são lidas em segundo plano em trechos, e uma IA de estudo ancorada responde a partir deles com citações.
- **Segurança**: auth JWT sem estado com refresh tokens rotativos, verificação de e-mail, exclusão de conta com segundo fator, quinze faixas de rate limit mais lockout de login, um buscador de links que recusa SSRF e validadores de boot que recusam uma produção mal configurada.
- **O agente de IA**: um chat com 54 ferramentas sobre os mesmos services de domínio, transmitido por SSE, rodando em uma cadeia de fallback de cinco LLMs gratuitos, mais o endpoint sem estado de sugestões de onboarding.
- **O sistema de docs**: importa esta documentação do GitHub para o Postgres e serve a API pública de leitura e busca em que o site de docs roda.

## Como é entregue

Fazer merge na main é o deploy: o CI roda os testes, constrói a imagem, publica no GHCR e o Watchtower no host de produção puxa e reinicia. Caches Caffeine ficam na frente das leituras quentes, e métricas Micrometer alimentam os dashboards do Grafana, com erros reportando ao GlitchTip auto-hospedado.

## Mergulhos profundos

Os tópicos de arquitetura cobrem este repositório em detalhe: modelo de domínio, segurança, gamificação, o agente de IA, o caderno de estudos, cache, logging AOP, o serviço de e-mail e o sistema de docs têm páginas próprias.
