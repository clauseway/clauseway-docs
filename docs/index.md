# Clauseway

**A logic programming stack for real problems.** Clauseway is a Java
library family for writing programs as relations: unification,
finite-domain constraints, tabling and weighted inference over real
data sources — SQL databases included — with transactional writes and
pinned, reproducible reads.

## The stack

| Part | What it is |
| --- | --- |
| [**functional**](https://github.com/clauseway/functional) | The substrate: fibers, fair schedulers, continuations — the machinery that keeps relational search complete and debuggable. |
| [**logic**](https://github.com/clauseway/logic) | The engine: unification, constraint stores (finite domains, disequality, nogoods), tabling with constraint-aware answer caching, weighted inference. |
| [**pldb**](https://github.com/clauseway/pldb) | The data boundary: relations backed by in-memory or SQL sources, constraint pushdown into WHERE clauses, transactions that certify their reads. |
| [**library**](https://github.com/clauseway/library-test) | The worked example: a lending-library domain built entirely from rules — policies as relations, denials as their complement. |

## Why

Most business logic is relations wearing imperative costume. Clauseway
exists to let the relations speak for themselves: a rule is stated once
and runs in every direction — computing due dates, inferring checkout
days, or refusing inconsistent data — over types that matter and data
that lives in a real database. The full argument is the opening essay:
[**The half of the Tar Pit we skipped**](blog/posts/the-half-of-the-tar-pit-we-skipped.md).

## Where to start

- The [blog](blog/index.md) — development notes along the way, opening
  with the essay that explains why this stack exists.
- The repositories above — each part builds with plain Maven.
