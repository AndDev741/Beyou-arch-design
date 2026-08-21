---
title: "Serviço de E-mail"
summary: "Seis e-mails transacionais, uma classe de serviço: como cada mensagem se desacopla da transação do banco, o que acontece quando o SMTP falha e os templates bilíngues inline."
---

Este documento cobre o subsistema de e-mail: quais mensagens existem, como cada uma é mantida segura em relação à transação, como falhas são tratadas por fluxo e como os templates são construídos e localizados.

## O que é enviado

O sistema envia exatamente seis e-mails, todos de uma única classe de serviço no pacote de notificação:

| # | E-mail | Gatilho | Contém |
|---|--------|---------|--------|
| 1 | Verificação de cadastro | Conta nova criada | Botão com link para a página de verificação, aviso de expiração em 24 horas |
| 2 | Redefinição de senha | Pedido de esqueci-a-senha | Botão com link para a página de reset, TTL em minutos |
| 3 | Código de exclusão de conta | Exclusão solicitada | Seis dígitos, deliberadamente sem link e sem botão: uma exclusão não pode estar a um clique de uma caixa de entrada |
| 4 | Confirmação de feedback | Usuário envia feedback | Eco da categoria e do texto enviado |
| 5 | Resposta de feedback | Admin responde | A resposta mais o original citado de volta |
| 6 | Aviso de feedback no console | Usuário envia feedback | Um link para o console de admin, e nada além disso |

Igualmente deliberado é o que nunca é enviado: uma mudança de status de feedback não avisa ninguém (não existe listener, então nenhum endpoint futuro pode enviar e-mail por acidente), cadastros via Google não recebem nada (o Google já verificou o endereço) e nada anuncia a conclusão do reset de senha nem a exclusão efetiva da conta.

O aviso é o único e-mail endereçado a quem opera o produto, e não a um usuário, e o vazio dele é o projeto. Leva um link e nenhuma categoria, nenhum remetente e nada do texto enviado: feedback pode ser pessoal, e copiar isso para dentro de um provedor de e-mail só para poupar um clique é um mau negócio. Quem recebe é quem tem ROLE_ADMIN no momento do envio, menos quem escreveu, então um admin que manda feedback recebe o recibo de sempre e nenhuma segunda mensagem sobre si mesmo. Um envio produz, portanto, um recibo mais uma mensagem por admin, cada uma endereçada individualmente — um To: compartilhado mostraria a cada admin o endereço dos outros.

```mermaid
flowchart LR
  REG["📝 Cadastro"] --> ES["✉️ EmailService"]
  PR["🔑 Reset de senha"] --> ES
  DEL["🗑️ Código de exclusão"] --> ES
  FB["💬 Feedback: confirmação + resposta"] --> ES
  FBA["🔔 Aviso de feedback no console"] --> ES
  ES --> SMTP["📤 SMTP (StartTLS)"] --> INBOX["📬 Caixa de entrada"]
```

## Três jeitos de desacoplar da transação

Todo e-mail precisa esperar o commit da sua transação; um link de reset apontando para um token que sofreu rollback seria pior que nenhum e-mail. O código chega a esse objetivo de três formas diferentes, e as diferenças importam:

| Fluxo | Mecanismo | Thread |
|-------|-----------|--------|
| Confirmação, resposta e aviso de feedback | Listener @Async em evento transacional AFTER_COMMIT | Segundo plano |
| Reset de senha, código de exclusão | Sincronização afterCommit registrada à mão | A thread da requisição: a resposta HTTP espera o SMTP |
| Verificação de cadastro | Listener @Async comum, sem fase transacional | Segundo plano |

O caminho do cadastro merece etiqueta de aviso própria: ele só é correto porque o `registerUser` não carrega `@Transactional`, então o save já commitou quando o evento é publicado. Adicionar `@Transactional` àquele método reintroduziria em silêncio a corrida de enviar-antes-do-commit que os outros fluxos evitam.

Nenhum executor é configurado; o `@Async` usa o padrão de virtual threads. Sem limite, sem fila, sem retry em lugar nenhum.

## Quando o SMTP falha

Cada fluxo responde à falha de um jeito, e as diferenças são o design:

- **Reset de senha**: o erro é registrado e a linha do token é apagada, para o cooldown de 5 minutos não deixar um usuário preso esperando um link que nunca saiu do prédio.
- **Código de exclusão**: mesma ideia, borda mais afiada. A linha do código é descartada por um helper REQUIRES_NEW, porque o descarte roda depois do commit, onde uma chamada comum de repositório entraria em uma transação morta. Ninguém recebeu o código, então ninguém deveria cumprir o cooldown.
- **E-mails de feedback**: registrados e engolidos. O envio ou a resposta sobrevivem; o recibo é melhor esforço. O recibo e o aviso ficam em blocos try separados de propósito, e o aviso captura de novo a cada destinatário, então uma caixa morta custa uma mensagem em vez de esconder um envio do console.
- **Verificação de cadastro**: nada captura a falha. A exceção morre no log padrão do handler assíncrono, a linha do usuário e seu token ficam, e aqui está a lacuna real: o login recusa contas não verificadas com EMAIL_NOT_VERIFIED, e não existe endpoint de reenvio. Um e-mail de verificação perdido encalha a conta.

