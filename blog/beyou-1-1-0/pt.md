---
title: "Beyou 1.1: A Versão Que Meus Testadores Escreveram"
summary: "Marquei a 1.0 no dia 17 de agosto e entreguei o app para pessoas que nunca tinham visto ele. Oito dias depois, a 1.1 fecha trinta tickets que elas acharam para mim: um onboarding que criou 58 hábitos quando alguém pediu 3, um fuso com que toda conta nascia e que ninguém nunca escolheu, e uma conta em que dava para se trancar para sempre perdendo um único e-mail."
---

Marquei a 1.0 no dia 17 de agosto. Depois dei o app para um punhado de pessoas que nunca tinham visto ele.

Oito dias depois estou marcando a 1.1. Ela fecha trinta tickets, e quase nenhum deles é feature. O que elas acharam descreveu meus pontos cegos melhor do que eu conseguiria, então este post é basicamente a lista delas.

## O onboarding que se empolgou

Um dos meus testadores escolheu três hábitos no onboarding com IA e terminou com cinquenta e oito.

O wizard cria hábitos de verdade pelos endpoints REST normais, uma chamada por hábito, e coloca um botão de Retry na sua frente quando alguma chamada falha. O Retry rodava o lote inteiro de novo. Cada clique somava mais um conjunto completo de tudo que já tinha dado certo, e as falhas mantinham o botão ali na tela. Clique após clique, a conta virou uma parede de duplicatas.

As criações agora leem a conta antes e pulam qualquer coisa que já esteja lá pelo nome, então o Retry virou idempotente. Também entrei no banco e limpei a conta na mão, que é o tipo de tarefa que faz você consertar um bug direito em vez de rápido.

## Um dia que virava uma hora atrasado

Esse aqui eu achei lendo o meu próprio código, e ele acabou sendo menor do que a minha primeira descrição dizia.

Toda conta era criada com o fuso em `UTC`, e nada nunca mudava isso a menos que a pessoa entrasse em Configurações e clicasse na sugestão. `UTC` também era indistinguível de uma escolha deliberada, porque não existia estado nulo nem flag, então nada podia fazer backfill com segurança.

Minha anotação original nesse card dizia que um usuário no Brasil estava perdendo hábitos toda noite. Isso não se sustentou. O Beyou ainda não tem usuários em produção, e as únicas contas alcançáveis são as minhas. A versão honesta é sazonal e pequena: Lisboa é UTC+0 no inverno, onde o fuso guardado é simplesmente o certo, e UTC+1 no verão, onde o dia vira uma hora atrasado. Um check feito entre meia-noite e 1h local cai no dia anterior, e se foi o único check daquele dia de calendário, o dia real fecha como perdido às 3h. Uma hora, sete meses por ano, em contas de teste.

Os números de offset grande continuam valendo a pena registrar, porque é isso que esse bug faz no instante em que alguém se cadastra a várias horas do UTC. Em UTC-3 o dia vira às 21h e fecha às 23h, então uma rotina da noite é arquivada no dia seguinte enquanto o dia de hoje leva o carimbo de perdido, e o `DayCloseService` é insert-only, então esse carimbo nunca se cura sozinho. Consertar antes de alguém estar nessa situação foi o mais barato que esse reparo vai ser algum dia.

Agora o navegador e o celular mandam o fuso em que o aparelho está de verdade, nos quatro caminhos de cadastro. Uma coluna `timezone_source` separa nunca-definido de escolhido, então a adoção única para contas existentes só passa por cima do default. Essa política mora no backend e não em cada cliente, porque um notebook aberto em outro país não deveria mover o limite do dia de quem está viajando.

Um segundo bug saiu da mesma auditoria e sobrevive à correção de fuso por conta própria: o `checkTime` era carimbado pelo relógio do servidor enquanto o `checkDate` usava o fuso do dono. Os dois leem o fuso do dono agora.

Deixei de fazer o header `X-Timezone` que o meu próprio escopo pedia. Ele é um segundo caminho de adoção que precisa ficar consistente com o primeiro, faz um GET escrever no banco, e as únicas contas que ele alcança são as que nunca mais abrem o app, com as quais nenhum mecanismo do cliente consegue ajudar.

## Trancado para fora por um e-mail perdido

