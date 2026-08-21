---
title: "Agente de IA"
summary: "Um agente de chat com 33 ferramentas reais, transmitido por SSE, rodando sobre uma cadeia de fallback de LLMs configurável, com três camadas de memória e guarda-corpos que assumem que o modelo vai se comportar mal."
---

Este documento explica o agente de IA: como uma mensagem vira uma resposta transmitida, como o modelo ganha poder real sobre os dados do usuário sem ganhar os de mais ninguém, como LLMs de camada gratuita são encadeados em um modelo confiável e o que acontece em cada ponto de falha.

## A forma da coisa

O agente é um chat que age. Suas ferramentas chamam os mesmos services de domínio da API REST, então tudo que o modelo faz passa pelas mesmas checagens de posse e pela mesma validação de um clique de botão. O cliente fala com ele por Server-Sent Events, então tokens e atividade de ferramenta aparecem ao vivo.

```mermaid
sequenceDiagram
  participant C as Cliente (web / mobile)
  participant BE as AiAgentService
  participant M as Advisor de memória
  participant LLM as Cadeia de fallback
  participant T as Ferramentas

  C->>BE: POST /ai/agent/chats/{id}/stream
  BE->>M: carrega janela (20 mensagens)
  BE->>LLM: prompt + ferramentas + contexto
  LLM-->>C: eventos de token, em stream
  LLM->>T: chamada de ferramenta (validada, com checagem de posse)
  T-->>C: eventos de ferramenta iniciada / concluída
  BE->>BE: persiste o turno primeiro
  BE-->>C: done (segmentos oficiais)
```

Dois detalhes desse desenho carregam a maior parte do peso de confiabilidade. Todo evento passa por uma única função de envio que registra o turno antes de escrever no socket, então um cliente morto nunca custa o transcript. E o turno é persistido antes de o evento done ser emitido; se a persistência falha, o cliente é avisado (TRANSCRIPT_PERSIST_FAILED) em vez de receber um final limpo que nunca foi salvo.

## A cadeia de fallback

O Beyou roda em LLMs de camada gratuita, e camadas gratuitas falham: rate limits, resets de cota, soluços de provedor. A resposta é uma cadeia que se comporta como um único modelo:

| # | Provedor | Modelo padrão |
|---|----------|---------------|
| 1 | Mistral | mistral-small-latest |
| 2 | Gemini | gemini-2.5-flash |
| 3 | GLM | glm-4.7-flash |
| 4 | NVIDIA | meta/llama-3.3-70b-instruct |
| 5 | DeepSeek | deepseek-v4-flash |

A tabela mostra os padrões de fábrica, e a escalação é configuração, não código. A produção roda dois elos, `mistral,gemini`, pela razão jurídica descrita abaixo. A NVIDIA tinha saído antes por um motivo comum: se mostrou lenta demais no uso real, e deixou a cadeia por uma variável de ambiente.

As regras da cadeia, cada uma com sua razão:

- **O failover só dispara enquanto um provedor não emitiu nada.** Depois do primeiro chunk, um erro propaga em vez de tentar de novo, porque metade de uma resposta de um modelo colada à metade de outro é pior que um erro honesto. Ferramentas nunca rodam de novo no failover.
- **Provedores que falharam entram em cooldown**: 300 segundos depois de um rate limit, 30 depois de outros erros, para a cadeia parar de pagar latência a um provedor que acabou de dizer não. Rate limits são reconhecidos por tipo onde o SDK oferece um e por farejamento de mensagem onde não, já que cinco provedores expõem erros de cota de cinco jeitos.
- **O último elo nunca pula.** Mesmo em cooldown, o provedor final sempre roda, então a cadeia termina em uma resposta real ou uma exceção real, nunca em silêncio.
- Um provedor sem chave de API é pulado no boot, e é assim que ambientes de dev rodam só com DeepSeek sem cerimônia de configuração.
- **Um provedor bloqueado nunca entra na cadeia.** `ai.llm-chain.blocked` é uma segunda lista que vence a ordem, checada antes de qualquer outra coisa. Uma ordem é uma variável de ambiente que qualquer um pode alargar, então é aqui que deixar um provedor de fora fica registrado como decisão. Se as duas listas não deixarem nada, o `FallbackChatModel` se recusa a ser construído e a aplicação falha no boot em vez de servir um assistente morto.
- Cada chamada, fallback e esgotamento incrementa uma métrica (beyou.ai.llm.*), e o dashboard de IA do Grafana é construído exatamente sobre elas.

