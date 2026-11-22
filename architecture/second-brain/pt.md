---
title: "Segundo Cérebro"
summary: "Uma página que diz à pessoa se ela está no rumo ou à deriva: uma visão geral com veredito reescrita uma vez por dia, uma pergunta por vez, um perfil curto que toda conversa do assistente lê, anotações com data sobre cada hábito, meta, rotina e categoria, e seis chaves para o que pode ser lido. As regras, as três chamadas ao modelo, o que chega ao provedor de IA e quando, e a classe dona de cada regra."
---

O Segundo Cérebro é a página em que alguém vê o que o Beyou entende da sua vida e ajusta isso. Ela responde a uma pergunta: essa pessoa está indo na direção do que disse que quer, ou se afastando disso. A página abre com uma visão geral curta que diz isso, faz uma pergunta por vez, guarda o que aprende num perfil curto que toda conversa do assistente lê e em anotações com data sobre os hábitos, as metas, as rotinas e as categorias da pessoa, e deixa a pessoa desligar qualquer parte dos dados que ela pode ler. Fica na navegação principal do app web e do app de celular, para toda conta logada, em inglês e em português.

Este documento cobre os blocos da página, as regras por trás de cada um e a classe onde cada regra mora, e depois as chamadas ao modelo e o que acontece com os dados. As rotas estão na API do Controller do Segundo Cérebro.

## Os blocos

`GET /second-brain` devolve a página inteira numa transação somente leitura e não grava nada. A resposta traz nove blocos:

| Bloco | O que mostra | De onde vem |
|-------|--------------|-------------|
| Visão geral | Um parágrafo que o modelo escreveu sobre como as coisas estão indo, com um veredito mostrado como etiqueta | `second_brain_insights` |
| Perguntas | As perguntas abertas, uma por vez | `second_brain_questions` |
| Metas | Até três metas abertas mais próximas da data final, com o texto de ritmo do Resumo do Dia | A fonte GOALS |
| Rotinas | Os últimos sete dias contra os sete anteriores, com os itens que escaparam e os que se mantiveram | A fonte ROUTINES |
| Áreas | As categorias da pessoa ordenadas pela fração dos itens de rotina feitos em 14 dias | A fonte ROUTINES |
| Perfil | Até 20 linhas curtas em três tipos | `second_brain_facts` |
| Nota antiga | As frases da nota que o assistente guardava escondida antes da memória existir, para manter ou apagar. Só aparece enquanto existir uma | `users.user_context` |
| Memória | Anotações com data sobre os hábitos, as metas, as rotinas e as categorias da pessoa, agrupadas por categoria | `second_brain_notes` |
| Fontes | Seis chaves para o que o Segundo Cérebro pode ler | `second_brain_sources` |

Toda porcentagem e todo veredito são calculados no servidor, então os dois apps não arredondam a mesma semana de dois jeitos. Um bloco cuja fonte está desligada diz isso no lugar do conteúdo. É um estado diferente de uma fonte ligada sem nada dentro, e a resposta deixa o cliente separar os dois: `routine` vem nulo quando ROUTINES está desligada, e a lista de chaves resolve as metas e as áreas.

Numa tela larga a página web tem duas colunas. A da esquerda é para ler: a visão geral, as metas, as rotinas e as áreas. A da direita é para responder e decidir: as perguntas, o perfil, a nota antiga, a memória e as fontes. Abaixo do breakpoint `lg` a página vira uma coluna só e as perguntas ficam em segundo lugar, logo abaixo da visão geral, porque esse é o bloco que precisa da pessoa. O app de celular usa essa mesma ordem em coluna única.

**Metas.** A lista é `DailyBriefingFactsBuilder.goalsAhead`, a mesma que o diálogo da manhã mostra: metas abertas que têm data final, ordenadas pela distância até hoje para qualquer lado, limitadas a três, com uma meta atrasada há mais de duas semanas mandada para o fim. Cada uma traz o veredito de ritmo do servidor, e as duas páginas escrevem a frase com a mesma função compartilhada, `goalPaceMessage` em `@beyou/state`. Essa função mudou duas vezes por causa desta página. Uma meta que pede menos de uma unidade por dia aparecia como "0 livros por dia ainda te leva lá", porque doze livros num ano dão 0,03 por dia e uma casa decimal arredonda isso para nada. A primeira correção passou a quantidade para a semana e depois para o mês, multiplicando o `requiredPerDay` já arredondado do servidor. Isso criou outro absurdo, visto na página em 22/11/2026: uma entrevista daqui a quinze dias chegava como 0,1 por dia e virava "3 Entrevista por mês", cinco documentos em nove dias viravam "3,5 Documentos por semana", e um livro em 39 dias ganhava só "Atrás do ritmo.", sem nada depois. A frase agora parte de `remainingValue` e `daysRemaining`. Ela dá a quantidade por dia de um por dia para cima, por semana ou por mês só quando o tempo que falta cabe dois desses períodos, e nos outros casos diz quanto falta e em quantos dias: "5 em 9 dias ainda te leva lá". Uma meta com alvo de um tem veredito próprio, `ONE_TIME`, porque ela está feita ou não, e uma linha reta a chama de atrasada poucos dias depois do início. Ela fica sem frase de ritmo e sem marca na barra, e o modelo fica sabendo que ela não tem ritmo para estar atrás. O Resumo do Dia lê a mesma função, então as metas dele são escritas do mesmo jeito.

