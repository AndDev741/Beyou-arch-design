---
title: "How I find and solve N+1 query problems in Beyou"
summary: "N+1 is a common problem in apps that rely on JPA and Hibernate. I stopped to investigate how they appear, how to identify them, and how to fix them."
---

## What is the N+1 problem

This problem is super common in apps that rely on ORMs to abstract SQL queries away from us. Let me use the first entity in Beyou where I found it: the `Habit`.

The `Habit` entity has these fields and relations:

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

I removed some annotations to make it easier to read, but those are the fields. Two `List`s are what matter here: `categories` and `habitGroups`. Categories is a `@ManyToMany` relationship, habitGroups is a `@OneToMany`.

The trap is that Hibernate hides what the SQL really looks like behind a method like `findAllByUserId`. By default, every `List` in an entity is **lazy**, only loaded when you actually access it. So when the service iterates each habit and reads `habit.getCategories()` to build the response, Hibernate fires one extra `SELECT` per habit. One query becomes 1 + N queries, and you have no idea unless you go look.

> N+1 literally means you are doing N extra queries on top of the one you wanted, and from the outside it still looks like one call.

If we enable SQL logging in the test profile, this is what shows up:

```sql
-- The 1
SELECT * FROM habit WHERE user_id = ?
-- The N (one per habit, on every loop iteration)
SELECT * FROM habit_category WHERE habit_id = ?
SELECT * FROM habit_category WHERE habit_id = ?
SELECT * FROM habit_category WHERE habit_id = ?
... (N times)
```

At 10 habits per user this is annoying. At 100, the dashboard takes half a second to load. At 1.000+ it gets bad fast, and that is the kind of thing you only see in production.

---

## How I proved the problem

Before fixing anything I needed to actually measure it. Otherwise I am guessing.

I wrote integration tests using `EntityManagerFactory` and `EntityManager` to read the metrics from the Hibernate session. I run them against real PostgreSQL using Testcontainers, so the test database engine is the same as production. The other piece is enabling `hibernate.generate_statistics: true` in the test profile, otherwise the counters are just zeroed-out no-ops.

To make this less ugly to use across tests, I wrote a small helper:

```java
public class HibernateStatistics {

    private final Statistics stats;

    /**
     * Creates a snapshot scoped to the calling test. Counters are reset on
     * construction so callers see only the queries triggered after this point.
     */
    public HibernateStatistics(EntityManagerFactory emf) {
        this.stats = emf.unwrap(SessionFactory.class).getStatistics();
        this.stats.setStatisticsEnabled(true);
        this.stats.clear();
    }

    /** Number of JDBC prepared statements sent to the database. Primary N+1 signal. */
    public long statementCount() {
        return stats.getPrepareStatementCount();
    }

    /** Number of entities hydrated from result sets. */
    public long entityLoadCount() {
        return stats.getEntityLoadCount();
    }

    /** Number of {@code @OneToMany}/{@code @ManyToMany} collection fetches. */
    public long collectionFetchCount() {
        return stats.getCollectionFetchCount();
    }

    /** Number of JPQL/HQL/Criteria queries executed. */
    public long queryExecutionCount() {
        return stats.getQueryExecutionCount();
    }

    /** Diagnostic dump for failure messages. */
    @Override
    public String toString() {
        return String.format(
                "statements=%d, entityLoads=%d, collectionFetches=%d, queries=%d",
                statementCount(), entityLoadCount(), collectionFetchCount(), queryExecutionCount());
    }
}
```

Of the four counters, the one that really matters for N+1 is `statementCount`, every JDBC round trip to the database. The other three help diagnose *why* the count is high (lots of `entityLoadCount` means each row triggered a fetch; high `collectionFetchCount` means a `List` is being initialized in a loop).

Then I wrote the test:

```java
@Test
@DisplayName("getHabits should fetch N habits + their categories in a bounded number of queries")
void getHabits_isBoundedRegardlessOfHabitCount() {
    // Arrange: seed user, categories, and habits
    User user = seedUser();
    List<Category> categories = seedCategories(user, 3);
    for (int i = 0; i < HABIT_COUNT; i++) {
        seedHabitWithCategories(user, "habit-" + i, categories);
    }
    entityManager.flush();
    entityManager.clear(); // discard L1 cache so reads hit the DB cold

    var stats = new HibernateStatistics(emf);

    // Act: same call path the controller uses in production
    var result = habitService.getHabits(user.getId());

    // Assert correctness
    assertThat(result).hasSize(HABIT_COUNT);
    assertThat(result.get(0).categories()).hasSize(3);

    // Assert performance: the fix should produce a small, bounded number of
    // queries, NOT 1 + N. Threshold of 5 covers the habits query, the
    // categories join/fetch, and a bit of fixed overhead.
    assertThat(stats.statementCount())
            .as("Should be bounded — N+1 would mean ~%d statements. Stats: %s",
                    HABIT_COUNT + 1, stats)
            .isLessThanOrEqualTo(5);

    System.out.println("[N+1 fix] HabitService.getHabits with " + HABIT_COUNT + " habits → " + stats);
}
```

One detail that bit me: `entityManager.clear()` is critical. Without it, the seeded entities are still living in the L1 cache from the same transaction, so `findAllByUserId` returns them without hitting the database, and the N+1 never shows up. The clear() call simulates a fresh request.

When I ran the test with `HABIT_COUNT = 10`, here is what came out:

```
statements=13, entityLoads=14, collectionFetches=11, queries=1
```

13 statements for 10 habits. Each habit triggered a separate `SELECT` on `habit_category`. The N+1 was real, not just theoretical.

---

## How I fixed it

There are three main tools, and the right one depends on what your entity looks like.

For `categories`, the simplest move is `@EntityGraph` on the repository method. This tells Spring Data JPA to do a single `LEFT JOIN` and bring categories along with the habits in one shot:

```java
public interface HabitRepository extends JpaRepository<Habit, UUID> {
    @EntityGraph(attributePaths = {"categories"})
    ArrayList<Habit> findAllByUserId(UUID userId);
}
```

For `habitGroups` I had to use a different approach. If I tried to add it to the same `@EntityGraph`, Hibernate throws `MultipleBagFetchException`.

> A "bag" in Hibernate is a `List` mapped without an `@OrderColumn`. Hibernate's rule: you can fetch at most ONE bag per query, because joining two unbounded `List`s produces a cartesian product (10 habits × 3 categories × M habitGroups gets very ugly very fast).

By design, `habitGroups` is small in this app, a habit only appears in a few routine sections, never hundreds. So `@BatchSize` is a clean fit:

```java
@JsonIgnore
@ToString.Exclude
@OneToMany(mappedBy = "habit", cascade = CascadeType.ALL, orphanRemoval = false)
@BatchSize(size = 10)
private List<HabitGroup> habitGroups;
```

`@BatchSize(10)` tells Hibernate: when one habitGroup proxy gets initialized, look around the persistence context for siblings and load them all in a single `IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)` query.

I ran the test again, and now:

```
statements=2, entityLoads=13, collectionFetches=1, queries=1
```

From 13 statements down to 2. With 10 habits we removed 11 round trips to PostgreSQL. With 100 habits, the difference would be ~101 vs still 2. The cost stops scaling with N.

---

## When to use what

If you are looking at an N+1 in your own code, this is the cheat sheet I keep in my head now:

| Situation | Tool |
|-----------|------|
| One lazy collection on the entity, you always need it loaded | `@EntityGraph(attributePaths = {"x"})` on the repository method |
| Multiple `List<>` collections on the same entity (hits `MultipleBagFetchException` if you try to JOIN both) | `@BatchSize(size = N)` on each |
| Deeply nested fetch (3+ levels of associations) | `JOIN FETCH` in a custom `@Query` for control |

And if you want to monitor it in production, `generate_statistics: true` is too expensive to leave on. The right move there is database-level slow query logs or APM tools (Datadog, New Relic) that count queries per request. But for tests it is exactly the right hammer.

---

The lesson I keep coming back to: lazy loading is not the bug. Iterating over lazy collections without realizing it is the bug. Once you can measure, you can fix. And once you have a test asserting "this endpoint must not exceed 5 statements", you have a regression guard that will save you the day someone introduces another `getXxx()` call inside a loop.
