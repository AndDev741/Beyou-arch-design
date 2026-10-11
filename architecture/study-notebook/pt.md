---
title: "Caderno de Estudos"
summary: "Tópicos organizados em quadros de roadmap, uma página de anotações atrás de cada nó, uma IA que responde a partir das anotações e fontes da própria pessoa com citações, flashcards em revisão espaçada e ciclos de estudo que fazem os tópicos girarem numa fila de blocos de tempo e de objetivo. O modelo, as regras e a classe dona de cada uma."
---

O caderno de estudos é onde alguém organiza um assunto que está aprendendo. A pessoa abre um tópico, digamos "Engenharia de Software", e monta um quadro de roadmap para ele. Cada nó desse quadro abre uma página de anotações, e essa página pode ter um quadro próprio, então "Estruturas de Dados" se divide em árvores e heaps sem sair do caderno. PDFs, páginas da web e texto colado entram nas páginas como fontes, e uma IA de estudo responde a partir delas com citações. Os flashcards escritos numa página voltam num cronograma. Um ciclo de estudos põe vários tópicos numa fila de blocos e diz à pessoa o que estudar hoje, e por quanto tempo.

Este documento cobre o modelo, as regras e a classe onde cada regra mora, e depois as escolhas que moldaram os dois clientes. As rotas estão na API do Controller do Caderno de Estudos.

## Tudo é uma página

```mermaid
flowchart TD
  T["📚 Tópico<br/>página · kind TOPIC · sem pai"]
  T -->|"quadro"| N1["nó → Estruturas de Dados"]
  T -->|"quadro"| N2["nó → Algoritmos"]
  N1 --> P1["📄 Estruturas de Dados<br/>anotações + quadro próprio"]
  P1 -->|"quadro"| N3["nó → Árvores"]
  P1 -->|"quadro"| N4["nó vinculado → Grafos<br/>(casa: outro tópico)"]
  P1 --- C["🃏 Flashcards"]
  P1 --- S["📎 Fontes<br/>PDF · link · texto"]
  P1 --- R["💬 Sala de estudo<br/>chat · visão geral · quiz"]
```

Um tópico é uma página com `kind = TOPIC` e sem pai. Toda outra página tem `kind = PAGE` e exatamente uma casa: `parent_id` a coloca na árvore e `topic_id` diz a qual tópico ela pertence, gravado na própria linha para que um tópico inteiro venha numa leitura indexada. Uma constraint CHECK na V34 recusa qualquer linha que fuja desse formato, então uma página sem tópico não existe nem quando um service a monta errado.

As anotações são um documento BlockNote guardado como string JSON em `content`. Ao lado fica `content_text`, o texto puro que o `BlockText` extrai no servidor a cada salvamento. A IA de estudo lê esse texto e a exportação da conta o leva. Os clientes nunca o enviam, porque um cliente poderia mandar qualquer texto junto do documento, e derivá-lo no servidor mantém os dois de acordo. O `BlockText` percorre o JSON pela estrutura e ignora os tipos de bloco, então um bloco que o editor ganhe depois continua sendo lido.

Uma página ou tópico pode ter um `icon`, um id do `@beyou/icons` escolhido no cabeçalho da página com o mesmo seletor que categorias e hábitos usam. Ele aparece onde a página aparece: a árvore, os nós do quadro, os cartões da página inicial e a tela da página no mobile, com o padrão de tópico ou de página quando não há um. Uma renomeação ou um ícone novo é gravado em todos esses lugares de uma vez pelo `applyPageDetails` do `@beyou/state`, a mesma propagação que o `applyStatuses` faz para o status.

Uma linha de página tem mais de um escritor, e cada um a carrega num momento diferente. O autosave é um. Um status que muda porque um nó abaixo terminou é outro. `NotebookPage` é `@DynamicUpdate`, então cada UPDATE leva só as colunas que aquela requisição mudou, e uma mudança de status não regrava mais um documento que carregou antes do autosave commitar. Abrir uma página grava `last_opened_at` por uma query de UPDATE que nunca suja a linha, já que a tela da página repete essa leitura enquanto a pessoa digita. Foi a suíte e2e que achou esse caso, e o `NotebookConcurrentWritesIT` reproduz as intercalações.

## Um quadro por página

O quadro pertence à página que o mostra. Suas linhas em `notebook_board_nodes` e `notebook_board_edges` têm `board_page_id` como chave, então uma página tem no máximo um quadro. O bloco "Quadro de roteiro" do editor (`roadmapBoard`, a constante `MarkdownBlocks.BOARD_BLOCK_TYPE`) só diz onde desenhá-lo na página. Apagar o bloco esconde o quadro e não apaga nenhum nó.

Um nó é um destes:

- Um nó PAGE criado por título. O servidor cria uma página filha da página do quadro e o nó a abre. É o caso comum.
- Um nó PAGE criado com `linkPageId`, que abre uma página que mora em outro lugar, quase sempre em outro tópico. Essa página continua com sua única casa e seu único status, e o nó vira uma segunda porta de entrada. Por isso o nó tem sua própria coluna `page_id` em vez de reaproveitar a árvore.
- Uma SECTION, uma faixa com rótulo, largura e altura que agrupa os nós desenhados por cima dela.

O `NotebookBoardService` recusa dois tipos de vínculo. Uma página aparece uma vez em cada quadro, garantido por `UNIQUE (board_page_id, page_id)` e devolvido como `NOTEBOOK_NODE_DUPLICATE`. E uma página que já alcança a página do quadro descendo pelos quadros colocaria essa página no próprio quadro, o que `ProgressGraph.reaches` pega como `NOTEBOOK_BOARD_CYCLE`. Toda caminhada no `ProgressGraph` guarda um conjunto de visitados de qualquer forma, então um ciclo que entrasse por outro caminho não trava a requisição.

As arestas ordenam os nós e desenham o caminho sugerido. Progresso e status nunca as leem, e elas não bloqueiam nada.

Uma página criada pelo "Adicionar página" da árvore mora sob o pai sem nó nenhum. Remover um nó deixa a página na árvore, fora do quadro. O quadro pode apagar a página junto (`deletePage`), e o servidor só aceita isso para uma página cuja casa é a página daquele quadro, então remover um nó vinculado nunca apaga a página no outro tópico.

## Status e progresso

São dois números com vidas diferentes.

O progresso é calculado a cada leitura pelo `ProgressGraph`, montado com duas queries por usuário: todas as páginas e todos os nós PAGE. Ele conta folhas. Uma página sem nós no quadro conta como uma, e uma página com quadro conta como a soma do que está nele, por todos os quadros aninhados e por toda página vinculada. "Engenharia de Software 11 de 56" quer dizer 56 coisas sem nada mais detalhado abaixo delas.

O status é gravado, e o `NotebookProgressService` é a única classe que o escreve. O XP é pago quando uma página chega a DONE pela primeira vez, e pagar numa transição exige um "antes". Um status calculado a cada leitura nunca saberia separar o momento em que a página terminou da centésima vez em que alguém olhou uma página terminada.

1. Uma folha tem o status que alguém deu a ela.
2. Uma página com nós segue esses nós pelo `ProgressGraph.derivedStatus`. Todos os nós concluídos deixam a página DONE, qualquer coisa concluída ou começada a deixa STUDYING, e fora isso ela fica TO_STUDY. Um status escolhido à mão numa página assim liga `status_manual`, e os nós param de mexer nela. `StatusChoice.AUTO` desliga a flag e recalcula na hora. A flag só importa onde os nós poderiam passar por cima, então numa folha ela fica desligada, e um nó adicionado depois assume.
3. Toda mudança sobe por todos os quadros que mostram a página (`ProgressGraph.boardsShowing`) e recalcula cada página que segue seus nós, até nada mais se mover. Uma página vinculada em dois tópicos move os dois. Uma página com status manual interrompe a subida naquele galho.
4. As edições de quadro passam pelo mesmo service (`boardChanged`), já que adicionar ou remover um nó muda o que os nós de uma página dizem. Uma página concluída que ganha um nó que ninguém começou volta para STUDYING.

Toda resposta que pode mexer num status traz `changed`, a lista de toda página que se moveu. No cliente, `applyStatuses` (em `@beyou/state`) grava isso na página, em todo nó de quadro que a abre, em toda linha da árvore e nas prévias de tópico da home. Um nó fica verde no quadro no instante em que sua página é marcada como concluída, sem refetch.

## XP

O `NotebookRewards` guarda todos os valores que o caderno paga, para que os três se comparem de relance:

| Evento | XP | O que impede pagar duas vezes |
|--------|----|-------------------------------|
| Uma página chega a DONE pela primeira vez | 15 (`PAGE_DONE_XP`) | `notebook_pages.done_xp_at`, gravado uma vez e nunca limpo |
| Um quiz é aprovado pela primeira vez, `score * 10 >= total * 7` | 20 (`QUIZ_PASSED_XP`) | `notebook_study_outputs.passed_at` |
| Um card revisado | 1 (`CARD_REVIEW_XP`), no máximo 30 por dia (`DAILY_REVIEW_XP_CAP`) | `notebook_card_reviews.xp_paid`, coletado por `POST /notebook/reviews/finish` |

O XP vai para a pessoa e para a categoria do tópico da página via `XpCalculatorService.addXpToUserAndCategoriesAndPersist`, que grava o ledger diário de XP como qualquer outro pagamento. Um tópico sem categoria paga só a pessoa. Uma única mudança de status pode concluir páginas em dois tópicos com categorias diferentes, então um pagamento é um `Payroll` agrupado por categoria, com um único repaint (`RefreshUiDTO`) para tudo.

Marcar uma página como concluída, desmarcar e marcar de novo paga uma vez só. Senão o status vira um botão de XP, e o `notebook-rules.spec.ts` segura essa regra.

