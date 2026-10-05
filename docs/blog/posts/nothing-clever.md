---
date: 2026-10-04
categories:
  - essays
---

# Nothing clever in the engine: relational programming for imperative programmers

If you come from the imperative world, you are probably sceptical about the premise of this library. Saying what is true about the result and expecting the engine to work it out sounds like intelligence, and these days we associate intelligence with LLMs, which take warehouses of GPUs to run. But logic programming — relational programming, if you prefer the name that stresses what it works on — is much older, and used to run on hardware that your backend server puts to shame. There is nothing clever in the engine. It is closer to a Sudoku solver than to a chatbot: your rules define a space of possible answers, and it walks that space methodically — try, record, undo, try the next — and hands back everything that holds. Unlike a chess engine, it isn't looking for the best answer; it's looking for all of them. A search space for chess is enormous and a lot of engineering goes into compressing it. The invoices and shipment legs you write are much simpler, so a small library can manage. This article is an outline of how it does that — how your declarations become a search — and stops before the question of how it knows when the search is done, which deserves its own.

<!-- more -->

## Functions vs relations

A function is the basic building block of almost every program: a named recipe that turns arguments into a return value. Functions have a direction — from inputs to an output. They are the how of your program, and writing without them seems impossible. It isn't, quite. The how still exists in relational programming; it has moved into the engine, written once, and what's left in your code is what holds.

Take the invariant `a + b = c`. As a function you write `add(a, b) -> c`, and it only goes one way: you need `a` and `b` to get `c`. But the invariant also says `a = c - b` and `b = c - a`, and the moment your program needs one of those, you write a second function, then a third, and now the same fact lives in three places that can drift apart. A relation is the invariant itself, with no direction attached. `plus(a, b, c)` holds or it doesn't, and whichever of the three you leave blank is the one the engine fills in. The three directions still have to be written somewhere — but somewhere is the library, once, and not your codebase, every time.

The simplest relation of all is equality, and the way the engine solves equality in every direction is called unification. Everything else is built on top of it.

## Unification

To say that two things are equal is easy; what it means is a deep topic, especially once things change. Java has two notions of it. `equals` compares contents, and is easy to understand: a mechanical comparison of state. `==` compares identity — two names pointing at the same object, whatever its state is now or becomes later. Even writing this down confuses me, so the good news is that Clauseway uses neither. In a relational program there is no how, so nothing changes, and the question "the same object?" has nothing to attach to.

What it has instead is unification. `x = y` is not a function that returns `true` or `false`. It's an invariant: once you've stated it, it holds for the rest of the program, and nothing you say afterwards can make it false. Say `x = 1` later and, by necessity, `y = 1`. Say `y = 1` and nothing new is learned, because it already followed. Say `y = 2` and you've contradicted yourself — `1` is not `2` — and the engine will not pretend otherwise. What it does when that happens is the subject of a later section; for now, it's enough that an invariant, once stated, is never weakened.

This is a different model from assignment in spirit. Assignment thinks of a variable as a box that holds a value and can hold another one later. Unification thinks of a variable as a symbol, and of what the engine has learned as a substitution: a record of which symbols can be replaced with which values, or with which other symbols. `x = y` adds "x can be replaced with y"; `x = 1` adds "x can be replaced with 1", and through the first entry, so can `y`. Nothing is ever written into a box. The substitution only grows, and each new entry is checked against the ones already there — which is how a contradiction is noticed the moment it appears.

## Propagation

A similar story holds for other relations, like `plus(a, b, c)` from earlier. The difference is that unification can act on symbols alone — `x = y` learns something with no value in sight — while addition can't do anything until enough of its symbols have values. So when you state addition as an invariant, the engine checks what is currently substitutable. If enough is known, it deduces the rest from the equation. If not, it records the invariant and revisits it whenever one of its symbols gets unified.

State `plus(a, b, c)` and `times(c, 2, d)`. Nothing happens; nothing has a value. Say `a = 1`; still nothing, a sum needs two. Say `b = 2`; `plus` wakes up and deduces `c = 3` — and that deduction is itself a unification. `times` has been waiting for exactly that, wakes up in turn, and deduces `d = 6`. You said two things and learned four. Say `c = 3` and `a = 1` instead, and `plus` deduces `b = 2`: the same invariant, a different unknown. If enough is never learned — only `c = 3`, say — the engine will not guess, and it will not pretend either: asked for an answer that hinges on an unresolved sum, it refuses loudly. To leave a sum legitimately open you give its symbols ranges, which is the next idea.

Values don't have to be fully known for a constraint to act. Say `a` is somewhere in `0..9` and `b` is somewhere in `0..9`, and `plus(a, b, c)` with `c = 10`: nothing is pinned down, yet `plus` can already tell you `a` is at least 1 and at most 9, and so is `b`. Add `a < b` and the ranges tighten again. A range like `0..9` is called a domain, and a constraint that works on ranges narrows them every time one of its neighbours narrows, until nothing more can be learned. This is cheaper than it sounds: most of the impossible combinations are ruled out before anything is tried.

This chain of deductions is called propagation, and relations that take part in it are called constraints. Unification is the simplest one. Clauseway ships with domains, addition, multiplication and ordering (`dom`, `addo`, `multo`, `leq`, `lss` and friends) over the common families — `Ints`, `Longs`, `BigDecimals`, `Dates`, `Instants` — and new constraint domains are one interface away when those aren't enough.

## Composition

So far I've been cheating a little. I've been stating invariants in natural language, and every time two of them held together I quietly used the word "and", as if that were free. It nearly is. Invariants are called goals in this engine, and goals compose with exactly the two connectives you'd expect, plus one with a reputation.

