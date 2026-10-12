---
title: "Beyou Dev Env"
summary: "A camada Compose que monta os outros repositórios em algo executável localmente: dev com hot reload, isolamento de e2e e o overlay de monitoramento. A produção foi para o Kubernetes em outubro de 2026."
---
# Beyou Dev Env

Este repositório não contém código de aplicação. É a camada de orquestração local: um arquivo Compose base com Postgres e a rede compartilhada, mais três overlays que se empilham por cima.

## Os overlays

- **dev** constrói o backend e o frontend dos checkouts irmãos com código montado e hot reload; artefatos de build vivem em volumes nomeados, então as ferramentas do host e do container nunca brigam.
- **e2e** sobe uma stack isolada com nome de projeto e banco próprios, o que a checagem de segurança do backend impõe.
- **monitoring** carrega Prometheus, Grafana, Loki com Alloy e GlitchTip, respondendo localmente as mesmas três perguntas que a stack de produção responde no cluster.

## Os scripts

`up-dev.sh` (aceitando `--monitoring`), `down.sh`, o `reset-db.sh` que falha alto, e o `bootstrap-glitchtip.sh`, que cria e reconcilia por código a organização do rastreador de erros, os projetos, os monitores de uptime, os heartbeats e as regras de alerta.

## A produção agora mora em outro lugar

Até outubro de 2026 este repositório também rodava a produção: um overlay `prod` puxando as imagens publicadas atrás do Watchtower, um `up-prod.sh`, e uma dupla de backup e teste de restauração em timers do systemd do host. Tudo isso foi para um cluster k3s de um nó gerido pelo Flux a partir do repositório público [be-infra](https://github.com/AndDev741/be-infra), onde deploys são commits, os backups e o teste semanal de restauração são CronJobs, e o monitoramento é uma stack de plataforma compartilhada. O que fica aqui é tudo que você precisa para rodar o Beyou na sua própria máquina.

## Mergulhos profundos

Os tópicos de arquitetura de infraestrutura e monitoramento documentam os dois lados, esta stack local e o cluster de produção, do laptop onde tudo roda ao alerta que dispara quando o scheduler de snapshots fica quieto.
