# Part 4: Any value, some value

So far in this tutorial we've been working our way up to expressing a useful derived rule that
we could query in multiple directions. This worked fine for the purpose of illustration, but
when solving real problems you will oftentimes be confused and try things that fail. So far
our examples didn't teach how to read the engine output in those cases, so today we're going
to break something. By the end of the article, you will know how to read the engine's symbolic
output, how it produces answers and when it will not. Finally, you will learn about quantifiers,
how to scope them and how they influence negation. Buckle up, because it's going to be a deep dive.

## Why not `<`?
The last part left us with a question - why did we choose to add `1` instead of using `<`?
After all, not every domain has contiguous indexes and sometimes there is no way to increment,
like with `Instant`s, or `BigDecimal`s. This is what we did:
``` java
static Literal<?> lastIndex(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar("next");  // you can name lvars for a better output
    return Literal.relation(Cargo.class, "lastIndex")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, index, lvar(), lvar(), lvar())
                    .and(Ints.addo(index, lval(1), next))
                    .and(exclude(leg(db, trackingId, next, ignored(), ignored(), ignored()))));
}
```
We could try to write it this way instead:
``` java
static Literal<?> lastIndexLss(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar("next");
    return Literal.relation(Cargo.class, "lastIndex")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, index, lvar(), lvar(), lvar())
                    .and(Ints.lss(index, next))
                    .and(exclude(leg(db, trackingId, next, ignored(), ignored(), ignored()))));
}
```
If you're adventurous, you might have tried it and were met with the following error:
``` java
Unifiable<Integer> index = lvar("index");
Query.of(lastIndexLss(db, lval("ABC123"), index))
        .select(index).stream()
        .map(row -> row.get(index))
        .collect(toList());
        // IllegalStateException:
        // holds only under ∃<next>. lss({1}, <next>), about a name the select did not name:
        // (<index>: {1}) : ∃<next>. lss({1}, <next>)
        // — a reading into Java needs a row that holds outright; select the name or read it raw
```
This is bad, why can't we get an answer? Let's print the row instead:
``` java
// the lss version: print the rows instead of reading them
Query.of(lastIndexLss(db, lval("ABC123"), index))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {1}) : ∃<next>. lss({1}, <next>)
// (<index>: {0}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>)) ∧ lss({0}, <next>)
```
This looks wrong twice over. First, we didn't get a straight answer but a bunch of squiggles, and more
than that - we got two answers, but there's only one last index. The good news is that this answer
isn't wrong - you just don't know how to read it yet. The bad news is that you accidentally described
something different from what you meant. Let's step back and talk about how the engine produces answers.  

## Any value, some value
Here is a simpler model:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Query.of(meaning.unifies(42))
    .select(meaning).stream()
    .forEach(System.out::println); 
// (<meaning>: {42})
```
Here we get the expected answer, but look what happens if we select an unrelated variable:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Unifiable<Integer> what = lvar("what");
Query.of(meaning.unifies(42))
    .select(what).stream()
    .forEach(System.out::println); 
// (<what>: _.0)
```
We got squiggles again, but this time they are simpler. The answer says that 
`what` has an identity: `_.0`. It doesn't say anything about its value because 
the model holds for every `what` you want. If the engine were to produce
all answers, it would have to enumerate all `Integer`s, which is usually 
not what you wanted. But why does `what` have an identity? The following 
example gives us a clue:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Unifiable<Integer> what = lvar("what");
Query.of(meaning.unifies(what))
    .select(meaning, what).stream()
    .forEach(System.out::println); 
// (<meaning>: _.0, <what>: _.0)
```
The model said that `meaning` and `what` unify, but hasn't specified any concrete
value, so the answer reflects that. They have the same identity: `_.0` because they
point to the same value, but this is any value, a parameter of the answer. 
Generally speaking, if the model doesn't unify a `select`ed variable with a value,
it will be assigned an identity and the answer will contain it. It signifies that
any value will do and the engine will not enumerate all possibilities. Let's look
at a different example involving `lss`:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Unifiable<Integer> what = lvar("what");
Query.of(Ints.lss(meaning, what))
    .select(meaning).stream()
    .forEach(System.out::println); 
// (<meaning>: _.0) : ∃<what>. lss(_.0, <what>)
```
Here we get something resembling our original answer: `meaning` is any value
`_.0` given (`:`) it is less than *some* (`∃`) `what`. We already discussed that `_.0`
means any number, but now we see a different name in the answer - `what`.
Here however, it has no identity and the engine prefixed it with `∃` (*some*). This
prefix is important because the engine gave us an answer that holds conditionally,
and it says that we need to check this condition ourselves. Why? Because of the
same reason we've seen before - `what` is not unified with any value so the engine
will not try and enumerate every valid value for it to find *some* that satisfies
the condition. This shows what Clauseway does whenever you `select` variables in
the query - it will produce answers for them, given that it can find *some* values
for the rest of the variables that make the model hold. If you didn't provide values
to try - it will hand you back a conditional answer naming those variables.