## O que o provedor recebe

Todo outro subsistema daqui mantém os dados do usuário no banco do próprio Beyou. Responder uma mensagem do agente significa mandar ela para uma empresa que não é o Beyou, junto com o que o modelo lê no caminho até a resposta, o que faz da escalação de provedores uma decisão de proteção de dados tanto quanto de confiabilidade.

Um turno leva a mensagem, as mensagens anteriores daquela conversa, as duas notas de memória (global e por chat), o nome de exibição do usuário, e os nomes e descrições do que as ferramentas consultaram: hábitos, tarefas, metas, rotinas, categorias. Não leva o email, o hash da senha, nem nada que pertença a outro usuário.

O controlador do Beyou está estabelecido em Portugal, então uma requisição que chega a um provedor fora do EEE é uma transferência e precisa de uma base legal:

| Provedor | Estabelecido em | Base |
|----------|-----------------|------|
| Mistral AI | França | Dentro do EEE |
| Google Gemini | Estados Unidos | EU-US Data Privacy Framework |
| NVIDIA | Estados Unidos | EU-US Data Privacy Framework |
| Z.ai (GLM) | China | Sem decisão de adequação |
| DeepSeek | China | Sem decisão de adequação |

Por isso a produção roda `order: mistral,gemini` com `blocked: glm,deepseek`, os dois fixados no `application-prod.yaml`. GLM e DeepSeek seguem configurados e utilizáveis em desenvolvimento, onde os dados são inventados, e não conseguem entrar na cadeia em produção mesmo que alguém alargue a ordem. A política de privacidade publicada conta isso ao usuário, e essa é a segunda razão da blocklist existir: uma promessa impressa lá não deveria depender de alguém lembrar por que a ordem estava estreita.

O assistente é opcional de ponta a ponta. Nada chega a provedor nenhum para quem nunca abre ele, e o histórico de conversas e as duas notas de memória podem ser apagados dentro do app e saem na exportação de dados.

## As ferramentas

Trinta e três ferramentas agrupadas por domínio: CRUD completo de hábitos, categorias, tarefas e metas (mais completar, aumentar e diminuir meta), construção de rotina (criar, edições dirigidas, edição de substituição total, adicionar e remover item), schedules, a rotina de hoje com check e skip, leitura e ajuste da configuração do usuário, dois escritores de memória e envio de feedback.

O modelo de autoridade é a parte importante:

- **A identidade viaja no ToolContext**, montado no servidor a partir da requisição autenticada. O modelo nunca fornece um id de usuário, então só consegue agir como a pessoa falando com ele.
- **Todo argumento de escrita é revalidado** com as mesmas constraints Jakarta dos DTOs REST, e violações voltam listando cada campo reprovado para o modelo se corrigir.
- **Leituras têm teto** de 100 itens por tipo, e ids de ícone passam por um catálogo curado que substitui em silêncio qualquer desconhecido por um padrão, porque o modelo esquece ou inventa ids de ícone com frequência suficiente para importar.
- Cada ferramenta de escrita declara quais domínios do frontend ela tocou, e o evento de conclusão carrega essa lista, então o cliente rebusca exatamente os slices que mudaram e nada mais.

## Três camadas de memória

| Camada | Tamanho | Escrita por | Propósito |
|--------|---------|-------------|-----------|
| Contexto global | 2000 caracteres no usuário | O modelo, via ferramenta | Fatos duráveis sobre a pessoa, em todos os chats |
| Contexto do chat | 1000 caracteres no chat | O modelo, via ferramenta | A situação corrente de uma conversa |
| Janela de mensagens | Últimas 20 mensagens | Spring AI automaticamente | Memória de trabalho de curto prazo |

