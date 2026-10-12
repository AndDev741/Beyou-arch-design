---
title: "Infraestrutura do Beyou"
summary: "A produção é um laptop de 2012 em um quarto: Debian 13, um cluster k3s de um nó gerido pelo Flux a partir do Git, um Cloudflare Tunnel discando para fora e zero portas abertas para a internet."
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

Dois núcleos e 8 GB não é muito, e acabou sendo o bastante. A stack inteira cabe: a API, o banco, os dois sites nginx, a stack de monitoramento e o próprio Kubernetes por cima.

## O sistema operacional

Debian 13 (trixie), headless, sem ambiente gráfico. A montagem foi deliberadamente manual: formatar o disco, instalar o Debian, aplicar os patches de segurança e construir a stack peça por peça. Fazer na mão primeiro é parte do objetivo. Cada tarefa que eu repito vira motivação para automatizá-la, e esse ciclo me ensinou mais sobre infraestrutura de baixo nível do que qualquer plataforma gerenciada.

Foi esse mesmo ciclo que tirou a produção do Docker Compose. Até outubro de 2026 a stack era um projeto Compose com um overlay de monitoramento, o Watchtower para atualizações e timers do systemd para os backups. Hoje o laptop roda **k3s**, um cluster Kubernetes de um nó só, e tudo nele está declarado num repositório Git público, o [be-infra](https://github.com/AndDev741/be-infra), que o [Flux](https://fluxcd.io) aplica. O Compose continua vivo como ambiente de desenvolvimento local e de e2e (o projeto [Beyou Dev Env](/projects/beyou-dev-env)); ele não roda mais a produção.

## O caminho para dentro: Cloudflare Tunnel

O roteador não encaminha nada. Nenhuma porta. Em vez disso, o `cloudflared` roda no laptop e disca para fora, até a borda da Cloudflare, mantendo uma conexão persistente. A Cloudflare termina o TLS e empurra as requisições de todos os hostnames públicos por esse túnel até um único lugar: o Traefik, o único ingress controller do cluster, numa node port. O Traefik roteia pelo header Host para um Ingress por hostname:

| Hostname | Para onde o Traefik envia |
|----------|---------------------------|
| **app.beyouweb.com** | o nginx do web app (faz proxy de /api/v1 para o backend) |
| **api.beyouweb.com** | o backend, direto (/api/v1) |
| **docs.beyouweb.com** | o nginx do site de documentação (mesmo proxy de /api/v1) |
| **obs.beyouweb.com** | Grafana |
| **mnt.beyouweb.com** | GlitchTip |

Nenhum app tem NodePort ou porta de host própria, então adicionar um hostname é um Ingress no Git, não um furo novo em lugar nenhum.

Esse desenho resolve em silêncio o problema clássico do servidor caseiro: o IP dinâmico. Como o túnel é uma conexão de saída, o IP público da casa é irrelevante. O DNS aponta para a Cloudflare, nunca para o meu roteador, e quando a operadora troca o endereço o cloudflared simplesmente reconecta. Eu nunca precisei pensar nisso, o que só percebi de verdade ao sentar para escrever este documento.

O site de marketing em **beyouweb.com** é a única superfície pública que nem passa pelo laptop: HTML estático no Cloudflare Pages.

## O cluster

Cada app ganha o seu próprio namespace, e cada namespace ganha as mesmas três cercas: uma **ResourceQuota** (um orçamento de memória, para que um app nunca deixe o nó sem recursos), um **LimitRange** com os padrões, e uma **NetworkPolicy default-deny** que depois é aberta um salto de cada vez. Para o Beyou essa cadeia é Traefik para os dois pods nginx, nginx e Traefik para o backend, backend para o banco, e o Prometheus e as sondas de uptime do GlitchTip para as portas que eles verificam. Todo o resto do cluster é recusado.

| Namespace | O que roda lá |
|-----------|---------------|
| **beyou** | web (nginx), docs (nginx), o backend Spring Boot e os CronJobs de backup |
| **data** | o cluster PostgreSQL 17 compartilhado, operado pelo CloudNativePG |
| **monitoring** | Prometheus, Grafana, Loki e Alloy (o [tópico de monitoramento](/architecture/monitoring)) |
| **glitchtip** | o rastreador de erros, com o seu Valkey |
| **ingress** | Traefik |

O backend roda com **uma réplica e a estratégia Recreate**, de propósito. Os uploads ficam num volume local-path que só um pod consegue montar, e os três jobs `@Scheduled` não podem rodar duas vezes. As duas coisas precisam mudar antes que uma segunda réplica faça sentido.

O banco é um cluster CloudNativePG chamado `pg`: uma instância PostgreSQL 17 com um banco e um role por app, com os roles declarados no Git e reconciliados pelo operador. O banco do Beyou é o `beyou`, dono do seu próprio role; um segundo role, somente leitura, existe só para que o dashboard de produto do Grafana consiga consultá-lo sem poder mudar nada.

Os segredos também estão no Git, criptografados com SOPS e age, e descriptografados pelo Flux dentro do cluster. O repositório é público, então esta é a linha que importa: a chave age fica no servidor (com uma cópia dentro do cluster, para o Flux) e num gerenciador de senhas, e em nenhum outro lugar.

## Mãos remotas: Tailscale, e o Git

O SSH não é alcançável pela internet. O laptop participa de uma rede Tailscale, e a administração acontece apenas de dispositivos dentro dela. O fail2ban roda como uma camada extra por baixo.

A consequência mais bonita: fazer deploy não precisa de sessão SSH. O CI constrói cada imagem e a publica no GHCR como `:latest`. A automação de imagens do Flux percebe o digest novo, faz commit dele no be-infra e faz o rollout do workload a partir desse commit. Fazer merge na main continua sendo o deploy; o laptop puxa, nada entra empurrado. O que mudou desde os tempos do Watchtower é o registro: o Git sempre diz exatamente qual build está rodando, e voltar atrás é um `git revert`, não um chute numa tag antiga. Qualquer coisa alterada na mão com `kubectl` em algo que o Flux gerencia volta ao que está no Git no próximo reconcile.

## O modelo de falha, com honestidade

Esta é uma máquina de casa, e o documento seria desonesto se fingisse o contrário.

- **Queda de energia ou de internet**: o app cai e fica fora até alguém ligar o laptop. Não há auto-recuperação para a máquina em si.
- **Depois do boot**: o k3s sobe sozinho, e o cluster traz cada workload de volta ao estado que o Git descreve. A recuperação é um botão de ligar.
- **Backups**: noturnos, fora da máquina, criptografados e rodados pelo próprio cluster. Um CronJob faz o dump do banco `beyou` com `pg_dump`, empacota o volume de uploads num tarball e entrega os dois ao restic, que os criptografa dentro do pod e os envia ao Cloudflare R2 com retenção de 7 diários / 4 semanais / 6 mensais. Uma trava de tamanho falha a execução se o repositório passar do orçamento, e um heartbeat vai para o GlitchTip só em caso de sucesso. Uma vez por semana um segundo CronJob restaura o snapshot mais recente num Postgres descartável dentro do próprio pod (nunca na instância compartilhada) e compara a contagem de linhas com o banco real; ele manda o seu próprio heartbeat só quando dá PASS, porque backup que ninguém restaurou é chute. A configuração não precisa mais de backup à parte: está no Git, criptografada. Essa foi a maior lacuna do setup por muito tempo. Está fechada para perda de dados e não faz nada por disponibilidade: continua um disco, uma máquina, nenhum failover.
- **Incidentes até agora**: exatamente uma categoria, que não está em nenhum template de postmortem: familiares desconectando o cabo de internet.

## Por que um laptop em um quarto

O Beyou é um app 100% gratuito, e eu não queria gastar uma moeda para hospedá-lo. Tentei o caminho comum primeiro: free tiers de nuvem e VPS baratos. Tecnicamente funcionavam, e me entediavam para fora do projeto toda vez.

A virada foi descobrir os Cloudflare Tunnels e perceber como tinha ficado fácil colocar um app rodando localmente na internet de verdade. Dali a coisa rolou: pegar a máquina antiga, formatar, instalar o Debian 13, aplicar os patches, montar a infraestrutura, comprar o domínio e colocar o web app, o backend e o banco no ar. Rodar o app só no localhost era desmotivador; ter uma produção de verdade, onde tudo persiste e eu abro o Beyou do celular em qualquer lugar, me fez querer usar o meu próprio produto. Depois vieram o monitoramento, o app mobile nativo, e cada camada deixou o conjunto melhor.

O Kubernetes veio por último, pelo mesmo motivo de todo o resto: aprender em algo de verdade. Um nó só não precisa de um orquestrador. Mas ganha com o que veio junto: cada app isolado atrás da sua própria quota e network policy, uma única stack de monitoramento olhando o host, o cluster e cada pod, e uma produção que eu consigo reconstruir a partir de um repositório Git.

## O que vem depois

O próximo passo são duas réplicas do backend atrás do Service, o que exige um lock para que só uma réplica rode cada job agendado, e tirar os uploads do volume local para um object storage. Depois disso, a lista de curiosidades continua a mesma: entender como seria uma história real de failover em um hardware desses. Backups lideravam essa lista. Agora que existem, sobrou a metade mais difícil — continuar de pé, e não apenas não perder nada.