## Exclusion is a complement

But does every unification mean that the engine will try to satisfy the model
with provided values? Not necessarily:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Query.of(exclude(meaning.unifies(42)))
    .select(meaning).stream()
    .forEach(System.out::println); 
// (<meaning>: _.0) : ¬(_.0 ≡ {42})
```
Here exclusion still hands us back a conditional answer. That is because straight
unification provides a value, but excluded - it is the value's complement. If the
engine were to provide all answers - it would have to list every integer except for
`42`. Still not what you'd want in an actual program. If you give it a range of values
to try, it will:
``` java
Unifiable<Integer> meaning = lvar("meaning");
Query.of(exclude(meaning.unifies(42))
            .and(FiniteDomain.dom(meaning, Ints.interval(40, 43))))
    .select(meaning).stream()
    .forEach(System.out::println);
// (<meaning>: {40})
// (<meaning>: {41})
// (<meaning>: {43})
// {42} is skipped
```
Even though `exclude(meaning.unifies(42))` makes `meaning` the complement of `42`,
assigning it values between `40` and `43` produces answers. That is because the engine
will try the values listed in the range and if the model holds, the answer will be produced.
Note that `42` is missing - the range provided the values and `exclude` filtered them.

With this, we've built enough vocabulary to unpack the original question.
## Where the *some* lives

``` java
Unifiable<Integer> index = lvar("index");
Unifiable<Integer> next = lvar("next");
Query.of(leg(db, lval("ABC123"), index, lvar(), lvar(), lvar())
        .and(Ints.lss(index, next))
        .and(exclude(leg(db, lval("ABC123"), next, ignored(), ignored(), ignored()))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {0}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>)) ∧ lss({0}, <next>)
// (<index>: {1}) : ∃<next>. lss({1}, <next>)
```
Let's first look at the answer that looks obviously wrong:
```
(<index>: {0}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>)) ∧ lss({0}, <next>)
```
`index` is `0`, given that there is *some* `next` that is greater than `0` and there 
is no `leg` with index `next`. First - why do we even get a conditional answer? `next`
is unified via `leg`, but it is negated so it takes every value that is not an index
of a `leg`. Since there is no other source of values for it, the answer contains the
quantifier. Let's see if there is a `next` like that that is greater than `0`. 
Yes - in fact, almost every positive integer satisfies that condition. There is no 
`leg` with index `100`, so this condition holds even if the engine wasn't able to 
prove it. The other answer looks less wrong, but still has a condition for a similar reason:
```
(<index>: {1}) : ∃<next>. lss({1}, <next>)
```
`index` is `1`, given that there is *some* `next` that is greater than `1`. You may be wondering at this
point why the answer does not mention `leg`; we will circle back to it later. Does it hold? Again, 
lots of integers satisfy this condition so the answer holds. This shows that the model we specified as 
`lastIndexLss` is not even the one that we meant:

``` java
leg(db, trackingId, index, lvar(), lvar(), lvar())
    .and(Ints.lss(index, next))
    .and(exclude(leg(db, trackingId, next, ignored(), ignored(), ignored())))
```
When we select `index`, this definition says that there is *some* `next` that is greater than 
`index` for which there is no `leg`. What you meant to say is that there is *no* `next` that 
is greater than `index` for which *there is* a `leg`. The difference is subtle, and it's the 
placement of negation, and where the `∃<next>` lives. Given this statement you may try this 
definition.

## Still a variable of the question

``` java
Unifiable<Integer> index = lvar("index");
Unifiable<Integer> next = lvar("next");
Query.of(leg(db, lval("ABC123"), index, lvar(), lvar(), lvar())
        .and(exclude(leg(db, lval("ABC123"), next, ignored(), ignored(), ignored()),
                     Ints.lss(index, next))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {0}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>) ∧ lss({0}, <next>))
