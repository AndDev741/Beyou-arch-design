---
title: "Beyou Docs UI"
summary: "Interface de utilizador de documentação para o projeto Beyou, fornecendo uma interface amigável para navegar na documentação de API, arquitetura e design."
---
# Beyou Docs UI

Esta é a interface de utilizador de documentação para o projeto Beyou, fornecendo uma interface amigável para navegar na documentação de API, arquitetura e design.

## Visão geral

Construída com React, TypeScript e Tailwind CSS, a UI de docs renderiza conteúdo markdown e especificações OpenAPI obtidas do backend. Suporta documentação multilingue (inglês e português) e oferece uma interface limpa e pesquisável.

## Funcionalidades

- Renderização em tempo real da documentação da API usando OpenAPI 3.0
- Diagramas interativos de arquitetura e design
- Suporte multilingue (en/pt)
- Pesquisa de texto completo em toda a documentação
- Design responsivo para dispositivos móveis e desktop
- Integração com o sistema de importação de documentação do backend Beyou

## Arquitetura

A UI está estruturada como uma Single Page Application (SPA) com:

- `src/pages` – páginas de alto nível (Projetos, API, Arquitetura, Design)
- `src/components` – componentes reutilizáveis (cartões, modais, barras de pesquisa)
- `src/lib` – utilitários para análise de OpenAPI e markdown
- `src/hooks` – hooks personalizados para obtenção de dados e gestão de estado

## Desenvolvimento

Para iniciar o servidor de desenvolvimento:

```bash
npm install
npm run dev
```

A aplicação ficará disponível em `http://localhost:3000`.

## Implementação

A UI de docs é implementada na **Vercel** e atualizada automaticamente quando são enviadas alterações para o branch `main`.