**Rotinas.** O bloco compara os sete dias que terminam ontem com os sete anteriores. Cada porcentagem é a fração dos itens de rotina pedidos que foram marcados. Um item pulado fica fora dos dois lados das porcentagens, porque um pulo é uma resposta que a pessoa deu. Abaixo dos dois números ficam os itens que escaparam e os que se mantiveram, no máximo três de cada, e essas duas listas contam os pulos: o mesmo item pulado de novo e de novo é um hábito sendo largado, e é nas listas que isso tem de aparecer. Um item escapou quando não foi feito (pulado ou deixado em aberto) em pelo menos duas das vezes em que apareceu nesta semana, e em pelo menos metade delas. Ele se manteve quando apareceu pelo menos quatro vezes e foi marcado em todas, sem nenhum pulo. Cada linha diz quantas vezes o item foi feito das vezes em que apareceu e, num item que escapou, quantas delas foram puladas, o que separa um hábito largado de propósito de um esquecido. A contagem é por vez e não por dia: um item em duas rotinas, ou duas vezes na mesma, aparece duas vezes no mesmo dia. `DailyBriefingFactsBuilder.buildSignals` calcula tudo isso. É a função de onde sai o texto do resumo, então esta página e o diálogo da manhã não discordam sobre a mesma semana.

**Áreas.** O bloco lê os mesmos 14 dias por categoria. Um check de snapshot guarda o hábito ou a tarefa de que veio, e esse item carrega as suas categorias. Todo check que apareceu e não foi pulado conta uma vez para cada categoria do item, então um item em duas categorias conta nas duas. Um check cujo hábito ou tarefa foi apagado depois não tem mais categoria onde contar e fica de fora. Só aparecem categorias com algo pedido, da maior porcentagem para a menor. A página etiqueta a primeira linha como a mais forte e a última como a mais fraca, e deixa as outras sem etiqueta. `secondBrainAreaEnds` em `@beyou/state` só devolve essas duas pontas quando pelo menos duas categorias têm porcentagem e as duas são diferentes.

## A visão geral e o veredito

A visão geral é um parágrafo sobre como a vida da pessoa está indo. O prompt (`prompts/secondBrainInsight.st`) pede de três a cinco frases. Os números em que o parágrafo se apoia estão nos blocos abaixo dele, então o prompt manda o modelo dizer o que eles somam e deixar os números onde estão.

O modelo devolve um veredito e um texto. O veredito é `SecondBrainVerdict`:

| Veredito | Significado |
|----------|-------------|
| ON_COURSE | O que a pessoa faz na maioria dos dias bate com as metas que ela tem e com os objetivos do perfil |
| DRIFTING | Não bate: metas atrás do ritmo, rotinas quase sempre em aberto, ou dias que vão para um lado diferente do que ela disse que quer |
| TOO_EARLY | Há pouco registrado para dizer uma coisa ou outra, como numa primeira semana, ou com rotinas e perfil ainda vazios |

O veredito tem coluna própria porque a página o mostra como etiqueta, e assim a resposta que a página existe para dar é lida de relance. O texto começa dizendo o veredito em palavras simples. Quando o veredito é DRIFTING, o prompt pede a única correção para a qual os dados apontam. Ou o plano pede mais do que as semanas da pessoa comportam, e uma meta ou uma rotina deveria mudar, ou o plano está certo e o comportamento escapou, e há algo para retomar. `SecondBrainVerdict.parse` lê com tolerância o que o modelo escreveu, e o que ele não reconhece vira TOO_EARLY, então um veredito que o modelo não deu com clareza nunca aparece como se fosse um.

`SecondBrainService.refreshInsight` monta o que o modelo lê. `SecondBrainSignals.promptData` escreve uma seção por fonte ligada e depois o perfil, e `recentAnswers` acrescenta as últimas respostas da pessoa: as respondidas entre as últimas 15 perguntas, oito no máximo, cada uma cortada em 300 caracteres. As respostas estão ali porque dizem o que os números não dizem, como o motivo de uma semana ter sido do jeito que foi. Todo número do bloco é calculado em Java e entregue pronto, a regra do Resumo do Dia. Os dias são escritos como distâncias a partir de hoje ("in 5 days, on a Tuesday", "overdue by 3 days").

O texto que volta é reduzido a um parágrafo e cortado na última frase inteira que cabe em 900 caracteres (`SecondBrainInsight.TEXT_MAX`). A coluna tem 1200, então esse limite pode mudar um pouco sem migração. Uma conta tem uma visão geral, substituída por um upsert a cada vez que é escrita.

### A primeira visão geral e a reescrita diária

A primeira visão geral nunca é escrita sem pedido. Uma conta que ainda não tem uma vê um botão, "Escrever minha visão geral", com o aviso de privacidade de IA ao lado: a visão geral e parte das perguntas são escritas por um provedor de IA externo, a partir das fontes ligadas, das linhas do perfil e das respostas. Apertar o botão chama `POST /second-brain/insight/refresh`, e essa chamada é o aceite da pessoa para o lado de IA da página (veja "Antes da primeira visão geral" abaixo).

Daí em diante a visão geral se reescreve uma vez por dia. `SecondBrainStore.insight` a marca como `stale` quando ela foi escrita antes do dia atual da conta. Vale o calendário da própria conta, pelo `UserDateResolver`, e não uma contagem de horas, então "hoje" significa aqui o mesmo que significa na rotina. Quando a página carrega e encontra uma visão geral existente vencida, com `aiAvailable` verdadeiro, ela chama a rota de refresh uma vez naquela visita. Assim a primeira visita de um dia reescreve a visão geral, uma segunda visita no mesmo dia a encontra em dia e não custa chamada de modelo, e um dia sem visita não escreve nada. O servidor não tem agendador para isso e nunca escreve uma visão geral sem que alguém peça.