As revisões pagam quando a sessão termina. Quem passa pelos cards vê um +XP no fim, sem um contador subindo a cada toque. O teto diário impede que um baralho de mil cards de uma palavra vire fazenda de XP, e revisar além dele continua movendo o cronograma. As revisões acima do teto são marcadas como pagas sem pagar nada: só poderiam ser pagas hoje, e hoje já bateu o teto.

## Flashcards

Os cards ficam numa página, em `notebook_cards`. O `SpacedRepetition` é o SM-2 do jeito que o Anki adaptou, em dias inteiros, sem relógio e sem repositório, para que cada regra seja um teste unitário:

- AGAIN quer dizer esqueci. O ease cai 0,2, as repetições zeram e o card vence de novo hoje. O cliente o leva para o fim da sessão.
- HARD baixa o ease em 0,15 e aumenta o intervalo em 1,2 vez.
- GOOD dá um dia, depois três, depois o intervalo vezes o ease.
- EASY sobe o ease em 0,15 e salta mais longe do que GOOD saltaria.

O ease nunca fica abaixo de 1,3 e nenhum intervalo passa de 365 dias. `due_on` é uma data no fuso do dono, resolvida pelo `UserDateResolver`, então "vence amanhã" é o amanhã de quem lê. A fila de revisão manda cada card com `intervals`, o resultado de cada botão, e os rótulos embaixo dos botões vêm da mesma função que agenda.

O `ReviewStreak` conta dias seguidos com alguma revisão, e a sequência pode terminar ontem. Às nove da manhã ninguém revisou ainda, e uma sequência que mostra 0 até o primeiro card castiga a pessoa por estar acordada.

## Fontes

Uma fonte é anexada a uma página, e uma página lê as próprias fontes e as de cada ancestral subindo a árvore. Um livro-texto no tópico responde perguntas em todos os nós dele. Um artigo num nó fica naquele nó, e páginas irmãs nunca veem as fontes umas das outras. O escopo segue só a árvore: uma página vinculada lê as fontes da sua própria casa.

Adicionar uma fonte grava a linha como PENDING e responde 202. O `SourceIngestionService` a lê num pool de duas threads, `notebook-ingest`, e o job só começa depois que a transação da requisição commita, para o pool nunca procurar uma linha que ainda não existe. A linha passa por READING com um percentual e termina READY ou FAILED com um `error_key` que os clientes traduzem. As gravações de status passam pelo `SourceWrites`, cada uma na própria transação, e uma linha apagada no meio da leitura é simplesmente pulada. No boot, as linhas que um restart pegou pela metade são marcadas FAILED com `NOTEBOOK_SOURCE_INTERRUPTED`, porque os bytes do PDF morreram com o processo.

- **PDF.** Até 15MB, reconhecido pelo número mágico `%PDF-`, seja qual for o tipo que o cliente declara. O `PdfTextExtractor` (PDFBox) o lê em memória, página por página, até 800 páginas, porque uma citação nomeia a página. Os bytes nunca são gravados em lugar nenhum. Quando o job termina, o texto é a única cópia que sobra, e não há arquivo para limpar quando uma conta é apagada. Um PDF sem camada de texto é recusado como ilegível, porque uma fonte que responde nada a toda pergunta é pior do que um erro com o qual a pessoa consegue agir.
- **Link.** O `LinkFetcher` é o único lugar em que o servidor busca uma URL digitada por um usuário, então é também onde o SSRF é recusado. As regras estão no tópico de segurança. O HTML volta como texto legível via jsoup, lendo no máximo 3MB. Um PDF atrás de um link segue o caminho do PDF, com o teto de 15MB.
- **Texto.** Colado, até 200.000 caracteres.

O `TextChunker` corta o texto em trechos de cerca de 1000 caracteres, seguindo os parágrafos e quebrando um parágrafo maior que 1600 em finais de frase, com no máximo 3000 trechos por fonte. Os trechos nunca se sobrepõem. Uma citação abre um trecho como excerto, e um excerto que começa no meio da última frase do trecho anterior parece bug. A busca devolve vários trechos por resposta, então uma ideia dividida entre dois chega inteira.

O `SourceChunkStore` lê e grava `notebook_source_chunks` com SQL puro via `JdbcTemplate`. A tabela tem um `tsvector` gerado e um `real[]`, dois tipos que o Hibernate precisaria aprender a validar e que nada aqui usa como objeto. Uma página guarda no máximo 20 fontes próprias (`NOTEBOOK_SOURCE_LIMIT_REACHED`), além das que herda.

### Encontrar fontes para mim

A pessoa descreve o que as fontes devem cobrir, e o `SourceDiscoveryService` faz uma busca na web com o tópico da página, o título dela e o objetivo da sala como contexto. Dois provedores ficam por trás do `WebSearchClient`, e as chaves escolhem um (`DiscoveryProperties`): Tavily quando `TAVILY_API_KEY` está definida, senão Gemini com Google Search quando `NOTEBOOK_DISCOVERY_GEMINI_API_KEY` está, senão nenhum e a sala de estudo esconde o botão. A busca do Gemini precisa de um projeto com faturamento; numa chave gratuita toda chamada com busca responde 429, e por isso a chave dela é própria e a `GEMINI_API_KEY` da cadeia do chat nunca é reaproveitada: um fallback mostraria um botão que falha toda vez. Os resultados do Gemini são os grounding chunks da resposta, páginas que o Google devolveu, nunca URLs escritas no texto do modelo, porque um modelo pode inventar uma URL.

Cada resultado é aberto antes de ser oferecido, pelo `LinkFetcher.resolve`: os redirecionamentos são seguidos até a página real, com as recusas de SSRF em cada salto, e só se lê HTML suficiente para o título. Um resultado do Gemini é um redirecionamento do Google, então é aqui também que ele vira o endereço real. Resultados que não abrem, apontam para algo privado, já são fontes da página (desligadas ou ainda em leitura incluídas) ou repetem outro resultado são descartados e contados. Eles são abertos em paralelo, e o que ainda estiver carregando depois de 20 segundos é descartado. Nada é guardado: a pessoa marca o que quer manter, e o cliente adiciona cada um como fonte de link, lida como qualquer link colado. A busca gasta a cota `notebook-ai` e cada fonte adicionada a `notebook-source`.

## Busca de trechos, e por que não há pgvector

O `NotebookRetriever` encontra os trechos que melhor respondem uma pergunta, nesta ordem:

1. Embeddings, quando há um modelo configurado e as fontes foram embedadas por esse mesmo modelo. A pergunta é embedada e pontuada por cosseno contra os vetores guardados dessas fontes, na JVM, até 6000 vetores. No tamanho das leituras de uma pessoa, isso leva milissegundos.
2. Busca full-text no `tsvector` gerado, casando qualquer palavra da pergunta. Uma pergunta em linguagem natural quase nunca tem todas as palavras num só trecho, então os termos vão com OR. A configuração de busca é `simple`, porque o caderno é bilíngue e um stemmer de uma língua estraga a outra.
3. Quando nada casa, os trechos iniciais de cada fonte, para que "do que se trata isto?" ainda tenha onde se apoiar.

Existe UM modelo de embedding, definido por `notebook.embedding.*` (`EmbeddingProperties`). O padrão é o `mistral-embed` da Mistral, com a mesma `MISTRAL_API_KEY` que a cadeia de chat usa, e qualquer endpoint `/embeddings` compatível com OpenAI serve via `NOTEBOOK_EMBEDDING_BASE_URL`, `NOTEBOOK_EMBEDDING_API_KEY` e `NOTEBOOK_EMBEDDING_MODEL`. Sem chave nenhuma, os embeddings ficam desligados. Vetores de dois modelos vivem em espaços diferentes, então uma cadeia de fallback tornaria inútil todo vetor guardado para a pergunta feita. Cada trecho registra `embedding_model`, e o retriever só compara uma pergunta com trechos embedados pelo modelo configurado agora. Quando o provedor falha durante a ingestão, a fonte termina READY do mesmo jeito: os trechos estão gravados e dá para buscá-los por full-text. Os perfis de teste e e2e fixam a chave vazia, então o CI e a stack e2e sempre rodam na busca full-text.

Os embeddings e a busca na web são as únicas chamadas de saída do caderno que não passam pela cadeia de fallback de LLM, então `ai.llm-chain.blocked` não consegue vetá-las. A política de privacidade cita a Mistral para a primeira e o Tavily ou o Gemini para a segunda; um endpoint novo para qualquer uma precisa entrar lá antes.

O pgvector chegou a ser planejado e foi descartado. Ele exige trocar a imagem do banco em produção, em três arquivos compose, em todo serviço de CI e no Testcontainers. Um backend mergeado antes dessa troca quebraria na hora do Flyway e derrubaria a API junto. As fontes de uma pessoa cabem com folga numa passada de cosseno na JVM, e migrar para pgvector depois é uma troca de tipo de coluna e uma query.

## A IA de estudo

Toda chamada ao modelo passa pelo `NotebookLlm`: a mesma cadeia de fallback do assistente, uma resposta estruturada via `ChatClient.entity()`, uma nova tentativa pedindo JSON válido, e depois `AI_UNAVAILABLE`. Sem tools, sem streaming e sem memória. Uma chamada do caderno é uma pergunta com o contexto anexado, montado por quem chama e enviado inteiro a cada vez, que é o contrato das sugestões do onboarding.

