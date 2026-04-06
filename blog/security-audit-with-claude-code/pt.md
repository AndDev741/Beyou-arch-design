---
title: "Como Usei o Claude Code para Encontrar 26 Vulnerabilidades de Seguranca Antes do Deploy em Producao"
summary: "Antes de colocar o Beyou em producao, fiz uma auditoria de seguranca adversarial usando o Claude Code como meu pen tester assistido por IA. Ele encontrou 26 vulnerabilidades em toda a stack — desde falhas de logica de negocio no sistema de gamificacao ate configuracoes incorretas de infraestrutura. Aqui esta o processo completo: o prompt, a metodologia, o que foi encontrado e como corrigi tudo em 3 grupos estruturados."
---

O Beyou estava com todas as funcionalidades prontas. Os habitos funcionavam, as rotinas rastreavam XP, o loop de gamificacao estava bom, e os 9 temas estavam bonitos. Eu estava pronto para fazer o deploy. Mas antes de colocar usuarios reais no app, eu tinha uma pergunta que nao conseguia responder com confianca: **esse app e realmente seguro?**

Eu sou um desenvolvedor full-stack, nao um engenheiro de seguranca. Conheco o basico — fazer hash de senhas, usar HTTPS, nao colocar secrets no frontend — mas tambem sei que existe uma diferenca entre "acho que esta tudo bem" e "eu realmente testei." Auditorias de seguranca de empresas especializadas custam milhares e levam semanas. Eu nao tinha esse orcamento nem esse prazo.

Entao tentei algo diferente: usei o **Claude Code** como um auditor de seguranca adversarial para todo o codebase.

## Por Que uma Auditoria Assistida por IA?

A ideia nao era substituir um teste de penetracao profissional. Era conseguir uma primeira passagem completa — o tipo de revisao que pega os erros obvios (e nao tao obvios) antes que um especialista humano comecasse.

O que torna o Claude Code interessante para isso e que ele consegue ler todo o seu codebase, entender a arquitetura, seguir fluxos de dados entre camadas e pensar como um atacante. Nao e apenas pattern matching para `eval()` ou secrets hardcoded. Ele consegue raciocinar sobre logica de negocio — coisas como "o que acontece se um usuario enviar esse ID que pertence a outra pessoa?" ou "e se esse campo inteiro nao tiver limite maximo?"

Dei acesso total ao monorepo do Beyou: o backend Spring Boot, o frontend React/Redux, a infraestrutura Docker Compose e as specs de API.

## O Prompt

E aqui que a magica acontece. A qualidade da auditoria depende inteiramente de como voce instrui a IA. Eu nao disse apenas "encontre bugs de seguranca." Escrevi um prompt detalhado que enquadra o Claude Code em um papel especifico, fornece um framework de analise estruturado e diz exatamente qual saida eu espero.

Aqui esta o prompt completo (e longo — e intencional):

> You are a Senior Application Security Engineer and Penetration Tester with 20+ years of experience conducting security audits for production web applications, with deep expertise in OWASP standards, threat modeling, and secure software development lifecycle (SSDLC). You have a track record of finding critical vulnerabilities before malicious actors do.
>
> Your task is to conduct a comprehensive, adversarial security review of a full-stack habit manager application called "BeYou". You are not a developer checking boxes — you are a security specialist who thinks like an attacker and protects like a guardian.

Depois defini 9 areas de analise para cobrir sistematicamente:

1. **Autenticacao e Gerenciamento de Sessao** — implementacao JWT, estrategia de refresh token, protecao contra brute force, politica de senha, enumeracao de contas
2. **Autorizacao e Controle de Acesso** — BOLA/IDOR, autorizacao a nivel de funcao quebrada, mass assignment, escalacao de privilegios
3. **Validacao de Input e Injecao** — SQL injection, XSS, SSTI, path traversal, ReDoS
4. **Seguranca de API** — politica CORS, rate limiting, mass assignment, enforcement de metodos HTTP
5. **Seguranca de Dados e Privacidade** — tratamento de PII, criptografia em repouso/transito, dados sensiveis em logs, conformidade GDPR
6. **Seguranca do Frontend** — headers CSP, armazenamento de tokens, exposicao de estado Redux, source maps
7. **Infraestrutura e Configuracao** — gerenciamento de secrets, seguranca Docker, interfaces admin expostas, CVEs de dependencias
8. **Seguranca de Logica de Negocio** — manipulacao de XP/gamificacao, condicoes de corrida, ataques de valor negativo, bypass de workflow
9. **Prontidao para Resposta a Incidentes** — logging de seguranca, deteccao de intrusao, revogacao de sessao

