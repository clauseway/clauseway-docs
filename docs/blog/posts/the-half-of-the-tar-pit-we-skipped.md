---
date: 2026-09-22
categories:
  - essays
---

# The half of the Tar Pit we skipped

Programming is irreducibly difficult. Fred Brooks diagnosed it as such four decades ago in "No Silver Bullet," and told us how the difficulty could be addressed. Two decades later, Mosley and Marks took up the same question in "Out of the Tar Pit," this time by surveying the tools we use and how they make our lives harder. Despite this long lineage of deliberation, mainstream languages still offer the approaches we had forty years ago. Programming is as expensive, stressful and unreliable as it was then. Why? Aren't the incentives good enough?

<!-- more -->

I think the reason is simpler than we want to admit: we just want it this way. And why wouldn't we? We know the problems we have, and even when the solutions are contrived, we know how to apply them. The patchwork of tools we use fights itself, but there is a path through that is just wide enough to be usable, and usable and well-trodden is good enough. Who is going to budget an experiment with a new approach when the old one has served for so many years? This isn't sloppiness, malice, human nature or original sin. Every intelligence works within a context, and solutions are only perfect if you imagine perfect conditions for them.

## What changes

When an electronics company ships a circuit board, or an engineering company builds a bridge, nobody expects the product to change after launch. Software is different, because bridges aren't trying to structure the companies that use them. Whenever the market forces an organization to change, the software that models its processes has to change with it. This is the biggest source of change in any software system, and change is its biggest cost.

The modern way to reduce that cost is to contain the change. It is the architecture's job to keep together the things that change together, and to stop change from spreading across system boundaries. Twenty years ago we invented the hexagon, which does exactly that: it keeps the business model in the center, builds stable walls around it, and forces technology out into adapters. The remaining question is what to do with the centerpiece.

## The half we skipped

Mosley and Marks surveyed the languages we use and told us how to avoid accidental complexity. Their advice is the functional programmer's favorite party topic: state and side effects are hard, so use a framework to do them for you. It took strongly typed functional languages years to invent monads, which we all love to use and struggle to explain, just so that we could use those frameworks elegantly.

What functional programmers won't tell you is that "Out of the Tar Pit" spends just as much time on the accidental complexity that comes from *control*. If you live in the imperative world, writing how to compute something *is* programming. Control is for-loops, if-statements, switches, order of evaluation, function application and assignment. How could *that* be accidental?

Mosley and Marks answer with a thought experiment. Imagine a perfect machine, one you hand the requirements to and it simply runs them. Whatever you would still have to tell that machine is the essential complexity: it belongs to the problem, and the users would have to know it too. Everything else — every loop, every ordering, every decision about what gets computed before what — is there because of our tools, not the task. Requirements speak in terms of what is true about the system. They never say how. So the how, the control, is accidental by definition.

## The machine is here

This is easier to see today than it was in 2006. "Out of the Tar Pit" imagines its ideal machine as science fiction. Well, to everybody's horror, the machine is here. It's called a large language model, and it takes requirements in plain English. But look at what it hands back: a for-loop. The how didn't disappear; it just stopped passing through your fingers. It still lands in your repository, still gets reviewed, and still has to change next quarter when the business does. Generation didn't remove accidental control from the artifact. It made more of it, faster.

And yet we once had something different.

## What we once had

Logic programming looks like the endpoint of the Tar Pit's vision. Functional languages delegate state and effects to a framework; logic languages delegate control as well. You declare the smallest surface that describes the problem, and the runtime goes looking for whatever satisfies it. There is no further to go in that direction; the framework is as fat as it gets.

So why isn't it mainstream? The truth is brutal: control leaked back in. Prolog took the how away from you, but the framework had a personality. Clause order matters, the cut exists, and the search is depth-first whether your problem likes it or not. Programmers who came for the elegance found themselves steering the engine by hand, which is exactly the thing they had been promised they wouldn't do. As an end-to-end paradigm it didn't survive, but its good parts were separated and assimilated elsewhere. The world runs on logic programming today, whether it looks like it or not — look no further than SQL, whose cousin Datalog is logic programming with the sharp edges filed off. In the 2000s, when network traffic was expensive, we built application logic inside relational databases, and it held up. We stopped for tooling and hiring reasons, not because it broke.

## Rules in the middle

So why build a logic programming library in Java? Because of the good parts.

Remember aggregate invariants, and making sure they still hold after every method call? Remember an invariant changing slightly and half the class being rewritten? Remember writing the how of "who holds this resource," and then the how of "which resources does this user hold," two questions that are logically the same and whose code says otherwise? Or maybe you're just tired of writing mappers between Hibernate entities and domain entities that look practically identical.