First, what an answer is: a substitution in which every stated goal holds. `x = y and x = 1` has one answer — a substitution in which both `x` and `y` are replaceable with 1 — and you can ask it for either. `x = y and x = 1 and y = 2` has none, because no substitution satisfies all three. This is what happens when you contradict yourself: the derivation dies, and the engine moves on. There is no exception and no error; a contradiction is just a derivation with no answer.

`or` is where a second substitution comes from. `x = 1 or x = 2` starts two derivations, each with its own substitution, and pursues both. Here that yields two answers; in general each side yields however many it yields, including none, and the contradicted ones drop out while the rest are produced. Because `and`, `or` and `not` of goals are themselves goals, they nest arbitrarily, and the whole program is one goal.

Two consequences. First, your programs compose like business requirements: a new rule is usually a new `and`; a new way to reach the same outcome is a new `or`. Second, the order you write them in does not change which answers exist. It can change how quickly they arrive and, at the edges, whether the search finishes at all — that is the how, and it's the engine's problem, not the requirement's. Clauseway ships with an optimizer that does the ordering for you.

The third connective is `not`, and here Clauseway parts ways with most of logic programming, where negation is notoriously fragile: Prolog's `not` only works once everything inside it already has a value, and silently misbehaves otherwise. In Clauseway, `not` is a constraint like `plus`. `exclude(x = 2)` states the invariant `x ≠ 2`. It doesn't need to know `x` yet; it records itself, and the moment anything would make `x` into 2, that derivation dies. Several equations together forbid a combination rather than a single value: `exclude(x = 3, y = 4)` allows either but not both. Like any constraint, a `not` that the search never gets to decide isn't thrown away — it comes back as part of the answer: "`x` is anything, provided it isn't 2." Combined with a domain, it does what you'd hope: `x ∈ {4, 5}` and `x ≠ 5` propagates straight to 4.

The same mechanism negates whole relations. Say `leg(from, to)` is a relation backed by your table of carrier legs — a parcel can be moved from one hub to the next. `exclude(leg(hub, anywhere))` computes everything `leg` can produce for that hub and forbids each of it — "none of these" — which is how "a hub nothing departs from", a dead end, is written as a rule rather than a check. The one thing this asks of the negated relation is that it have finitely many answers, so the engine can know when it has seen them all.

## Naming and recursion

Everything so far composes into one goal, and one goal is a mouthful. So goals get names. A relation is a goal with a name and arguments — `plus(a, b, c)` has been one all along, and so has `leg` — and once a goal has a name, its body can mention other names, and its own. That is recursion, and it's the only way to say "and keep going" in a relational program. There are no loops.

Back to the shipment legs. A route is a leg, or a leg followed by a route:

```java
Literal route(Unifiable<String> from, Unifiable<String> to) {
    return Literal.relation(Rules.class, "route")
            .arg("from", from).arg("to", to)
            .solving(leg(db, from, to)
                    .or(exist(via -> leg(db, from, via).and(route(via, to)))));
}
```

Read it as the requirement reads, and ask it in any direction: where can Warsaw reach, what can reach Lisbon, the whole table.

It looks like a how — recursion in a function is a how, a call stack hiding in a keyword. Here it's a definition by induction: `route` is the smallest relation containing every leg and closed under leg-then-route. Write it as leg-then-route, route-then-leg or route-then-route and you get the same relation; only the engine's effort changes. In classic Prolog, a definition like this over data with cycles never finishes. In Clauseway, every named relation remembers what its calls have already produced, so it does — and gives each answer once. How that works is its own article.

## Asking

Everything so far has been stating. Asking is the same thing with blanks in it: the symbols you leave open are the question, and the answers are the substitutions that fill them.

```java
Unifiable<String> hub = lvar();
Query.of(route(lval("Warsaw"), hub)).solve(hub);   // everywhere Warsaw reaches
Query.of(route(hub, lval("Lisbon"))).solve(hub);  // everything that reaches Lisbon
```

`solve` hands back a stream of answers, one per substitution, and stops when the engine has run out. There's no second method for the second direction; the blank moved.

## Magic

If I've done my job so far, you may be asking: this is all pretty basic, so where's the magic? There isn't any in the engine, as promised. Every invariant you've seen is obvious in the small — `x = y`, a sum, a leg followed by a route. The magic, such as it is, is in what they do together. Goals compose by connectives, and they compose by nesting: a relation refers to other relations and, as in the last section, to itself. The answers are the closure of all of that at once, and closure is the thing no one can hold in their head. Each Sudoku constraint is trivial; the solved grid isn't. At some point you lose track of what implies what, and at that point the engine takes over and hands you answers that look as though intelligence produced them. That is the promise of logic programming: state the invariants, and the answers follow.

It is also where the cost lives, and I'd rather say so here than have you find out. When an answer is wrong, you can't step through it the way you'd step through a loop, because there was no loop. What you can do is watch the engine work: it will narrate the search for you — every rule as it is tried, succeeds or fails, in order — and reading that narration is a different skill from reading a stack trace. When an answer is slow, you are back to thinking about the how: a domain you should have declared, an index you should have asked for, a rule written in a shape the engine takes the long way round. Memoization and propagation remove most of that burden, the optimizer removes some more, and what's left is the residue that no declarative system has quite eliminated. It's smaller than it used to be. It isn't zero. There are no silver bullets, after all.

Thankfully, Clauseway is a library, not a paradigm you adopt end to end. When the residue outweighs the win for some piece of your system, write that piece's how yourself and keep the rest.