---
title: "Como Usei o Claude Code para Encontrar 26 Vulnerabilidades de Segurança Antes do Deploy em Produção"
summary: "Antes de colocar o Beyou em produção, fiz uma auditoria de segurança adversarial usando o Claude Code como meu pen tester assistido por IA. Ele encontrou 26 vulnerabilidades em toda a stack — desde falhas de lógica de negócio no sistema de gamificação até configurações incorretas de infraestrutura. Aqui está o processo completo: o prompt, a metodologia, o que foi encontrado e como corrigi tudo em 3 grupos estruturados."
---

O Beyou estava com todas as funcionalidades prontas. Os hábitos funcionavam, as rotinas rastreavam XP, o loop de gamificação estava bom, e os 9 temas estavam bonitos. Eu estava pronto para fazer o deploy. Mas antes de colocar usuários reais no app, eu tinha uma pergunta que não conseguia responder com confiança: **esse app é realmente seguro?**

Eu sou um desenvolvedor full-stack, não um engenheiro de segurança. Conheço o básico — fazer hash de senhas, usar HTTPS, não colocar secrets no frontend — mas também sei que existe uma diferença entre "acho que está tudo bem" e "eu realmente testei." Auditorias de segurança de empresas especializadas custam milhares e levam semanas. Eu não tinha esse orçamento nem esse prazo.

Então tentei algo diferente: usei o **Claude Code** como um auditor de segurança adversarial para todo o codebase.

## Por Que uma Auditoria Assistida por IA?

A ideia não era substituir um teste de penetração profissional. Era conseguir uma primeira passagem completa — o tipo de revisão que pega os erros óbvios (e não tão óbvios) antes que um especialista humano começasse.

O que torna o Claude Code interessante para isso é que ele consegue ler todo o seu codebase, entender a arquitetura, seguir fluxos de dados entre camadas e pensar como um atacante. Não é apenas pattern matching para `eval()` ou secrets hardcoded. Ele consegue raciocinar sobre lógica de negócio — coisas como "o que acontece se um usuário enviar esse ID que pertence a outra pessoa?" ou "e se esse campo inteiro não tiver limite máximo?"

Dei acesso total ao monorepo do Beyou: o backend Spring Boot, o frontend React/Redux, a infraestrutura Docker Compose e as specs de API.

## O Prompt

É aqui que a mágica acontece. A qualidade da auditoria depende inteiramente de como você instrui a IA. Eu não disse apenas "encontre bugs de segurança." Escrevi um prompt detalhado que enquadra o Claude Code em um papel específico, fornece um framework de análise estruturado e diz exatamente qual saída eu espero.

Aqui está o prompt completo (é longo — é intencional):

> You are a Senior Application Security Engineer and Penetration Tester with 20+ years of experience conducting security audits for production web applications, with deep expertise in OWASP standards, threat modeling, and secure software development lifecycle (SSDLC). You have a track record of finding critical vulnerabilities before malicious actors do.
>
> Your task is to conduct a comprehensive, adversarial security review of a full-stack habit manager application called "BeYou". You are not a developer checking boxes — you are a security specialist who thinks like an attacker and protects like a guardian.

Depois defini 9 áreas de análise para cobrir sistematicamente:

1. **Autenticação e Gerenciamento de Sessão** — implementação JWT, estratégia de refresh token, proteção contra brute force, política de senha, enumeração de contas
2. **Autorização e Controle de Acesso** — BOLA/IDOR, autorização a nível de função quebrada, mass assignment, escalação de privilégios
3. **Validação de Input e Injeção** — SQL injection, XSS, SSTI, path traversal, ReDoS
4. **Segurança de API** — política CORS, rate limiting, mass assignment, enforcement de métodos HTTP
5. **Segurança de Dados e Privacidade** — tratamento de PII, criptografia em repouso/trânsito, dados sensíveis em logs, conformidade GDPR
6. **Segurança do Frontend** — headers CSP, armazenamento de tokens, exposição de estado Redux, source maps
7. **Infraestrutura e Configuração** — gerenciamento de secrets, segurança Docker, interfaces admin expostas, CVEs de dependências
8. **Segurança de Lógica de Negócio** — manipulação de XP/gamificação, condições de corrida, ataques de valor negativo, bypass de workflow
9. **Prontidão para Resposta a Incidentes** — logging de segurança, detecção de intrusão, revogação de sessão

