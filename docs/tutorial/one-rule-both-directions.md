# Part 3: One rule, both directions

Part 2 ended with a promise: one rule that answers in both directions.
This part writes it. By the end you will have stored the two relations
the shipping domain turns on, the itinerary and the handling history,
and written `plannedHandling`, the handling the itinerary expects at
each step. Asked with a step it says what should happen. Asked with an
event it says which step that was. Asked with nothing it lists the plan.

Three things are new: relations over integers, a rule that calls
another rule, and a negation inside a real rule. We take them one case
at a time and put the
rule together at the end.

We start with the data.

## A leg
When we want to move `cargo` from one place to another, rarely can we do it in one go.
Usually it takes several stops to achieve that goal, so the cargo needs an itinerary. 
Each segment of its journey is called a `leg`, and it forms the plan of how to get `cargo`
from its source to destination. It is the essential state of our domain, so we store it:

``` java
static External<?> leg(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index,
        Unifiable<String> voyage, Unifiable<String> loadLocation, Unifiable<String> unloadLocation) {
    return Literal.relation(Cargo.class, "leg")
            .arg("trackingId", trackingId).indexed()
            .arg("index", index)
            .arg("voyage", voyage)
            .arg("loadLocation", loadLocation)
            .arg("unloadLocation", unloadLocation)
            .from(db);
}
``` 
## A handling event
For a real system, recording a plan is rarely enough. People make mistakes, cargos get misdirected,
so we also need to record the actual history of a voyage that is sent to us by the carriers. Keeping
track of how the actual events correspond with a plan is an important feature of our application. This
is also essential state of our application, so we store it:

``` java
public enum HandlingType {
    LOAD,
    UNLOAD,
    RECEIVE,
    CLAIM,
    CUSTOMS;
}

static External<?> handlingEvent(AnswerSource db, Unifiable<String> trackingId, Unifiable<HandlingType> type,
        Unifiable<String> location, Unifiable<Instant> completed) {
    return Literal.relation(Cargo.class, "handlingEvent")
            .arg("trackingId", trackingId).indexed()
            .arg("type", type)
            .arg("location", location)
            .arg("completed", completed)
            .from(db);
}
```
## The plan, case by case
Now, in order to construct a rule that shows the correspondence 
between `leg`s and `event`s, let's make a few observations. Let's look
at `HandlingType`:

* In order to work with `cargo` at all, we first need to have it, which is the `RECEIVE` event.
* Whenever we start a `leg`, we need to `LOAD` the `cargo` onto the carrier.
* Whenever a `leg` ends, the `cargo` needs to be `UNLOAD`ed.
* When the voyage ends, the `cargo` has to be `CLAIM`ed.
* `CUSTOMS` may happen at any step, so we don't plan for it.

With this relation we're interested in `trackingId`, `type`, `location` and the `step`:

``` java
static Literal<?> plannedHandling(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> step,
        Unifiable<HandlingType> type, Unifiable<String> location)
```
Now, I'll break down the specification separately, step by step, and we'll put it back together at the end.

Let's try to encode the logic behind the `RECEIVE` event:
``` java
// In order to work with `cargo` at all, we first need to have it, which is the `RECEIVE` event.
type.unifies(HandlingType.RECEIVE)
            .and(leg(db, trackingId, lval(0), lvar(), location, lvar()))
            .and(step.unifies(0))
```
In this case, we know that the `type` is `RECEIVE` and that the `step` is `0`. Similarly,
this also applies to the `leg`'s `index`. The only surprise you may be seeing is the naked `lvar()`
for a column we don't need. This fragment is a part of the whole relation we'll show at the end,
and a column no `.arg` names stays inside its body, as in Part 2. Onto the next case:
``` java
// Whenever we start a `leg`, we need to `LOAD` the `cargo` onto the carrier.
Unifiable<Integer> index = lvar();
Unifiable<Integer> twice = lvar();
type.unifies(HandlingType.LOAD)
            .and(leg(db, trackingId, index, lvar(), location, lvar()))
            .and(Ints.multo(index, lval(2), twice))
            .and(Ints.addo(twice, lval(1), step))
```
Here I snuck in relational operators over integers, so let me explain. The `step` is
the step of the plan, but `leg`s have their own `index`. Each `leg` starts with `LOAD` and ends
with `UNLOAD`. Since we start with `RECEIVE` as `step` `0`, the relationship between them for the
`LOAD` event is: `step=2*index+1`. `Ints.multo` and `Ints.addo` are arithmetic relations
over integers. We need two helper variables `index` and `twice` to express
this equation. The `o` at the end of their names is a convention that denotes that 
they are relational: they work regardless of whether you ask about `index` or `step` - 
the engine will deduce the missing one if there's enough information. The similar case holds for `UNLOAD`:
``` java
// Whenever a `leg` ends, the `cargo` needs to be `UNLOAD`ed.
Unifiable<Integer> index = lvar();
Unifiable<Integer> twice = lvar();
type.unifies(HandlingType.UNLOAD)
            .and(leg(db, trackingId, index, lvar(), lvar(), location))
            .and(Ints.multo(index, lval(2), twice))
            .and(Ints.addo(twice, lval(2), step))
```
The difference here is that since `UNLOAD` happens at the end of each `leg`, the
relationship is `step=2*index+2` (`1` for the `0`-th `RECEIVE` and `1` to offset to the end of a `leg`).
## The last leg