A pessoa também pode pedir uma reescrita a qualquer momento com "Escrever de novo". Enquanto uma roda, a página esconde a etiqueta antiga, que seria lida como a de hoje. Uma reescrita que falha deixa a visão geral guardada no lugar e mostra o erro abaixo dela.

## As perguntas

A página faz uma pergunta por vez, e a pessoa responde em texto livre ou pula. Uma por vez é de propósito: uma lista de cinco parece um formulário para preencher, e formulário fica para depois.

São quatro tipos:

| Tipo | Quando é feita | Quem escreve |
|------|----------------|--------------|
| Inicial (`PROFILE_OBJECTIVES`, `PROFILE_LIKES`, `PROFILE_ABOUT`) | O perfil não tem fato daquele tipo, e aquela pergunta nunca foi feita antes, respondida ou pulada | Texto fixo em inglês e em português |
| Check-in (`CHECK_IN`) | Uma vez por dia do calendário da própria conta, quando a fonte ROUTINES está ligada e alguma rotina roda naquele dia | Texto fixo |
| Acompanhamento (`FOLLOW_UP`) | O modelo está em uso para a conta, há espaço, e nenhuma pergunta de acompanhamento foi guardada nas últimas seis horas | O modelo, sobre algo específico nos dados da pessoa |
| Coisa nova (`NEW_THING`) | Um hábito, uma meta ou uma rotina foi criado depois da primeira visita da conta, há menos de cinco perguntas abertas, e nunca se perguntou sobre aquela coisa | Texto fixo com o nome da coisa, escrito quando ela é criada (veja Memória abaixo) |

`POST /second-brain/questions/refill` roda os três primeiros nessa ordem (a pergunta sobre uma coisa nova é escrita pela própria criação): `SecondBrainStore.seed` grava os dois primeiros, que não precisam de modelo, depois `SecondBrainAi.followUps` é chamado e `SecondBrainStore.addFollowUps` guarda o que ele escreveu. Os limites:

- **Cinco perguntas abertas ao mesmo tempo** (`SecondBrainQuestion.MAX_OPEN`). Os passos param aí, e o modelo recebe o pedido de tantas perguntas de acompanhamento quanto ainda cabem.
- **Seis horas entre dois conjuntos de perguntas de acompanhamento** (`SecondBrainStore.FOLLOW_UP_PAUSE`). A pausa conta a partir da última pergunta de acompanhamento guardada, então uma chamada que voltou sem nada não a inicia.
- **Um assunto não é perguntado duas vezes.** Uma pergunta inicial é feita uma vez por conta, seja qual for o destino dela. O prompt de acompanhamento (`prompts/secondBrainQuestions.st`) recebe as últimas 15 perguntas com o que aconteceu com cada uma (respondida e com o quê, pulada, ainda esperando) e a instrução de não repetir nem reformular nenhuma, de não perguntar o que uma resposta ou uma linha do perfil já resolve, e de deixar em paz o assunto de uma pergunta pulada. O servidor também descarta uma pergunta de acompanhamento que repete uma dessas 15 palavra por palavra.
- **Uma pergunta tem até 300 caracteres e uma resposta até 2000.**

O texto fixo é guardado na pergunta no idioma em que a conta lê o Beyou quando ela é feita, então a pergunta continua igual depois que a pessoa troca de idioma.

Abrir a página nem sempre pede mais perguntas. A rota pode terminar numa chamada de modelo, então entrar e sair da página não pode virar uma chamada a cada vez. `secondBrainRefillIsDue` em `@beyou/state` decide para os dois apps: uma lista vazia sempre pede, uma lista cheia nunca pede, e qualquer coisa entre as duas pede no máximo uma vez por hora. O web guarda a hora do último pedido em sessionStorage e o celular guarda em memória.

Dois refills da mesma conta podem rodar juntos, já que a página pede ao montar. O `SecondBrainService` mantém num conjunto em memória as contas cujas perguntas de acompanhamento estão sendo escritas e pula o modelo para a segunda. Isso vale numa instância. Entre instâncias, `addFollowUps` lê de novo o espaço e a pausa sob o lock da conta, e o segundo conjunto a chegar é descartado.

### O que uma resposta faz

`SecondBrainService.answer` fecha a pergunta primeiro. `SecondBrainStore.answer` a lê sob uma trava de linha, marca como ANSWERED com o texto sem espaços nas pontas e faz commit antes de pedir qualquer coisa ao modelo. A resposta fica guardada aconteça o que acontecer depois, e o segundo envio de um clique duplo recebe `SECOND_BRAIN_QUESTION_CLOSED` (409) onde, sem isso, sairia um segundo conjunto de fatos.

Depois o serviço aprende com a resposta, pelo `prompts/secondBrainFacts.st`:

- **Uma resposta inicial vira linhas do perfil.** O modelo a divide nas coisas separadas que ela diz, no máximo três fatos, todos do tipo da pergunta, seja qual for o tipo que o modelo escreveu. Quando o modelo não está em uso, a chamada falha ou a divisão volta vazia, a própria resposta vira um fato, palavra por palavra, cortada em 200 caracteres quando é mais longa.
- **Uma resposta de check-in ou de acompanhamento acrescenta só o que ainda vai ser verdade depois.** O prompt pede o que continua valendo daqui a um mês, como um objetivo novo, uma preferência duradoura ou uma mudança de circunstância. Voltam no máximo dois fatos, com o modelo escolhendo o tipo de cada um, e o resultado mais comum é nenhum. A mesma chamada pode deixar anotações nas coisas da pessoa: o prompt lista os hábitos, as metas e as rotinas dela com os ids (20 de cada, 60 no máximo), e uma anotação que o modelo põe num id que não é da conta é descartada. Quando o modelo não está em uso, nada é aprendido.
- **Uma resposta sobre uma coisa nova vira anotações sobre ela** (modo NEW_THING). Com o modelo em uso ela é dividida em anotações sobre a coisa que a pergunta nomeou, e o que for duradouro sobre a pessoa vai para o perfil. Sem ele, ou quando a divisão volta vazia, a própria resposta vira uma anotação sobre aquela coisa, cortada em 500 caracteres. O campo `notes` da resposta lista o que foi gravado, e os apps recarregam a página quando ele não vem vazio, porque a memória é agrupada no servidor.