Para cada área, dei coisas específicas para verificar. Por exemplo, em Lógica de Negócio:

> XP/gamification manipulation: can users forge XP gains, bypass habit checks, or manipulate level-up logic from the client? Race conditions in critical operations (double-submitting habit completions for double XP). Negative value attacks: what happens with negative habit counts, XP values, or progress?

Também especifiquei o formato exato de saída — IDs de achados no estilo CVE, pontuações CVSS, cenários de prova de conceito de ataque, um roadmap de remediação com prioridades por tempo, e um veredito final de deploy/não-deploy.

Restrições-chave que incluí:

> Think like an attacker first, then a defender — what would you target in the first 10 minutes? Never mark something as "low risk" just because it requires authentication — authenticated attacks are the most common in real breaches. Flag any finding that could violate GDPR given the app handles EU user personal data.

Também forneci as credenciais de teste e o endpoint da API live para que ele pudesse fazer requisições HTTP reais e confirmar os achados contra a aplicação em execução — não apenas adivinhar pelo código-fonte.

A chave foi o enquadramento: **"You are not a developer checking boxes — you are a security specialist who thinks like an attacker."** Isso mudou toda a análise de "esse código segue boas práticas?" para "como eu quebraria isso?" — uma perspectiva muito mais valiosa.

**A lição: quanto mais estruturado e específico seu prompt, mais estruturada e acionável a saída.** Prompts vagos geram resultados vagos. Um framework de análise detalhado gera um relatório detalhado.

## A Metodologia

O Claude Code não fez apenas grep por palavras-chave. Ele realizou uma análise em camadas:

1. **Revisão de código-fonte** — lendo cada controller, service, filter e arquivo de configuração. Seguindo os dados desde a requisição HTTP até a query no banco e de volta.

2. **Análise de configuração** — verificando `application.yaml`, arquivos Docker Compose, configuração do Spring Security, CORS e políticas de cookies.

3. **Testes na API live** — fazendo requisições HTTP reais para confirmar vulnerabilidades. Por exemplo, testou o CORS enviando uma requisição com `Origin: https://evil-attacker.com` e verificando se o servidor aceitava com credenciais.

4. **Raciocínio de lógica de negócio** — entendendo o modelo de gamificação (fórmula de XP, níveis, sequências) e perguntando "como alguém poderia explorar isso?"

5. **Análise cross-layer** — encontrando problemas que só aparecem quando você olha como o frontend e o backend interagem. Coisas como um campo que o frontend nunca envia mas o backend aceita, ou um header de resposta que só funciona por causa de um comportamento específico de uma biblioteca.

A saída foi um relatório estruturado com cada achado categorizado por severidade (Crítico, Alto, Médio, Baixo), com uma pontuação CVSS e uma recomendação de correção específica.

## O Que Foi Encontrado

Dos 26 achados, vou destacar os que mais me ensinaram — o tipo de problema que você não encontra em um code review típico.

### Falhas de Lógica de Negócio na Gamificação

Esta foi a categoria mais reveladora. O sistema de XP do Beyou funciona assim: quando você marca um hábito na sua rotina diária, você ganha XP baseado em uma fórmula (`10 × dificuldade × importância`). Parece simples. Mas o Claude Code encontrou duas formas de explorar:

**Dificuldade e importância sem limites** — O endpoint de criação de hábito aceitava qualquer inteiro para dificuldade e importância. O frontend mostrava um dropdown de 1-5, mas um atacante poderia chamar a API diretamente com `difficulty: 1000, importance: 1000` e ganhar 10.000.000 de XP por check. A correção foi direta: anotações `@Min(1) @Max(5)` no DTO com `@Valid` no controller.