A chamada inteira, nova tentativa incluída, tem 90 segundos (`NotebookLlm.BUDGET`). O Cloudflare derruba um pedido à API aos 100 segundos, e o cliente web desiste no mesmo ponto. Sem esse limite, um provedor travado podia segurar um pedido por minutos, e a pessoa via um erro de um trabalho que o servidor terminava depois, ou salvava duas vezes quando tentava de novo. Então o servidor para primeiro e responde `AI_UNAVAILABLE`. A nova tentativa só começa se sobrarem pelo menos 20 segundos. Cada tentativa roda numa virtual thread, para o pedido poder parar de esperar no prazo; a chamada HTTP abandonada termina no próprio read timeout.

Nenhuma transação de banco fica aberta enquanto o modelo responde. Uma chamada que pode levar 90 segundos seguraria uma das dez conexões do pool o tempo todo, e dez respostas lentas ao mesmo tempo deixariam o resto do app esperando por uma. Então todo caminho que chama o modelo lê o que o prompt precisa numa transação curta, chama o modelo sem nenhuma aberta e salva o que voltou numa segunda transação curta, pelo `NotebookTransactions`. O salvamento carrega a página de novo, porque a da leitura já está desanexada, e uma página apagada enquanto o modelo pensava recusa o salvamento em vez de juntar linhas órfãs. O `NotebookModelCallTransactionIT` pergunta a cada um desses caminhos se há uma transação aberta no momento da chamada. Um limite que vale saber: o open-session-in-view do Spring está ligado, então num pedido HTTP a sessão do Hibernate, e a conexão que ela pegou, vive até a resposta ser escrita. A divisão libera a conexão nas chamadas de ferramenta do assistente e na thread de fundo do rascunho de roteiro. Nas rotas HTTP isso depende de desligar essa configuração, o que é outra mudança.

Enquanto uma chamada roda, toda tela do caderno que espera por ela mostra o tempo decorrido e, depois de 30 segundos, um aviso de que ainda está em andamento e quanto pode levar. O rascunho do roteiro também mostra linhas de esqueleto onde os nós vão aparecer, e pode ser fechado a qualquer momento sem perder nada, porque o rascunho fica guardado (abaixo).

O prompt de sistema é `prompts/notebookTutor.st`, e cada mensagem começa com um modo:

| Modo | Usado por | O modelo pode |
|------|-----------|---------------|
| GROUNDED | Chat, resumo, guia de estudo, visão geral, quiz, flashcards da IA, "sugerir nós" a partir das fontes | Responder só a partir dos trechos numerados, citando `[n]`. Quando eles não cobrem a pergunta, diz isso numa frase e para |
| SUPPORTED | "Explicar o bloco acima" | Explicar com o próprio conhecimento e citar os trechos que o sustentam |
| PLANNING | Rascunho de roadmap, "sugerir nós" sem fontes | Usar conhecimento geral. Não há trechos |

O `StudyContextBuilder` numera os trechos e é o dono desses números. As anotações da pessoa vêm primeiro, lidas frescas da página e dos ancestrais, e depois os trechos que o `NotebookRetriever` escolheu. A resposta que concorda com o que a pessoa escreveu é a que fixa. Qualquer número que o modelo cite fora da lista é descartado, e marcadores `[n]` mortos saem do texto, porque uma citação que o leitor não consegue abrir é pior do que nenhuma. Uma chamada GROUNDED numa página sem anotações e sem fontes legíveis levanta `NOTEBOOK_NOTHING_TO_STUDY` antes de chamar o modelo.

O `NotebookStudyService` cuida da sala de estudo. As mensagens do chat ficam em `notebook_chat_messages`, e as últimas 6 vão junto com cada pergunta para dar conta das perguntas de seguimento. As saídas ficam em `notebook_study_outputs` como JSON: OVERVIEW (uma por página, substituída a cada vez), SUMMARY, STUDY_GUIDE e QUIZ. Um quiz guarda as respostas no servidor. O cliente recebe as perguntas, manda as escolhas para correção e só então descobre o que estava certo, que é também onde os 20 de XP são pagos.

O `NotebookAiService` cuida do resto. Um rascunho de roteiro fica guardado, porque uma chamada leva até um minuto e meio e o diálogo perdia um rascunho pronto num clique fora dele. O `RoadmapDraftService` grava o pedido em `notebook_roadmap_drafts` (V35) e responde na hora, DRAFTING. A chamada ao modelo roda numa virtual thread depois que esse pedido faz commit, a mesma passagem da leitura de fontes, e o `RoadmapDraftWrites` guarda o resultado, mas só numa linha que ainda está DRAFTING, então um rascunho excluído no meio da chamada continua excluído. O diálogo relê o rascunho a cada poucos segundos. A página inicial do caderno lista todos os rascunhos e reabre cada um com o formulário, os nós e as marcações da pessoa, que o diálogo salva conforme mudam. Uma chamada por rascunho de cada vez: um rascunho DRAFTING recusa redesenho e novas marcações (`NOTEBOOK_DRAFT_BUSY`). Criar o tópico exclui o rascunho na mesma transação, um reinício marca como FAILED as chamadas que cortou, com `NOTEBOOK_DRAFT_INTERRUPTED`, e cada pessoa guarda no máximo 20 rascunhos. As leituras ficam em `/notebook/drafts`, fora de `/notebook/ai`, então reler um rascunho gasta a cota de leitura, não a de IA. Nada no caderno muda antes de "Criar".

O rascunho também oferece vínculos. Cada título rascunhado é normalizado (sem caixa, acentos, pontuação e o s do plural) e comparado com as páginas que a pessoa já tem, então um rascunho de "Fundamentos da Computação" pode dizer "você já tem Sistemas Operacionais, 4 de 9 feitos, ligar". Criar a partir do rascunho roda numa transação só pelo `NotebookBoardService.addChain`, que dispõe os nós três por linha, a 240 por 140 de distância, ligados por arestas, e dá a cada nó novo com subtópicos um quadro próprio. O modelo propõe texto. Ele nunca grava no banco e nunca escolhe um id ou uma coordenada.

Toda rota em `/notebook/ai/**` gasta o bucket de rate limit `notebook-ai`, 60 chamadas por usuário por hora. Dividir os 30 do assistente deixaria uma noite de estudo trancar a pessoa fora do assistente. A ferramenta de cartões do assistente chama o mesmo modelo sem passar por essa rota, então o `NotebookAiQuota` gasta o mesmo bucket por ela (mesmo cache, mesma chave), e uma resposta do chat não consegue rascunhar cartões além do limite da hora.

### O preparo da sala de estudo

Antes da primeira pergunta a sala abre no preparo (`StudySetup` na web), e "Editar preparo", na linha acima do chat, traz ele de volta. Ele pergunta três coisas. Um objetivo para estudar a página, guardado em `notebook_pages.study_goal` e colocado antes dos trechos no contexto de toda resposta, para as respostas mirarem nele; ele nunca é um trecho e nunca é citado. As anotações de quem a IA lê (`study_scope`, V36): PAGE, a página e as páginas acima dela, que é o que a sala sempre leu e continua sendo o padrão; SUBTREE, também todas as páginas abaixo; TOPIC, todas as páginas do tópico. O `StudyScopes` monta a lista, e a tela de preparo mostra quantas páginas com anotações e quantas palavras cada escolha cobre. E o que ela pode citar: as fontes ficam no painel ao lado do preparo, com seus interruptores, e o preparo adiciona mais, à mão ou com "encontrar fontes para mim". Objetivo e escopo valem para toda chamada de IA na página, não só o chat: o estúdio, a explicação, os flashcards de IA e as sugestões de nós leem o mesmo contexto. `study_setup_at` nulo é o que abre a tela de preparo; uma sala com mensagens de antes da V36 abre no chat.

## O assistente num quadro

O chat do assistente (o [agente de IA](/architecture/ai-agent)) muda um quadro quando a pessoa pede. O `getStudyBoard` lê um: os nós de página na ordem do caminho com seus status, as ligações entre eles por título e as seções. Nove ferramentas escrevem: as oito abaixo e o `generateStudyCards`, mais adiante. O `addStudyNode` põe um nó na próxima célula livre da grade e pode ligá-lo depois de outro nó. O `editStudyNode` renomeia um nó ou troca o ícone, e como o título de um nó é o título da página dele, a página muda de nome em todo lugar onde aparece. O `setStudyNodeStatus` define o status de um nó ou da própria página do quadro. O `connectStudyNodes` e o `disconnectStudyNodes` criam e removem ligações. O `removeStudyNode` tira um nó do quadro e mantém a página, a não ser que venha `deletePage`. O `reorderStudyBoard` refaz o caminho, e o `addStudyNotes` escreve markdown no fim de uma página.

O `StudyBoardEditor` transforma os nomes que a pessoa usa em linhas e depois chama os mesmos services que o quadro na tela chama, então posse, as recusas de vínculo e o XP de uma página concluída são os mesmos de um clique. Um quadro é nomeado pelo id da página, que a rota onde a pessoa está carrega, ou pelo título exato da página. Um nó é nomeado pelo título ou pelo id. Um nome que não bate com nada recebe de volta a lista do que existe. Um nome que bate com duas páginas, ou dois nós, recebe de volta uma pergunta, então o modelo nunca escolhe um. Cada chamada de ferramenta é uma transação, menos a `generateStudyCards`: ela resolve a página numa transação curta e deixa a chamada ao modelo fora de qualquer uma, como o botão faz.

O `reorderStudyBoard` recebe cada nó de página do quadro uma vez. Eles viram uma corrente só na grade, nessa ordem, e a corrente substitui todas as ligações que o quadro tinha (`NotebookBoardService.restructure`), então um ramo que a pessoa desenhou some depois. O prompt faz o assistente repetir a ordem e esperar um sim. Quando o assistente adiciona o primeiro nó numa página cujo documento não tem bloco de quadro, ele grava o bloco também (`MarkdownBlocks.withBoardBlock`). Sem ele o quadro não teria onde ser desenhado.

