# memory-write-contention

What happens when several agents write to the same memory at once, & what each fix actually costs.

## Why

Anthropic's [The components of agent memory](https://www.anthropic.com/research/components-of-agent-memory) has a contract-terms task where several agents wrote into one shared table at the same time; only 41 of 57 facts survived. The fix (one page per contract, one writer per page, terms quoted word for word) got it to 57/57, but it changed three things at once & came from Claude rewriting its own instructions, so nobody knows which part did the work.

I think the overwrite half is a plain lost-update bug, the same one databases solved decades ago. So this repo takes the standard database fixes, runs them against agents, and prices each one in facts kept, tokens & time.

**Status:** work in progress, no results yet. Every number that ends up in here traces to a run.

## How it works

corpus -> N concurrent agents -> write strategy -> memory store -> grader -> metrics

- ~50 synthetic contracts with known terms (change of control, assignment, notice period, payment, governing law, ...) spread across ~100 short documents
- N agents (asyncio) read the documents & write the facts they find to a shared store on disk
- the grader checks the final store against ground truth by exact match (no LLM judge)

## Strategies

| strategy | reads before writing | writes |
|---|---|---|
| `naive-rewrite` | whole file | replaces the whole file (the baseline that should fail) |
| `naive-append` | whole file | adds lines at the end |
| `shard-per-entity` | its own contract's file | its own file only, enforced in code |
| `compare-and-swap` | whole file + version | replaces it if the version hasn't moved, else retries |
| `append-only` | nothing | adds an event to a log; code folds it, last line wins |
| `lock` | whole file | replaces it while holding a lock |

## Metrics

- `retention`: facts in the final store / ground-truth facts
- `loss_cause`: why each missing fact is missing (never extracted, overwritten, wrongly corrected, bad format)
- `tokens_write` / `tokens_read`: what each fix costs to write & to query
- `retries`: compare-and-swap saves that got rejected & retried
- `value_conflicts`: terms stored with two different values
- `wall_time`: how long the run takes (a lock costs time, not tokens)
- `store_ratio`: store size / corpus size

The result is two charts: retention vs tokens & retention vs wall_time, one line per strategy, swept over agent count.

## Not in v1

Retrieval, embeddings, wikis, & any claim about whether memory helps. Amendments (later documents changing earlier terms) are flagged for v2.

(the writeup will live on jinsoo.space once there's data :) )
