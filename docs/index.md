# Clauseway

**Rules over your database, stated once and asked in any direction.**
Clauseway is a logic programming library for Java 8 and later. A table
is a relation, a rule is a relation written in terms of others, and the
same rule answers whichever of its columns you leave open.

## One rule, two questions

A cargo travels in legs, and the legs are rows of a table. The first
declaration reads that table. The second is a rule over it: a cargo's
last leg is a leg with no leg after it.

``` java
// A stored relation: the table `leg`, one argument per column.
static External<?> leg(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index,
        Unifiable<String> from, Unifiable<String> to) {
    return Literal.relation(LastLeg.class, "leg")
            .arg("trackingId", trackingId).indexed()
            .arg("index", index)
            .arg("loadLocation", from)
            .arg("unloadLocation", to)
            .from(db);
}

// A rule: a cargo's last leg is a leg with no leg after it.
static Literal<?> lastIndex(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar();
    return Literal.relation(LastLeg.class, "lastIndex")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, index, lvar(), lvar())
                    .and(Ints.addo(index, lval(1), next))
                    .and(exclude(leg(db, trackingId, next, ignored(), ignored()))));
}
```

Give the rule a cargo and leave the index open, and it finds the last
leg. Give it an index and leave the cargo open, and it finds the cargos:

``` java
// Which leg of ABC123 is the last one?
Unifiable<Integer> index = lvar();
Query.of(lastIndex(db, lval("ABC123"), index))
        .select(index).stream()
        .map(row -> row.get(index))
        .collect(toList());                      // [1]

// Which cargos travel on a single leg?
Unifiable<String> trackingId = lvar();
Query.of(lastIndex(db, trackingId, lval(0)))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // [JKL567]
```

Over PostgreSQL, the value you gave becomes the `WHERE` clause. The
first question sends one statement:

``` sql
SELECT index, loadLocation, unloadLocation FROM leg WHERE trackingId = ?
```

The second sends this one, then one lookup for each cargo it found, to
check that no leg follows:

``` sql
SELECT trackingId, loadLocation, unloadLocation FROM leg WHERE index = ?
```

Nothing in the rule says which column is the input.

## Try it

``` xml
<dependency>
    <groupId>org.clauseway</groupId>
    <artifactId>pldb</artifactId>
    <version>0.3.0</version>
</dependency>
```

The two declarations above run as they stand over an in-memory store,
with no database to set up:

``` java
AnswerSource db = AnswerStore.empty().asserting(
        leg(null, lval("ABC123"), lval(0), lval("CNHKG"), lval("USNYC")),
        leg(null, lval("ABC123"), lval(1), lval("USNYC"), lval("FIHEL")),
        leg(null, lval("JKL567"), lval(0), lval("CNHGH"), lval("DEHAM")));
```

The [tutorial](tutorial/index.md) builds this up from the first
declaration, and moves the same relations onto PostgreSQL in Part 1.

## What it is, vs what you know

- **vs a query builder.** A query builder writes one query in one
  direction. A Clauseway rule is written once and the question decides
  the direction; rules build on other rules, and may be recursive.
- **vs a rule engine.** A rule engine reacts: when the facts change,
  its rules fire and act. Clauseway answers: a rule states what holds,
  the facts stay in your database, and nothing runs until a question is
  asked.
- **vs Prolog and Datalog.** It is a library, not a language: rules
  are Java methods and columns are typed Java values. The engine is of
  the miniKanren family, with finite-domain constraints, negation and
  tabling.

## The stack

| Part                                                      | What it is                                                                                                                                           |
|-----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| [**functional**](https://github.com/clauseway/functional) | The substrate: fibers, fair schedulers, continuations — the machinery that keeps relational search complete and debuggable.                          |
| [**logic**](https://github.com/clauseway/logic)           | The engine: unification, constraint stores (finite domains, disequality, nogoods), tabling with constraint-aware answer caching, weighted inference. |
| [**pldb**](https://github.com/clauseway/pldb)             | The data boundary: relations backed by in-memory or SQL sources, constraint pushdown into WHERE clauses, transactions that certify their reads.      |
| [**library**](https://github.com/clauseway/library-test)  | The worked example: a lending-library domain built entirely from rules — policies as relations, denials as their complement.                         |
| [**cargo**](https://github.com/clauseway/cargo)           | Eric Evans' DDD cargo shipping domain over the Clauseway stack.                                                                                      |

## Why

Most business logic is relations wearing imperative costume. Clauseway
exists to let the relations speak for themselves: a rule is stated once
and runs in every direction — computing due dates, inferring checkout
days, or refusing inconsistent data — over types that matter and data
that lives in a real database. The full argument is the opening essay:
[**The half of the Tar Pit we skipped**](blog/posts/the-half-of-the-tar-pit-we-skipped.md).

## Where to start

- The [tutorial](tutorial/index.md) — the features one at a time, over
  the cargo domain, every listing run as a test.
- The [blog](blog/index.md) — development notes along the way, opening
  with the essay that explains why this stack exists.
- The repositories above — each part builds with plain Maven.