O `generateStudyCards` rascunha cartões na página de um nó, ou na página do quadro, com a mesma chamada do "Rascunhar com IA" do bloco de cartões (`NotebookAiService.cards`), a partir das anotações e fontes da página ou de um texto de foco. Ele gasta a cota notebook-ai e dá à página um bloco de cartões se ela não tinha, para os cartões aparecerem onde a pessoa lê. O bloco de cartões relê o baralho quando o número de cartões da página muda, e é assim que cartões criados pelo chat aparecem numa página aberta.

Toda ferramenta que escreve informa o domínio `notebook`, e as que podem mudar um status informam `perfil` também. Com isso, os dois clientes releem a home, cada quadro e árvore que já carregaram e a página na tela (`refreshNotebook` em `@beyou/state`). Uma revisão mais nova da página na tela é juntada ao editor aberto (veja "Uma página, dois escritores" abaixo), então as anotações do assistente aparecem sem recarregar e o que foi digitado nesse meio tempo fica.

## Uma página, dois escritores

O documento é uma coluna JSON, salva inteira pelo autosave do editor, e até a V37 o último a salvar vencia. Um celular e um computador na mesma página, ou o assistente acrescentando anotações com a página aberta, faziam uma escrita apagar a outra sem aviso. Agora cada página tem um `content_revision` que sobe a cada escrita do documento, e toda escrita passa por um único compare-and-set, `NotebookPageRepository.writeContentIfAt` (UPDATE ... WHERE content_revision = :read). O autosave manda a revisão de onde partiu como `baseRevision`; um salvamento de uma revisão mais antiga é recusado com `NOTEBOOK_CONTENT_CONFLICT` e não muda nada. As escritas do próprio servidor (um append, um bloco de quadro ou de cartões que ele acrescenta) leem de novo e tentam outra vez quando alguém escreveu antes. Um pedido sem `baseRevision`, de um cliente anterior às revisões, ainda sobrescreve.

O editor recusado junta. O `DocumentSync` em `@beyou/state` guarda a revisão que o editor viu por último e o documento nessa revisão, a base. Numa recusa, ou quando chega uma revisão mais nova de qualquer outro lugar (um reload, uma resposta do assistente), ele lê a página e junta a base, o documento do editor e o do servidor bloco a bloco (`mergeDocuments`). Um bloco mudado de um lado fica com a versão desse lado, blocos acrescentados de qualquer lado ficam onde o lado deles os pôs, e só um bloco mudado dos dois lados, ou removido de um e mudado no outro, vai para a pessoa: um diálogo mostra as duas versões e mantém as duas a não ser que ela escolha uma. Nada é salvo enquanto ele está aberto. O documento juntado entra no editor trocando só o trecho que difere, então o quadro, os cartões e o cursor em outro lugar ficam como estão, e é salvo quando tem algo que o servidor não tem. Só um salvamento vai por vez, então um editor nunca entra em conflito consigo mesmo. O `DocumentSync` não tem React nem rede própria, e é isso que deixa o editor do celular usá-lo também.

A junção casa os blocos pelo id. Os blocos que o servidor escreve ganham um UUID aleatório, como os do BlockNote (`MarkdownBlocks`), e uma página salva antes disso, com blocos sem id, é salva uma vez quando abre, o que grava os ids que o editor deu. Um bloco sem id que chegue a uma junção é casado pelo tipo e pelo texto.

## Onde cada regra mora

| Regra | Mora em |
|-------|---------|
| A página pertence a quem chama. Todo caminho verifica isso primeiro, inclusive os ciclos de foco que nomeiam uma página | `NotebookOwnership` |
| O status, a subida pelos quadros, o pagamento no primeiro DONE e o aviso aos ciclos de estudo sobre as páginas que terminaram | `NotebookProgressService` |
| Progresso, status derivado, alcance, a próxima folha a estudar, a subárvore de uma página | `ProgressGraph` |
| Todo valor de XP e o teto diário de revisão | `NotebookRewards` |
| O texto puro de um documento | `BlockText` |
| A revisão do documento: um compare-and-set para toda escrita | `NotebookPageRepository.writeContentIfAt`, `NotebookPageService` |
| Juntar um editor aberto com o que foi salvo em outro lugar, um salvamento por vez | `DocumentSync`, `mergeDocuments` em `@beyou/state` |
| O que os dois editores fazem com um documento fora do BlockNote: linguagens de código, a checagem de página vazia, o JSON salvo como conteúdo inicial, trocar só o trecho que mudou | `codeLanguages`, `editorDocument` em `@beyou/state` |
| O markdown da IA virando blocos no "Salvar na página", o tipo do bloco de quadro | `MarkdownBlocks` |
| As recusas de nó vinculado, o `deletePage`, a disposição em corrente do rascunho, a próxima célula livre, a ordem do caminho, refazer um quadro como um caminho só | `NotebookBoardService` |
| As ferramentas de quadro do assistente: de nomes para linhas, e as recusas que o impedem de adivinhar | `StudyBoardEditor` |
| O bucket notebook-ai gasto fora de `/notebook/ai/**` (a ferramenta de cartões do assistente) | `NotebookAiQuota` |
| O cronograma SM-2 | `SpacedRepetition` |
| O escopo das fontes, o limite de 20 por página, as checagens de PDF | `NotebookSourceService` |
| As recusas de SSRF, os saltos de redirect, os tetos de corpo | `LinkFetcher` |
| Leitura em segundo plano, embeddings, recuperação após restart | `SourceIngestionService` |
| A escolha dos trechos e seus fallbacks | `NotebookRetriever` |
| A numeração dos trechos e a limpeza das citações, o objetivo no contexto | `StudyContextBuilder` |
| De quem são as anotações que as respostas de uma página leem | `StudyScopes` |
| Busca na web, abrir cada resultado, descartar o que está morto, é privado ou já é fonte | `SourceDiscoveryService`, `WebSearchClient`, `LinkFetcher.resolve` |
| A chamada ao modelo, a nova tentativa, o limite de 90 segundos e o `AI_UNAVAILABLE` | `NotebookLlm` |
| Nenhuma transação aberta enquanto o modelo responde: uma leitura curta antes, uma gravação curta depois | `NotebookTransactions` |
| Um rascunho de roteiro guardado: uma chamada por vez, resultado só numa linha DRAFTING, apagado quando o tópico existe | `RoadmapDraftService`, `RoadmapDraftWrites` |
| Os blocos de um ciclo a partir da semana e dos tópicos: pesos, minutos por tópico, tamanho dos blocos, a intercalação | `StudyCyclePlanner` |
| Como minutos e entendimento movem uma volta: o caminho do crédito, o transbordo, a sobra, a espera pelo quiz, o estorno, a volta completa | `StudyLap` |
| A fatia de hoje: o orçamento, a virada para a próxima volta, os horários | `StudyDaySlice` |
| Um ciclo ativo por tópico, os limites, o replanejamento e a junção, o lock da linha, fechar uma volta, as leituras que fixam um conceito, as sessões | `StudyCycleService` |
| O que um bloco de OBJETIVO mira (a página escolhida ou a próxima folha do roadmap), e os fatos do caderno que um ciclo lê uma vez por requisição | `NotebookFacts` |
| O histórico de um ciclo, dia a dia | `StudyCycleHistoryService` |
| As ferramentas de ciclo do assistente: de nomes e valores soltos para ids e requisições | `StudyCycleAssistant` |
| Como um bloco aparece nos dois apps (o título e o número ao lado), e a prévia, no web, de onde uma sessão lançada vai cair | `studyCycleView` em `@beyou/state` |

## O editor web e o quadro

O editor web é o BlockNote 0.55 (`@blocknote/core`, `@blocknote/react`, `@blocknote/mantine`) sobre o Mantine 8. O Mantine 9 exige React 19.2 e o app web está no React 18, então o `@mantine/core` fica no `^8.3.18` até o React subir. O schema tira os blocos de áudio, vídeo e arquivo, porque eles precisam de um backend de upload que o caderno não tem, e um bloco que oferece upload e depois falha é pior do que bloco nenhum. Imagens ficam, embutidas por link. Dois blocos próprios entram junto dos padrões, `roadmapBoard` e `flashcards`, e o menu de barra ganha "Quadro de roteiro", "Cartões" e "Explicar o bloco acima". O editor salva 900 ms depois da última mudança com `PUT /notebook/pages/{id}/content`. As cores vêm de variáveis de tema, então ele segue o tema que estiver ligado.

Enquanto uma página não tem nada, o editor mostra dois botões iniciais, "Adicionar um quadro de roteiro" e "Adicionar cartões". Eles moram dentro do editor porque é o editor que sabe que o documento está vazio, e inserem o bloco na frente do que ele tiver. Antes ficavam na tela da página, decidiam a partir de uma cópia da página que o autosave nunca atualiza, e gravavam um documento novo no servidor: numa página que começou vazia eles continuavam na tela enquanto a pessoa escrevia, e "Adicionar cartões" trocava as anotações por um bloco de cartões. O bloco de cartões lista os três primeiros cartões, cada um uma pergunta cuja resposta abre num clique. Blocos de código ganham um seletor de linguagem sobre uma lista fixa (`codeLanguages.ts`); o BlockNote lança erro ao desenhar uma linguagem que não está na lista, então a lista responde sim a qualquer nome e os blocos de código de uma página passam para um id da lista (ou `text`) quando ela carrega. Links são desenhados como no Notion: a cor do próprio texto com um sublinhado suave.

