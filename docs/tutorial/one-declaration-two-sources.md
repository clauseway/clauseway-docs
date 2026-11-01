# Part 1: One declaration, two sources

This is the first part of a tutorial on using Clauseway, over the cargo shipping domain the DDD community has discussed to death. By the end of it you will have declared one relation, filled it from memory and from a database, and asked it questions in both directions. You need Java and SQL; the two essays before this, on [why the stack exists](../blog/posts/the-half-of-the-tar-pit-we-skipped.md) and [how it works in broad strokes](../blog/posts/nothing-clever.md), explain the motivation, but nothing here depends on them. The place we'll start may seem backwards, because it's the engine's end, but every program needs its data to reason over, and Clauseway has a named way to do just that:

```java
static External<?> cargo(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin) {
    return Literal.relation(Cargo.class, "cargo")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .from(db);
}
```

This is what the hexagon's port looks like from inside of the engine. It declares a relation whose facts are owned `External`ly. The type of the storage is deliberately opaque, because it may be in-memory, or accessed via JDBC, and its arguments are wrapped in `Unifiable` because it treats knowns (inputs) and unknowns (outputs) uniformly. Let's look into how to provide facts:

```java
// Two cargos, as dddsample seeds them: ABC123 from Hong Kong, JKL567 from Hangzhou.
AnswerSource db = AnswerStore.empty().asserting(
        cargo(null, lval("ABC123"), lval("CNHKG")),
        cargo(null, lval("JKL567"), lval("CNHGH")));
```

Several things to note here:

- The store is immutable, so every assertion returns a new one.
- Every declared fact re-uses the cargo relation to provide values.
- Every provided value is wrapped in an `lval`.
- The `AnswerSource` argument to cargo is left as null. It is unused in fact statements, but cargo declaration requires it. This is the least of all the evils I considered and the alternatives cost heavily in ergonomics. I assure you that I share your hatred of nulls, so if you see any - it means that I mourn over their sites.

`Unifiable` is an interface with two important implementors - `LVal` and `LVar`. The former is used to provide literal values, knowns, while the other is used to ask questions. Since facts are the knowns, we use `lval`s. Now, onto asking questions:

```java
// Where does ABC123 come from? The tracking id is known, the origin is the unknown.
Unifiable<String> origin = lvar();
Query.of(cargo(db, lval("ABC123"), origin))
        .select(origin).stream()
        .map(row -> row.get(origin))
        .collect(toList());                      // ["CNHKG"]
```

Here, we've built a `Query` over cargo, stored in the in-memory store we just defined. We declared `origin` as the unknown and ask about trackingId `ABC123` with `lval("ABC123")`. The output is chosen with `.select(origin)`, which returns generic `Row`s that are keyed by unknowns. We extract the typed value of `origin` with `row.get(origin)` and collect it to a list. The reverse lookup is done with the same relation, but a different `Query`:

```java
// Which cargos come from Hangzhou? The same relation, the unknown on the other side.
Unifiable<String> trackingId = lvar();
Query.of(cargo(db, trackingId, lval("CNHGH")))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // ["JKL567"]
```

Here, the known is the location, `lval("CNHGH")`, and `trackingId` is unknown. This is what "relational" programming stands for - there is no direction in the model and it can be queried in any direction - just like SQL. At this point, you may want to actually use it over a database, so here it is, over Postgres:

```java
Connection connection = DriverManager.getConnection("jdbc:postgresql://localhost/cargo", "cargo", "cargo");

try (CachingSqlFetch db = CachingSqlFetch.pinned("cargo", connection)) {
   // Any previously shown query
   // The connection is automatically closed after the try-with-resources block
}
```

The catch is the things I skimmed over so far. Let's conjure back the original relation and discuss it in detail:

```java
static External<?> cargo(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin) {
    return Literal.relation(Cargo.class, "cargo")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .from(db);
}
```

The return statement builds a port declaration with two named arguments. It will be used to retrieve facts by the underlying source, but the meaning of this declaration is source-dependent. For `CachingSqlFetch` this is a table named `cargo` with two non-null columns: `trackingId` and `origin`. This table needs to exist in the database because it is the schema authority:

```sql
-- A table named like the relation, columns named like its arguments. Nothing else is configured.
CREATE TABLE cargo(trackingId VARCHAR(16) NOT NULL, origin VARCHAR(5) NOT NULL);
INSERT INTO cargo VALUES ('ABC123', 'CNHKG'), ('JKL567', 'CNHGH');
```

Another aspect to consider is indexing. `AnswerStore` (that also underlies `CachingSqlFetch`) will use `.indexed()` to assume lookup patterns. This will speed up repeated queries whenever the engine is able to deduce the specific value for this parameter before lookup.

Lastly, if you want to use custom types with CachingSqlFetch, you'll have to register codecs for them. Let's look at another one of cargo's relations:

```java
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

Nothing new to discuss so far. Here are the associated DDL statements to create the table in Postgres and add data:

```sql
CREATE TABLE routeSpecification(trackingId VARCHAR(16) NOT NULL, origin VARCHAR(5) NOT NULL,
                                destination VARCHAR(5) NOT NULL, arrivalDeadline TIMESTAMP WITH TIME ZONE NOT NULL);
INSERT INTO routeSpecification VALUES ('ABC123', 'CNHKG', 'FIHEL', TIMESTAMP WITH TIME ZONE '2026-03-15 00:00:00Z');
```

And here we register codecs for the Instant class. These are jdbc-driver dependent and the following ones are tailored towards Postgresql:

```java
// An Instant crosses the wire as a java.sql.Timestamp. The two functions are the two directions.
static final Codec<Instant> INSTANT = Codec.of(Instant.class, Timestamp.class, Timestamp::from, Timestamp::toInstant);

// The codec is bound to the column by position: the relation called with the codec's marker where the Instant is.
try (CachingSqlFetch db = CachingSqlFetch.pinned("routing", connection)
        .withCodec(routeSpecification(null, lvar(), lvar(), lvar(), INSTANT.arg()))) {
    Unifiable<Instant> deadline = lvar();
    Query.of(routeSpecification(db, lval("ABC123"), lvar(), lvar(), deadline))
            .select(deadline).stream()
            .map(row -> row.get(deadline))
            .collect(toList());              // [2026-03-15T00:00:00Z]
}
```

Here, we use the `.withCodec(routeSpecification(null, lvar(), lvar(), lvar(), INSTANT.arg()))` to tell Clauseway to use `INSTANT.arg()` codec for the required column. The `lvar()` columns mean "use the default". The default is the driver's object as it comes, and nothing checks it against the type you declared: a column without a codec is not refused, it fails at the first typed use. 

Look back at what you actually wrote: one declaration. It did not change when the facts moved from a list in memory to a table, and it did not change when the question turned around. It names the relation and its columns and says nothing about storage, direction or lookup. Everything that did change, the source, the codec, the index, was said beside it, to the source, never inside it.

You may object that so far this is SQL with extra steps, and at this layer that is fair. A relation over a table is a table. The reason to declare it this way shows up when the columns stop being read and start being reasoned over, and that is the next part: the cargo's itinerary, and the one rule that replaces two switches in the original.
