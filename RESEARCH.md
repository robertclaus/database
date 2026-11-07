# Finding a niche for an experimental database

_Research notes, November 2026._

## Question

Which gap in today's databases is real enough, and small enough, to justify
building a new experimental database from scratch?

## Candidate gaps

| Gap | Who covers it today | Verdict |
|---|---|---|
| Copy-on-write branches for AI agents | Neon, Xata, Dolt, OceanBase "Fork Table" | **Crowded.** Copy-on-write branching is now a standard Postgres-hosting feature. |
| Local-first / sync engines | Zero, ElectricSQL, PowerSync, Jazz, LiveStore, InstantDB, Automerge | **Crowded and changing fast.** Many projects, no clear winner, frequent pivots. |
| Incremental view maintenance / streaming SQL | Feldera (DBSP), Materialize (differential dataflow), RisingWave | **Well served.** Its math (Z-sets) is still a good foundation to build on. |
| Verifiable / ledger databases | immudb, Azure SQL Ledger. Amazon QLDB was retired on 2025-07-31. | **Small gap.** QLDB's exit left room, but buyers are few and want compliance features, not novel ideas. |
| Embedded Datalog with graph and vector features | CozoDB (no clearly dated upstream release since about 2024; forks such as cozo-ce and mnestic) | **Moderate gap**, but most of it is upkeep, not new work. |
| Probabilistic / uncertain data | Research only: ProvSQL (Postgres extension, 2025), FastPDB (SIGMOD 2025), MayBMS/Orion (unmaintained) | **Real gap.** No production system. |
| Provenance: "why is this row here?" / "why is this row missing?" | Research only: ProvSQL, GProM, PUG/PUGS (UIC) | **Real gap.** Every system is a prototype or an extension added to an existing database. |
| Hypothetical queries: "what would this return if…?" | Research only: HypeR (SIGMOD 2022 / ICDE 2023), historical what-if (Glavic et al.) | **Real gap.** Production systems can fork data, but none can diff a query's answer across a hypothetical change. |

## The pattern

The crowded categories (branching, sync, streaming) all answer **"what is
the data, quickly, wherever I need it."** The empty ones ask the database
to **reason about its own answers**:

- **Why** is this tuple in my result? Why is that one *not*?
- **How sure** am I about this tuple?
- **What if** I applied this change, or this agent's plan? Which results
  would change, and why?

Research has studied each of these questions for 15+ years, but no
production engine supports them natively. They arrive as add-ons: Postgres
extensions or query-rewriting middleware. Each add-on handles one question,
pays 2–10× overhead, and does not work with the others.

Demand for these questions is also growing:

- **AI agents** write to databases. Operators want to know *why* a row
  changed and *what would happen* before a plan commits. A forked branch
  answers "let the agent try it." It does not answer "what changed in my
  results because of it."
- **Interactive analytics:** a user clicks a bar in a chart and asks which
  rows produced it. A slider asks what the chart would look like if the
  input changed.
- **Data integration and ML pipelines** produce uncertain values, which
  people currently store as plain facts.

## Proposed thesis: a database where every row carries an annotation

There is a known theoretical result that ties these gaps together:
**provenance semirings** (Green, Karvounarakis & Tannen, PODS 2007). Annotate
each tuple with a value from a commutative semiring K. Joins multiply
annotations and unions or projections add them. The choice of K then
determines what the database computes:

| Semiring K | Annotation means | Feature you get |
|---|---|---|
| ℤ (integers) | multiplicity / delta | **Incremental maintenance.** This is exactly DBSP's Z-sets, so Feldera already runs on it. |
| ℕ[X] (polynomials over tuple IDs) | how each output was derived | **Why-provenance and lineage** |
| Boolean formulas over events | when the tuple exists | **Probabilistic queries** (probability of the formula) |
| Tropical (min, +) | cheapest derivation | Shortest-path or cost queries, confidence scores |
| Security lattice | minimum clearance needed | Fine-grained access control derived from the inputs |

Streaming engines already use the ℤ case without presenting it this way.
The experiment is to **make the semiring a first-class, pluggable part of
the engine** and build hypotheticals out of the pieces:

1. **What-if = a ℤ-delta that is never committed.** A hypothetical update
   is a Z-set Δ. Pushing Δ through the incremental dataflow gives the
   *change in every view*, with no fork and no copy, in time proportional
   to the change.
2. **Explain the what-if.** Carry ℤ × ℕ[X] (a product semiring) and the
   delta also says *which inputs caused each changed output*.
