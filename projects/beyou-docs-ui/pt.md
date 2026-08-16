---
title: "Beyou Docs UI"
summary: "O site que você está lendo: um visualizador React para os docs importados, com navegador de endpoints OpenAPI, Mermaid ciente de tema, busca e páginas pré-renderizadas para SEO."
---
# Beyou Docs UI

O próprio site de documentação. Uma SPA React e Vite que lê a API pública de docs do backend e renderiza quatro coleções: tópicos de arquitetura com diagramas Mermaid cientes de tema e um trilho de sumário, o blog de engenharia, a referência de API com um navegador de endpoints OpenAPI por controller e o catálogo de projetos, mais a busca entre coleções com destaques.

## A metade de SEO

Conteúdo renderizado no cliente é invisível para a maioria dos crawlers, então um script de pré-renderização busca cada tópico nos dois idiomas na mesma API pública e escreve um arquivo HTML real por rota, com títulos, canonicals, pares de hreflang, dados estruturados e o próprio texto do artigo. A SPA sobe por cima e assume. Mudanças de conteúdo reconstroem a imagem inteira: o workflow do repositório de conteúdo dispara o CI deste repositório após cada importação, e o Watchtower implanta o resultado.

## Detalhes que valem saber

Páginas de detalhe são endereçadas por caminho sob um prefixo de idioma, e as listagens de arquitetura e API abrem sua primeira entrada automaticamente. O sistema de tema de duas bases e cinco acentos casa com o do app, com um script de pré-pintura evitando o flash do tema errado.
