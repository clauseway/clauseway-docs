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
But the engine did produce answers. What failed was our attempt to read one as an ordinary Java value. 
Let's print the rows instead.
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
Two answers again. At first glance, this looks like the question we meant to ask: there is no `leg` 
with an index greater than `index`. Since `ABC123` has legs at `0` and `1`, we'd expect `0` to fail 
and `1` to succeed. So why did both survive?

Look carefully at where `∃<next>` appears. It is outside the `¬`. The answer for `index = 0` doesn't 
say "there is no later leg". It says "there is some `next` for which it is not true that a `leg` exists at 
`next` and `next` is greater than `0`".

That's a very different statement. Take `next = 0`. There is a leg at `0`, but `0` is not greater than `0`, 
so the conjunction is false and its exclusion holds. Or take `next = 100`: there is no `leg` there, so 
the exclusion holds again. The same reasoning works for `index = 1`. Both answers have conditions that 
can be satisfied, even though the engine hasn't settled them.

What went wrong? We moved `lss` inside the exclusion, but `next` is still a variable of the whole `Query`. 
The engine is constraining which values `next` may take, not asking `leg` to find all later indices and 
then denying their existence. Inside `exclude`, the lookup cannot provide values for `next`: its 
complement becomes a condition on that variable.

We need to ask whether a later `leg` exists first, with `next` belonging only to that question. Then 
we can `exclude` the answer to the whole question, rather than `exclude` a condition on an exposed 
variable.

That's what a derived relation lets us do.

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
settled: `0` is less than `1`, out; `1` is not less than either, in. The reading
we wanted above - no `leg` with an index greater than `index`, with `leg` providing `next` -
was right all along. It just has to happen inside a literal, where `next` is a local, before
the complement is taken.

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
The first `leg` provides `index`, `addo` determines `next`, and `exclude` checks whether a `leg` 
exists at that index. There is no unresolved `next` left for the engine to reason about.

We used this definition in Part 3 because it let us introduce one idea at a time. But it also 
avoided a problem that is worth understanding: the engine does not always manage to settle 
every condition, even when the information needed to do so is available. I promised to circle 
back to the answer that was inexplicably simplified:
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
For `index = 1`, by the time the engine checked the exclusion, `lss` had already pushed `next` 
above `1`, and no `leg` of `ABC123` has such an index, so the lookup could match nothing and the 
exclusion was settled. For `index = 0`, `next` above `0` still leaves the leg at `1`, so the 
condition stayed. Nothing in the rules I gave you says this. They tell you what you expressed, 
not how far the engine gets with it, and how far it gets depends on order. Take the query from 
"Still a variable of the question" and swap the two conjuncts inside `exclude`:
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
This time the engine proves that `index = 1` holds outright, while `index = 0` still carries a condition. 
We haven't changed the logical meaning of the conjunction, but we have changed how much the engine 
managed to prove before returning its answers.

This doesn't mean the earlier answers were wrong. A conditional answer describes what must still hold; 
an unconditional one means the engine has already established it. Different execution orders can leave 
different amounts of work unresolved.

So why not just rearrange the statements until the engine produces the answers we want?

Because the whole point of declaring a relation is that we can ask it different questions. An order 
that helps the engine answer one question may be a poor choice for another. If we have to arrange 
every rule around a particular execution strategy, we're back to writing algorithms instead of 
describing the domain.

This is why Clauseway has pluggable schedulers and optimizers, configured on the Query rather than 
embedded in the model. The rule describes what must hold; the execution machinery decides how to 
work toward an answer. That doesn't guarantee every condition will be settled, but it lets us improve 
execution without changing the domain rule.

And that's the important distinction between the two fixes we've seen: changing the order can change 
how much the engine proves; changing the scope of a variable changes what question we asked. 
`hasLaterLeg` fixes the question itself, rather than relying on a favorable execution order.

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