`SecondBrainStore.saveLearned` acrescenta esses fatos sob o lock da conta: quantos couberem no limite de 20, cada um cortado em 200 caracteres, com origem ANSWER, deixando de fora um fato que o perfil já tem palavra por palavra. A lista `learned` da resposta pode então vir vazia, e a página diz que a resposta foi anotada e que nada novo foi para o perfil. O prompt também manda o modelo nunca guardar senha, código, número de cartão ou de documento, nem qualquer outro segredo, mesmo quando a resposta traz um.

Pular fecha a pergunta sem resposta e sem chamada de modelo. A linha fica guardada, e é isso que impede uma pergunta inicial pulada de voltar.

### A passagem para o assistente

Uma resposta nunca muda uma rotina, um hábito ou uma tarefa. Depois de um check-in ou de uma pergunta de acompanhamento, o resultado oferece "Deixar o assistente agir". `canActOnAnswer` em `@beyou/state` guarda essa regra para os dois apps, e deixa de fora as perguntas iniciais e as de coisa nova, porque uma resposta inicial já é o próprio fato e uma resposta sobre uma coisa nova é um motivo, não um pedido. O botão abre o assistente numa conversa nova com um rascunho no campo de texto: a pergunta, a resposta e uma linha pedindo que o assistente faça o que a resposta pede nas rotinas, nos hábitos e nas tarefas da pessoa. Conversa nova de propósito, porque jogar o rascunho no chat que estava aberto por último misturaria dois assuntos. Nada é enviado até a pessoa enviar, e daí em diante é uma mensagem comum do assistente, tratada com as ferramentas e as regras dele. O texto do check-in diz para que serve a passagem: "Como está o seu dia? Me descreve e eu marco ou pulo os itens da sua rotina."

## O perfil

O perfil é uma lista de linhas curtas que a pessoa adiciona, edita e apaga na página, em três tipos: objetivos principais (`OBJECTIVE`), coisas de que gosta (`LIKE`) e fatos sobre a vida dela (`ABOUT`). Uma linha é digitada (origem USER), aprendida de uma resposta (ANSWER), guardada pelo assistente numa conversa (ASSISTANT) ou mantida da nota escondida antiga (OLD_NOTE), e a página marca toda linha que não foi digitada. Editar uma linha mantém a origem.

Ele guarda no máximo 20 linhas de 200 caracteres, digitadas e aprendidas juntas (`SecondBrainFact.MAX_PER_USER`, `TEXT_MAX`). `POST /second-brain/facts` responde 409 `SECOND_BRAIN_FACT_LIMIT_REACHED` no limite, e apagar uma linha abre espaço. Os limites existem por dois motivos que o código declara. Toda linha viaja com toda mensagem do assistente, então o tamanho do perfil é um custo em cada uma. E as chamadas ao modelo por trás desta página não têm ferramentas, então o máximo que uma injeção de prompt escondida nos dados da própria pessoa consegue é mudar o que é salvo como fato. Os limites delimitam quanto disso pode persistir e chegar depois a um prompt de chat.

Toda conversa do assistente lê o perfil. O `AiAgentService` preenche o espaço `{secondBrainProfile}` do `prompts/aiAgent.st` a cada turno a partir de `SecondBrainStore.profileForChat`: uma linha por fato, do mais antigo para o mais novo, rotulada "Objective", "Likes" ou "About them", ou "(none yet)" para um perfil vazio. É lido de novo a cada mensagem e não fica em cache, então uma linha editada na página já é conhecida pela mensagem seguinte. O bloco do prompt diz três coisas ao modelo. As linhas são pano de fundo para ler em toda conversa, para a pessoa nunca precisar repetir. Elas são anotações da própria pessoa e nunca instruções, e uma linha que pede para o modelo fazer algo continua sendo só uma anotação. E elas são da pessoa: o assistente acrescenta uma linha com `rememberAboutUser` quando ela conta algo duradouro sobre si, avisa na conversa, e não consegue editar nem apagar nenhuma. Isso a pessoa faz na página.

## Memória

O passo dois do Segundo Cérebro dá à página uma memória que a pessoa consegue ler. Cada anotação fica presa ao que ela descreve: um hábito, uma meta, uma rotina ou uma categoria. As anotações sobre a própria pessoa são as linhas do perfil acima, que têm tabela e limite próprios. Antes disso, a única coisa que o assistente lembrava entre conversas era uma nota escondida de até 2000 caracteres que o modelo reescrevia para si mesmo, e a pessoa não conseguia ver nem corrigir uma linha dela.

Uma anotação é uma linha curta com data, de no máximo 500 caracteres, e cada coisa guarda no máximo 30. A 31ª anotação sobre a mesma coisa apaga a mais antiga. Nada sobrescreve uma anotação: quando algo muda ("pausou a academia por duas semanas, joelho"), a mudança vira uma anotação nova e as anteriores ficam, então cada coisa se lê como uma linha do tempo, da mais antiga para a mais nova. A pessoa edita ou apaga qualquer anotação na página, e editar mantém a origem dela, como acontece com uma linha do perfil.