O quadro é React Flow (`@xyflow/react` 12). O "Organizar" põe os nós de página de volta na grade em que um rascunho começa: na ordem do caminho, então um nó vem depois dos nós que apontam para ele, três por linha, cada linha continuando onde a anterior terminou. Onde as arestas deixam escolha, decide a ordem que a pessoa deixou no quadro. As seções ficam onde a pessoa as colocou. O layout da web (`boardLayout.ts`) e o do servidor (`NotebookBoardService`) usam a mesma grade, então organizar um rascunho recém-criado não move nada. Um nó novo ocupa a próxima célula livre. O Organizar era um layout do dagre, que desenhava qualquer cadeia como uma linha comprida. O `useBoard` reúne toda ação de quadro, para o quadro dentro da página e para o de tela cheia. Toda escrita chega ao store a partir da resposta do servidor. A exceção são as posições, desenhadas primeiro e salvas depois, para que arrastar nunca espere a rede.

As rotas:

| Rota | Tela |
|------|------|
| `/notebook` | Home: cards de tópico com miniatura do quadro, "continuar estudando", as revisões do dia |
| `/notebook/review` | Uma sessão de revisão, opcionalmente restrita com `?page=` |
| `/notebook/:pageId` | Uma página: árvore, anotações, o quadro embutido, status |
| `/notebook/:pageId/board` | O quadro em tela cheia, cobrindo o shell como `/focus` |
| `/notebook/:pageId/study` | A sala de estudo em tela cheia: fontes, chat, saídas do estúdio, ou as anotações da página ao lado do chat |
| `/notebook/cycles` | Os ciclos de estudo: os ciclos ativos em chips, a volta até aqui, os blocos em ordem, a semana e as voltas anteriores |
| `/notebook/cycles/new` | O editor de ciclo com a calculadora, para um ciclo novo |
| `/notebook/cycles/:cycleId/edit` | O mesmo editor num ciclo que já existe |
| `/notebook/cycles/:cycleId/history` | O histórico de um ciclo, uma semana por vez |

O Caderno fica no grupo principal da sidebar depois de Metas (ícone `NotebookPen`) e na folha do bottom nav. Cada tela do caderno é um chunk lazy. O editor vai dentro do chunk da tela de página e nunca carrega no boot.

No desktop a tela da página reserva o próprio espaço no fim: 40px abaixo da linha vazia final do editor, mais enquanto a pílula do timer flutua sobre o rodapé. Ela desliga o espaçador de 96px do shell com `useNoDesktopSpacer`; esse espaçador continua em todas as outras páginas para o botão do assistente nunca cobrir o fim de uma, e o celular o mantém por causa da barra inferior. Na sala de estudo os painéis de fontes e de estúdio podem ser redimensionados arrastando as barras ao lado do chat (de 220 a 560px; duplo clique volta ao padrão, setas e Enter pelo teclado) ou recolhidos em trilhos finos para o chat ocupar a tela. O layout é uma preferência por navegador guardada no localStorage (`useStudyLayout`), e nada disso vale abaixo de `lg`, onde as colunas se empilham.

Um seletor no cabeçalho da sala escolhe o que ela mostra: a sala, as anotações da página ao lado do chat, ou só as anotações (`useStudyView`, também guardado por navegador). As anotações ali são o editor da tela da página, com a mesma sincronização. "Salvar na página" no chat coloca no store a página que volta da chamada, e as anotações abertas juntam a resposta na hora. Sem isso, o próximo salvamento automático delas seria recusado como desatualizado e teria que se atualizar depois. Trocar de modo esconde painéis e nunca os desmonta, porque a conversa e as saídas do estúdio feitas desde que a sala abriu vivem neles; só as anotações montam ao aparecer, e leem a página de novo. Lado a lado, cada coluna rola sozinha e a caixa de pergunta fica presa ao pé do chat. A caixa de pergunta é um textarea: Enter envia, Shift+Enter quebra a linha, e o Enter que encerra uma composição de IME nunca envia.

## Mobile

O celular lê e escreve anotações. As telas são a home de tópicos (`app/(app)/notebook/index.tsx`), um tópico ou página com as abas Trilha e Notas (`app/(app)/notebook/[id].tsx`), o editor de notas (`app/notebook-editor.tsx`), a revisão (`app/notebook-review.tsx`), a sala de estudo (`app/notebook-study.tsx`), o rascunho de roteiro (`app/notebook-draft.tsx`) e as três telas do ciclo de estudos (`app/notebook-cycles.tsx`, `app/notebook-cycle-edit.tsx`, `app/notebook-cycle-history.tsx`, descritas em "Ciclos de estudo" mais abaixo). O editor, a revisão, a sala de estudo, o rascunho e as telas do ciclo são em tela cheia e ficam fora do grupo `(app)`, para que a barra de baixo nunca fique entre o texto e o teclado nem numa sessão que quer a tela inteira.

Um canvas arrastado com um polegar não serve para ler um roadmap, então a aba Trilha lê o quadro em níveis (`pathLevels` em `@beyou/state`). Cada nó vem depois dos nós que apontam para ele, e um nível com mais de um nó ganha "Depois, em qualquer ordem". A aba Notas desenha o documento com views nativas (`src/notebook/BlockRenderer.tsx`), e "Editar notas" ou um toque nelas abre o editor. As páginas abaixo desta que não estão no quadro aparecem listadas embaixo, lidas da árvore do tópico, já que o celular não tem barra lateral.

O roteiro também é editado ali. O "⋯" de um nó abre a folha dele, o inspetor de nó do web no tamanho de um polegar: o título e o ícone da página, o status, os nós de que ele vem depois (uma ligação pode ser tirada e outra adicionada), e então abrir a página, adicionar um nó depois deste, tirar do roteiro (a página continua abaixo desta) ou apagar a página com as notas. "Adicionar nó", abaixo da trilha, põe um depois do último nó na ordem em que a trilha lê. O celular não manda coordenadas: o servidor põe o nó na próxima célula livre da grade, liga ele depois do nó indicado e dá à página o bloco do quadro, então um roteiro começado no celular aparece no web. "Reordenar" move os nós com setas e salva a ordem inteira de uma vez (`PUT /notebook/pages/{id}/board/order`), o que transforma o quadro num caminho só; num quadro desenhado com galhos, a folha avisa que eles viram uma linha.

Uma terceira aba guarda os flashcards da página, o bloco de cartões do web em forma de aba: quantos vencem hoje, com a revisão a um toque, "Rascunhar com IA" (a IA de estudo lê a página e as fontes dela e salva os cartões que escreve), "Escrever um" e o baralho, onde um cartão mostra a pergunta e a resposta abre embaixo com um toque, com Editar e Excluir ali. O rótulo da aba leva a contagem de cartões da página. Os blocos de quadro e de cartões no editor abrem a tela da página nas abas Trilha e Cartões (`/notebook/{id}?tab=`).

A sala de estudo da página abre em "Sala de estudo", ao lado do Foco (`app/notebook-study.tsx`): a sala do web em três abas, Chat, Fontes e Estúdio, embaixo de uma barra com o objetivo da sala. Uma sala que ninguém configurou abre na configuração, como no web, e "Editar" na barra do objetivo abre ela de novo. Cada resposta do chat lista de onde vieram os pontos numerados, e pode ir para o fim da página ou virar três cartões. Fontes aceita um link, um texto colado, "Encontrar fontes para mim" ou um PDF escolhido no celular, recusado acima de 15 MB antes de sair. Toda fonte da página tem um interruptor que diz se as respostas usam ela. As fontes adicionadas numa página acima aparecem à parte, e enquanto o servidor ainda está lendo uma, a lista é lida de novo a cada 3 segundos. O estúdio faz um resumo, um guia de estudo, um quiz ou seis cartões. Um resumo ou um guia abre para ler. Um quiz é feito na folha dele e corrigido no servidor, e passar paga o XP na primeira vez, com o mesmo aviso de subida de nível que um check-in recebe. A visualização de markdown do celular não tem títulos, então as linhas de título da IA aparecem em negrito.

O PDF sobe pelo `src/lib/uploadFile.ts`, que a foto de perfil também usa agora. O fetch do React Native não manda uma uri `file://` como parte multipart, então o expo-file-system monta a requisição no nativo, e o helper põe os headers de autenticação e a única nova tentativa depois de um 401 que o cliente da API faria. O nome do arquivo na parte multipart é o último segmento da uri, e a cópia que o seletor guarda no cache tem um nome aleatório, então o helper recebe o nome do arquivo e manda uma cópia com ele, apagada depois; sem isso a fonte do PDF ficava com um UUID de título. O `expo-document-picker` é um módulo nativo: um build de desenvolvimento feito antes dele não tem o seletor.

O editor é o BlockNote, o mesmo do web, dentro de uma web view: um componente DOM do Expo (`src/notebook/editor/NotebookEditorDom.tsx`, `'use dom'`). O schema é o do web, bloco por bloco, então uma página escrita num abre no outro; os blocos do quadro e dos cartões continuam no lugar deles no documento e aparecem como cartões que abrem a aba Trilha e a revisão. O editor e o `DocumentSync` dele rodam juntos na web view, então um merge lê o documento que a pessoa está vendo. A tela empresta a rede (salvar, ler a página), mostra o estado do salvamento e faz a pergunta do conflito numa folha. O `editorBridge.ts` é o contrato entre as duas metades: a barra de ferramentas da tela manda comandos para a web view, e a web view conta onde o cursor está, para que Negrito, Itálico, Link e os botões de lista acendam onde valem. O menu lateral e a barra flutuante do BlockNote ficam desligados (um precisa de um hover que o dedo não tem, a outra briga com as alças de seleção do Android); o "+" adiciona um bloco abaixo do cursor, o "Aa" transforma o bloco atual em outro tipo, e digitar "/" ainda abre o menu de blocos.

Sair com uma edição ainda no debounce de 900ms do salvamento automático manda essa edição na hora e espera a resposta antes de a tela sair, então a página por trás mostra o que acabou de ser escrito. Ir para segundo plano também manda, e voltar junta o que foi salvo em outro lugar nesse meio tempo. Links abrem no navegador do celular, nunca dentro da web view.

