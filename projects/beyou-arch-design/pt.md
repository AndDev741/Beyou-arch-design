---
title: "Beyou Arch & Design"
summary: "Repositório central para documentação de arquitetura e design do projeto Beyou, incluindo diagramas, descrições de tópicos e especificações OpenAPI."
---
# Beyou Arch & Design

Este repositório contém a documentação de arquitetura e design do projeto Beyou, servindo como fonte única de verdade para decisões de design do sistema, especificações de API e fluxos de utilizador.

## Visão geral

O repositório está organizado em três secções principais:

- **arquitetura** – diagramas de alto nível do sistema, descrições de componentes e topologias de implementação
- **design** – maquetes de interface de utilizador, fluxos de interação e diretrizes de usabilidade
- **api** – especificações OpenAPI para cada grupo de endpoints, juntamente com documentação ao nível do controlador

## Funcionalidades

- Documentação baseada em Markdown com metadados YAML
- Suporte multilingue (inglês e português)
- Ficheiros fonte de diagramas (Mermaid) que podem ser renderizados como SVG
- Importação automatizada para o backend Beyou para visualização em tempo real na UI
- Controlo de versão juntamente com alterações de código

## Utilização

Programadores e designers contribuem para este repositório adicionando ou atualizando ficheiros markdown nas pastas apropriadas. A documentação é importada automaticamente para o backend Beyou através da API de Importação de Documentação.

## Integração

O conteúdo é consumido pela UI de Documentação Beyou, que o renderiza numa interface amigável. As alterações enviadas para o branch `main` disparam um webhook de importação que atualiza a documentação em direto.