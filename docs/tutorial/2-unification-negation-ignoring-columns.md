# Part 2: Unification, negation, and ignoring columns

Part 1 ended with a promise: declaring a relation pays off when its
columns are reasoned over, not merely read. This part is the runway
for that. By the end you will know what a query means, and you will
have written a relation in terms of another, denied one, and seen
the one way a denial asks a different question than you meant.

Nothing new is stored. The `cargo` relation and the two cargos from
Part 1 are still there, and every example below asks them something.
If you skipped Part 1, its first section is all you need.

We start smaller than that, with two variables and a string.

## Unification, and, or

Before we build anything more impressive, we need to step back and
learn some basics. First, we need to learn how to say that things are
equal, and in relational programming the name for that is
unification. This is how a variable value gets deduced:

``` java
Unifiable<String> first = lvar();
Query.of(first.unifies("Hello"))
        .select(first).stream()
        .map(row -> row.get(first))
        .collect(toList());              // ["Hello"]
```
What `Query.of` takes is called a goal: an invariant all answers must satisfy. 
A unification is the simplest goal, and there are two ways to combine
them into a bigger one. They can be declared to hold together, in conjunction:

``` java
Unifiable<String> first = lvar();
Unifiable<String> second = lvar();
Query.of(first.unifies("Hello").and(second.unifies("world!")))
        .select(first, second).stream()
        .map(row -> row.get(first) + " " + row.get(second))
        .collect(toList());              // ["Hello world!"]
```

Whenever the engine finds an `and`, it will make sure that both sides
of it hold in a single answer. Here, it is visible in two ways: we
select both `first` and `second`, and we read them from a single row.
When solutions are collected, we only have one, and it is the mapped
`Hello world!`. If the two sides cannot both hold, this line of
reasoning is abandoned and produces no answer:

``` java
Unifiable<String> first = lvar();
Query.of(first.unifies("Hello").and(first.unifies("world!")))
        .select(first).stream()
        .map(row -> row.get(first))
        .collect(toList());              // []
```

Here the engine fails because you specified that `first` is `Hello`
**and** `world!`, which is a contradiction. The second connective is
`or`, disjunction, and it has different rules:

``` java
Unifiable<String> first = lvar();
Query.of(first.unifies("Hello").or(first.unifies("world!")))
        .select(first).stream()
        .map(row -> row.get(first))
        .collect(toList());              // ["Hello", "world!"]
```

Whenever the engine finds an `or`, it will branch and reason about
the two sides separately. They are allowed to succeed or fail
independently, and they may produce distinct answers that obey the
rules declared in each branch. Here, it is again seen in two ways: we
have a single variable, `first`, that we select, and only `first` is
in the row. However, it produces two distinct answers, `Hello` and
`world!`. Because the engine reasons about the branches separately,
there is no contradiction between them.

Of course, we can mix the two connectives to create more complicated
logic:

``` java
Unifiable<String> first = lvar();
Unifiable<String> second = lvar();
Query.of(first.unifies("Hello").and(second.unifies("world!"))
        .or(first.unifies("Goodbye").and(
                second.unifies("clauseway!").or(second.unifies("reader!"))))
        .or(first.unifies("this").and(first.unifies("fails"))))
        .select(first, second).stream()
        .map(row -> row.get(first) + " " + row.get(second))
        .collect(toList());              // ["Hello world!", "Goodbye clauseway!", "Goodbye reader!"]
```

A more contrived example, with branch nesting and failure. The first
answer is, unsurprisingly, `Hello world!`, like in the previous
examples. The second alternative holds two branches, `Goodbye` with
`clauseway!` or `reader!`, and produces two answers. The fourth branch
produces nothing, because `first` cannot be both `this` and `fails`.

## A literal is a goal

Now that we have the vocabulary, we can say what the relation from
last time means. Here it is again:

``` java
static External<?> cargo(AnswerSource db, Unifiable<String> trackingId, Unifiable<String> origin) {
    return Literal.relation(Cargo.class, "cargo")
            .arg("trackingId", trackingId).indexed()
            .arg("origin", origin)
            .from(db);
}
```