A gestão de páginas fica no "⋯" da página: renomear, ícone, uma página nova abaixo desta (que já abre no editor) e excluir, que avisa o que vai junto como o diálogo do web e volta para a página pai. "Novo tópico" na home começa um tópico em branco ou rascunha um roteiro com IA.

O rascunho é o diálogo do web numa tela só (`app/notebook-draft.tsx`), o formulário e o rascunho embaixo. Depois que um rascunho é pedido, o formulário vira uma linha com "Editar", e a tela lê o rascunho de novo a cada 2,5 segundos enquanto o modelo escreve, com o tempo decorrido na tela. Um toque mantém um nó ou deixa ele de fora. Um nó que a pessoa já estuda em outro lugar oferece "Ligar" ou "Nova cópia", e as marcações são salvas 600ms depois da última, como no web. Uma mudança pedida em palavras é digitada embaixo do último nó; a tela então volta para cima, onde está a espera. Sair não perde nada, porque os rascunhos esperam na home em "Rascunhos". Um que o modelo ainda está escrevendo mostra ali o tempo correndo e vira "Pronto para revisar" sozinho, já que a home lê a lista de novo a cada 4 segundos enquanto há um em andamento. O que os dois clientes fazem com um rascunho (as linhas com as marcações, os pedidos, o tópico criado a partir das linhas mantidas, as semanas que ele leva) fica em `roadmapDraft.ts` no `@beyou/state`, que o diálogo do web agora usa também.

A web view precisou de três coisas que o resto do app não precisa. `react` e `react-dom` são redirecionados para as cópias do mobile no `metro.config.js`, porque o BlockNote fica no topo do monorepo ao lado do React 18 do web e o dedupe do próprio Expo só cobre as plataformas nativas. O `@expo/metro-runtime` é dependência direta, porque a entrada do bundle DOM importa ele. E as folhas de estilo do BlockNote são importadas uma a uma, a do Mantine inteira inclusive, porque o bundle DOM não segue `@import`s de CSS com nome de pacote. O build de produção empacota a web view em `www.bundle`, ao lado do app.

O Caderno está na folha do bottom nav do mobile. O reducer é registrado em `apps/mobile/src/store.ts`, e o redux do mobile fica em memória como sempre.

## Ciclos de estudo

O ciclo de estudos é o método que todo concurseiro conhece: uma fila ordenada de blocos de estudo, sem dia da semana fixo. Cada bloco pertence a um tópico do caderno. Fechou um bloco, entra o próximo; quando todos os blocos da fila fecham, a volta está completa e a fila recomeça. Quem estuda para um concurso enquanto termina a faculdade pode tocar dois ciclos ao mesmo tempo.

Um tópico fica em no máximo um ciclo ativo. O `StudyCycleService` confere isso antes, para a recusa poder dizer qual ciclo já tem o tópico: `STUDY_CYCLE_TOPIC_TAKEN`, com `topicId`, `cycleId` e `cycleName` nos `details`. Dois salvamentos podem passar juntos por essa checagem, e aí quem segura é o banco: um índice único parcial, `uq_study_cycle_topics_active`, em `study_cycle_topics (topic_id) WHERE active`. Tirar um tópico de um ciclo libera o tópico. Arquivar um ciclo põe `active` em false em todos os tópicos dele, o que também os libera.

Cada ciclo tem uma semana, com os minutos de cada dia (0 é dia de descanso) e um horário de início opcional por dia. Os minutos da semana são a soma dos sete dias, então ninguém digita o mesmo número duas vezes. Cada pessoa tem no máximo 10 ciclos ativos, e um ciclo comporta até 30 tópicos e 60 blocos; passar de qualquer um desses limites responde `STUDY_CYCLE_LIMIT`.

A funcionalidade não paga XP em lugar nenhum. O caderno já paga por página concluída e por quiz aprovado, e o ciclo só decide o que vem a seguir. Também não tem cache, como o resto do caderno: as leituras escrevem (veja abaixo), e cada pomodoro e cada mudança de status em outro lugar mexem nos números, então um cache precisaria ser invalidado pelos caminhos do foco, da página, do quadro e do quiz.

### Blocos de tempo e blocos de objetivo

Todo bloco é TIME ou GOAL ("Tempo" e "Objetivo" nas telas), escolhido bloco a bloco. O ciclo guarda um modo padrão para os blocos que o planejador cria, GOAL a menos que a pessoa escolha TIME.

Um bloco de tempo é uma quantidade de minutos num tópico, "70 minutos de Português". Ele fecha como DONE quando chega aos minutos previstos.

Um bloco de objetivo é um conceito a entender, e um conceito é uma página do caderno. Com `goalSource = PAGE`, ele mira uma página que a pessoa escolheu, que precisa ser do tópico do bloco (a própria página do tópico vale). Com `goalSource = NEXT`, mira o que o roadmap do tópico diz que vem a seguir, `ProgressGraph.nextLeaf`: a primeira folha, na ordem de leitura, que está em estudo, ou então a primeira ainda não começada. Esse alvo é sempre uma folha, e ele anda pelo roadmap sozinho. O bloco fica UNDERSTOOD quando a página chega a DONE. Um bloco também pode exigir o quiz da página, e aí uma página DONE sem quiz aprovado deixa o bloco aberto; as telas avisam que a página está feita e falta o quiz.

O bloco de objetivo também tem minutos previstos. Para ele, são uma estimativa, que serve para montar o dia e fazer as matérias girarem. Quando esses minutos acabam antes de o conceito ser entendido, o bloco fecha como TURN_USED ("vez usada") e o ciclo segue. O próximo bloco do mesmo tópico pega o mesmo conceito, já que aquela página continua sem estar feita. Entender antes fecha o bloco na hora, com quantos minutos ele tiver.

Um bloco de objetivo sem nada para entender funciona como um de tempo e fecha como DONE pelos minutos. Isso acontece com um bloco NEXT cujo roadmap já terminou, que as telas mostram como revisão, e com um bloco PAGE cuja página já estava feita (com o quiz aprovado, quando o bloco pede). Desmarcar uma página depois nunca reabre um bloco fechado.

### De onde vêm os minutos

Só duas coisas creditam minutos. Uma é um pomodoro terminado em qualquer página do tópico, que o timer de foco já registra com o `notebookPageId`; o tópico é a raiz daquela página, e pausa nunca conta. O `FocusService.recordCycle` chama o `StudyCycleService.credit` depois de salvar o ciclo de foco. A outra é uma sessão lançada à mão, para estudo feito longe do timer, pelo `POST /notebook/sessions`.

O `StudyLap.credit` é o caminho. Ele acha o ciclo ativo que tem o tópico e trabalha na volta atual desse ciclo. Os blocos abertos do tópico, na ordem da fila, recebem cada um o que ainda falta, até onde os minutos derem. O bloco que chega aos minutos previstos fecha: DONE, ou TURN_USED num bloco de objetivo cujo conceito continua aberto. O que sobrar depois do último bloco aberto do tópico vai para o último bloco do tópico, que passa a mostrar algo como "80 de 70 min", e nenhum minuto estudado some. Estudar fora de ordem não tem problema: os minutos vão para o primeiro bloco aberto daquele tópico, esteja ele onde estiver na fila. Um tópico sem bloco na volta não credita nada, e a sessão lançada fica guardada para o histórico mesmo assim.

Quando todos os blocos da volta estão fechados, a volta fecha. Uma linha em `study_cycle_laps` congela os números dela (minutos estudados e previstos, conceitos entendidos, vezes usadas, blocos), o número da volta do ciclo sobe um e a próxima volta começa naquele instante. O excedente da sessão que fechou a volta fica na volta fechada. Esses números são guardados porque um bloco editado ou apagado depois mudaria o passado se eles fossem recalculados.

O progresso fica por volta em `study_cycle_lap_blocks`, uma linha por bloco por volta, criada na primeira vez que a volta credita, resolve ou fecha aquele bloco. Bloco sem linha está aberto e sem minutos. A linha do bloco em si é configuração e não guarda progresso nenhum, e é isso que deixa um replanejamento reescrever os blocos sem mexer no que foi estudado.

Toda escrita que mexe numa volta pega primeiro o lock da linha do ciclo: `StudyCycleRepository.lockById`, ou `lockActive` para todos os ciclos da pessoa, pegos em ordem de id para dois escritores não travarem um ao outro. Sem isso, dois pomodoros terminando no mesmo segundo leriam um bloco em 50 de 70 e cada um gravaria o próprio total. `StudyCycle` e `StudyCycleLapBlock` também são `@DynamicUpdate`, porque uma renomeação e um crédito podem chegar juntos.

### Os ganchos no caderno

O entendimento chega ao ciclo por dois funis. O `NotebookProgressService` é a única classe que grava o status de uma página, e no fim de cada mudança de status ele passa ao `StudyCycleService.onPagesDone` toda página que a mudança concluiu, incluindo as páginas pai que subiram junto. Todo jeito de concluir uma página passa por ali, as ferramentas do assistente inclusive, então um jeito novo que aparecer depois não tem como esquecer o ciclo. O `NotebookStudyService` chama o `onQuizPassed` na primeira aprovação de um quiz, onde o `passed_at` é gravado, e isso fecha os blocos que esperavam o quiz de uma página já DONE.

O `NotebookFacts` lê o grafo de páginas para o ciclo direto dos repositórios. Passar pelo `NotebookProgressService` criaria um laço de construtores, já que esse service chama o ciclo.

### Por que ler um ciclo pode gravar nele