**Nível de experiência como inteiros brutos** — Ao criar um hábito, usuários podem selecionar seu nível de experiência (Iniciante, Intermediário, Avançado). O frontend mapeava esses para pares fixos de XP/nível, mas o backend aceitava inteiros brutos. Alguém poderia começar um hábito no nível 100. Substituímos os inteiros por um enum `ExperienceLevel` no lado do servidor que mapeia cada tier para seus valores corretos.

```java
public enum ExperienceLevel {
    BEGINNER(0, 0),
    INTERMEDIARY(5, 750),
    ADVANCED(8, 1800);
    // servidor resolve os valores reais
}
```

### Lacunas de Autorização e Propriedade

O Claude Code traçou o fluxo de dados pelo endpoint de check de rotina e descobriu que, embora a requisição incluísse um parâmetro `userId`, o service **nunca o usava** para verificar propriedade. Qualquer usuário autenticado poderia marcar hábitos na rotina de qualquer outro usuário — e o XP seria concedido ao atacante, não ao dono da rotina.

Era um IDOR clássico (Insecure Direct Object Reference), mas era sutil porque o `userId` estava na assinatura do método — parecia que estava sendo usado. Você tinha que ler a implementação cuidadosamente para notar que era silenciosamente ignorado.

A correção foi um método `verifyRoutineOwnership()` que carrega a rotina e compara o dono com o usuário autenticado antes de qualquer operação prosseguir.

### Endurecimento de Infraestrutura

Vários achados eram sobre prontidão para produção em vez de exploits ativos:

- O **Spring Actuator** estava expondo todos os endpoints sem autenticação — health, metrics, variáveis de ambiente, até heap dumps. Restringimos para `health,metrics,prometheus` e vinculamos ao `localhost`.

- **Dashboards de monitoramento** estavam acessíveis sem autenticação. Adicionamos credenciais adequadas.

- **Senhas de banco hardcoded** no arquivo de configuração principal foram substituídas por variáveis de ambiente, com um validador de startup que rejeita configurações inseguras quando rodando em modo produção.

### As Discussões Que Mudaram Achados

Nem tudo que o Claude Code sinalizou era realmente uma vulnerabilidade. Esta é uma parte importante do processo — **revisar os achados criticamente**.

Por exemplo, ele sinalizou a configuração CORS como crítica porque o padrão wildcard (`*`) estava sendo usado. Mas após discutir, percebemos que a origem CORS é externalizada via variável de ambiente — `*` é o padrão do dev, e produção usa o domínio real via AWS secrets. A arquitetura estava correta; apenas adicionamos um guard de startup que rejeita wildcard no profile `prod` do Spring.

O mesmo com bindings de portas Docker — as portas do banco de dados e gerenciamento estavam expostas em todas as interfaces, mas apenas nos arquivos Docker Compose usados para desenvolvimento local. Produção usa infraestrutura AWS completamente diferente. Rebaixamos esses de "Alto" para "Informativo."

Essas discussões melhoraram a precisão do relatório e me ensinaram a distinguir entre **vulnerabilidades de código** e **configurações de contexto de deploy**.

## A Estratégia de Remediação

Com 26 achados para corrigir, eu precisava de um plano. Corrigir coisas aleatoriamente seria caótico e arriscaria regressões. Então organizei a remediação em 3 grupos:

### Grupo 1: Fundação e Críticos (7 itens)
As coisas que devem ser corrigidas antes de qualquer deploy. Isso incluiu a correção do IDOR, o problema do logging AOP, bloqueio do actuator, e — mais importante — **Spring Profiles**. Criamos `application-dev.yaml` e `application-prod.yaml` para separar claramente as configurações de dev e produção, com um `SecurityConfigValidator` que falha rápido no startup se as configurações de produção forem inseguras.

Essa fundação tornou tudo mais fácil porque agora podíamos dizer "no dev isso é ok, em prod isso é forçado."

### Grupo 2: Lógica de Negócio e Endurecimento de Auth (9 itens)
Rate limiting, correção do bypass de data, o enum ExperienceLevel, limites de dificuldade, proteção CSRF do OAuth, verificação de email no registro, política de senha mais forte (12 caracteres, 2 classes de caracteres) e externalização de segurança de cookie.

