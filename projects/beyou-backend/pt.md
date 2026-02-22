# Beyou Backend
Backend Spring Boot para a aplicação Beyou, fornecendo APIs REST para metas, hábitos, tarefas e gestão de utilizadores.

## Visão Geral
Este backend foi construído com Java 17, Spring Boot 3, PostgreSQL e Docker. Segue uma arquitetura limpa com camadas separadas para controladores, serviços e repositórios.

## Histórico
- **2024‑Q4**: Configuração inicial do projeto com autenticação básica de utilizador e seguimento de metas.
- **2025‑Q1**: Adicionado seguimento de hábitos, agendamento de rotinas e gestão de categorias.
- **2025‑Q2**: Implementado sistema de importação de documentação para API, arquitetura e design.
- **2025‑Q3**: Adicionado cálculo de Xp, tokens de refresh e notificações por email.

## Estado Atual
O backend está em desenvolvimento ativo com atualizações regulares. Serve como núcleo do ecossistema de produtividade Beyou.

## Destaques da Arquitetura
- Autenticação baseada em JWT com tokens de refresh
- Autorização baseada em roles (USER, ADMIN)
- Migrações de base de dados com Flyway
- Testes de integração com Testcontainers
- Tratamento centralizado de erros e logging

## Implantação
- Container Docker implantado num VPS privado
- CI/CD via GitHub Actions
- Configuração específica por ambiente