Um bloco NEXT ganha o conceito na primeira vez que alguma coisa trabalha nele. O conceito fica então guardado na linha da volta (`target_page_id`), para o bloco saber que página esperar. As leituras também fixam. O `GET /notebook/cycles/today`, o `GET /notebook/cycles/{cycleId}` e o ciclo que volta ao criar ou editar um ciclo gravam o conceito do primeiro bloco de objetivo aberto de cada tópico.

O motivo é a pessoa que abre a página que o card de Hoje aponta, estuda sem timer e marca como feita. O bloco tem que fechar como UNDERSTOOD, e só consegue se o conceito foi gravado antes de a página mudar. Os blocos abertos seguintes do mesmo tópico mostram o que o roadmap diz agora, com `targetPreview: true`, e não gravam nada: o conceito que vão receber depende do que acontecer antes. Essas leituras pegam o mesmo lock das escritas, então duas telas abrindo juntas não criam a mesma linha de volta duas vezes.

Mudar o alvo de um bloco (o modo, a origem ou a página) esquece o conceito que a linha aberta dele tinha fixado, e a próxima leitura ou crédito resolve de novo. O Resumo do Dia lê o dia pelo `todayPreview`, dentro da sua própria transação somente leitura, e não fixa nada.

### O planejador, o mesmo para o editor e para o assistente

O `StudyCyclePlanner` transforma uma semana e um conjunto de tópicos em blocos. É aritmética pura:

1. Cada tópico pesa `importance × (6 − mastery)`, importância e domínio, os dois numa escala de 1 a 5. Uma matéria importante que a pessoa mal conhece pesa 25; uma secundária que ela domina pesa 1.
2. Os minutos sugeridos de um tópico por volta são a fatia dele nos minutos da semana, arredondada para 5 e nunca abaixo de 5. A pessoa pode definir os minutos de um tópico por conta própria (`minutesOverride`, de 5 a 600), e o número dela vale.
3. Esses minutos são cortados em blocos de no máximo `maxBlockMinutes` (15 a 240), tão iguais quanto os passos de 5 minutos deixam. Os primeiros blocos ficam com os cincos a mais, e o que sobrar abaixo de 5 vai para o primeiro bloco.
4. Os tópicos são ordenados por peso, com empates mantendo a ordem recebida, e a rodada r pega o r-ésimo bloco de cada tópico que ainda tiver um. A intercalação é a graça do ciclo. Toda matéria volta a cada poucos blocos, e nenhuma engole a semana sozinha.

O `lapDays` diz quantos dias uma volta leva no ritmo da semana, no mínimo 1, e vem null numa semana sem minutos.

Tudo que cria blocos chama o planejador: criar um ciclo, replanejar um, o `createStudyCycle` do assistente e o `POST /notebook/cycles/plan`. Essa última rota é uma prévia sem estado. O editor dos dois apps pede de novo uns 300 ms depois da última mudança, descarta uma resposta que chegue atrasada e nunca faz a conta sozinho. Assim, o que o editor mostra é o que fica salvo, e um ciclo que o assistente monta com as mesmas entradas sai igual. Por ser POST, a prévia gasta o orçamento genérico de escrita, como as outras escritas do caderno.

### Editar um ciclo no meio da volta

O `PUT /notebook/cycles/{cycleId}` recebe a configuração inteira de novo, e só replaneja quando as entradas do plano mudaram: o total de minutos da semana, o tamanho do bloco, ou os tópicos com a ordem, os pesos e os minutos próprios. Qualquer outra coisa, uma renomeação ou horários novos por exemplo, deixa a fila como está, e uma ordem que a pessoa arrumou à mão sobrevive.

O replanejamento é juntado à lista antiga tópico por tópico. O k-ésimo bloco de um tópico no plano novo herda o id do k-ésimo bloco desse tópico na lista antiga, e com o id fica com o progresso daquele bloco na volta e com as configurações dele: modo, origem do objetivo, página, quiz e nota. A posição e os minutos vêm do plano novo. Os blocos antigos que ficaram sem par são apagados junto com o progresso, e os novos recebem o modo padrão do ciclo.

Os blocos têm rotas próprias. O `PUT /notebook/cycles/{cycleId}/blocks` troca a lista ordenada. Um bloco enviado com id mantém o progresso, um sem id é novo, e um bloco não pode mudar de tópico, porque os minutos dele foram estudados naquele. O `PATCH /notebook/cycles/{cycleId}/blocks/{blockId}` muda alguns campos de um bloco: campo ausente fica como está, e um null explícito limpa. Um bloco que já fechou mantém o estado, mude o que mudar. Baixar os minutos de um bloco até o que ele já tem fecha o bloco ao salvar (`StudyLap.closeFull`), e tirar o último bloco aberto pode fechar a volta.

O `POST /notebook/cycles/{cycleId}/archive` libera os tópicos do ciclo e mantém o ciclo e o histórico dele legíveis. Editar um ciclo arquivado, nas configurações ou nos blocos, responde `STUDY_CYCLE_NOT_FOUND`, ele não recebe mais créditos, e não existe desarquivar. O `DELETE` remove o ciclo e tudo que está embaixo dele; as sessões lançadas ficam na conta, sem ciclo, como histórico do tópico. Um ciclo de outra pessoa responde `STUDY_CYCLE_NOT_OWNED` em todas as rotas, e os tópicos e páginas que uma requisição cita recebem as recusas do próprio caderno (`NOTEBOOK_PAGE_NOT_FOUND`, `NOTEBOOK_PAGE_NOT_OWNED`, e `NOTEBOOK_TOPIC_REQUIRED` para uma página que não é tópico).

### Hoje

O `GET /notebook/cycles/today` monta o dia local do dono para cada ciclo ativo que tem minutos naquele dia da semana, e o `StudyDaySlice` monta a parte de cada ciclo:

- Os minutos já estudados hoje nos tópicos do ciclo saem primeiro: pomodoros com data de hoje nas páginas deles, mais sessões com data de hoje. O orçamento do dia é o que o dia prevê menos isso, nunca abaixo de zero.
- Os blocos fechados hoje vêm primeiro na lista, como feitos.
- A partir do primeiro bloco aberto da volta, na ordem da fila, cada bloco aberto recebe o que ainda falta até o orçamento acabar.
- Quando a volta termina dentro do orçamento, a fatia continua na próxima volta a partir do primeiro bloco, marcada com `nextLap` e ainda sem progresso. Ela vira no máximo uma vez. Quem tem mais horas num dia do que duas voltas precisa de um ciclo maior, e uma lista mais comprida não ajudaria.
- Com horário de início, cada entrada ganha um intervalo de relógio. O cursor começa no horário de início mais o que já foi estudado hoje, então quem fez uma hora a partir das 19:00 vê o próximo bloco às 20:00.

Os ciclos com horário de início hoje vêm primeiro, por esse horário, e os outros mantêm a ordem deles. Um ciclo sem minutos hoje vai para `restingCycles` pelo nome, para a tela poder dizer que ele descansa. Cada entrada também traz os flashcards do tópico que vencem hoje.

O card de Hoje na home do caderno nos dois apps, o widget `studyToday` do dashboard e o `getStudyToday` do assistente leem essa rota. O Resumo do Dia lê a mesma fatia pelo `todayPreview`.

### Histórico e sessões lançadas

O `GET /notebook/cycles/{cycleId}/history` cobre no máximo 31 dias (`StudyCycleHistoryService`). Cada dia traz os minutos que a semana previa para aquele dia, os minutos estudados nos tópicos do ciclo com a divisão por tópico, e as entradas do dia em ordem de horário: FOCUS para um pomodoro, MANUAL para uma sessão lançada, UNDERSTOOD para um bloco que a página fechou, e LAP_CLOSED. Os minutos contam no dia em que aconteceram, seja qual for a volta onde caíram, então uma sessão lançada hoje para a terça passada aparece na terça. Os tópicos são os que o ciclo tem agora, ativos ou não, e por isso um ciclo arquivado também tem histórico.

O `POST /notebook/sessions` credita a sessão na hora, na volta atual, diga o `studiedOn` o que disser. A data só posiciona a sessão no histórico, e precisa ser o hoje do dono ou um dos 30 dias anteriores (`STUDY_SESSION_DATE`, conferido contra `UserDateResolver.today`). A linha guarda o `cycle_id` e o `lap_number` onde entrou, e o `DELETE /notebook/sessions/{sessionId}` usa os dois. Quando a sessão entrou na volta que ainda está correndo, os minutos dela voltam dos blocos do tópico, do último para trás, tirados só de blocos que têm minutos. Um bloco DONE ou TURN_USED que fica abaixo dos minutos previstos abre de novo. Um bloco UNDERSTOOD continua fechado, porque o conceito foi entendido, digam os minutos o que disserem. Uma sessão que entrou numa volta que já fechou só é apagada, e a volta fechada mantém os números. A sessão de outra pessoa responde `STUDY_SESSION_NOT_FOUND`.

### As telas, e o resto do Beyou

No web, o card de Hoje fica no topo de `/notebook`: para cada ciclo com minutos hoje, os blocos que cabem no dia com os horários, "Foco 25" e "Abrir" no bloco em que o dia está, e um jeito de lançar uma sessão. Quem não tem ciclo vê um convite para montar um, e um dia de descanso é uma linha só. `/notebook/cycles` reúne os ciclos ativos. Um bloco abre um diálogo com as configurações dele, salvas num PATCH, e os blocos sobem e descem com um PUT da ordem inteira. O editor em `/notebook/cycles/new` e `/notebook/cycles/:cycleId/edit` mostra a resposta do planejador enquanto a pessoa mexe na semana e nos tópicos. O histórico em `/notebook/cycles/:cycleId/history` desenha uma semana de minutos empilhados por tópico contra a meta de cada dia, e uma sessão lançada pode ser apagada nas entradas do dia. O celular tem as mesmas três telas em `app/notebook-cycles.tsx`, `app/notebook-cycle-edit.tsx` e `app/notebook-cycle-history.tsx`, e o card de Hoje na home do caderno. No web, o diálogo de lançar sessão mostra antes onde os minutos vão cair, percorrendo o ciclo carregado do mesmo jeito que o servidor credita (`previewCredit`), e o aviso depois lê a resposta do servidor.