O fluxo de verificação de email foi a maior adição — usando Spring Events com `@Async` para enviar emails de verificação bilíngues sem bloquear a resposta do registro.

### Grupo 3: Compliance e Endurecimento (10 itens)
Headers de segurança (CSP, Referrer-Policy, Permissions-Policy), endpoint de exportação de dados GDPR, limpeza do redux-persist, comparação de secret em tempo constante, remoção de source maps, escaneamento de vulnerabilidades de dependências (plugin OWASP + npm audit), e constraint única nos checks de hábito para prevenir condições de corrida.

Cada correção foi commitada individualmente com uma mensagem clara referenciando o ID do achado (ex: `fix(security): add ownership verification to check/skip endpoints — CVE-BYU-003`). Isso tornou o histórico do git uma trilha de auditoria rastreável.

## Decisões Arquiteturais Chave

Algumas correções foram simples de uma linha. Outras exigiram decisões arquiteturais:

**Rate limiting em tiers com Bucket4j** — Em vez de um único rate limit global, implementamos 4 tiers: endpoints de auth (5 requisições por 15 minutos por IP), escritas de domínio (30/min por usuário), leituras de domínio (60/min por usuário) e docs (30/min por IP). O filtro roda em `@Order(1)` e é condicionalmente desabilitado em ambientes de teste.

**Spring Profiles como backbone de segurança** — O `SecurityConfigValidator` roda no `@PostConstruct` e rejeita configurações inseguras em produção: CORS wildcard, JWT secret curto, cookie security desabilitado. Isso significa que mesmo se alguém esquecer de setar uma variável de ambiente, o app não inicia em vez de rodar de forma insegura.

**Exportação self-service GDPR** — Em vez de construir um painel de admin complexo, adicionamos um simples endpoint `GET /user/export` que retorna todos os dados do usuário (perfil, categorias, hábitos, metas, tarefas) como JSON estruturado. Autenticado, somente leitura, e rápido graças à camada de cache Caffeine existente.

## Os Números

| Métrica | Valor |
|---------|-------|
| Total de achados | 26 |
| Críticos | 4 |
| Altos | 8 |
| Médios | 10 |
| Baixos / Informativos | 4 |
| Tempo de auditoria | ~2 horas (com Claude Code) |
| Tempo de remediação | ~1 dia (3 grupos) |
| Novos arquivos de teste | 12 |
| Linhas alteradas | ~2.500 em 3 repos |

## O Que Aprendi

**Segurança é uma mentalidade diferente.** Escrever funcionalidades é sobre "como isso funciona?" Segurança é sobre "como isso quebra?" Pedir ao Claude Code para pensar como um atacante mudou minha perspectiva de maneiras que revisar meu próprio código nunca faria.

**A maioria das vulnerabilidades são bugs de lógica, não exploits técnicos.** Ninguém ia fazer SQL injection no Beyou — JPA parametriza tudo. Mas a lógica de negócio em torno de XP, propriedade e limites de entrada? É aí que os riscos reais estavam.

**Separação dev/prod é fundamental.** Metade dos achados "críticos" acabaram sendo configurações apenas de dev. Uma vez que tínhamos Spring Profiles adequados, a superfície de risco real de produção era muito menor do que o relatório inicial sugeria.

**Auditoria assistida por IA não é substituto — é um acelerador.** O Claude Code encontrou coisas que eu não teria pensado em verificar, e encontrou rápido. Mas a fase de discussão — onde eu expliquei minha arquitetura e refinamos os achados juntos — foi igualmente valiosa. A ferramenta é mais poderosa quando você a trata como um colaborador, não como um oráculo.

Se você está construindo um app e pensando em fazer deploy, eu recomendo fortemente rodar esse tipo de auditoria antes. O custo de encontrar uma vulnerabilidade antes do lançamento é algumas horas de trabalho. O custo de encontrar depois é muito maior.