Para cada area, dei coisas especificas para verificar. Por exemplo, em Logica de Negocio:

> XP/gamification manipulation: can users forge XP gains, bypass habit checks, or manipulate level-up logic from the client? Race conditions in critical operations (double-submitting habit completions for double XP). Negative value attacks: what happens with negative habit counts, XP values, or progress?

Tambem especifiquei o formato exato de saida — IDs de achados no estilo CVE, pontuacoes CVSS, cenarios de prova de conceito de ataque, um roadmap de remediacao com prioridades por tempo, e um veredito final de deploy/nao-deploy.

Restricoes-chave que inclui:

> Think like an attacker first, then a defender — what would you target in the first 10 minutes? Never mark something as "low risk" just because it requires authentication — authenticated attacks are the most common in real breaches. Flag any finding that could violate GDPR given the app handles EU user personal data.

Tambem forneci as credenciais de teste e o endpoint da API live para que ele pudesse fazer requisicoes HTTP reais e confirmar os achados contra a aplicacao em execucao — nao apenas adivinhar pelo codigo-fonte.

A chave foi o enquadramento: **"You are not a developer checking boxes — you are a security specialist who thinks like an attacker."** Isso mudou toda a analise de "esse codigo segue boas praticas?" para "como eu quebraria isso?" — uma perspectiva muito mais valiosa.

**A licao: quanto mais estruturado e especifico seu prompt, mais estruturada e acionavel a saida.** Prompts vagos geram resultados vagos. Um framework de analise detalhado gera um relatorio detalhado.

## A Metodologia

O Claude Code nao fez apenas grep por palavras-chave. Ele realizou uma analise em camadas:

1. **Revisao de codigo-fonte** — lendo cada controller, service, filter e arquivo de configuracao. Seguindo os dados desde a requisicao HTTP ate a query no banco e de volta.

2. **Analise de configuracao** — verificando `application.yaml`, arquivos Docker Compose, configuracao do Spring Security, CORS e politicas de cookies.

3. **Testes na API live** — fazendo requisicoes HTTP reais para confirmar vulnerabilidades. Por exemplo, testou o CORS enviando uma requisicao com `Origin: https://evil-attacker.com` e verificando se o servidor aceitava com credenciais.

4. **Raciocinio de logica de negocio** — entendendo o modelo de gamificacao (formula de XP, niveis, sequencias) e perguntando "como alguem poderia explorar isso?"

5. **Analise cross-layer** — encontrando problemas que so aparecem quando voce olha como o frontend e o backend interagem. Coisas como um campo que o frontend nunca envia mas o backend aceita, ou um header de resposta que so funciona por causa de um comportamento especifico de uma biblioteca.

A saida foi um relatorio estruturado com cada achado categorizado por severidade (Critico, Alto, Medio, Baixo), com uma pontuacao CVSS e uma recomendacao de correcao especifica.

## O Que Foi Encontrado

Dos 26 achados, vou destacar os que mais me ensinaram — o tipo de problema que voce nao encontra em um code review tipico.

### Falhas de Logica de Negocio na Gamificacao

Esta foi a categoria mais reveladora. O sistema de XP do Beyou funciona assim: quando voce marca um habito na sua rotina diaria, voce ganha XP baseado em uma formula (`10 x dificuldade x importancia`). Parece simples. Mas o Claude Code encontrou duas formas de explorar:

**Dificuldade e importancia sem limites** — O endpoint de criacao de habito aceitava qualquer inteiro para dificuldade e importancia. O frontend mostrava um dropdown de 1-5, mas um atacante poderia chamar a API diretamente com `difficulty: 1000, importance: 1000` e ganhar 10.000.000 de XP por check. A correcao foi direta: anotacoes `@Min(1) @Max(5)` no DTO com `@Valid` no controller.

