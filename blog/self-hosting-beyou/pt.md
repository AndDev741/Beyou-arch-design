---
title: "Self-Host do Beyou: Os Desafios e as Coisas Boas de Tentar"
summary: "A produção do Beyou é um laptop de 2012 no meu quarto, alcançado por um túnel, com zero portas abertas no roteador. Este é o porquê de escolher self-host em vez de free tiers, o que quebra de verdade, e por que abrir meu próprio app todo dia ainda é essa sensação."
---

Todo dia eu abro o Beyou na web, depois o app mobile, às vezes a página de docs, depois o Grafana e o GlitchTip. E toda vez, alguma parte do meu cérebro lembra: tudo isso está rodando no meu laptop. Cada requisição de cada uma dessas telas viaja até uma máquina no meu quarto. Todo o controle, toda a arquitetura que construí, funcionando lisinho. É uma das melhores sensações que esse projeto me deu.

Este post é sobre como essa montagem nasceu, o que custa de verdade rodar produção em casa, e por que eu faria de novo.

## Por que não simplesmente usar uma nuvem?

O Beyou é um app 100% gratuito, e decidi cedo que não queria gastar uma moeda para hospedá-lo. Então fiz o que todo mundo faz primeiro: free tiers de nuvem e VPS baratos. Tecnicamente funcionavam. Também me entediavam para fora do projeto todas as vezes. Gerenciar limites de cota e dashboards feitos para me vender upgrade era o oposto da energia que eu queria colocar no Beyou.

A virada foi descobrir os Cloudflare Tunnels, e perceber como tinha ficado fácil colocar um app rodando localmente na internet de verdade. Sem port forwarding, sem IP fixo, sem expor minha rede de casa. Um daemon na máquina disca para fora até a Cloudflare, e a Cloudflare manda os visitantes túnel abaixo. No momento em que entendi isso, o plano se escreveu sozinho.

## A máquina

Aqui está minha parte favorita da história inteira. A produção roda em um laptop LG S460: dois núcleos, 8 GB de DDR3, um SSD de 120 GB, hardware de por volta de 2012. Minha mãe o comprou para mim, usado, em 2020, para eu estudar durante o ensino médio. É a máquina em que aprendi a programar.

Formatei, instalei Debian 13 sem interface gráfica, apliquei os patches de segurança e montei a stack peça por peça: os containers, o túnel, Tailscale para o SSH nunca tocar a internet pública, fail2ban por baixo. O roteador não encaminha nada. Nenhuma porta. E o problema clássico do servidor caseiro, o IP dinâmico, se dissolve por design: o túnel é uma conexão de saída, então quando a operadora troca meu endereço, o daemon reconecta e o DNS nunca apontou para a minha casa mesmo.

O computador em que aprendi a programar agora serve cada requisição que meu produto recebe. Existe uma simetria nisso sobre a qual me recuso a ser cínico.

## Os desafios, com honestidade

Self-host em uma máquina de casa significa ser dono de um modelo de falha que a maioria dos tutoriais pula:

- **Energia e internet são o meu SLA.** Se qualquer uma piscar por tempo suficiente, o app fica fora até alguém (eu) apertar o botão de ligar. Depois do boot, as políticas de restart do Docker trazem cada container de volta sozinhas, então a recuperação é exatamente um botão, mas o botão é físico.
- **Meu único incidente de produção até agora**: um familiar desconectando o cabo de internet. Nenhum template de postmortem cobre isso. Eu verifiquei.
- **Disciplina substitui a plataforma.** Ninguém mais rotaciona meus segredos nem endurece meus padrões, então as regras são rígidas: tudo escuta em loopback, o túnel e o Tailscale são as únicas entradas, e o servidor sobe se recusando a aceitar configuração insegura de produção.
- **Backups são a lacuna honesta.** O banco ainda não tem rotina de backup, e isso está acima de todo o resto na lista da infraestrutura. Self-host torna lacunas assim pessoais: não existe ticket de suporte para se esconder atrás.

Também existe um padrão de que passei a gostar: as coisas começam manuais, e o incômodo de repeti-las vira a motivação para automatizar. Os deploys foram de manuais para o Watchtower puxando imagens sozinho. A configuração do rastreador de erros foi de cliques para um script de bootstrap. Toda tarefa chata que sobreviveu tempo suficiente foi automatizada, e eu aprendi algo em cada uma.

## As coisas boas

O custo é zero, e para um app gratuito isso importa, mas o dinheiro é honestamente a menor parte.

Tudo é meu. Quando algo quebra, o caminho inteiro do navegador ao banco é inspecionável por mim, em hardware que posso tocar. Quando adicionei a stack de monitoramento, eu não estava lendo uma página de preços, estava aprendendo como alvos de coleta e labels de log funcionam de verdade. O self-host transformou infraestrutura de uma conta em um currículo.

E mudou minha relação com o próprio produto. Quando o Beyou só rodava no localhost, usá-lo parecia teste. Uma produção de verdade, onde tudo persiste e posso abrir minhas rotinas de qualquer lugar, me fez querer usar meu próprio app todo dia. Esse uso diário é de onde vem metade das ideias de produto agora.

## Se você está pensando em tentar

Meu conselho honesto, baseado em nada além dessa experiência: o custo de entrada é bem menor do que parece. Uma máquina velha, um Linux instalado e um túnel te dão uma URL de produção real sem abrir uma única porta. Seja franco consigo mesmo sobre o modelo de falha (o seu também vai ser um cabo de energia e um familiar), mantenha toda superfície administrativa fora da internet pública, e coloque backups mais alto na sua lista do que eu coloquei.

A recompensa é aquela sensação com que abri este post. Abrir o seu próprio app, no seu próprio domínio, servido pela sua própria máquina, e ver cada camada que você construiu fazendo o seu trabalho. Não acho que uma plataforma gerenciada consiga vender isso.

## O que vem depois

A montagem fica como está por enquanto. A lista de curiosidades para ir mais fundo: replicação do backend, load balancing e entender como seria uma história real de failover em um hardware desses. Mas backups vêm primeiro. Escrevi isso na documentação, então agora é uma promessa.

## Atualização, agosto de 2026

A promessa foi cumprida. Os backups rodam toda noite: o banco, o volume de uploads e o arquivo `.env` vão para o Cloudflare R2 via restic, criptografados antes de sair da máquina, e um job semanal restaura o snapshot mais recente num banco descartável e compara a contagem de linhas com o banco real. Esse segundo job é o que eu defenderia com mais força. Backup que ninguém restaurou é chute, e eu prefiro descobrir isso numa segunda-feira de manhã do que no pior dia possível.

O que me surpreendeu no caminho é que a minha primeira versão silenciosamente nunca apagava nada: o restic agrupa snapshots por caminho quando aplica a política de retenção e, como eu montava cada execução num diretório temporário novo, cada snapshot caía num grupo só dele e o "manter 7 diários" mantinha todos para sempre. Só peguei isso porque rodei a coisa três vezes seguidas e contei.

O que isso não resolve é disponibilidade. Continua sendo um disco num laptop só, sem failover, então fecha a metade de perda de dados do modelo de falha e deixa a metade difícil exatamente onde estava.