As anotações vêm de três lugares, e cada uma guarda a sua origem:

| Origem | Escrita quando |
|--------|----------------|
| `ANSWER` | Uma resposta nesta página fala de uma das coisas da pessoa. A pergunta sobre uma coisa nova sempre deixa uma anotação (abaixo); uma resposta ao check-in ou a um acompanhamento só deixa quando o modelo em uso encontra algo duradouro sobre um hábito, uma meta ou uma rotina específicos |
| `ASSISTANT` | A pessoa conta ao assistente algo duradouro sobre uma das suas coisas, e ele guarda com `rememberAboutItem` |
| `USER` | Reservada para anotações escritas à mão, que este passo ainda não oferece |

### A pergunta sobre uma coisa nova

Quando um hábito, uma meta ou uma rotina é criado, pela página dele ou pelo assistente, a página pergunta sobre ele: o que levou a isso e o que costuma atrapalhar. O gancho fica nos métodos de criação dos serviços, então as rotas REST e as ferramentas do assistente passam pelo mesmo lugar. A pergunta (`NEW_THING`) só é escrita para uma conta que já abriu o Segundo Cérebro, só enquanto houver menos de cinco perguntas abertas, e uma vez por coisa. O texto traz o nome da coisa e é guardado no idioma da conta, como as perguntas iniciais. Responder nunca depende do modelo: sem ele, ou antes da primeira visão geral, a própria resposta vira uma anotação sobre aquela coisa. Com o modelo em uso, a resposta é dividida em anotações sobre aquela coisa, e o que for duradouro sobre a pessoa vai para o perfil.

A passagem para o assistente não aparece depois de uma resposta sobre uma coisa nova. Ela é um motivo, e não um pedido.

### O que os acompanhamentos leem

As perguntas de acompanhamento e a visão geral leem uma seção NOTES no bloco de dados: as três anotações mais novas de até doze coisas, com preferência para os itens de rotina que escaparam ou se mantiveram na semana e para as metas à frente. É isso que permite a um acompanhamento perguntar "você começou a academia para ter mais energia, e faltou três vezes esta semana: parou, ou aconteceu outra coisa?". A resposta entra na linha do tempo da academia.

### Para onde vão as anotações quando a coisa some

As anotações não têm chave estrangeira para a coisa de que falam, porque o assunto pode estar em quatro tabelas diferentes. Os caminhos de exclusão movem as anotações na mesma transação:

- Um hábito ou uma meta apagados passam as suas anotações para a primeira categoria por nome. Elas guardam o nome da coisa e o que ela era (`former_type`), então a página mostra essas anotações dentro da categoria como "Academia, hábito apagado".
- Uma rotina apagada, uma categoria apagada, ou um hábito ou meta sem categoria nenhuma deixam as anotações como `LOOSE`. Elas continuam lá, no grupo da página para coisas sem categoria.
- Uma pergunta aberta sobre a coisa apagada é removida, porque não sobrou nada para perguntar.

### A página

O bloco de memória agrupa as coisas por categoria: um hábito ou uma meta na sua primeira categoria por nome, as anotações da própria categoria no topo do grupo, e as rotinas e as anotações soltas num grupo próprio no fim. O agrupamento é montado no servidor (`memory` em `GET /second-brain`), para os dois apps não agruparem as mesmas anotações de dois jeitos.

### A nota escondida antiga

O assistente não escreve mais `users.user_context`. O que uma conta já tinha ali aparece na página como linhas para revisar (`oldNote`): o texto é dividido em quebras de linha e fins de frase, e cada linha pode virar uma linha `ABOUT` do perfil (origem `OLD_NOTE`) ou ser apagada, uma por uma ou todas de uma vez. A divisão é uma única função no servidor, usada para listar as linhas e para achar a que o cliente devolve. Enquanto a pessoa não resolve, a nota antiga continua indo com cada mensagem ao assistente, como antes, para nada que o assistente sabia sumir sem a pessoa ver.

## As fontes

Seis chaves dizem o que o Segundo Cérebro pode ler. Cada linha abaixo é exatamente o que o `SecondBrainSignals` consulta quando a chave está ligada.

| Fonte | Padrão | O que é lido |
|-------|--------|--------------|
| ROUTINES | Ligada | Os snapshots de rotina dos 14 dias que terminam ontem. São os blocos de rotinas e de áreas, e num prompt as duas porcentagens semanais, os itens que escaparam ou se mantiveram e a fração feita por categoria. A pergunta diária de check-in também precisa desta chave |
| GOALS | Ligada | As metas abertas com data final, as três mais próximas dela: nome, progresso, dias até a data final e ritmo |
| MOOD | Ligada | O nível registrado em cada um desses 14 dias, entregue ao modelo como as médias desta semana e da anterior. Nenhum bloco da página o mostra |
| CALENDAR | Desligada | Tarefas abertas do calendário com data até 21 dias à frente, as atrasadas incluídas, 30 no máximo: o título e a quantos dias está |
| NOTEBOOK | Desligada | Os 20 tópicos de estudo atualizados mais recentemente: o título e quantos subtópicos estão concluídos |
| IDEAS | Desligada | O texto das 15 notas mais novas do quadro de Ideias. As arquivadas ficam de fora |

Digam o que disserem as chaves, os prompts da visão geral e das perguntas de acompanhamento também levam as linhas do perfil. O prompt da visão geral acrescenta as respostas recentes, e o das perguntas de acompanhamento acrescenta as últimas 15 perguntas com as respostas. O prompt de fatos não lê fonte nenhuma: recebe a pergunta, a resposta dela e o perfil. O Beyou não acrescenta nome de exibição nem e-mail a nenhum deles.