// (<index>: {1}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>) ∧ lss({1}, <next>))
```
Two answers again, but let's see if they hold. `index` is `0` given that there is
no `leg` whose index `next` is greater than `0`. `next` here is an index of a `leg`
with `trackingId=ABC123`, so it is `0` or `1`. Is either greater than `0`? Yes, `1` is -
so there is such a `leg` and the answer does not hold. The answer `(<index>: {1})`
holds if no `leg` index is greater than `1`. None is - so it holds. This looks like
the right question, so why wasn't the engine able to prove it?

Because we cheated a little. Look at the row again: `∃<next>.` lives in front
of `¬`, not behind it. We read `next` as "an index of a `leg`" - we let `leg`
provide its values. But `next` is a variable of the whole `Query`, and inside
`exclude` the engine does not let `leg` provide values - it takes the complement.
`next` ranges over every value that is *not* an index of a `leg` greater than
`index`. For `index = 0` that is every integer but `1`; for `index = 1` it is every
integer. Moving `lss` inside changed *which* complement `next` ranges over, but
`next` still ranges over a complement, the complement is still almost everything,
so *some* `next` always exists and both answers survive. The engine will not go looking
for a `next` inside that complement for us, so it hands it back as a condition naming `next`.

So - are we doomed? Not quite. The problem is that `next` exists outside
the exclusion at all: it is a variable of our question. We need a question 
that has no `next` in it and still asks about the legs after `index`.

## A literal is a question of its own

``` java
static Literal<?> hasLaterLeg(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar("next");
    return Literal.relation(Cargo.class, "hasLaterLeg")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, next, lvar(), lvar(), lvar())
                    .and(Ints.lss(index, next)));
}