The last case is a little bit of a curveball, but we'll muscle through it:
``` java
// When the voyage ends, the `cargo` has to be `CLAIM`ed
Unifiable<Integer> index = lvar();
Unifiable<Integer> twice = lvar();
type.unifies(HandlingType.CLAIM)
            .and(leg(db, trackingId, index, lvar(), lvar(), location))
            .and(lastIndex(db, trackingId, index))
            .and(Ints.multo(index, lval(2), twice))
            .and(Ints.addo(twice, lval(3), step)))
```
Similar statement, with two differences: `step=2*index+3`, with `index` being the 
index of the last `leg`. The last `UNLOAD` was `2*index+2` and this is the next step,
but how do we assert that `index` is the last? Explicitly - the index is last, if there
are no `leg`s with next index:
``` java
static Literal<?> lastIndex(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar();
    return Literal.relation(Cargo.class, "lastIndex")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, index, lvar(), lvar(), lvar())
                    .and(Ints.addo(index, lval(1), next))
                    // note the ignored() marker under exclusion
                    .and(exclude(leg(db, trackingId, next, ignored(), ignored(), ignored()))));
}
```
Here, we again see a `leg` lookup with `index` named and `next` described with `addo`. Then we state
that there is no `leg` with `next` `index`. This is the same negation as in Part 2, with the
row's identity computed by `addo` first. Two things to note - `lvar`s and `ignored()`. In the
first `leg` the unneeded columns are `lvar()`s: the `.arg`s don't name them, so they stay inside
the body. The second `leg` stands under `exclude` with no head above it to drop a column, so
there they are `ignored()`, like we previously discussed.

Like in part 2, there are nuances to negation that we won't be discussing for now. 
You may be wondering why we're adding `1` to index instead of saying that `index < next`.
This assumes that there are no gaps in the itinerary and wouldn't work for non-discrete domains
like time. We'll get to it in part 4 when talking about conditional answers.

## The whole rule

Now that every case is specified we can write the full relation:

``` java
static Literal<?> plannedHandling(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> step,
        Unifiable<HandlingType> type, Unifiable<String> location) {
    Unifiable<Integer> index = lvar();
    Unifiable<Integer> twice = lvar();
    return Literal.relation(Cargo.class, "plannedHandling")
            .arg("trackingId", trackingId)
            .arg("step", step)
            .arg("type", type)
            .arg("location", location)
            .solving(type.unifies(HandlingType.RECEIVE)
                    .and(leg(db, trackingId, lval(0), lvar(), location, lvar()))
                    .and(step.unifies(0))
                    .or(type.unifies(HandlingType.LOAD)
                            .and(leg(db, trackingId, index, lvar(), location, lvar()))
                            .and(Ints.multo(index, lval(2), twice))
                            .and(Ints.addo(twice, lval(1), step)))
                    .or(type.unifies(HandlingType.UNLOAD)
                            .and(leg(db, trackingId, index, lvar(), lvar(), location))
                            .and(Ints.multo(index, lval(2), twice))
                            .and(Ints.addo(twice, lval(2), step)))
                    .or(type.unifies(HandlingType.CLAIM)
                            .and(leg(db, trackingId, index, lvar(), lvar(), location))
                            .and(lastIndex(db, trackingId, index))
                            .and(Ints.multo(index, lval(2), twice))
                            .and(Ints.addo(twice, lval(3), step))));
}
```
## Three cargos

Let's populate an in-memory database with example events:
``` java
static final Instant T1 = Instant.parse("2026-03-01T08:00:00Z");
static final Instant T2 = Instant.parse("2026-03-02T08:00:00Z");
static final Instant T3 = Instant.parse("2026-03-09T08:00:00Z");

static final AnswerSource db = AnswerStore.empty().asserting(
        // ABC123: Hong Kong to Helsinki via New York, handled as planned so far
        leg(null, lval("ABC123"), lval(0), lval("V100"), lval("CNHKG"), lval("USNYC")),
        leg(null, lval("ABC123"), lval(1), lval("V200"), lval("USNYC"), lval("FIHEL")),
        handlingEvent(null, lval("ABC123"), lval(HandlingType.RECEIVE), lval("CNHKG"), lval(T1)),
        handlingEvent(null, lval("ABC123"), lval(HandlingType.LOAD), lval("CNHKG"), lval(T2)),
        handlingEvent(null, lval("ABC123"), lval(HandlingType.UNLOAD), lval("USNYC"), lval(T3)),
        // JKL567: Hangzhou to Hamburg, unloaded in Rotterdam instead
        leg(null, lval("JKL567"), lval(0), lval("V300"), lval("CNHGH"), lval("DEHAM")),
        handlingEvent(null, lval("JKL567"), lval(HandlingType.RECEIVE), lval("CNHGH"), lval(T1)),
        handlingEvent(null, lval("JKL567"), lval(HandlingType.LOAD), lval("CNHGH"), lval(T2)),
        handlingEvent(null, lval("JKL567"), lval(HandlingType.UNLOAD), lval("NLRTM"), lval(T3)),
        // XYZ556: routed, nothing handled yet
        leg(null, lval("XYZ556"), lval(0), lval("V400"), lval("SESTO"), lval("FIHEL")));
```
## Both directions

