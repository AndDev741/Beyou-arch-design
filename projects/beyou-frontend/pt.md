# Beyou Frontend

Esta é a aplicação frontend para a plataforma de produtividade Beyou.

## Visão geral

Construída com tecnologias web modernas:

- **React** para componentes de UI
- **TypeScript** para segurança de tipos
- **Vite** para builds rápidos e desenvolvimento
- **Tailwind CSS** para estilização
- **shadcn/ui** para biblioteca de componentes

## Funcionalidades

- Autenticação de utilizadores e gestão de sessões
- Acompanhamento de objetivos com progresso visual
- Formação de hábitos com sequências
- Gestão de tarefas com arrastar e largar
- Agendamento de rotinas e integração de calendário
- Notificações em tempo real

## Arquitetura

O frontend segue uma estrutura de pastas orientada a funcionalidades, separando responsabilidades em:

- `src/components` – componentes UI reutilizáveis
- `src/pages` – componentes de página de alto nível
- `src/lib` – utilitários, clientes API e lógica partilhada
- `src/context` – provedores de contexto React (tema, autenticação, etc.)
- `src/hooks` – hooks React personalizados

## Desenvolvimento

Para iniciar o servidor de desenvolvimento:

```bash
npm install
npm run dev
```

A aplicação ficará disponível em `http://localhost:5173`.

## Testes

Utilizamos **Vitest** para testes unitários e **Playwright** para testes end‑to‑end.

Executar testes com:

```bash
npm run test
```

## Implementação

O frontend é automaticamente implementado na **Vercel** em cada push para o branch `main`.