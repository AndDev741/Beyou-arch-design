---
title: "Como eu identifico e resolvo o problema N+1 no Beyou"
summary: "O problema N+1 é comum em apps que dependem de JPA e Hibernate. Parei pra investigar como eles aparecem, como identificar e como resolver."
---

## O que é o problema N+1

Esse problema é super comum em apps que dependem de ORMs pra abstrair as queries SQL. Vou usar a primeira entidade do Beyou onde eu encontrei isso: a `Habit`.

A entidade `Habit` tem esses campos e relações:

```java
public class Habit {
    private UUID id;

    private String name;

    private String description;

    private String iconId;

    @ManyToMany
    @JoinTable(
    name = "habit_category",
    joinColumns = @JoinColumn(name = "habit_id"),
    inverseJoinColumns = @JoinColumn(name = "category_id"))
    private List<Category> categories;

    @JsonIgnore
    @ToString.Exclude
    @OneToMany(mappedBy = "habit", cascade = CascadeType.ALL, orphanRemoval = false)
    private List<HabitGroup> habitGroups;

    private Integer importance;

    private Integer dificulty;

    private String motivationalPhrase;

    private XpProgress xpProgress = new XpProgress();

    private int constance;

    private Date createdAt;

    private Date updatedAt;

    private User user;
}
```

Removi algumas annotations pra ficar mais fácil de ler, mas esses são os campos. O que importa aqui são duas `List`s: `categories` e `habitGroups`. Categories é um `@ManyToMany`, habitGroups é um `@OneToMany`.

A pegadinha é que o Hibernate esconde da gente como o SQL realmente fica por trás de um método tipo `findAllByUserId`. Por padrão, toda `List` em uma entidade é **lazy**, só carrega quando você acessa. Então quando o service itera cada habit e lê `habit.getCategories()` pra montar a resposta, o Hibernate dispara um `SELECT` extra pra cada habit. Uma query vira 1 + N queries, e você não faz ideia disso a menos que vá olhar.

> N+1 literalmente significa que você está fazendo N queries extras em cima da que você queria fazer, e por fora ainda parece uma chamada só.

Se eu habilito o log de SQL no profile de teste, é isso que aparece:

```sql
-- O 1
SELECT * FROM habit WHERE user_id = ?
-- O N (uma por habit, em cada iteração do loop)
SELECT * FROM habit_category WHERE habit_id = ?
SELECT * FROM habit_category WHERE habit_id = ?
SELECT * FROM habit_category WHERE habit_id = ?
... (N vezes)
```

Com 10 habits por usuário isso é chato. Com 100, o dashboard demora meio segundo pra carregar. Com 1.000+ vira um problema sério, e é o tipo de coisa que você só percebe em produção.

---

## Como eu provei o problema

Antes de consertar qualquer coisa eu precisava medir de verdade. Senão tô só chutando.

Escrevi testes de integração usando `EntityManagerFactory` e `EntityManager` pra ler as métricas da sessão do Hibernate. Rodo eles contra um PostgreSQL real usando Testcontainers, então o banco do teste é o mesmo que o de produção. A outra peça é habilitar `hibernate.generate_statistics: true` no profile de teste, senão os contadores ficam zerados e não servem pra nada.

Pra não ficar feio de usar nos testes, escrevi essa classe helper:

```java
public class HibernateStatistics {

    private final Statistics stats;

    /**
     * Cria um snapshot no escopo do teste atual. Os contadores são zerados na
     * construção pra que o chamador veja só as queries disparadas a partir
     * daqui.
     */
    public HibernateStatistics(EntityManagerFactory emf) {
        this.stats = emf.unwrap(SessionFactory.class).getStatistics();
        this.stats.setStatisticsEnabled(true);
        this.stats.clear();
    }

    /** Número de prepared statements JDBC enviados ao banco. Sinal primário de N+1. */
    public long statementCount() {
        return stats.getPrepareStatementCount();
    }

    /** Número de entidades hidratadas a partir do result set. */
    public long entityLoadCount() {
        return stats.getEntityLoadCount();
    }

    /** Número de fetches em coleções {@code @OneToMany}/{@code @ManyToMany}. */
    public long collectionFetchCount() {
        return stats.getCollectionFetchCount();
    }

    /** Número de queries JPQL/HQL/Criteria executadas. */
    public long queryExecutionCount() {
        return stats.getQueryExecutionCount();
    }

    /** Dump diagnóstico pra mensagens de falha. */
    @Override
    public String toString() {
        return String.format(
                "statements=%d, entityLoads=%d, collectionFetches=%d, queries=%d",
                statementCount(), entityLoadCount(), collectionFetchCount(), queryExecutionCount());
    }
}
```

Dos quatro contadores, o que realmente importa pra detectar N+1 é o `statementCount`, toda ida e volta JDBC pro banco. Os outros três ajudam a diagnosticar *por que* a contagem tá alta (muito `entityLoadCount` significa que cada linha disparou um fetch; `collectionFetchCount` alto significa que uma `List` tá sendo inicializada num loop).

Então escrevi o teste:

```java
@Test
@DisplayName("getHabits deve buscar N habits + categories em um número limitado de queries")
void getHabits_isBoundedRegardlessOfHabitCount() {
    // Arrange: cria user, categories e habits no banco
    User user = seedUser();
    List<Category> categories = seedCategories(user, 3);
    for (int i = 0; i < HABIT_COUNT; i++) {
        seedHabitWithCategories(user, "habit-" + i, categories);
    }
    entityManager.flush();
    entityManager.clear(); // descarta o cache L1 pra leitura ir no banco "fria"

    var stats = new HibernateStatistics(emf);

    // Act: mesmo caminho que o controller usa em produção
    var result = habitService.getHabits(user.getId());

    // Assert de correção
    assertThat(result).hasSize(HABIT_COUNT);
    assertThat(result.get(0).categories()).hasSize(3);

    // Assert de performance: a correção deve produzir um número pequeno e
    // limitado de queries, NÃO 1 + N. O limite de 5 cobre a query de habits,
    // o join/fetch das categories e um pouco de overhead fixo.
    assertThat(stats.statementCount())
            .as("Deveria ser limitado. N+1 daria ~%d statements. Stats: %s",
                    HABIT_COUNT + 1, stats)
            .isLessThanOrEqualTo(5);

    System.out.println("[N+1 fix] HabitService.getHabits com " + HABIT_COUNT + " habits → " + stats);
}
```

Um detalhe que me pegou: o `entityManager.clear()` é crítico. Sem ele, as entidades que eu acabei de criar ainda estão no cache de primeiro nível (L1) da mesma transação, então o `findAllByUserId` retorna elas sem nem ir no banco, e o N+1 não aparece. O `clear()` simula uma request nova.

Quando rodei o teste com `HABIT_COUNT = 10`, isso é o que saiu:

```
statements=13, entityLoads=14, collectionFetches=11, queries=1
```

13 statements pra 10 habits. Cada habit disparou um `SELECT` separado em `habit_category`. O N+1 era real, não só teoria.

---

## Como eu corrigi

Existem três ferramentas principais, e a escolha certa depende de como sua entidade tá montada.

Pra `categories`, a opção mais simples é `@EntityGraph` no método do repository. Isso diz pro Spring Data JPA fazer um único `LEFT JOIN` e trazer as categories junto com as habits numa tacada só:

```java
public interface HabitRepository extends JpaRepository<Habit, UUID> {
    @EntityGraph(attributePaths = {"categories"})
    ArrayList<Habit> findAllByUserId(UUID userId);
}
```

Pra `habitGroups` precisei de outra abordagem. Se eu tentasse adicionar no mesmo `@EntityGraph`, o Hibernate cuspiria `MultipleBagFetchException`.

> Um "bag" no Hibernate é uma `List` mapeada sem `@OrderColumn`. Regra do Hibernate: você só pode fazer fetch de UM bag por query, porque juntar duas `List`s sem limite produz um produto cartesiano (10 habits × 3 categories × M habitGroups vira um pesadelo rapidinho).

Por design, `habitGroups` é uma lista pequena nesse app, uma habit só aparece em algumas seções de rotina, nunca centenas. Então `@BatchSize` cai bem:

```java
@JsonIgnore
@ToString.Exclude
@OneToMany(mappedBy = "habit", cascade = CascadeType.ALL, orphanRemoval = false)
@BatchSize(size = 10)
private List<HabitGroup> habitGroups;
```

`@BatchSize(10)` diz pro Hibernate: quando um proxy de habitGroup for inicializado, dá uma olhada no contexto de persistência atrás dos irmãos e carrega todos numa query só usando `IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`.

Rodei o teste de novo, e agora:

```
statements=2, entityLoads=13, collectionFetches=1, queries=1
```

De 13 statements pra 2. Com 10 habits eliminei 11 idas ao PostgreSQL. Com 100 habits a diferença seria ~101 vs ainda 2. O custo para de escalar com N.

---

## Quando usar o quê

Se você tá olhando pra um N+1 no seu próprio código, esse é o resumo que eu carrego na cabeça agora:

| Situação | Ferramenta |
|----------|------------|
| Uma coleção lazy na entidade, e você sempre precisa dela carregada | `@EntityGraph(attributePaths = {"x"})` no método do repository |
| Várias `List<>` na mesma entidade (cai no `MultipleBagFetchException` se tentar JOIN nas duas) | `@BatchSize(size = N)` em cada uma |
| Fetch profundamente aninhado (3+ níveis de associação) | `JOIN FETCH` num `@Query` customizado, pra ter mais controle |

E se quiser monitorar em produção, `generate_statistics: true` é caro demais pra deixar ligado. O caminho lá é log de slow query no banco ou ferramentas de APM (Datadog, New Relic) que contam queries por request. Mas pra teste é exatamente a ferramenta certa.

---

A lição que eu sempre volto: lazy loading não é o bug. Iterar sobre coleções lazy sem perceber é o bug. Quando você consegue medir, consegue corrigir. E quando você tem um teste afirmando "esse endpoint não pode passar de 5 statements", você ganha uma proteção contra regressão que vai te salvar o dia que alguém introduzir outra chamada `getXxx()` dentro de um loop.