**Nivel de experiencia como inteiros brutos** — Ao criar um habito, usuarios podem selecionar seu nivel de experiencia (Iniciante, Intermediario, Avancado). O frontend mapeava esses para pares fixos de XP/nivel, mas o backend aceitava inteiros brutos. Alguem poderia comecar um habito no nivel 100. Substituimos os inteiros por um enum `ExperienceLevel` no lado do servidor que mapeia cada tier para seus valores corretos.

```java
public enum ExperienceLevel {
    BEGINNER(0, 0),
    INTERMEDIARY(5, 750),
    ADVANCED(8, 1800);
    // servidor resolve os valores reais
}
```

### Lacunas de Autorizacao e Propriedade

O Claude Code tracou o fluxo de dados pelo endpoint de check de rotina e descobriu que, embora a requisicao incluisse um parametro `userId`, o service **nunca o usava** para verificar propriedade. Qualquer usuario autenticado poderia marcar habitos na rotina de qualquer outro usuario — e o XP seria concedido ao atacante, nao ao dono da rotina.

Era um IDOR classico (Insecure Direct Object Reference), mas era sutil porque o `userId` estava na assinatura do metodo — parecia que estava sendo usado. Voce tinha que ler a implementacao cuidadosamente para notar que era silenciosamente ignorado.

A correcao foi um metodo `verifyRoutineOwnership()` que carrega a rotina e compara o dono com o usuario autenticado antes de qualquer operacao prosseguir.

### Endurecimento de Infraestrutura

Varios achados eram sobre prontidao para producao em vez de exploits ativos:

- O **Spring Actuator** estava expondo todos os endpoints sem autenticacao — health, metrics, variaveis de ambiente, ate heap dumps. Restringimos para `health,metrics,prometheus` e vinculamos ao `localhost`.

- **Dashboards de monitoramento** estavam acessiveis sem autenticacao. Adicionamos credenciais adequadas.

- **Senhas de banco hardcoded** no arquivo de configuracao principal foram substituidas por variaveis de ambiente, com um validador de startup que rejeita configuracoes inseguras quando rodando em modo producao.

### As Discussoes Que Mudaram Achados

Nem tudo que o Claude Code sinalizou era realmente uma vulnerabilidade. Esta e uma parte importante do processo — **revisar os achados criticamente**.

Por exemplo, ele sinalizou a configuracao CORS como critica porque o padrao wildcard (`*`) estava sendo usado. Mas apos discutir, percebemos que a origem CORS e externalizada via variavel de ambiente — `*` e o padrao do dev, e producao usa o dominio real via AWS secrets. A arquitetura estava correta; apenas adicionamos um guard de startup que rejeita wildcard no profile `prod` do Spring.

O mesmo com bindings de portas Docker — as portas do banco de dados e gerenciamento estavam expostas em todas as interfaces, mas apenas nos arquivos Docker Compose usados para desenvolvimento local. Producao usa infraestrutura AWS completamente diferente. Rebaixamos esses de "Alto" para "Informativo."

Essas discussoes melhoraram a precisao do relatorio e me ensinaram a distinguir entre **vulnerabilidades de codigo** e **configuracoes de contexto de deploy**.

## A Estrategia de Remediacao

Com 26 achados para corrigir, eu precisava de um plano. Corrigir coisas aleatoriamente seria caotico e arriscaria regressoes. Entao organizei a remediacao em 3 grupos:

### Grupo 1: Fundacao e Criticos (7 itens)
As coisas que devem ser corrigidas antes de qualquer deploy. Isso incluiu a correcao do IDOR, o problema do logging AOP, bloqueio do actuator, e — mais importante — **Spring Profiles**. Criamos `application-dev.yaml` e `application-prod.yaml` para separar claramente as configuracoes de dev e producao, com um `SecurityConfigValidator` que falha rapido no startup se as configuracoes de producao forem inseguras.

Essa fundacao tornou tudo mais facil porque agora podiamos dizer "no dev isso e ok, em prod isso e forcado."

