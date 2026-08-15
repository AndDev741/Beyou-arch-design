---
title: "Infraestrutura do Beyou"
summary: "A produção é um laptop de 2012 em um quarto: Debian 13, Docker, um Cloudflare Tunnel discando para fora e zero portas abertas para a internet."
---

A [Visão Geral da Arquitetura](/architecture/overview) mostra o que roda. Este tópico é sobre onde tudo isso roda, e é a minha parte favorita do projeto inteiro: o ambiente de produção do Beyou é a minha primeira infraestrutura self-hosted, construída no primeiro computador que eu tive.

## A máquina

A produção é um laptop LG S460. Minha mãe o comprou para mim em 2020, usado, para eu estudar durante o ensino médio, e foi nele que aprendi a programar. Hoje ele atende cada requisição que o Beyou recebe, de um quarto da minha casa.

| | |
|---|---|
| **Modelo** | LG S460 (comprado usado em 2020) |
| **CPU** | Intel Core i3-3120M, 2 núcleos / 4 threads @ 2,5 GHz (Ivy Bridge, geração de 2012) |
| **RAM** | 8 GB DDR3 |
| **Armazenamento** | SSD SATA de 120 GB |
| **Rede** | Um cabo de rede comum até o roteador de casa |

Dois núcleos e 8 GB não é muito, e acabou sendo o bastante. A stack inteira cabe: a API, o banco, os dois sites nginx e todo o overlay de monitoramento.

## O sistema operacional

Debian 13 (trixie), headless, sem ambiente gráfico. A montagem foi deliberadamente manual: formatar o disco, instalar o Debian, aplicar os patches de segurança e construir a stack peça por peça. Fazer na mão primeiro é parte do objetivo. Cada tarefa que eu repito vira motivação para automatizá-la, e esse ciclo me ensinou mais sobre infraestrutura de baixo nível do que qualquer plataforma gerenciada.

## O caminho para dentro: Cloudflare Tunnel

O roteador não encaminha nada. Nenhuma porta. Em vez disso, o `cloudflared` roda no laptop e disca para fora, até a borda da Cloudflare, mantendo uma conexão persistente. A Cloudflare termina o TLS e empurra as requisições dos hostnames públicos por esse túnel até serviços em loopback:

| Hostname | Serviço no laptop |
|----------|-------------------|
| **app.beyouweb.com** | nginx do web app em 127.0.0.1:3000 (faz proxy de /api/v1 para o backend) |
| **docs.beyouweb.com** | nginx da documentação em 127.0.0.1:3002 (mesmo proxy de /api/v1) |
| **obs.beyouweb.com** | Grafana em 127.0.0.1:3001 |
| **mnt.beyouweb.com** | GlitchTip em 127.0.0.1:8000 |

Esse desenho resolve em silêncio o problema clássico do servidor caseiro: o IP dinâmico. Como o túnel é uma conexão de saída, o IP público da casa é irrelevante. O DNS aponta para a Cloudflare, nunca para o meu roteador, e quando a operadora troca o endereço o cloudflared simplesmente reconecta. Eu nunca precisei pensar nisso, o que só percebi de verdade ao sentar para escrever este documento.

O site de marketing em **beyouweb.com** é a única superfície pública que nem passa pelo laptop: HTML estático no Cloudflare Pages.

## Mãos remotas: Tailscale

O SSH não é alcançável pela internet. O laptop participa de uma rede Tailscale, e a administração acontece apenas de dispositivos dentro dela. O fail2ban roda como uma camada extra por baixo.

A consequência mais bonita: fazer deploy não precisa de sessão SSH. O CI publica as imagens no GHCR, o Watchtower no laptop as busca e reinicia o que mudou. Fazer merge na main é o deploy; o laptop puxa, nada entra empurrado.

## O modelo de falha, com honestidade

Esta é uma máquina de casa, e o documento seria desonesto se fingisse o contrário.

- **Queda de energia ou de internet**: o app cai e fica fora até alguém ligar o laptop. Não há auto-recuperação para a máquina em si.
- **Depois do boot**: as políticas de restart do Docker trazem todos os containers de volta sozinhas. A recuperação é um botão de ligar.
- **Backups**: ainda não existem. É a maior lacuna conhecida de todo o setup, e vem antes de qualquer coisa mais sofisticada no roadmap.
- **Incidentes até agora**: exatamente uma categoria, que não está em nenhum template de postmortem: familiares desconectando o cabo de internet.

## Por que um laptop em um quarto

O Beyou é um app 100% gratuito, e eu não queria gastar uma moeda para hospedá-lo. Tentei o caminho comum primeiro: free tiers de nuvem e VPS baratos. Tecnicamente funcionavam, e me entediavam para fora do projeto toda vez.

A virada foi descobrir os Cloudflare Tunnels e perceber como tinha ficado fácil colocar um app rodando localmente na internet de verdade. Dali a coisa rolou: pegar a máquina antiga, formatar, instalar o Debian 13, aplicar os patches, montar a infraestrutura, comprar o domínio e colocar o web app, o backend e o banco no ar. Rodar o app só no localhost era desmotivador; ter uma produção de verdade, onde tudo persiste e eu abro o Beyou do celular em qualquer lugar, me fez querer usar o meu próprio produto. Depois vieram o monitoramento, o app mobile nativo, e cada camada deixou o conjunto melhor.

## O que vem depois

Por enquanto fica exatamente como está. A lista de curiosidades para ir mais fundo: replicação do backend, load balancing e entender como seria uma história real de failover em um hardware desses. Mas backups vêm primeiro.
