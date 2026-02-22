---
title: "Beyou Dev Environment"
summary: "Configuração do ambiente de desenvolvimento para o projeto Beyou, incluindo configurações Docker Compose, seeds de base de dados e ferramentas de desenvolvimento local."
---
# Beyou Dev Environment

Este repositório fornece um ambiente de desenvolvimento local completo para o projeto Beyou, permitindo que os programadores executem toda a stack (backend, frontend, base de dados) com um único comando.

## Visão geral

O ambiente de desenvolvimento é construído em torno do Docker Compose e inclui:

- **PostgreSQL** base de dados com esquemas pré‑configurados
- **Backend** aplicação Spring Boot com suporte a hot‑reload
- **Frontend** aplicação React servida pelo Vite
- **UI de Docs** a correr em conjunto com o backend
- **Serviços opcionais** como MailHog para testes de email

## Funcionalidades

- Arranque com uma linha: `docker‑compose up`
- Seeds de base de dados com dados de teste realistas
- Gestão de variáveis de ambiente através de ficheiros `.env`
- Verificações de saúde e logging integrados
- Suporte para múltiplos perfis (desenvolvimento, teste, CI)

## Utilização

Clone o repositório e execute:

```bash
docker‑compose up -d
```

Aceda aos serviços em:

- Backend API: `http://localhost:8080`
- Frontend: `http://localhost:5173`
- UI de Docs: `http://localhost:3000`
- MailHog: `http://localhost:8025`

## Personalização

O ambiente pode ser adaptado editando o ficheiro `docker‑compose.yml` e ajustando as variáveis de ambiente no ficheiro `.env`. Serviços adicionais (por exemplo, Redis, Elasticsearch) podem ser adicionados conforme necessário.