Now we can run some queries. These variables serve every query below:
``` java
Unifiable<String> id = lvar();
Unifiable<Integer> step = lvar();
Unifiable<HandlingType> type = lvar();
Unifiable<String> location = lvar();
```
``` java
// step to activity
Query.of(plannedHandling(db, lval("ABC123"), lval(2), type, location))
        .select(type, location).stream()
        .map(row -> row.get(type) + "@" + row.get(location))
        .collect(toList());                            // [UNLOAD@USNYC]
```
This query asks about the step `2` of the plan for cargo `ABC123`. We can also reverse it:
``` java
// activity to step
Query.of(plannedHandling(db, lval("ABC123"), step, lval(HandlingType.UNLOAD), lval("USNYC")))
        .select(step).stream()
        .map(row -> row.get(step))
        .collect(toList());                            // [2]
```
Here we're asking which step is `UNLOAD` at `USNYC`. Or leave everything free to ask about the whole plan:
``` java
// the whole plan
Query.of(plannedHandling(db, lval("ABC123"), step, type, location))
        .select(step, type, location).stream()
        .sorted(Comparator.comparing(row -> row.get(step)))
        .map(row -> row.get(step) + " " + row.get(type) + "@" + row.get(location))
        .collect(toList());
// [0 RECEIVE@CNHKG, 1 LOAD@CNHKG, 2 UNLOAD@USNYC,
//  3 LOAD@USNYC, 4 UNLOAD@FIHEL, 5 CLAIM@FIHEL]
```
Was this handling ever planned?
``` java
// was the Rotterdam unload ever planned?
Query.of(plannedHandling(db, lval("JKL567"), step, lval(HandlingType.UNLOAD), lval("NLRTM")))
        .select(step).stream()
        .map(row -> row.get(step))
        .collect(toList());                            // [] (No)
```
What is the next step of the plan?
``` java
// what is the next step of the plan: backwards to its step, forwards to the next
Unifiable<Integer> next = lvar();
Query.of(plannedHandling(db, lval("ABC123"), step, lval(HandlingType.UNLOAD), lval("USNYC"))
                .and(Ints.addo(step, lval(1), next))
                .and(plannedHandling(db, lval("ABC123"), next, type, location)))
        .select(type, location).stream()
        .map(row -> row.get(type) + "@" + row.get(location))
        .collect(toList());                            // [LOAD@USNYC]
```
And finally, every unplanned event, for every cargo. The step is `ignored()`: any step
would do, and the literal stands under a bare `exclude`, which is what the marker is for:
``` java
// every event the plan never planned, across all cargos
Query.of(handlingEvent(db, id, type, location, lvar())
                .and(exclude(plannedHandling(db, id, ignored(), type, location))))
        .select(id, type, location).stream()
        .map(row -> row.get(id) + " " + row.get(type) + "@" + row.get(location))
        .collect(toList());                            // [JKL567 UNLOAD@NLRTM]
```
## The trade

Notice what it would take to support these queries in Java: a method
per question, each encoding the same invariants about legs and steps.
Even in one file they drift apart, and the bugs that come from that
are subtle. Here the invariants are written once, in `plannedHandling`,
and every question is a different way of calling it.

That is the trade Clauseway makes. Java makes the algorithm explicit
and hides the logic: the rule for "last leg" is spread over loops and
early returns, and you reconstruct it by reading them. Clauseway makes
the logic explicit and hides the algorithm: the rule is the code, and
the engine finds a way to answer each question. The price is precision.
You have to say exactly what you mean, because the engine does exactly
what you said. Part 2 showed one way a denial says less than it looks,
and the next part shows another.

Unlike general purpose logic programming languages, Clauseway is embedded
in Java, so the algorithm-logic trade is not absolute. Each side can
support the other and help you build more reliable software. If the
logic some algorithm encodes is invisible - refactor it into Clauseway
to make it explicit. If some Clauseway query is too slow for you - write
the algorithm yourself and use the model as the oracle for its tests.

## Where this leads

The plan is half the story. The history is stored but only read once
here, and one question was left open at `lastIndex`: why `addo` and
not `index < next`. The next part starts there, with what the engine
does with an answer it cannot settle, and then reads the history
against the plan: what happened last, whether the cargo is where the
itinerary says, and what should happen next.
