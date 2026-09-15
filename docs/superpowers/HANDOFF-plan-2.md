# Handoff to the session that writes plan 2

Written 2026-09-15, at the end of plan 1. Everything a fresh session needs that is not already obvious from the code, the spec or the git history. Read the spec first (`specs/2026-09-14-llm-test-forge-design.md`); this file only adds what happened during execution.

## Where plan 1 stopped, and why there

Plan 1 built the forge half: `describe` → `scenarios` → `cases` → `dedupe` → `import` → `review`. Plan 2 is the other half of the MVP as designed: `estimate` (cost before the run), `emit` (a promptfoo config plus a target shim, and a flat JSONL as the portable format), and `report` (judge disagreement and per-case stability read back from promptfoo's `results.json`).

The split was not arbitrary. Plan 1 produces reviewed cases on disk and nothing else; plan 2 is everything that happens once those cases exist. `SuiteSchema` already exists in `src/core/schemas.ts` with `target`, `judges`, `repeat` and `include`, and `forgePaths()` already returns a `suite` path. Nothing reads or writes `suite.yaml` yet. That is deliberately plan 2's first seam.

## The five things most likely to bite plan 2

1. **The coverage gate measures lines and functions only. Bun tracks no branch coverage.** Every "100%" in this repository is blind to an untested guard arm, and that blindness accounted for most of the defects found during plan 1 — including five of the six that only the whole-branch review caught. Do not let a plan-2 brief treat "100% covered" as "this case is tested". It means "no statement is unreachable", nothing more. Pin guard arms with assertions, deliberately, one per arm.

2. **A test that counts is weaker than a test that identifies.** Six of the eleven tasks produced a finding of exactly this shape: a test asserting "two came back" passing when the wrong two came back. `emit` and `report` are full of list transformations, so this will recur. Assert content: which case, which assertion, which scenario id.

3. **Mutation is the only evidence that a guard is guarded.** Every fix round in plan 1 ended by breaking the covered code on purpose and confirming the named test went red. Three separate times the mutation instrument itself failed silently — a `sed` pattern that never matched, a deletion that broke syntax so the suite failed for the wrong reason, a regex against the wrong shape. So: assert the mutant landed (grep), assert it still compiles (`tsc`), then read the result.

4. **The plan's example code was wrong in several places, and being wrong was the useful part.** The plan's mock shape did not match the installed `ai` 7.0.100 (`finishReason` is `{unified, raw}`, not a string). One task's test was structurally incapable of failing. One test forced dead defensive code into production, which then justified a forbidden cast. Write plan 2's code blocks knowing they are hypotheses; tell the implementers to take mock shapes from the existing test files, never from the brief.

5. **Run the binary, not the diff.** Every cross-cutting defect in plan 1 was found by driving `src/cli/bin.ts` in a temp directory, and none by reading. The end-to-end test and the live test are both cheap; extend them rather than adding another unit test at the same seam.

## Facts about the environment, measured

- `bun` 1.4.2. Coverage thresholds are **per file**, not aggregate — a single file below the floor fails the run even when the aggregate is far above it. This is why `src/cli/bin.ts` exists at all: it holds the shebang, the real stdin reader and `process.exit`, and it is the only file `bunfig.toml` excludes. Do not widen that exclusion; extract glue instead.
- `bun test --coverage=false` **does not exist**. Use a separate config file, as `bunfig.live.toml` does for `test:live`.
- Providers: `GOOGLE_API_KEY` is in the shell and is free tier — Flash models work, Pro answers `429 … limit: 0`, and Flash itself returns `503 high demand` intermittently, varying by model and by minute. The OpenAI account has **zero credits**; every call is `429 insufficient_quota`, and any tool that treats 429 as a rate limit will retry for minutes before failing. The Anthropic key is **not** in the environment — it lives in `~/.config/anthropic/vim-ai-autocomplete.env` and must be sourced.
- The live test (`tests/live/`) is off unless `FORGE_LIVE=1`, defaults to `google/gemini-3.5-flash`, and retries **only** on a transient 503, three attempts. It ran for real once and produced 12 scenarios covering all six kinds.

## Design decisions plan 2 inherits, with the reasoning

- **`review --all` refuses to approve a case with no `expected`.** Imported cases arrive without one by design; approving them in bulk would produce exactly the "inputs only" suites this product exists to replace. The interactive path now validates the human's answer against the feature's declared labels, with the same `expectedMatchesOracle` applied to the model (`src/core/oracle.ts`).
- **A changed `scenario` field on a case is refused by name, not supported.** That is a file move — two files, id renumbering, possibly a different oracle — and not a review decision. A `forge move` verb is the honest way to support it, if it is ever wanted.
- **Duplicate placement is single-hop.** Chains of duplicates-of-duplicates are order-dependent and documented as such in `src/core/review.ts`. Nothing is lost when one occurs.
- **`slugify` drops accents rather than transliterating.** Not a correctness bug, since `uniqueId` disambiguates, but the first real target classifies Portuguese email, so ids read worse than they should. It is in the backlog.
- **`sameInput` is duplicated** between `src/core/cases.ts` and `src/core/import.ts`, deliberately. Eight lines, two call sites, no behavioural coupling; extracting it would be the drive-by refactor rather than the fix.

## What plan 2 should verify early, because plan 1 could not

- Whether promptfoo's `results.json` `cost` field includes judge calls or only the target's. The investigation could not tell, and `report`'s "real versus estimated" comparison depends on it.
- Whether promptfoo's Python provider can call moonlighter's real `classify_response` through the emitted shim. The whole point of the target shim is that the suite measures the application, not the prompt's formatting — the investigation found eight false failures from testing a raw prompt instead.
- How promptfoo's MCP server behaves with a config the forge wrote, rather than one written by hand.

## The first real target

moonlighter's `classify_response` (`~/Programming/moonlighter/packages/email/moonlighter/tracking/classification.py`): one email in, one of seven labels out, small input, and an explicit rule about automated acknowledgements that a prompt tweak could break. Its code does **not** validate the returned label against the enum, so an invented label passes silently — a regression class the generated suite would catch and its unit tests do not. `examples/` is the place for its `.forge/` once plan 2 can produce a full one.

## Backlog

`BACKLOG.md` carries 18 items, 15 of them lifted out of plan 1's execution ledger before that ledger was deleted. Read it before writing plan 2; several items are cheap enough to fold into a task rather than live on as separate work.

## Addendum, 2026-09-15: the three open questions, measured

Measured by running promptfoo 0.123.0 from a scratch directory before plan 2 was written; the full list of facts is in `plans/2026-09-15-run-half.md` under "Verified facts", and the raw output of the run is `tests/fixtures/promptfoo-results-0.123.0.json`.

- **`cost` in `results.json` is the target's cost only.** Verified by arithmetic on the spike's Haiku rows (403 in × $1/M + 89 out × $5/M = $0.000848, exactly the row's `cost`). Judges appear as tokens under `gradingResult.componentResults[].tokensUsed`, never as dollars; `report` prices them from `prices.yaml`. A Python target reports `cost: 0` and zero tokens unless the shim returns `tokenUsage`/`cost`, which promptfoo then keeps verbatim.
- **promptfoo's Python provider does call moonlighter's real `classify_response`.** An `async def call_api` importing `classify_response` and `make_api_caller`, run through moonlighter's venv via `pythonExecutable`, passed 4/4 against Anthropic Haiku. `options["config"]` carries the provider's config block, which is how the per-model providers pass the model; the spec's `FORGE_MODEL` environment variable is not a promptfoo mechanism and plan 2 uses `config.model` instead.
- **The MCP server works with a config the forge shape produces.** `promptfoo mcp --transport stdio` lists 14 tools; `validate_promptfoo_config` returned `isValid: true` on the probe config, and `run_evaluation` takes `configPath` and `repeat`. Nothing in plan 2 depends on it.

Two more facts that changed the plan: `tests[].metadata` reaches every result row (so `report` matches on it, not on `testIdx`, which `repeat` renumbers), and installing promptfoo costs 2.1 GB and 79 s, so it is not a devDependency — CI validates the example with `bunx promptfoo@0.123.0 validate`.