Unifiable<Integer> index = lvar("index");
Query.of(leg(db, lval("ABC123"), index, lvar(), lvar(), lvar())
        .and(exclude(hasLaterLeg(db, lval("ABC123"), index))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {1})
```
Finally - a correct answer! But why does a derived literal help? Look at `hasLaterLeg`
as a `Query`: its body is the model and its `arg`s are its `select`. Everything we
said about `select` holds here. `hasLaterLeg` selects `trackingId` and `index`, and for
the rest of its variables - `next` - the engine must find *some* values that make
the body hold. Inside the body `leg` provides them. But wait - you may ask, doesn't
`hasLaterLeg` exist under `exclude` in our top-level `Query`? It does, but `next` does
not. That is because a `Literal` is a boundary that the engine does not look across. 
It treats this as a separate question that only communicates through the `args` and 
since we haven't declared `next` as a selected variable - it does not leak upwards. 
To see that, we can ask `hasLaterLeg` directly:
``` java
Query.of(hasLaterLeg(db, lval("ABC123"), index))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: _.0) : lss(_.0, {0}) ∨ lss(_.0, {1})
```
`next` is gone. The body spent it on `0` and `1`, and the only condition left is
about `index`. Now `exclude(hasLaterLeg(...))` takes the complement of *that*: every
`index` that is not less than `0` and not less than `1`. The difference from before
is who provides the values. Nothing outside the `exclude` provided `next`, so its
complement came back as a condition. `index` is provided by the `leg` lookup in
our query - it is `0` or `1` - so the complement is checked against a value and
settled: `0` is less than `1`, out; `1` is not less than either, in. The natural
reading we used above - letting `leg` provide `next` - was right all along. It just
has to happen inside a literal, where `next` is a local, before the complement is
taken.

One thing you can now explain that Part 2 only asserted. There we wrote
`exclude(leg(db, trackingId, next, ignored(), ignored(), ignored()))` and said that an ignored
column is not a variable of the question, and that `ignored()` is a shortcut for a one-line
derived relation. Before reading on, say in this part's words what changes if those three
columns are `lvar()` instead. Where does their *some* live? Here is the check, at an index
that does have a `leg`:
``` java
Unifiable<Integer> index = lvar("index");
Query.of(index.unifies(1)
        .and(exclude(leg(db, lval("ABC123"), index, lvar("voyage"), lvar("load"), lvar("unload")))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {1}) : ∃<voyage>,<load>,<unload>. ¬(leg@2c34f934({ABC123}, {1}, <voyage>, <load>, <unload>))
```
With `ignored()` in those columns the same query prints nothing: there is a `leg` at `1`, so the
exclusion fails and the row is gone. With `lvar()` the three columns are variables of the
question, their *some* lands at the row, outside the `¬`, and the row survives on a condition
that reads "for some voyage, there is no `leg` at `1` with that voyage" - true, and useless.
The rows above held the clue all along: every excluded `leg` in this part printed as
`leg[trackingId,index]@tabled`, two columns and tabled, because that is the derived relation
`ignored()` wrote for you. Here the engine excludes the lookup itself, all five columns, against
the store - the hash after `@` is the store's address and will differ on your machine.

## Order is not yours to pick

Now that we've been through the whole arc, we can also see why the original relation
worked:
``` java
static Literal<?> lastIndex(AnswerSource db, Unifiable<String> trackingId, Unifiable<Integer> index) {
    Unifiable<Integer> next = lvar("next");  // you can name lvars for a better output
    return Literal.relation(Cargo.class, "lastIndex")
            .arg("trackingId", trackingId)
            .arg("index", index)
            .solving(leg(db, trackingId, index, lvar(), lvar(), lvar())
                    .and(Ints.addo(index, lval(1), next))
                    .and(exclude(leg(db, trackingId, next, ignored(), ignored(), ignored()))));
}
```
The first `leg` provides a value for `index` and `next` has its value from `addo` - 
`exclude` simply filters it. The reason to write it this way is artificial - I need to
present ideas in this tutorial in a way that you can digest them - one at a time. This 
requires me to choose examples that illustrate the points I'm trying to get across. 
For example, I promised to circle back to the answer that was inexplicably simplified:
``` java
Unifiable<Integer> index = lvar("index");
Unifiable<Integer> next = lvar("next");
Query.of(leg(db, lval("ABC123"), index, lvar(), lvar(), lvar())
        .and(Ints.lss(index, next))
        .and(exclude(leg(db, lval("ABC123"), next, ignored(), ignored(), ignored()))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {0}) : ∃<next>. ¬(leg[trackingId,index]@tabled({ABC123},<next>)) ∧ lss({0}, <next>)
// (<index>: {1}) : ∃<next>. lss({1}, <next>)
```
The engine settled the condition `¬(leg[trackingId,index]@tabled({ABC123},<next>))` for
`(<index>: {1})` on its own: once `lss` had pushed `next` above `1`, no `leg` had such an index.
Our current rules never say why.
That is because the logical rules I presented allow you to reason about what you express, but not
how the engine reasons about it. The sole reason to use Clauseway is that you do not want to know
how the engine works, and how exactly it produces answers. If you wanted to think about it, you 
would just write the algorithm yourself. This is why the engine never promises the order in which
it will execute your statements, and yet some orders happen to produce the right answer:
``` java
Unifiable<Integer> index = lvar("index");
Unifiable<Integer> next = lvar("next");
Query.of(leg(db, lval("ABC123"), index, lvar(), lvar(), lvar())
        .and(exclude(Ints.lss(index, next),
                     leg(db, lval("ABC123"), next, ignored(), ignored(), ignored()))))
        .select(index).stream()
        .forEach(System.out::println);
// (<index>: {1})
// (<index>: {0}) : ∃<next>. ¬(lss({0}, <next>) ∧ leg[trackingId,index]@tabled({ABC123},<next>))
```
Here we took the example from `Still a variable of the question` and we swapped the order
of conjuncts inside `exclude`. This allowed the engine to find an unconditional answer for
`index = 1` because of the algorithm it uses, while `index = 0` came back as conditional as
before. The point is, that you shouldn't rely on it. That is because the ability of the engine
to answer different questions about the same model makes any hand-picked order right for one
question only. Similarly, if you ordered your model to make it run faster, it could run faster
for a single query, which would make this paradigm less useful. This is why Clauseway has 
pluggable schedulers and optimizers that are parameters of the `Query` and not of the model.
This way they can figure out proper order and execution strategy for you. With the rules I
gave you, they should work for all of them.

## Where this leads

You came here to find out why `lastIndex` adds `1` instead of asking for `index < next`, and
you leave with a way of reading what the engine hands back. A row is the cells you selected and,
after the `:`, the condition they hold under. `_.0` is any value: the model holds for all of
them, so the engine leaves the choice to you. `∃<name>` is some value: one has to exist for the
row to hold, the engine could neither find it nor rule it out, so the condition is yours to check.
`¬` is a complement: it filters the values it is given and never provides any. A literal is a
question of its own: what it does not select cannot leak out of it, which is where a *some*
belongs when you want it decided before the complement is taken. And the order in which any of
this gets checked is the engine's to pick, not yours.

The history is still waiting. It is a list of handling events, each stamped with an `Instant`,
and an `Instant` is exactly the kind of value you cannot add `1` to - the reason we wanted `<` in
the first place. The next part reads the history against the plan: what happened last, whether
the cargo is where the itinerary says it is, and what should happen next. Each of those is a
"nothing later" question, and you already know how to ask it: a literal of its own, with
`Instants.lss` inside.
