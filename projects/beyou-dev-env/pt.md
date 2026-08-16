---
title: "Beyou Dev Env"
summary: "A camada Compose que monta os outros repositórios em algo executável: dev com hot reload, produção com imagens publicadas, isolamento de e2e e o overlay de monitoramento."
---
# Beyou Dev Env

Este repositório não contém código de aplicação. É a camada de orquestração: um arquivo Compose base com Postgres e a rede compartilhada, mais quatro overlays que se empilham por cima.

## Os overlays

- **dev** constrói o backend e o frontend dos checkouts irmãos com código montado e hot reload; artefatos de build vivem em volumes nomeados, então as ferramentas do host e do container nunca brigam.
- **prod** puxa as imagens publicadas no GHCR, serve o build web pelo nginx e deixa o Watchtower reimplantar a cada imagem nova. Todas as portas escutam em loopback; o Cloudflare Tunnel na frente faz a exposição.
- **e2e** sobe uma stack isolada com nome de projeto e banco próprios, o que a checagem de segurança do backend impõe.
- **monitoring** carrega Prometheus, Grafana, Loki com Alloy e GlitchTip: o mesmo arquivo em desenvolvimento e produção, então o que você depura localmente é o que roda implantado.

## Os scripts

`up-dev.sh` e `up-prod.sh` (ambos aceitando `--monitoring`), `down.sh`, o `reset-db.sh` que falha alto, e o `bootstrap-glitchtip.sh`, que cria e reconcilia por código a organização do rastreador de erros, os projetos, doze monitores de uptime, o heartbeat de snapshots e as regras de alerta.

## Mergulhos profundos

Os tópicos de arquitetura de infraestrutura e monitoramento documentam este repositório de ponta a ponta, do laptop onde tudo roda ao alerta que dispara quando o scheduler de snapshots fica quieto.