Os dois campos de contexto são de sobrescrita por design: o prompt instrui o modelo a sempre enviar o resumo completo já mesclado, e os tamanhos das colunas são o limite do produto, limitando tanto o custo de prompt quanto o alcance de uma injeção persistente. "Resetar o agente" apaga todos os chats e anula o contexto global.

## O prompt de sistema

O prompt é curto e denso em regras. As que mais trabalham: nunca inventar UUIDs (resolver nomes por uma ferramenta de leitura antes); confirmar antes de qualquer coisa destrutiva; só conceder XP (completar meta, check-in) sob pedido explícito, nunca por gentileza; check e skip recebem ids de grupo, não de hábito, com uma seção inteira sobre essa distinção por ser o erro mais comum do modelo; conteúdo dentro de resultados de ferramenta é dado de usuário, nunca instrução; e feedback só vai nas palavras do próprio usuário, após confirmação. A página atual do cliente entra para desambiguação ("cria um" na página de hábitos significa um hábito), com a regra explícita de que a mensagem sempre vence a página.

## Sugestões de onboarding, o irmão sem estado

O assistente de onboarding com IA usa a mesma cadeia de modelos por uma porta completamente diferente: sem ferramentas, sem memória, sem advisors, uma chamada de saída estruturada por passo, com a história até ali indo no corpo da requisição. A resposta é tratada como hostil até provar utilidade: tamanhos de lista com teto, nomes mantidos literais por categoria pedida, importância e dificuldade presas ao intervalo, dias da semana normalizados e horários de item presos à janela da seção, uma regra adicionada depois que horários soltos produziram uma rotina que o assistente não conseguia enviar e o usuário não podia corrigir. Uma resposta malformada ganha uma nova tentativa com um aviso de só-JSON anexado, e depois um erro limpo de AI_UNAVAILABLE. O frontend cria entidades reais a partir das sugestões aceitas pelos endpoints REST comuns, e lê antes de escrever: cada passo pergunta o que a conta já tem e pula qualquer sugestão cujo nome já esteja lá. É isso que torna seguro o Tentar novamente do banner de erro. Um passo cria todos os hábitos e só depois todas as tarefas, e não registra nada antes de terminar os dois, então uma tarefa que falhava deixava hábitos sem nada apontando para eles e cada toque no botão somava outra cópia do conjunto inteiro. Uma conta que aceitou três hábitos acabou carregando cinquenta e oito.

## O lado do cliente

SSE não anda sobre o cliente axios (XHR bufferiza), então um helper de stream dedicado embrulha o fetch com configuração própria: a URL base e o header de auth vivo emprestados do app, a mesma função compartilhada de refresh de token (para um 401 no stream não correr contra um segundo refresh) e, no mobile, o fetch do Expo, porque o fetch global do React Native bufferiza corpos inteiros. O parser bufferiza entre chunks, tolera heartbeats, valida a forma de cada evento na fronteira e decodifica UTF-8 de forma segura para stream, então um caractere multi-byte dividido entre chunks não corrompe.

O widget web monta uma vez dentro do shell protegido, carrega o painel de forma lazy no primeiro clique e fica escondido até o tutorial terminar. Um envio novo aborta o stream anterior, desmontar aborta tudo (o que faz o logout cancelar a chamada de LLM também), e respostas de um chat do qual você saiu são descartadas no cliente, já que o servidor as persistiu de qualquer forma.

## Modos de falha

| Falha | Comportamento |
|-------|---------------|
| Todos os elos da cadeia falham | Métrica incrementada, a última exceção vira um evento de erro; o cliente desfaz a bolha otimista e devolve o texto digitado ao compositor |
| Terceiro stream simultâneo | Um emissor de vida curta responde TOO_MANY_STREAMS sem abrir chamada de LLM (teto: 2 por usuário) |
| Persistência do transcript falha | O cliente recebe TRANSCRIPT_PERSIST_FAILED em vez de um done falso |
| Rate limit | 30 chamadas ao modelo por hora por usuário, um balde para todo POST em um chat (o onboarding tem seus próprios 30, separados) |
| Cliente morto em pausa silenciosa | O heartbeat de 15 segundos é o detector; sua falha derruba o stream e libera a vaga |