The centerpiece of the hexagon is a set of rules, and Clauseway is what runs them. It lets you declare your invariants as rules, explicitly. It's relational, so you write your logic once and query in every direction. There is no object-relational impedance, because the rule is the port: it stays relational, the JDBC adapter compiles it to SQL, and the domain never learns where the data came from. It owns your state, and because that state is immutable, the search is embarrassingly parallel — write the rule once, run it on as many threads as you have. It's extensible, so you can write your own constraints and integrations.

Here is the resource example, worked up from the bottom. Start with a fact: a loan is a checkout event.

```java
public static Literal loan(AnswerSource db, Unifiable<Integer> loanId, Unifiable<Integer> copyId,
        Unifiable<Integer> memberId, Unifiable<LocalDate> dueDay) {
    return Literal.relation(Schema.class, "loan")
            .arg("loanId", loanId).indexed()
            .arg("copyId", copyId).indexed()
            .arg("memberId", memberId).indexed()
            .arg("dueDay", dueDay)
            .from(db);
}
```

Read it as two halves. Everything up to `.from(db)` is the relation's head: its name and its columns, with hints about which ones are worth indexing. That head is the port. The last line is the adapter: `db` is whatever answers questions about loans — an in-memory value today, a PostgreSQL table over JDBC when you're ready. The method signature is the arity, the builder is the schema, and there is no entity class, no repository, and no mapper.

A return is a separate event, with the same shape and one column. And now the first rule: an active loan is a loan with no return.

```java
Literal activeLoan(Unifiable<Integer> loanId, Unifiable<Integer> copyId,
        Unifiable<Integer> memberId, Unifiable<LocalDate> dueDay) {
    return Literal.relation(Rules.class, "activeLoan")
            .arg("loanId", loanId).arg("copyId", copyId)
            .arg("memberId", memberId).arg("dueDay", dueDay)
            .solving(loan(db, loanId, copyId, memberId, dueDay)
                    .and(exclude(returned(db, loanId))));
}
```

Same head. The only new thing is the body: instead of `.from(db)`, the relation is `.solving(...)` a goal, and the goal is the whole of the business logic — one conjunction and one negation. In the Tar Pit's terms, `loan` and `returned` are essential state: the events the users actually caused. `activeLoan` is derived state, and the paper's rule for derived state is that you never store it, you only define it. There is no `active` flag on the loan to flip and forget to flip; the engine computes it from the events every time it's asked. From the outside, `activeLoan` is indistinguishable from `loan`; callers can't tell a stored fact from a derived one, and neither can the rules built on top of it.

There is no `findByCopy` and no `findByMember`. You pick a direction by deciding which arguments are values and which are variables:

```java
// Who holds copy 1?
Unifiable<Integer> who = lvar();
Query.of(rules.activeLoan(lvar(), lval(1), who, lvar())).solve(who);

// Which copies does member 100 hold?
Unifiable<Integer> what = lvar();
Query.of(rules.activeLoan(lvar(), what, lval(100), lvar())).solve(what);
```

One more layer. The invariant "a copy on loan can't be lent again" isn't a check inside a method; it's a relation built by negating the previous one — derived from derived, with the same rule applying all the way up:

```java
Literal availableCopy(Unifiable<Integer> copyId, Unifiable<String> isbn) {
    return Literal.relation(Rules.class, "availableCopy")
            .arg("copyId", copyId).arg("isbn", isbn)
            .solving(copy(db, copyId, isbn)
                    .and(exclude(activeLoan(projected(), copyId, projected(), projected()))));
}
```

Three relations, two lines of logic between them, and the store is mentioned exactly once per base fact. `Rules` never learns where the data came from.

## The experiment nobody has to budget

Most importantly, Clauseway is a library. Use it where it fits, because debugging nontermination isn't always the right use of your time.

Which brings me back to where I started. We don't change how we program because nobody budgets the experiment, and that's a sane thing not to budget: a new paradigm means a new language, a new runtime, a new hiring pool, and a rewrite of things that already work. Clauseway is the experiment nobody has to budget. It's a Maven dependency. Pick one aggregate whose invariants you've rewritten three times, put its rules in a class, and let the engine derive the rest — in memory today, over your tables when you're ready, with the domain code unchanged in between. Better yet, let the machine rewrite it for you and judge for yourself whether it reads better. If it doesn't earn its place, delete the class.

Brooks was right that there is no silver bullet. There is, though, a lot of accidental control in the middle of your hexagon, and forty years is a long time to keep writing it by hand.