Os dois apps guardam o que carregaram no slice `studyCycles` (a fatia de hoje, a lista, cada ciclo aberto por inteiro). O `refreshStudyCycles` lê tudo de novo depois das escritas do próprio app e quando uma ferramenta do assistente avisa o domínio `studyCycles`. O `studyCycleView` em `@beyou/state` escreve um bloco para as duas plataformas, "Entender Crase" para um conceito e o nome do tópico nos outros casos, com o número ao lado, para o card, a tela do ciclo, o widget e o resumo dizerem a mesma coisa.

A página Hoje do Resumo do Dia lista cada ciclo com minutos no dia, o horário de início e os três primeiros blocos abertos (`TodayAhead.study`). O servidor monta o título de cada bloco no idioma da conta, porque os clientes mostram o texto do jeito que chega. O resumo de ontem ganha os minutos estudados nos tópicos dos ciclos ativos e os conceitos entendidos. Um ciclo com minutos hoje, ou estudo ontem, faz o resumo valer a pena de mostrar, e o narrador recebe uma linha com esses fatos junto do que já está na tela. O assistente tem quatro ferramentas para ciclos, listadas no [tópico do agente de IA](/architecture/ai-agent).

## O timer de foco

O "Foco 25 min" numa página do caderno inicia o pomodoro único do app, o mesmo timer que a tela de foco usa. O estado `FocusTimer` carrega um `notebookPageId` opcional, mantido na passagem para a pausa, e o `PomodoroOwner` do web e do mobile reporta o ciclo com ele. O botão recusa enquanto um ciclo está rodando ou pausado em qualquer lugar do app, porque substituir esse ciclo jogaria fora minutos que alguém está no meio de usar, e o ciclo substituído nunca seria reportado. Essa regra é o `notebookFocusStart` em `@beyou/state`, e os dois apps só despacham o que ele devolve. O web saiu sem essa checagem no começo, e por isso ela foi parar lá. O servidor grava isso em `focus_cycles.notebook_page_id` depois de checar a posse, e o `focusMinutes` de uma página soma os minutos de POMODORO feitos nela e em toda página cuja casa fica abaixo dela. Pausas não contam. Apagar a página põe a coluna em null e mantém o ciclo. O mesmo registro também credita o ciclo de estudos do tópico, quando algum ciclo tem o tópico (veja "Ciclos de estudo" acima).

O servidor não faz check-in nenhum. Quando o tópico está vinculado a um hábito que está na rotina de hoje, o timer pré-seleciona aquele item da rotina, a tela de foco abre nele e o check continua a um toque, nas mãos da pessoa.

## Privacidade

As anotações são tão pessoais quanto o diário de humor e recebem o mesmo tratamento. O slice `notebook` está na blacklist de persistência do web ao lado de `mood`, e o mobile guarda tudo em memória, então nenhuma página chega ao disco do aparelho. O `PERSIST_VERSION` não mudou por causa da blacklist em si, porque uma chave na blacklist nunca existiu no estado persistido. Ele foi para 7 por algo que as primeiras versões erraram: o timer de foco, que persiste, também guardava o título da página ao lado do id. Nada mostrava esse título, e era o título de uma anotação de estudo no localStorage. Agora o timer guarda só o id, e a migração v7 apaga qualquer título já guardado.

O `GET /user/export` leva cada página como texto puro e os quadros que as organizam, com posições e ligações, porque um quadro é um layout que a pessoa montou. Os flashcards vão com as datas de vencimento e cada revisão feita neles, as fontes com seus detalhes, e os chats e as saídas da sala de estudo com as notas dos quizzes, além dos rascunhos de roteiro. O texto lido das fontes aparece em `notIncluded` como `notebookSourceText`: é uma cópia de documentos que a pessoa já tem, e um livro inteiro por fonte enterraria o resto do arquivo. Toda tabela do caderno tem cascade a partir de `users`, então a exclusão da conta leva o caderno inteiro.

Os ciclos de estudo recebem o mesmo tratamento no aparelho. O slice `studyCycles` é dado do servidor lido sob demanda, então fica na blacklist de persistência do web ao lado de `notebook` e vive em memória no celular; o `PERSIST_VERSION` não mudou por causa dele. A exportação leva cada ciclo em `notebook.studyCycles`, os arquivados inclusive, com a semana, os tópicos e os pesos deles, os blocos, o progresso de cada bloco em cada volta e as voltas fechadas. As sessões lançadas vão em `notebook.studySessions`, com o ciclo e a volta onde entraram. É uma consulta por tabela, tenha a pessoa quantos ciclos tiver. `study_cycles` e `study_sessions` têm cascade direto de `users`, e as outras cinco tabelas vêm em cascade por `study_cycles`, então apagar a conta leva as sete. Apagar um tópico tira o tópico do ciclo, junto com os blocos e as sessões dele, e uma página apagada deixa sem página o bloco ou a linha de volta que apontava para ela.

## Testes

No backend, as regras puras têm testes unitários em `unit/notebook/` (`ProgressGraphTest`, `SpacedRepetitionTest`, `LinkFetcherTest`, `BlockTextTest` e outros), e `integration/notebook/` roda os services contra o Postgres, incluindo escritas concorrentes, a ingestão de fontes e o `StudyBoardEditor` do assistente. Seis specs e2e dirigem a stack: `notebook.spec.ts` para o caminho de UI do tópico ao nó concluído, `notebook-rules.spec.ts` para posse, pagar uma vez, a recusa de ciclo, SSRF e escopo de fontes direto na API, `notebook-study.spec.ts` para o XP da revisão e as telas de IA, incluindo o Shift+Enter da caixa de pergunta e as anotações ao lado do chat recebendo uma resposta salva sem nenhum salvamento recusado, com só as rotas do modelo simuladas, `notebook-agent.spec.ts`, que reproduz uma resposta do assistente sobre uma página aberta e confere que o quadro, a árvore, o título e o editor mostram a mudança, e `notebook-sync.spec.ts`, duas abas na mesma página: anotações acrescentadas a uma página aberta sobrevivem à próxima edição nela, parágrafos diferentes se juntam sem pergunta, o mesmo parágrafo pergunta, e `notebook-trail.spec.ts`, um roteiro editado do jeito do celular direto na API: um nó sem coordenadas na próxima célula livre, depois do nó indicado, com o bloco do quadro na página, e uma ordem que transforma o quadro num caminho só. O `NotebookConcurrentWritesIT` reproduz as corridas da revisão numa thread só, e `mergeDocuments`, `DocumentSync` e `editorDocument` têm testes unitários em `@beyou/state`. No mobile, o `notebook-editor-screen.test.tsx` põe um stub no lugar da web view e cobre a metade da tela na ponte: a página de onde o editor começa, salvar a partir da revisão dele, a folha de conflito, os comandos da barra de ferramentas e sair só depois que a última edição foi salva. O `notebook-page-actions.test.tsx` cobre renomear, página nova, excluir e tópico novo, o `notebook-trail.test.tsx` a folha do nó, um nó adicionado depois de outro e a reordenação, e o `notebook-cards.test.tsx` a aba de cartões: a contagem do dia, a resposta com um toque, rascunhar com IA, escrever, editar e excluir. O `notebook-study.test.tsx` cobre a sala de estudo: a primeira configuração, uma pergunta cuja resposta citada é salva na página e vira cartões, fontes adicionadas e uma desligada, um PDF do celular com a checagem de tamanho e um quiz corrigido e pago. O `notebook-draft.test.tsx` cobre um rascunho do formulário ao tópico (lido de novo até ficar pronto, as marcações salvas, o tópico criado a partir dos nós mantidos), uma mudança aplicada a um rascunho reaberto, um rascunho que falhou e os rascunhos na home.

O ciclo de estudos tem testes próprios. Em `unit/notebook/cycle`, o `StudyCyclePlannerTest` cobre a aritmética e a intercalação do planejador, o `StudyLapTest` o caminho do crédito com o transbordo e a sobra, o entendimento, a espera pelo quiz e o estorno, e o `StudyDaySliceTest` a fatia do dia com a virada e os horários. O `StudyCycleServiceIT` e o `StudyCycleApiIT`, em `integration/notebook`, rodam contra o Postgres: o índice único parcial, a posse, os ganchos de um pomodoro, de uma página concluída e de um quiz aprovado, a junção no replanejamento, os limites, o dia de hoje, o histórico e o formato das respostas. O `StudyCycleAssistantTest` cobre o lado do assistente, e o `DailyBriefingServiceIT` os campos de estudo do resumo. Dois specs e2e dirigem a stack. O `study-cycle-rules.spec.ts` trabalha direto na API: um ciclo ativo por tópico, a divisão do planejador, um pomodoro que enche um bloco, transborda para o próximo e fecha a volta, um conceito que passa para o próximo bloco do tópico e é entendido quando a página fica feita, uma sessão lançada creditada e depois estornada, a fatia de hoje com os horários, e o ciclo de outra pessoa recusado. O `study-cycle.spec.ts` passa pela UI: um ciclo montado no editor, o card de Hoje, uma sessão lançada a partir dele caindo no primeiro bloco aberto do tópico, e o widget do dashboard. Nos clientes, o `studyCycleView` tem testes unitários em `@beyou/state`, as telas, os diálogos e o card de Hoje do ciclo no web têm testes de componente, e o celular testa as três telas, o card de Hoje e o widget em `__tests__/`.
