# Finding a niche for an experimental database

_Research notes, November 2026. This is a second pass. The first pass
(commit `51870d0`) proposed a database built on provenance semirings. This
pass checked that proposal against prior art. It is still the best
direction, but the gap is narrower than the first pass claimed._

## Summary

- **The first pass's main claim is now out of date.** It said no production
  engine supports provenance and uncertainty natively. But
  [ProvSQL](https://pgxn.org/dist/provsql/) released 1.0 on 2026-04-05, and
  its changelog calls it "fully documented and usable in production." It
  has had at least 17 releases since, up to 1.13.1 on 2026-11-01. It
  supports:
  - provenance in any semiring the user picks;
  - probabilities and Shapley values;
  - aggregates and recursion;
  - tracking of INSERT, UPDATE and DELETE "enabling temporal queries and undo";
  - clearance and consent semirings.

  A semiring engine that supports provenance and probabilities is no longer
  a gap.
- **The first pass's design had an algebra error.** DBSP-style incremental
  computation needs weights with additive inverses (an abelian group).
  ℕ[X], the Boolean semiring and the tropical semiring have no inverses.
  Turning the Booleans into a ring collapses them to the trivial ring {0}
  (Suciu 2024). So "ℤ × ℕ[X]" and "Boolean formulas, maintained
  incrementally" do not work as described.
- **A narrower gap is still open, and the earlier work points straight at
  it:** **speculative, incremental provenance**. It needs an engine that:
  1. maintains provenance-annotated SQL views under both inserts and deletes,
     without recomputing them;
  2. answers "if I applied this change, which results would change, and
     which inputs would cause each change?";
  3. does this in time proportional to the change, not to the data.

  Nobody ships this:
  - ProvSQL recomputes materialized views in full on `REFRESH`.
  - Feldera and Differential Dataflow maintain views incrementally, but
    without provenance.
  - The newest theory on incremental maintenance with semirings handles
    inserts only. Its authors say deletes are out of scope ("Our setting
    forbids the deletes").
- **The strongest alternative is a memory store for robots and other
  embodied agents.** That is a real gap with stronger outside demand. It is
  mostly domain engineering and is filling quickly (details below).

## What the first pass got wrong or missed

| First pass said | Corrected |
|---|---|
| Provenance: "every system is a prototype or an extension" | ProvSQL is an extension, but now production-labelled and very active ([changelog](https://pgxn.org/dist/provsql/1.4.0/CHANGELOG.html), [1.13.0 release](https://github.com/PierreSenellart/provsql/releases/tag/v1.13.0)). |
| Probabilistic data: "research only" | ProvSQL computes probabilities (d-DNNF compilation and Monte Carlo). [FastPDB](https://odin.cse.buffalo.edu/papers/2025/SIGMOD-FastPDB.pdf) (SIGMOD 2025) is still research; its code is listed as "To Be Released." |
| "Make the semiring a pluggable part of the engine" was new | It is not new. Scallop, Dyna, ProvSQL and the payload types of Differential Dataflow all have pluggable semirings. [F-IVM](https://arxiv.org/abs/2303.08583) maintains views over pluggable *ring* payloads. |
| What-if as a delta that is never committed was new | Datomic's `d/with` runs speculative transactions (with no provenance or view maintenance). Griffin & Hull (SIGMOD 1997) did hypothetical queries with first-class deltas. Mahif (SIGMOD 2022) answers historical what-if queries. |
| Product semiring ℤ × ℕ[X] inside DBSP | Not a group, so it cannot be a DBSP weight. Use ℤ[X] (integer-coefficient polynomials). See "Why the gap exists". |
| Missed: incremental explanations | [Chothia, Liagouris, McSherry & Roscoe, VLDB 2016](https://www.vldb.org/pvldb/vol9/p1137-chothia.pdf). On Differential Dataflow, it "computes and updates explanatory inputs even as the inputs … change." Its explanations are sets of inputs, not semiring polynomials, and it has no speculative API. |
| Missed: incremental provenance in Datalog | [Zhao et al. 2025](https://arxiv.org/abs/2501.09225): incremental provenance in Soufflé suggests which recent input changes to roll back. |
| CozoDB: no release "since about 2024" | The last release is v0.7.6, from December 2023. |

## Landscape (updated)

| Gap | Who covers it today | Verdict |
|---|---|---|
| Copy-on-write branches for AI agents | Neon, Xata, Dolt, OceanBase seekdb `FORK TABLE` (experimental in V1.1.0) | **Crowded** |
| Local-first / sync engines | Zero, ElectricSQL, PowerSync, Jazz, LiveStore, InstantDB | **Crowded** |
| Incremental view maintenance / streaming SQL | Feldera (DBSP), Materialize, RisingWave | **Served** |
| Bitemporal | XTDB v2 (GA 2025, 2.2 in 2026) | **Served** |
| Graph queries inside SQL | SQL/PGQ committed to PostgreSQL 19 on 2026-03-16 | **Served** |
| LLM / semantic operators in queries | BigQuery, LOTUS, Palimpzest, ThalamusDB. There is already a benchmark comparing them (SemBench, VLDB 2026). | **Crowded** |
| Provenance + probability (static) | ProvSQL 1.x | **Served** (as of 2026) |
| Ledger / verifiable | immudb, Azure SQL Ledger. QLDB was retired on 2025-07-31. | Small gap, narrow market |
| Embedded Datalog | CozoDB (last release Dec 2023), forks | Moderate gap, mostly upkeep |
| Several schema versions live at once, inside the engine | pgroll and Reshape (active, built on Postgres views). Cambria and InVerDa (research, inactive). | Moderate gap. View-based tools cover most practical needs. |
| Access control that checks what queries reveal | Blockaid (OSDI 2022, inactive). Oracle has announced "Deep Data Security" in 26ai for agents (vendor claim, not checked). | Moderate gap. Vendors are absorbing the simple end. |
| Optimisation / decision queries inside the database | SketchRefine, DeQL, DeciDB (active research groups) | Moderate gap. You would compete with the field's founders. |
| Causal inference over relations | CaRL, CauSumX (research) | Research only, weak practitioner demand |
| **Memory store for embodied agents** | Custom stacks only (see below) | **Real gap, filling fast** |
| **Speculative, incremental provenance** | Pieces exist separately (see below) | **Real but narrow gap** |

## Recommended thesis: speculative, incremental provenance

### The question it answers

> "If this change were applied, which rows in which views would change,
> and which input tuples would cause each change?" The answer should cost
> time proportional to the change, and nothing is committed.

### Demand

AI agents now write to production databases, and tooling for 2026 has
grown around *previewing* their writes. Examples:

- pg-dry-run
- dry-run MCP tools for Postgres
- Harness "DB Test and Preview"
- [writing on agent dry-run modes](https://dev.to/mads_hansen_27b33ebfee4c9/your-ai-database-agent-needs-dry-run-mode-3kdk)

All of them preview the **rows the write touches**, such as affected row
counts and before/after values. None of them answer what reviewers
actually need: **what does this do downstream**? Which dashboard numbers,
derived tables or agent-facing views change, and why? A copy-on-write
branch lets an agent try the change. It cannot tell you how the results
differ, or why.

The same question appears in other places:

- interactive analytics: "if I exclude this supplier, what changes in this
  chart?";
- data debugging: "which upstream rows caused this metric to move?";
- access control: "if this grant is revoked, what disappears from this
  user's views?", using a clearance semiring.

### Prior art, by piece

| Piece | Who has it | What is missing |
|---|---|---|
| Semiring provenance, probability, aggregates | ProvSQL | Views are recomputed in full; nothing is incremental or speculative |
| Incremental views over rings | DBSP/Feldera (weights fixed to `i64`), Differential Dataflow (generic payload types), F-IVM | No provenance exposed |
| Incremental views over semirings | [Chmielewski, Draghici, Olteanu & Zhang, 2026](https://arxiv.org/abs/2606.07795) | Inserts only; deletes "forbidden" |
| Incremental explanations | Chothia et al. 2016, Soufflé incremental provenance 2025 | Explanations are input sets or proof trees, not algebraic; no SQL-view API; no speculation |
| Speculative transactions | Datomic `d/with`, Griffin & Hull 1997, Mahif 2022 | No incremental provenance |

No system combines all three: incremental maintenance, provenance under
deletes, and a speculative API.

### Why the gap exists, and the design that gets around it

The algebra makes this hard, and that is what makes it a research
project rather than plain engineering:

1. **Weights must form a ring.** DBSP incremental computation subtracts,
   so weights need additive inverses. ℤ[X] (polynomials over input-tuple IDs
   with integer coefficients) is a commutative ring, so it works as a DBSP
   weight. Insert tuple *t* with weight +x\_t, delete it with −x\_t.
2. **Positive queries stay positive.** † _Our own reasoning, to be checked._
   For select, project, join and union over the current set of tuples, the
   stored annotation equals what recomputation would give in ℕ[X]. Because
   of that, the usual mappings still apply when results are *read*:
   - ℕ[X] → ℕ gives counts;
   - → Why(X) gives why-provenance;
   - → PosBool(X), then a probability, gives lineage confidence.

   Uncertainty therefore fits, but it is computed when a result is read,
   not maintained incrementally.
3. **Non-linear operators fall back to recomputing per key.** Distinct,
   difference, MIN/MAX and selections on aggregates have no linear
   incremental form over polynomials. ℤ-difference also breaks expected
   identities (Amsterdamer, Deutch & Tannen, TaPP 2011). DBSP already
   handles `distinct` by recomputing each affected key. The research
   question is what the provenance of those per-key recomputations should
   be.
4. **Speculation is close to free for one step.** † _Design reasoning._ A
   DBSP incremental operator computes its output delta from (current
   state, Δ). A single hypothetical step is therefore a *read-only*
   evaluation, with no fork or copy needed. A chain of hypothetical steps
   needs a copy-on-write overlay of operator state.
5. **Zero-testing is a new systems problem.** Polynomials blow up, so store
   them as shared circuits (as ProvSQL does). But DBSP must know when a
   weight is zero, so it can drop the row, and testing whether a circuit
   with subtraction equals zero is polynomial identity testing.
   † _Idea:_ store a fingerprint next to each circuit, its value at a
   random point mod a large prime (Schwartz–Zippel). This gives a fast
   zero test that is wrong with only tiny probability.
6. **Cancellation is part of the meaning.** x − x = 0 hides that a tuple
   was inserted and then deleted. That is correct for "net effect" but
   wrong for an audit trail. Decide which one the engine promises.

### Scope for a first version

- Non-recursive SQL: select, project, join, union, group-by with SUM and
  COUNT. Distinct is recomputed per key.
- In-memory state plus an append-only log. Storage is not the experiment.
- ℤ[X] weights stored as hash-consed circuits with fingerprints.
- Read-time evaluators for count, why-provenance and lineage probability,
  with Monte Carlo as the fallback for probability.
- API: `WHAT IF <delta> SHOW CHANGES IN <view>`. It returns, for each
  changed row, the change in its weight and the input tuples that caused
  it.

### Milestones

1. A ℤ-weighted incremental engine (a small DBSP clone). Test it against
   recomputation from scratch.
2. Switch the weights to ℤ[X] circuits, add fingerprints and add
   `EXPLAIN WHY`.
3. Speculative single-step `WHAT IF`, then multi-step with copy-on-write
   overlays.
4. Read-time semiring evaluators, including probability.
5. Provenance through per-key recomputation of distinct and MIN/MAX.
6. Stretch goal: maintain provenance correctly under deletes for some
   class of recursive queries.

### How to evaluate it

- **Baselines:**
  - ProvSQL with `REFRESH MATERIALIZED VIEW` (full recomputation);
  - Feldera or DBSP with no provenance, to measure what provenance costs;
  - Chothia et al. for explanation quality.
- **Claim to test:** the cost of `WHAT IF` and of keeping provenance up to
  date grows with |Δ| and the size of the affected keys, not with |DB|.
  Provenance should add a constant-factor overhead over plain ℤ weights.
- **Workloads:** TPC-H or TPC-DS views with small update batches. Also a
  replay of an agent's write log against a set of dashboards.

### Kill criteria

- Circuit size or fingerprint overhead makes provenance more than about
  10× slower than ℤ-only DBSP on simple joins.
- Most useful queries contain non-linear operators, so per-key
  recomputation dominates and the engine is effectively ProvSQL with
  caching.

## Strongest alternative: a memory store for embodied agents

- **Gap.** Robots and embodied agents need a persistent world memory. It
  has to cover:
  - objects whose identity persists over time;
  - uncertain positions;
  - embeddings;
  - queries that mix meaning, space, time and graph relations.

  No database is designed around that.
- **Evidence.** Research groups keep building their own stores:
  - [eMEM](https://arxiv.org/abs/2606.03374) (June 2026) combines SQLite,
    hnswlib and an R-tree running inside the agent's process.
  - An [MIT scene-graph paper](https://arxiv.org/abs/2510.16643) (October
    2025) stores 3D scene graphs in a graph database that an LLM queries
    through Cypher.
  - Others: ReMEmbR (2024) and RoboMemory (2025).

  Well-funded tools (Foxglove, Rerun) target offline fleet logs, not
  memory on the robot.
- **Why it ranks second.**
  - Demand is still mostly academic.
  - The work is heavy on domain engineering: spatial, vector and temporal
    indexes plus a planner across them.
  - New projects appear every few months, so the gap may close within
    about a year.
- **Overlap with the main thesis.** "Why do I believe the mug is in the
  kitchen?" and "how sure am I?" are provenance and uncertainty questions.
  An incremental-provenance core could later serve this domain.

## Sources

The claims behind the recommendation were checked at the source on
2026-11-07: ProvSQL, the 2026 semiring-maintenance paper, Chothia et al.,
FastPDB, eMEM, the MIT scene-graph paper, QLDB, CozoDB, SQL/PGQ and
OceanBase. The status details for the moderate-gap rows of the landscape
table come from a broader sweep and were only spot-checked.

- **ProvSQL:** [PGXN](https://pgxn.org/dist/provsql/), [1.13.0 release notes](https://github.com/PierreSenellart/provsql/releases/tag/v1.13.0), [1.0–1.4 changelog](https://pgxn.org/dist/provsql/1.4.0/CHANGELOG.html), [ProvSQL paper (arXiv 2504.12058)](https://arxiv.org/abs/2504.12058)
- **Semiring incremental maintenance:** [The Role of Semirings in Incremental View Maintenance (arXiv 2606.07795)](https://arxiv.org/abs/2606.07795), [F-IVM (arXiv 2303.08583)](https://arxiv.org/abs/2303.08583)
- **Differences in semirings:** [Suciu, "Different Differences in Semirings" (OASIcs, Tannen Festschrift, 2024)](https://drops.dagstuhl.de/entities/document/10.4230/OASIcs.Tannen.10)
- **Incremental explanations:** [Chothia et al., VLDB 2016](https://www.vldb.org/pvldb/vol9/p1137-chothia.pdf), [Zhao et al., "Provenance Guided Rollback Suggestions" (arXiv 2501.09225)](https://arxiv.org/abs/2501.09225)
- **Probabilistic data:** [FastPDB (SIGMOD 2025)](https://odin.cse.buffalo.edu/papers/2025/SIGMOD-FastPDB.pdf)
- **Why-not provenance:** [PUG (arXiv 1808.05752)](https://arxiv.org/pdf/1808.05752)
- **Agent dry-run demand:** [dev.to: "Your AI database agent needs dry-run mode"](https://dev.to/mads_hansen_27b33ebfee4c9/your-ai-database-agent-needs-dry-run-mode-3kdk), [Harness DB Test and Preview](https://developer.harness.io/docs/database-devops/use-database-devops/step-guide/test-and-preview)
- **Agent branching:** [OceanBase Fork Table](https://en.oceanbase.com/blog/26640018176), [Xata](https://xata.io/blog/every-ai-agent-needs-a-database-it-can-break)
- **Embodied memory:** [eMEM (arXiv 2606.03374)](https://arxiv.org/abs/2606.03374), [3D scene graphs + Cypher (arXiv 2510.16643)](https://arxiv.org/abs/2510.16643), [ReMEmbR (arXiv 2409.13682)](https://arxiv.org/pdf/2409.13682), [RoboMemory (arXiv 2508.01415)](https://arxiv.org/html/2508.01415v6)
- **Served categories:** [SQL/PGQ commitfest](https://commitfest.postgresql.org/58/4904), [XTDB v2](https://www.xtdb.com/v2), [SemBench (arXiv 2511.01716)](https://arxiv.org/abs/2511.01716)
- **Moderate gaps:** [pgroll](https://github.com/xataio/pgroll), [Blockaid (OSDI 2022)](https://www.usenix.org/conference/osdi22/presentation/zhang), [Oracle Deep Data Security](https://blogs.oracle.com/database/oracle-deep-data-security-is-now-available-in-oracle-ai-database-26ai), [Stochastic SketchRefine (PVLDB)](https://www.vldb.org/pvldb/vol18/p3106-haque.pdf)
- **Ledger and Datalog:** [QLDB end of support](https://dev.classmethod.jp/articles/amazon-qldb-eos-announcement/), [CozoDB releases](https://github.com/cozodb/cozo/releases)
- **Cited from memory, not re-checked:**
  - Green, Karvounarakis & Tannen, PODS 2007
  - Amsterdamer, Deutch & Tannen, PODS 2011 and TaPP 2011
  - Budiu et al., DBSP, VLDB 2023
  - Griffin & Hull, SIGMOD 1997
  - Datomic `d/with`