### Grupo 2: Logica de Negocio e Endurecimento de Auth (9 itens)
Rate limiting, correcao do bypass de data, o enum ExperienceLevel, limites de dificuldade, protecao CSRF do OAuth, verificacao de email no registro, politica de senha mais forte (12 caracteres, 2 classes de caracteres) e externalizacao de seguranca de cookie.

O fluxo de verificacao de email foi a maior adicao — usando Spring Events com `@Async` para enviar emails de verificacao bilingues sem bloquear a resposta do registro.

### Grupo 3: Compliance e Endurecimento (10 itens)
Headers de seguranca (CSP, Referrer-Policy, Permissions-Policy), endpoint de exportacao de dados GDPR, limpeza do redux-persist, comparacao de secret em tempo constante, remocao de source maps, escaneamento de vulnerabilidades de dependencias (plugin OWASP + npm audit), e constraint unica nos checks de habito para prevenir condicoes de corrida.

Cada correcao foi commitada individualmente com uma mensagem clara referenciando o ID do achado (ex: `fix(security): add ownership verification to check/skip endpoints — CVE-BYU-003`). Isso tornou o historico do git uma trilha de auditoria rastreavel.

## Decisoes Arquiteturais Chave

Algumas correcoes foram simples de uma linha. Outras exigiram decisoes arquiteturais:

**Rate limiting em tiers com Bucket4j** — Em vez de um unico rate limit global, implementamos 4 tiers: endpoints de auth (5 requisicoes por 15 minutos por IP), escritas de dominio (30/min por usuario), leituras de dominio (60/min por usuario) e docs (30/min por IP). O filtro roda em `@Order(1)` e e condicionalmente desabilitado em ambientes de teste.

**Spring Profiles como backbone de seguranca** — O `SecurityConfigValidator` roda no `@PostConstruct` e rejeita configuracoes inseguras em producao: CORS wildcard, JWT secret curto, cookie security desabilitado. Isso significa que mesmo se alguem esquecer de setar uma variavel de ambiente, o app nao inicia em vez de rodar de forma insegura.

**Exportacao self-service GDPR** — Em vez de construir um painel de admin complexo, adicionamos um simples endpoint `GET /user/export` que retorna todos os dados do usuario (perfil, categorias, habitos, metas, tarefas) como JSON estruturado. Autenticado, somente leitura, e rapido gracas a camada de cache Caffeine existente.

## Os Numeros

| Metrica | Valor |
|---------|-------|
| Total de achados | 26 |
| Criticos | 4 |
| Altos | 8 |
| Medios | 10 |
| Baixos / Informativos | 4 |
| Tempo de auditoria | ~2 horas (com Claude Code) |
| Tempo de remediacao | ~1 dia (3 grupos) |
| Novos arquivos de teste | 12 |
| Linhas alteradas | ~2.500 em 3 repos |

## O Que Aprendi

**Seguranca e uma mentalidade diferente.** Escrever funcionalidades e sobre "como isso funciona?" Seguranca e sobre "como isso quebra?" Pedir ao Claude Code para pensar como um atacante mudou minha perspectiva de maneiras que revisar meu proprio codigo nunca faria.

**A maioria das vulnerabilidades sao bugs de logica, nao exploits tecnicos.** Ninguem ia fazer SQL injection no Beyou — JPA parametriza tudo. Mas a logica de negocio em torno de XP, propriedade e limites de entrada? E ai que os riscos reais estavam.

**Separacao dev/prod e fundamental.** Metade dos achados "criticos" acabaram sendo configuracoes apenas de dev. Uma vez que tinhamos Spring Profiles adequados, a superficie de risco real de producao era muito menor do que o relatorio inicial sugeria.

**Auditoria assistida por IA nao e substituto — e um acelerador.** O Claude Code encontrou coisas que eu nao teria pensado em verificar, e encontrou rapido. Mas a fase de discussao — onde eu expliquei minha arquitetura e refinamos os achados juntos — foi igualmente valiosa. A ferramenta e mais poderosa quando voce a trata como um colaborador, nao como um oraculo.

Se voce esta construindo um app e pensando em fazer deploy, eu recomendo fortemente rodar esse tipo de auditoria antes. O custo de encontrar uma vulnerabilidade antes do lancamento e algumas horas de trabalho. O custo de encontrar depois e muito maior.
