---
title: "Beyou Arch Design"
summary: "A fonte de verdade de tudo neste site: tópicos de arquitetura, posts do blog, as specs OpenAPI dos controllers e este catálogo de projetos, publicados por um workflow a cada merge."
---
# Beyou Arch Design

O repositório de conteúdo. Tudo que este site de documentação renderiza vive aqui como arquivo: dezesseis tópicos de arquitetura (inglês, português e um diagrama Mermaid cada), o blog de engenharia, a referência de API como uma spec OpenAPI por controller do backend e este catálogo de projetos.

## Como a publicação funciona

Um merge na main é um deploy. O workflow de publicação faz login no backend, aciona os quatro endpoints de importação atrás do portão duplo de JWT mais segredo, e então dispara o CI do site de docs, que pré-renderiza cada rota contra o conteúdo fresco e entrega uma imagem nova. Tópicos removidos do repositório são arquivados no banco em vez de apagados, então despublicar é reversível.

## Convenções

Um diretório por tópico, um descritor YAML, um arquivo markdown por idioma e um diagrama onde a área exige. As specs de API são geradas da saída springdoc viva do backend, divididas por controller com cada spec carregando só os schemas que referencia.