Uma fonte desligada não é consultada de jeito nenhum, então não chega à página nem a um prompt por nenhum caminho que passe pelo `SecondBrainSignals`. Uma fonte ligada sem nada dentro ganha uma linha dizendo isso, o que impede o modelo de inventar o que não recebeu, e uma desligada fica ausente. Os prompts avisam o modelo de que uma seção ausente foi desligada pela pessoa e não deve ser mencionada nem pedida.

Uma linha em `second_brain_sources` só existe para uma chave em que a pessoa mexeu. Linha ausente significa o padrão, que mora no enum `SecondBrainSource`, e `SecondBrainStore.effectiveSources` é o único lugar em que essa regra é lida. Nada foi preenchido para as contas que já existiam, e uma fonte adicionada depois também não precisa de linhas.

**As chaves valem só para o Segundo Cérebro.** O chat do assistente mantém as próprias ferramentas e o próprio acesso: com IDEAS desligada aqui, o assistente ainda lê o quadro de Ideias pelo `getQuickNotes` quando a pessoa pede. Desligar uma fonte muda o que a próxima leitura e o próximo prompt contêm. Não reescreve a visão geral nem as perguntas já guardadas.

**O texto do diário nunca é lido, em nenhuma configuração.** O humor vem pelo `MoodEntryRepository.findLevels`, uma consulta que seleciona a data e o nível e deixa a anotação de fora, então o texto não sai do banco. Não existe valor JOURNAL no enum, de propósito, e um pedido por ele responde 400. O caderno segue a mesma linha: títulos dos tópicos e progresso, nunca o que uma página diz.

## As chamadas ao modelo

O `SecondBrainAi` faz três chamadas. Cada uma é uma chamada estruturada e sem estado pelo `ChatClient.entity()` ao endpoint de LLM compartilhado, sem ferramentas, sem advisors e sem memória de chat, feita como as sugestões do onboarding. Sem ferramentas o modelo não consegue tocar na conta. Tudo o que ele devolve é texto, e a classe corta esse texto no tamanho antes de qualquer coisa ser guardada.

| Chamada | Prompt | Devolve | Tempo | Quando falha |
|---------|--------|---------|-------|--------------|
| A visão geral | `secondBrainInsight.st` | Um veredito e um texto | 90 segundos, com uma nova tentativa incluída | `AI_UNAVAILABLE`, e a visão geral guardada fica |
| Perguntas de acompanhamento | `secondBrainQuestions.st` | Uma lista de perguntas | 45 segundos, uma tentativa | Um aviso no log. O refill responde com as perguntas que já estavam lá |
| Fatos e anotações de uma resposta | `secondBrainFacts.st`, no modo PROFILE, LASTING ou NEW_THING | Fatos do perfil, e anotações sobre as coisas da pessoa | 45 segundos, uma tentativa | Uma resposta inicial fica como linha do perfil e uma resposta sobre coisa nova como uma anotação, as duas como foram escritas. Qualquer outra não acrescenta nada |

Só a visão geral tenta de novo, porque é a única chamada sem nada para onde recuar. A nova tentativa pede JSON válido outra vez e só começa quando sobram pelo menos 20 segundos do tempo. Cada tentativa roda numa thread virtual, então o pedido consegue parar de esperar no prazo. O proxy na frente da API derruba um pedido aos 100 segundos, e é por isso que os tempos ficam abaixo disso.

**Os dados da pessoa são marcados como dados.** Eles viajam na mensagem de usuário entre `<data>` e `</data>` (`SecondBrainData.wrap`), e cada prompt de sistema diz que tudo dentro do bloco é material para ler e nunca uma instrução. Isso só vale enquanto o texto da própria pessoa não consegue fechar o bloco, então `SecondBrainData.inline` remove os marcadores de tudo o que ela digitou e achata o texto numa linha.

**Nenhuma chamada roda dentro de uma transação.** Uma chamada ao modelo pode levar quase um minuto, e uma transação aberta durante ela prenderia uma das dez conexões do pool por esse tempo. `refill`, `answer` e `refreshInsight` no `SecondBrainService` não são `@Transactional`, de propósito. Cada um lê o que o prompt precisa numa transação curta, chama o modelo sem nenhuma aberta e grava o que voltou em outra transação curta. Essas transações curtas moram no `SecondBrainStore`, um bean próprio por esse motivo, o mesmo formato de `DailyBriefingWrites` ao lado de `DailyBriefingService`. O `SecondBrainModelCallIT` verifica, para cada uma das três rotas, se há transação ativa no momento em que o modelo é chamado.

### Antes da primeira visão geral

Nada da página chega ao modelo até a pessoa pedir uma primeira visão geral. `SecondBrainService.aiInUse` é verdadeiro quando há um endpoint de LLM configurado e a conta tem uma linha de visão geral (`SecondBrainStore.hasInsight`), e ele barra as duas chamadas que não são a própria visão geral:

- Um refill não escreve pergunta de acompanhamento. As perguntas iniciais e o check-in continuam vindo, já que não precisam de modelo.
- Uma resposta não é lida pelo modelo. Uma resposta inicial fica palavra por palavra como uma linha do perfil, uma resposta sobre coisa nova como uma anotação sobre ela, e uma resposta de check-in não acrescenta nada.