Não há retry, outbox nem rastreio de entrega em lugar nenhum. O que torna as falhas engolidas visíveis é o pipeline de logs: toda falha registra em ERROR, e linhas ERROR viram eventos no GlitchTip. O rastreador de erros é o sino do retry.

Um efeito colateral operacional: o indicador de saúde de mail do Spring fica ligado, então um servidor SMTP morto derruba o `/actuator/health` para DOWN.

## Configuração de SMTP

| Variável | Propósito |
|----------|-----------|
| MAIL_HOST / MAIL_PORT | Servidor SMTP, StartTLS habilitado, auth ligado |
| MAIL_USERNAME / MAIL_PASSWORD | Credenciais |
| MAIL_FROM | Endereço remetente, padrão MAIL_USERNAME |

E-mail não é opcional. Os quatro valores centrais vêm sem defaults, e a resolução do remetente falha na criação do bean quando as variáveis estão ausentes, então o app não sobe sem um ambiente de mail. Valores vazios sobem e falham a cada envio. O único modo sem SMTP sancionado é o perfil e2e, que contorna o e-mail em vez de desligá-lo: o cadastro auto-verifica e pula o evento, e o código de exclusão volta no corpo da resposta. As duas saídas de emergência são bloqueadas em produção pelo validador de boot.

## Templates e idiomas

Sem engine de template e sem arquivos de template: cada corpo é um text block Java inline, só HTML, formatado com String.formatted. Com dois idiomas por mensagem, isso dá onze templates hardcoded dividindo o mesmo cabeçalho, o azul da marca e o rodapé com o ano — onze e não doze porque o aviso do console é só em inglês. Todos os outros templates escolhem idioma porque quem lê é um usuário; esse é endereçado a quem opera o produto, tem duas frases, e o conteúdo dele é uma URL.

A escolha de idioma é uma decisão de dois ramos por mensagem: qualquer coisa começando com "pt" recebe português, todo o resto (incluindo nulo) recebe inglês. A parte interessante é de onde cada fluxo lê o idioma:

- A confirmação de feedback prefere o idioma capturado no contexto de UI do envio ao invés da preferência do perfil, porque o recibo chega na hora e o campo do perfil fica nulo até o usuário abrir as configurações. Ler o perfil primeiro mandava um recibo em inglês para toda conta nova.
- A resposta de feedback prefere a preferência atual do perfil, porque uma resposta pode chegar dias depois, quando o contexto capturado já envelheceu.

Texto escrito por usuário ou admin passa por escape de HTML antes da interpolação, então um corpo de feedback não consegue injetar markup no próprio recibo.

## Cooldowns e rate limits

Duas camadas independentes controlam os endpoints que produzem e-mail:

| Fluxo | Cooldown no service | Balde de rate limit |
|-------|---------------------|---------------------|
| Reset de senha | 5 min entre pedidos por conta; cada token novo invalida o anterior | 5 / 15 min por IP (faixa auth) |
| Código de exclusão | 60 segundos, em segundos de propósito: o usuário espera na tela e o reenvio precisa funcionar na mesma sentada | 10 / hora por usuário |
| Cadastro | Nenhum | 5 / 15 min por IP (faixa auth) |
| Feedback | Nenhum | 10 / hora por usuário |

Uma minúcia de configuração registrada por honestidade: o default do yaml para o TTL do reset é 15 minutos, enquanto o template de env ainda entrega 30. O yaml vence, a menos que o operador copie o valor do template.

## Cobertura de testes

Nenhum teste fala SMTP. Os fluxos de feedback mockam o próprio JavaMailSender e verificam as mensagens capturadas (destinatários, assunto, corpo e o caso que sustenta tudo: um envio de feedback sobrevive a um send que lança exceção). O fluxo de exclusão mocka o EmailService e fixa o formato de seis dígitos, o armazenamento só do hash e o descarte em caso de falha. O aviso do console tem suíte própria, verificando por destinatário em vez de contar envios: cada admin avisado exatamente uma vez, quem escreveu nunca avisado sobre a própria mensagem, nenhum texto de feedback no corpo, e um recibo que lança exceção ainda deixando o console avisado. A lacuna: o fluxo de reset de senha não tem teste dedicado, então a limpeza do token em falha de envio depende só de revisão de código.

## O que pode melhorar

| Área | Estado atual | Nota |
|------|--------------|------|
| Reenvio de verificação | Sem endpoint | O único modo de falha que encalha uma conta; o candidato número um |
| Retry | Tentativa única em tudo | Uma tabela de outbox ou retry no provedor removeria o padrão do GlitchTip-como-sino |
| Thread de envio de reset/exclusão | A requisição espera o SMTP | Migrar para o padrão de listener assíncrono dos fluxos de feedback cortaria a latência |
| Templates | Onze blocos HTML inline | Extrair o chrome compartilhado reduziria a duplicação; engine de template segue exagero |
| Testes do fluxo de reset | Nenhum | O único fluxo de e-mail sem cobertura direta |