Se o seu e-mail de verificação nunca chegou, não existia jeito de pedir outro. A conta existia, se recusava a te deixar entrar, e não oferecia saída nenhuma. Cadastrar de novo com o mesmo endereço era bloqueado, corretamente, porque o endereço já estava em uso.

Agora tem um reenvio, e ele fica na tela que avisa que o e-mail não está verificado, que é onde a pessoa nessa situação realmente está. O endpoint responde igual para um endereço que existe e para um que não existe, então ninguém consegue usar ele para descobrir quem tem conta. O login com Google também passava direto pelo portão de verificação. Isso foi fechado.

## O agente aprendeu a contar

O agente de IA levou mais correções do que qualquer outra coisa nesta versão.

Ele simplesmente não conseguia mexer no progresso das metas. Aumentar e diminuir davam erro de lazy-initialization, porque a tool rodava fora de uma sessão e as coleções da meta nunca eram carregadas. Ele também escolhia metas por um id que tinha chutado em vez de pelo nome, então de vez em quando criava uma segunda meta em vez de editar a que você queria.

A tool de agenda anunciava MONDAY até SUNDAY enquanto o enum era Monday. Toda agenda que o agente tentava montar falhava na primeira tentativa, voltava como erro, e custava um round trip inteiro a mais no modelo antes de sair. Adicionar um hábito a uma seção de rotina devolvia o grupo novo com id nulo, porque o mapper rodava antes do flush. Editar uma rotina que já tinha ids de grupo dava erro de detached entity.

Nenhum desses bugs é interessante sozinho. Juntos, faziam o agente parecer pouco confiável, o que é pior do que um agente que recusa com todas as letras.

O chat também parou de dar de ombros. Quando um provedor falha, ou quando você chega no limite por hora, ele agora diz qual das duas coisas aconteceu. O rate limit chega no endpoint de streaming, o que não acontecia antes, e a chamada ao provedor não é mais cortada em dois minutos numa resposta longa.

## Finalmente sei se alguém está usando

Durante toda a 1.0 eu não fazia ideia de quantas pessoas abriam o app num dia qualquer.

O Beyou agora manda eventos de produto para o PostHog a partir do app web, do app mobile, do site de docs e da landing page, passando por um proxy first-party para que um bloqueador de anúncios não apague metade do retrato caladinho. O backend informa quem está ativo e quando cada pessoa entrou pela última vez, e o Grafana tem um painel para isso. O dia do cadastro viaja junto com o identify, então dá para envelhecer um cohort.

Tirei os códigos de OAuth e os tokens de reset de senha da captura antes de qualquer coisa ir para o ar. Bibliotecas de analytics pegam a URL inteira por padrão, e os dois moram na query string.

## Backups, enfim

No post sobre self-host eu escrevi que backup era a lacuna honesta e que ele estava acima de tudo na minha lista de infraestrutura. Está feito. O restic manda para o Cloudflare R2 toda noite, tem um teste de restauração semanal, e o tamanho do repositório tem teto, porque o R2 não tem limite duro de gasto e eu prefiro saber disso por uma trava do que por uma fatura.

## Preparando a Play Store

Uma boa fatia desta versão começou como papelada da Play Store e virou correção de verdade. O app declarava permissões que nunca usava, o que o Android mostra para as pessoas como se usasse. A política de privacidade para onde a listagem aponta agora existe, diz o que os analytics mandam de fato, e dá base legal para os e-mails de rotina antes desses e-mails serem ligados.

Fotos de perfil podem ser removidas agora, e aparecem na exportação dos seus dados, o que não acontecia. A exportação tinha um N+1 e ganhou o próprio bucket de rate limit, já que é de longe a coisa mais pesada que uma conta sozinha pode pedir.

## O que ficou em aberto

Os e-mails de engajamento estão prontos e desligados. O disparo, o consentimento, o toggle de preferência nas duas plataformas e o descadastro estão todos lá, atrás de uma flag que eu não liguei, porque quero ler umas duas semanas de analytics antes de começar a mandar e-mail para alguém.

Notificações push seguem sendo a maior ausência. Uma constância prestes a quebrar é exatamente o momento em que um lembrete valeria a pena, e o Beyou não tem nada ali ainda.

A listagem na Play Store também não está no ar. O bundle assinado é gerado sob demanda. O resto é papelada.