`refreshInsight` não é barrado, porque ele é o próprio aceite: o pedido sai de uma tela que diz que o texto vai para um provedor de IA externo. A regra mora no servidor para que um cliente não ligue o lado de IA por acidente e um cliente novo não esqueça de perguntar. A barreira é a existência da linha de visão geral, e nenhuma rota apaga essa linha, então o lado de IA continua ligado para a conta depois que começou. As chaves continuam decidindo o que ele lê.

A barreira cobre as chamadas ao modelo desta página. O perfil também faz parte do prompt de sistema do assistente, então quem usa o assistente envia as linhas do perfil ao provedor a cada mensagem, tenha ou não pedido uma visão geral.

### Com o endpoint de IA sem configurar

`LlmProperties.isConfigured` é falso enquanto qualquer um de `LLM_BASE_URL`, `LLM_API_KEY` e `LLM_MODEL` não está definido. A página então informa `aiAvailable` falso, e os refills e as respostas se comportam como antes de uma primeira visão geral. Uma conta sem visão geral lê que a IA está desligada neste servidor no lugar onde ficaria o botão, e `POST /second-brain/insight/refresh` responde 400 `AI_UNAVAILABLE`. Todos os outros blocos continuam funcionando.

## Rate limiting

As três rotas que podem chamar um modelo gastam um balde próprio no `RateLimitFilter`: `POST /second-brain/questions/refill`, `POST /second-brain/questions/{id}/answer` e `POST /second-brain/insight/refresh`, 30 chamadas por usuário por hora sob a chave `second-brain`. Ele tem o formato da cota do assistente e uma chave separada, então a página nunca gasta o orçamento do assistente. Uma chave só cobre as três rotas para que alternar entre elas não compre um segundo orçamento.

As rotas são nomeadas uma a uma no filtro. A leitura da página fica na faixa de leitura genérica (60 por minuto), e as chaves, os fatos, as anotações, a nota antiga e pular ficam na faixa de escrita genérica (30 por minuto), já que nenhum deles chega a um modelo. A reescrita diária é uma dessas três rotas, então sai deste balde e não do balde do assistente.

## As cinco tabelas

As quatro chegaram na V45, cada uma pertencendo a uma conta e cada uma com `ON DELETE CASCADE` a partir de `users`.

| Tabela | Uma linha por | Colunas que vale conhecer |
|--------|---------------|---------------------------|
| `second_brain_sources` | Conta e fonte em que a pessoa mexeu. Chave primária `(user_id, source)` | `enabled`, `updated_at`. Gravada por um único `INSERT ... ON CONFLICT`, então dois toques numa chave ainda sem linha não colidem |
| `second_brain_facts` | Linha do perfil | `kind`, `text` (varchar 200), `origin`, `created_at`, `updated_at`. Índice em `user_id`, usado por toda leitura e pela contagem por trás do limite |
| `second_brain_questions` | Pergunta já feita, aberta ou fechada | `kind`, `text` (varchar 300), `status` (padrão OPEN), `answer` (varchar 2000, nulo até ser respondida), `created_at`, `answered_at`, e desde a V47 `subject_type` e `subject_id`, a coisa de que trata uma pergunta de coisa nova. Índice em `(user_id, status)` |
| `second_brain_insights` | Conta. Chave primária `user_id` | `text` (varchar 1200), `verdict`, `generated_at`. Substituída a cada reescrita |
| `second_brain_notes` | Anotação sobre uma coisa | `subject_type`, `subject_id` (nulo só para LOOSE), `subject_name`, `former_type`, `text` (varchar 500), `origin`, `created_at`, `updated_at`. Sem chave estrangeira para a coisa, veja Memória. Índice em `(user_id, subject_type, subject_id)` |

Todo enum é um `varchar` com uma CHECK que o espelha: as seis fontes, os três tipos de fato, as quatro origens de fato, os seis tipos de pergunta, os três status, os três vereditos, os cinco tipos de assunto e as três origens de anotação. Adicionar um valor em Java sem adicioná-lo à constraint faz toda gravação desse valor falhar. As perguntas fechadas ficam guardadas de propósito. A regra das perguntas iniciais ("nunca perguntar isto duas vezes") e o prompt de acompanhamento ("não se repita") leem essas perguntas.

O diário não tem coluna aqui e nunca é copiado para cá, e os níveis de humor são lidos na hora do pedido.

## Onde cada regra mora

| Regra | Mora em |
|-------|---------|
| Nenhuma transação aberta durante uma chamada ao modelo, e nenhum modelo antes da primeira visão geral | `SecondBrainService` |
| Chave ausente é o padrão | `SecondBrainStore.effectiveSources`, `SecondBrainSource.enabledByDefault` |
| Um fato ou uma pergunta só é alcançável pelo dono: toda busca leva o id do dono, então a linha de outra pessoa e uma que não existe recebem o mesmo 404 | `SecondBrainStore`, `SecondBrainFactRepository.findByIdAndUserId`, `SecondBrainQuestionRepository.lockByIdAndUserId` |
| No máximo 20 fatos e 5 perguntas abertas, contados sob um advisory lock de transação com chave na conta, porque uma conta nova não tem linha para travar | `SecondBrainStore`, `SecondBrainFactRepository.lockProfile` |
| Uma pergunta fecha uma vez: responder e pular a leem sob uma trava de linha | `SecondBrainStore.answer`, `SecondBrainStore.skip` |
| Quando uma visão geral está vencida | `SecondBrainStore.insight` |
| Uma anotação só é alcançável pelo dono, guarda 500 caracteres, e uma coisa fica com 30, saindo primeiro a mais antiga | `SecondBrainNotes` |
| Como a página agrupa a memória | `SecondBrainNotes.memory` |
| Quando se pergunta sobre uma coisa nova, e para onde vão as anotações quando a coisa é apagada, na própria transação da exclusão | `SecondBrainHooks`, chamado por `HabitService`, `GoalService`, `DiaryRoutineService` e `CategoryService` |
| Como a nota escondida antiga se divide em linhas | `OldNote` |
| O que cada fonte lê, e que uma fonte desligada não é consultada | `SecondBrainSignals` |
| A ordenação das áreas, e os números de rotina compartilhados com o resumo | `SecondBrainSignals.areas`, `DailyBriefingFactsBuilder.buildSignals` |
| As três chamadas ao modelo, os tempos de cada uma e os cortes de tamanho | `SecondBrainAi` |
| O bloco de dados, e manter o texto da pessoa dentro dele | `SecondBrainData` |
| O veredito, e como o texto do modelo é lido | `SecondBrainVerdict` |
| O perfil como o assistente o lê | `SecondBrainStore.profileForChat`, `SecondBrainSignals.profileLines` |
| O orçamento por hora compartilhado pelas três rotas que chamam modelo | `RateLimitFilter.isSecondBrainLlmPath` |
| Quando um cliente pede mais perguntas, quais respostas podem ir para o assistente, quais áreas ganham etiqueta, como um veredito é desenhado, como se rotula uma origem e uma coisa apagada | `secondBrainRules.ts` em `@beyou/state` |
| Como o ritmo de uma meta é escrito, para esta página e para o resumo | `goalPaceMessage` em `@beyou/state` |

