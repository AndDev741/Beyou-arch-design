---
title: "Beyou Frontend"
summary: "O monorepo com os dois clientes: o web app React, o app mobile nativo, oito pacotes de código compartilhado e o site de marketing."
---
# Beyou Frontend

Uma base TypeScript, dois clientes. npm workspaces com Turborepo guardam o web app React 18 (`apps/web`), o app mobile React Native 0.85 + Expo (`apps/mobile`) e oito pacotes compartilhados consumidos como código-fonte cru, então uma única edição recarrega os dois apps.

## O que é compartilhado, e o que não é

O estado (17 slices Redux e a lógica de gamificação), a camada de API atrás de uma interface HttpClient estreita, os tokens de tema, as traduções, os schemas de validação e o registro de ícones são uma implementação só. Cada plataforma mantém apenas o que precisa diferir: persistência, navegação, armazenamento de tokens e toda a camada de renderização. O site de marketing vive na raiz do repositório, deliberadamente fora do workspace, e vai sozinho para o Cloudflare Pages.

## Como é entregue

Um único grafo de CI faz typecheck, build e testes de cada workspace, e então roda a suíte e2e Playwright contra a stack completa. Um push na main publica a imagem web no GHCR (o Watchtower a implanta) e, quando o app mobile ou qualquer pacote compartilhado mudou, constrói um APK Android assinado publicado em um release rolante do GitHub.

## Mergulhos profundos

Os tópicos de arquitetura do monorepo, componentes de UI, Redux e dados, idioma e tema, e segurança na UI documentam este repositório.