3. **Uncertainty comes from the same machinery.** A probabilistic row has
   a Boolean-formula annotation, and a query's confidence is computed from
   the annotations of its results.
4. **Why-not as a later goal.** Missing-answer explanations are the
   hardest (PUG shows that the explanation output grows very large).
   Start with "why", then add "why-not" with summarization.

### Why this is a good from-scratch project

- **New idea, small engine.** The interesting part is the algebra and the
  query engine. Storage can be simple, for example in-memory plus an
  append-only log.
- **Clear contrast with existing systems.** The research systems are
  add-ons with overhead. This would be an engine *designed around* the
  annotations. That makes one question testable: what does provenance cost
  when the engine is built for it?
- **Can be built in stages.** Each step below works on its own.

### Possible milestones

1. A relational core with Datalog or a SQL subset, where every relation is
   a K-relation and the semiring is a type parameter. Start with ℤ to get
   correct incremental views.
2. ℕ[X] provenance with `EXPLAIN WHY <tuple>`, plus a compact
   representation of the polynomials (circuits or DAG sharing, as ProvSQL
   does).
3. Product semirings and `WHAT IF <delta> SHOW CHANGES IN <view>` that
   returns changed rows with their causes.
4. A probabilistic semiring that computes probabilities by knowledge
   compilation, with sampling as a fallback.
5. Why-not explanations with summarization.

### Known risks

- **Provenance size.** Polynomials can grow exponentially. Use circuit
  representations, and let the user choose detail per query.
- **Aggregates.** Semiring provenance for aggregation needs semimodules
  (Amsterdamer, Deutch & Tannen 2011). This is the hardest part in theory.
- **Negation and recursion.** Each semiring needs its own treatment for
  negation, and recursion requires ω-continuous semirings.
- **Probability is #P-hard in general.** Plan for approximations from
  the start.

## Runner-up ideas

- **An embedded Datalog database that is maintained.** CozoDB showed the
  demand. This is useful but mostly engineering.
- **An open-source, verifiable ledger for small teams to replace QLDB.**
  It needs cryptographic proofs of history, such as Merkle trees, and
  bitemporal queries. The market is narrow but concrete.

## Sources

- Agent branching: [Xata, "Every AI agent needs a database it can break"](https://xata.io/blog/every-ai-agent-needs-a-database-it-can-break), [OceanBase Fork Table](https://en.oceanbase.com/blog/26640018176), [Postgres as AI-agent substrate](https://www.softwareseni.com/how-postgres-became-the-ai-agent-substrate-for-memory-branching-and-modern-hosting)
- Local-first: [Choosing a sync engine in 2026](https://johnny-sh.pages.dev/blog/choosing-a-sync-engine-in-2026), [Verso sync-engine notes](https://next.tangled.org/wlls.dev/verso/blob/main/docs/prototype/sync-engine.md)
- IVM: [RisingWave on IVM](https://risingwave.com/blog/what-is-incremental-view-maintenance), [dbt-feldera](https://pypi.org/project/dbt-feldera/0.339.0/)
- Ledger: [QLDB → immudb](https://codenotary.com/blog/navigating-the-transition-from-amazon-qldb-to-immudb-vault), [QLDB → Azure SQL ledger](https://techcommunity.microsoft.com/t5/azure-sql-blog/moving-from-amazon-quantum-ledger-database-qldb-to-ledger-in/ba-p/4246237)
- Datalog: [CozoDB on DBDB](https://dbdb.io/db/cozodb), [cozo-ce](https://lib.rs/crates/cozo-ce)
- Probabilistic and provenance: [ProvSQL (arXiv 2504.12058)](https://arxiv.org/abs/2504.12058), [FastPDB](https://odin.cse.buffalo.edu/research/fastpdb), [DuBio](https://www.utwente.nl/en/eemcs/dmb/research/de/DuBio/), [PUG / PUGS](https://www.cs.uic.edu/~bglavic/dbgroup/projects/pug.html)
- Hypotheticals: [HypeR](https://cris.huji.ac.il/en/publications/causal-what-if-and-how-to-analysis-using-hyper/), [Historical what-if queries (arXiv 2203.12860)](https://arxiv.org/pdf/2203.12860v1)
- Theory, cited from memory: Green, Karvounarakis & Tannen, "Provenance Semirings," PODS 2007. Amsterdamer, Deutch & Tannen, "Provenance for Aggregate Queries," PODS 2011. Budiu et al., "DBSP: Automatic Incremental View Maintenance," VLDB 2023.