## Os clientes

Os tipos estão em `@beyou/types`, as catorze chamadas em `@beyou/api`, e os julgamentos que os dois apps fazem em `@beyou/state`. O Segundo Cérebro não tem slice do Redux. Nada fora da página mostra os dados dele, então cada app guarda a resposta no estado da própria tela e aplica nela as próprias escritas: uma resposta tira a pergunta da lista e acrescenta o que foi aprendido, e a troca de uma chave é seguida de uma recarga silenciosa, porque uma fonte pode alimentar mais de um bloco.

No web a página é `/second-brain`, uma rota lazy dentro do shell protegido. Ela fica em segundo lugar na sidebar, logo abaixo do dashboard, e em primeiro na folha de Mais da barra inferior numa tela do tamanho de um celular. O prompt do assistente lista `/second-brain` entre as páginas do app, então o assistente pode criar links para ela.

## Privacidade e direitos sobre os dados

O que chega ao provedor de IA, e quando, está descrito acima: nada desta página antes da primeira visão geral, depois as fontes ligadas, as linhas do perfil, as anotações e as respostas. O [tópico do agente de IA](/architecture/ai-agent) lista o Segundo Cérebro entre as funcionalidades que enviam texto ao provedor.

`GET /user/export` traz uma seção `secondBrain` com cinco chaves. `notes` tem toda anotação com o tipo e o nome da coisa, `former_type`, origem e datas. `facts` tem toda linha do perfil com tipo, origem e datas. `questions` tem toda pergunta com a resposta, as fechadas incluídas, da mais antiga para a mais nova, já que uma resposta é algo que a pessoa escreveu. `insight` é a visão geral guardada com o veredito e a hora em que foi escrita, ou nulo para uma conta que nunca teve uma. `sources` lista as seis chaves com o valor em vigor, então o arquivo diz o que o Segundo Cérebro podia ler mesmo onde a pessoa nunca mexeu numa chave. São cinco consultas, seja qual for o tamanho da conta.

Limpar as conversas do assistente deixa o perfil, as anotações e a nota antiga onde estão: elas se apagam nesta página. Apagar a conta remove as cinco tabelas pelo cascade de `users`. Os testes de exclusão colocam uma linha em cada uma e esperam não sobrar nenhuma.

No dispositivo, nada da página chega ao Redux, então nada dela é persistido pelo redux-persist. O web guarda um único horário em sessionStorage, o do último pedido de mais perguntas.

## Testes

No backend, o `SecondBrainServiceIT` roda as regras do store contra o Postgres: os padrões, os limites, as regras das perguntas iniciais e do check-in, a posse e o vencimento pelo dia da própria conta. O `SecondBrainModelCallIT` verifica que nenhuma transação está aberta durante uma chamada ao modelo e que nada chega ao modelo antes da primeira visão geral. O `SecondBrainAiTest` cobre os cortes de tamanho, a nova tentativa e o bloco de dados, o `SecondBrainControllerTest` os códigos de status, e o `RateLimitFilterTest` o orçamento compartilhado pelas três rotas que chamam modelo. O `AgentSystemPromptTest` renderiza o prompt do assistente com um perfil dentro. O `SecondBrainMemoryIT` cobre as anotações: o agrupamento, o corte em 30, a posse, as regras de coisa nova pelos caminhos reais de criação, as mudanças na exclusão e a nota antiga. O `MemoryToolsIT` cobre as três ferramentas de memória do assistente, e o `SecondBrainMemoryTextTest` a divisão da nota antiga. Os testes de exportação e de exclusão (`UserExportCompletenessIntegrationTest`, `UserExportQueryCountTest`, `AccountDeletionIntegrationTest`) cobrem as cinco tabelas.

Nos clientes, `secondBrainRules.test.ts` e `briefingRules.test.ts` em `@beyou/state` cobrem as regras compartilhadas, o texto das metas lentas entre elas, e a página web tem o próprio arquivo de teste. O `second-brain.spec.ts` na suíte e2e exercita a API e a página, a posse e a ordem na largura de celular incluídas, e o `second-brain-memory.spec.ts` as anotações: a pergunta sobre coisa nova, a posse, a mudança na exclusão, que limpar as conversas as mantém, e o bloco de memória.
