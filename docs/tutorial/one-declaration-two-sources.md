# Part 1: One declaration, two sources

This is the first part of a Clauseway tutorial over the cargo shipping
domain the DDD community has discussed to death. By the end, you will
have declared one relation, filled it from memory and a database, and
asked it questions in both directions.

You need Java and SQL. The two essays before this, on [why the stack
exists](../blog/posts/the-half-of-the-tar-pit-we-skipped.md) and [how it
works in broad strokes](../blog/posts/nothing-clever.md), explain the
motivation, but nothing here depends on them.

We will start at what may seem like the wrong end of the engine: its
data. Every program needs something to reason over, and Clauseway has a
named way to provide it.

## Declaring a relation

``` java
static External<?> cargo(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin) {
    return Literal.relation(Cargo.class, "cargo")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .from(db);
}
```

This is what the hexagon's port looks like from inside the engine. It
declares an `External` relation named `cargo` with two arguments. Its
storage is deliberately opaque: the `AnswerSource` may be in memory or
backed by JDBC.

Its arguments are `Unifiable` because they have no fixed input/output
direction. A value says "I know this"; a variable says "tell me this."
The same relation can therefore be asked in either direction.

## Providing facts

``` java
// Two cargos, as dddsample seeds them: ABC123 from Hong Kong, JKL567 from Hangzhou.
AnswerSource db = AnswerStore.empty().asserting(
        cargo(null, lval("ABC123"), lval("CNHKG")),
        cargo(null, lval("JKL567"), lval("CNHGH")));
```

`Unifiable` has two important implementations: `LVal` and `LVar`.
`lval(...)` provides a known value; `lvar()` creates an unknown. Facts
contain knowns, so both arguments above are `lval`s.

The store is immutable, so each assertion returns a new one, and each
fact reuses the same `cargo` relation to provide its values.

There is one wart worth naming: the `AnswerSource` argument is `null`. A
fact needs the relation's shape but does not read from a source, while
the declaration requires one. This is the least of the evils I
considered; the alternatives cost heavily in ergonomics. I assure you
that I share your hatred of nulls, so if you see any, it means that I
mourn over their sites.

## Asking questions

``` java
// Where does ABC123 come from? The tracking id is known, the origin is unknown.
Unifiable<String> origin = lvar();
Query.of(cargo(db, lval("ABC123"), origin))
        .select(origin).stream()
        .map(row -> row.get(origin))
        .collect(toList());                      // ["CNHKG"]
```

`lval("ABC123")` says "I know the tracking id"; `origin = lvar()` says
"tell me the origin." `.select(origin)` returns rows keyed by the
selected unknowns, and `row.get(origin)` extracts the typed value.

Turn the question around and the relation does not change:

``` java
// Which cargos come from Hangzhou?
Unifiable<String> trackingId = lvar();
Query.of(cargo(db, trackingId, lval("CNHGH")))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // ["JKL567"]
```

Now the origin is known and the tracking id is unknown. This is the
first meaning of "relational" here: the model has no input/output
direction and can be queried either way, just like SQL.

SQL already has this property. The interesting part of Clauseway is
keeping it when we leave stored facts and start writing domain logic.

## The same over Postgres

``` java
Connection connection = DriverManager.getConnection(
        "jdbc:postgresql://localhost/cargo", "cargo", "cargo");

try (CachingSqlFetch db = CachingSqlFetch.pinned("cargo", connection)) {
   // Any previously shown query
   // The connection is automatically closed after the try-with-resources block
}
```

At this point the central idea of Part 1 is complete: **one declaration,
either source, either direction.**

The rest is boundary machinery: schema, indexes and JDBC codecs.

## The strings attached

Here is the declaration again:

``` java
static External<?> cargo(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin) {
    return Literal.relation(Cargo.class, "cargo")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .from(db);
}
```

The declaration names a relation and its arguments. How it is
interpreted depends on the source. For `CachingSqlFetch`, it means a
table named `cargo` with two non-null columns named `trackingId` and
`origin`. The database is the schema authority, so the table must
already exist:

``` sql
-- A table named like the relation, columns named like its arguments. Nothing else is configured.
CREATE TABLE cargo(trackingId VARCHAR(16) NOT NULL, origin VARCHAR(5) NOT NULL);
INSERT INTO cargo VALUES ('ABC123', 'CNHKG'), ('JKL567', 'CNHGH');
```

`AnswerStore`, which also underlies `CachingSqlFetch`, uses `.indexed()`
to prepare for lookup patterns. It speeds repeated queries when the
engine can deduce a specific value for that argument before lookup.

Custom types used with `CachingSqlFetch` need codecs. Consider another
cargo relation:

``` java
static External<?> routeSpecification(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin,
        Unifiable<String> destination, Unifiable<Instant> arrivalDeadline) {
    return Literal.relation(Cargo.class, "routeSpecification")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .arg("destination", destination)
            .arg("arrivalDeadline", arrivalDeadline)
            .from(db);
}
```

Its Postgres table follows the same convention:

``` sql
CREATE TABLE routeSpecification(trackingId VARCHAR(16) NOT NULL, origin VARCHAR(5) NOT NULL,
                                destination VARCHAR(5) NOT NULL, arrivalDeadline TIMESTAMP WITH TIME ZONE NOT NULL);
INSERT INTO routeSpecification VALUES ('ABC123', 'CNHKG', 'FIHEL', TIMESTAMP WITH TIME ZONE '2026-03-15 00:00:00Z');
```

`Instant` needs a PostgreSQL-specific codec:

``` java
// An Instant crosses the wire as a java.sql.Timestamp. The two functions are the two directions.
static final Codec<Instant> INSTANT = Codec.of(Instant.class, Timestamp.class, Timestamp::from, Timestamp::toInstant);

try (CachingSqlFetch db = CachingSqlFetch.pinned("routing", connection)
        .withCodec(routeSpecification(null, lvar(), lvar(), lvar(), INSTANT.arg()))) {
    Unifiable<Instant> deadline = lvar();
    Query.of(routeSpecification(db, lval("ABC123"), lvar(), lvar(), deadline))
            .select(deadline).stream()
            .map(row -> row.get(deadline))
            .collect(toList());              // [2026-03-15T00:00:00Z]
}
```

The codec is bound by position: `INSTANT.arg()` marks the column that
uses it, while the `lvar()` arguments mean "use the default." Without a
codec, Clauseway uses the JDBC driver's object as-is. The declared Java
type is not checked at the boundary; a mismatch fails at the first typed
use.

## What you wrote

Look back at what you actually built: one declaration. The relation did not change when the
facts moved from memory to a table, and it did not change when the
question turned around. It names the relation and its columns without
saying how they are stored or which direction they will be queried.

You may object that this is SQL with extra steps, and at this layer that
is fair. A relation over a table is a table. The reason to declare it
this way appears when the columns stop merely being read and start being
reasoned over.

We'll get there, but I'll need some runway. The next part introduces
unification, derived relations and negation.