`cargo` returns an `External`, and an `External` is a literal: a
relation's name applied to arguments, which is what
`Literal.relation(...).arg(...)` builds. A literal is a goal like the
ones above, and over a stored relation it means the disjunction of
its rows: every row is one way for the relation to hold, 
and a row holds when each variable unifies with the
value in its column. With the two cargos from Part 1 in the store,
`cargo(db, trackingId, origin)` reads as

``` java
trackingId.unifies("ABC123").and(origin.unifies("CNHKG"))
        .or(trackingId.unifies("JKL567").and(origin.unifies("CNHGH")))
```

and because a literal is a goal, it composes with the connectives.
Add a conjunction:

``` java
Unifiable<String> trackingId = lvar();
Unifiable<String> origin = lvar();
Query.of(cargo(db, trackingId, origin).and(trackingId.unifies("ABC123")))
        .select(origin).stream()
        .map(row -> row.get(origin))
        .collect(toList());                      // ["CNHKG"]
```

Only one answer is produced, because the question is equivalent to
the disjunction above with `trackingId.unifies("ABC123")` conjoined
to each branch, and the second branch cannot hold. Ask for an id
nobody booked:

``` java
Query.of(cargo(db, trackingId, origin).and(trackingId.unifies("NONEXISTENT")))
        .select(origin).stream()
        .map(row -> row.get(origin))
        .collect(toList());                      // []
```

and neither branch can hold. That is what a literal means, and every
answer you get is one you could derive this way. It is not how the
engine computes it. A thousand rows and a conjunction do not become a
thousand branches, and when the cost of a question matters we will
come back to why. For now, the meaning is all you need.

Notice what this buys you: one language for describing data and computations 
over it. You no longer have to fetch data with SQL, map it into Java objects, 
and only then express the computation. Both are described uniformly, which 
opens up interesting possibilities, including choosing where the work happens: 
inside the JVM or in the database.

## Derived relations

At some point you will want relations defined in terms of other
relations. We call them derived. They are built the same way as an
`External`, with a different terminal: `.solving(Goal)`. Any goal
will do, and we have just seen that a literal is one:

``` java
public static Literal<?> known(AnswerSource db, Unifiable<String> trackingId) {
    return Literal.relation(Cargo.class, "known")
            .arg("trackingId", trackingId)
            .solving(cargo(db, trackingId, lvar()));
}
```

Here we're saying that `known` holds whenever `cargo` has a
`trackingId`. The `.arg`s are the relation's arguments, and they are
all a caller of `known` sees. The goal under `.solving` is its body,
and a variable the body makes for itself stays inside. We don't care
about `origin`, so it is a fresh variable that nothing else mentions.
You can use this relation the same way you'd use `cargo`:

``` java
// Which cargos are booked? Nothing is known, only the tracking id is asked for.
Unifiable<String> trackingId = lvar();
Query.of(known(db, trackingId))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // ["ABC123","JKL567"]
```

The important bit is that derived relations are never stored in an
`AnswerSource` and are recomputed per `Query`. They are, however,
memoized, so that repeated uses with the same arguments are reused.
In logic programming this technique is called tabling, and for now
it's enough to know that it's there. Note that you still need to
provide the `AnswerSource` for the downstream `cargo`.

## Negation

If you want to add a new `cargo` to the database, you'll need to make
sure that its `trackingId` is unique. In order to do that, you need
to say that `known` does not hold for a specific value. This is what
negation means: `exclude(G)` holds exactly when `G` cannot. For
example:

``` java
Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("NONEXISTENT").and(exclude(known(db, trackingId))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // ["NONEXISTENT"]
```

Here, we ask whether `NONEXISTENT` is known. It isn't, so the
exclusion holds and the engine answers. Alternatively, we could ask:

``` java
Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("ABC123").and(exclude(known(db, trackingId))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // []
```

In this case, `ABC123` is a known `trackingId`, so the branch fails
and no answer is produced. A bound value is checked like this, at
once. A variable that is still free when the exclusion is stated gets
the denial attached to it as a condition, and the check happens when
the variable binds. You can also exclude a unification directly:

``` java
Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("ABC123").and(exclude(trackingId.unifies("ABC123"))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // []
```

Since we're saying that `trackingId` is `ABC123` and cannot be
`ABC123`, this must fail, and so it does. If you play with exclusion
a bit more, you may want to try this:

``` java
Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("XYZ556").and(exclude(trackingId.unifies("JKL567").or(trackingId.unifies("ABC123")))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // does not compile
```

This will not compile, and the reason is worth knowing. `exclude`
takes a statement the engine can hold on to while it reasons: a
piece of knowledge it adds to what it already knows and checks
everything that follows against. A unification is one. A literal is
one. A conjunction of them is one, because adding both halves can
only narrow what is possible, never widen it, and that is what lets
the engine settle. A disjunction is not: it opens two worlds instead
of narrowing one, and there is nothing to settle on. So the engine
refuses to take it directly. What it can take is a derived relation,
because a derived relation is computed to completion for the
question asked, its answers tabled, and a finite table of answers is
knowledge the engine can hold on to and deny. Give the disjunction a
name:

``` java
public static Literal<?> eitherOne(AnswerSource db, Unifiable<String> trackingId) {
    return Literal.relation(Cargo.class, "eitherOne")
            .arg("trackingId", trackingId)
            .solving(trackingId.unifies("JKL567").or(trackingId.unifies("ABC123")));
}

Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("XYZ556").and(exclude(eitherOne(db, trackingId))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // ["XYZ556"]
```

This compiles and produces the expected answer.

## Ignoring columns

`known` is a one-line derived relation over `cargo`, and you may be wondering
whether you need it at all. Use `cargo` directly under `exclude`, with
the fresh variable for the origin, the way `known` spelled it:

``` java
Unifiable<String> trackingId = lvar("trackingId"); // you can name lvars for a better output
Query.of(trackingId.unifies("ABC123")
            .and(exclude(cargo(db, trackingId, lvar("origin")))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());
        // java.lang.IllegalStateException: 
        // holds only under ∃<origin>. ¬(cargo@7a5d012c({ABC123}, <origin>)), 
        // about a name the select did not name: (<trackingId>: {ABC123}) : ∃<origin>. ¬(cargo@7a5d012c({ABC123}, <origin>)) 
        // — a reading into Java needs a row that holds outright; select the name or read it raw
```

This is a new kind of outcome - the engine produced an answer, 
but refused to hand it to Java, because it holds only under a condition. 
How to deal with this kind of situation is the subject of part 4 so for 
now, I'll translate the answer for you: `trackingId` is `ABC123` if 
there is some `origin` for which there is no `cargo` with `trackingId=ABC123`. 
There are a lot of origins like that: `USNYC`, `DEHAM`, and in 
fact every string except `CNHKG`, the one that actually sits next 
to `ABC123` in the row. This tells us that the model specifies 
something different from "I don't care about this column". It says: 
"there is some value for which the row exists". The difference is
visible here because it's `exclude`d - "there is some value for
which the row doesn't exist".  

What you meant was "no row with this id, whatever its origin". The
engine has a marker for exactly that, `ignored()`:

``` java
Unifiable<String> trackingId = lvar();
Query.of(trackingId.unifies("ABC123")
            .and(exclude(cargo(db, trackingId, ignored()))))
        .select(trackingId).stream()
        .map(row -> row.get(trackingId))
        .collect(toList());                      // []
```

An `ignored()` column is not a variable of the question at all. The
literal's answers are a set over the columns that remain, and under
`exclude` the denial reads the way you meant it. Outside a negation
the two spellings differ only in how often an answer repeats, which
is why nothing in the sections above could show you the difference.

So `ignored()` is a shortcut: it writes the one-line derived relation
for you, at the place you use the literal, and that is all it is.
Inside a derived relation you do not need it. Only the `.arg`s are
visible from outside, so a fresh `lvar()` for a column the body does
not care about stays inside, and `known` is right as it was written.
Where you need the shortcut is a stored literal standing under
`exclude` on its own, with no derived relation around it to keep the
column inside, as in the example above. Spell that column
`ignored()`, or write the relation.

## Where this leads

That is all the runway. Connectives, a relation written over
another, and a denial that means what it says are enough to write a
rule that does real work.

The next part writes one: `plannedHandling`, the handling a cargo's
itinerary expects at each step. Ask it with the step bound and it
says what should happen next. Ask it with an event bound and it says
which step the event was. One rule, both directions, and that is the
point of declaring relations instead of writing lookups.
