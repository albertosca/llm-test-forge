# Backlog

Deferred on purpose during the design of 2026-09-14 (see `specs/2026-09-14-llm-test-forge-design.md`).

- **MCP server** — a second thin shell over the same core verbs, so the review loop can run inside a conversation. Comes after the CLI proves the library.
- **`forge run`** — invoke `promptfoo eval` from inside the forge and produce the report in one step. Costs a dependency on the installed promptfoo binary and handling its errors; `emit` + `report` ship, and the person runs `bunx promptfoo eval` between them by hand.
- **Multi-turn with a simulated user** — cases where the application's own reply shapes the next turn. Per-turn testing (`history` + `message` as input fields) is already covered; simulation is a different product.
- **Multi-run history** — keep `report.json` across runs in one place and plot trends. `report --baseline` compares exactly two runs and nothing accumulates them.
- **More providers** — OpenRouter, Mistral and others in the registry. MVP ships Anthropic, Google, OpenAI, Ollama.

## Carried out of the core-pipeline execution ledger (2026-09-15)

Decisions taken and deliberately deferred while executing `plans/2026-09-14-core-pipeline.md`. They lived in `.superpowers/`, which is gitignored, so they are written down here before that branch merges and they evaporate. None of them is a known wrong answer; each is a place where the answer was postponed.

- **`fakeCursors` is process-lifetime module state** — `src/llm/generate.ts` keys a module-level `Map` by fixture path plus verb, so two runs of the same verb against the same fixture inside one process continue the sequence rather than restarting it. Fine for the CLI (one process, one run) and relied upon by the tests; a second consumer in the same process would inherit a cursor it never set.
- **`pendingItems` places duplicates single-hop only** — a case carrying `duplicate_of` is moved next to its target in one left-to-right pass; chains of duplicates-of-duplicates come out order-dependent. Ruled on, documented in the function's own doc comment, and pinned by a test. Revisit only if real suites grow chains.
- **The coverage gate is blind to branches** — bun measures lines and functions and tracks no branch coverage, so a file at 100% can still have guard arms nobody executed. Five of the six defects the whole-branch review found were exactly that. Either move to a tool that measures branches, or treat the number as a floor and keep hunting by hand.
- **`import --oracle` is silently ignored when the `imported` scenario already exists** — `import` only writes the scenario when it is absent, so the flag has no effect on the second import onwards and says nothing. Decide: refuse the flag, or apply it to the existing scenario and say what changed.
- **`review --all` parses and validates `--only`, then ignores it** — an unknown value is still rejected, but a valid one silently does nothing on that path. Either honour it or refuse the combination.
- **`oracleOf` falls back to `rubric` for a case whose scenario is missing** — `src/cli/commands/review.ts` asks the person for a rubric sentence when it cannot find the case's scenario, which is the wrong question rather than an error. Now harder to reach (a bad `--scenario` is refused up front), still reachable through a cases file whose `scenario` field names nothing.
- **Only one of the three badges the spec asks for is shipped** — the README carries the CI badge. A lint badge is honest, since a real `lint` job exists in `ci.yml`, but GitHub's `badge.svg` is per workflow rather than per job, so it needs the lint job split into its own workflow file. A coverage badge needs a service (Codecov or similar) that the project does not use yet.
- **Test scaffolding is in the production import graph** — `src/llm/generate.ts` imports `MockLanguageModelV4` from `ai/test` to back the documented `fake/<file>` provider. It works and the provider is a real feature, not a test-only hook, but shipping a dependency's test module deserves a deliberate yes or no.
- **A rename can still leave a `duplicate_of` pointing at nothing** — review writes an edited item back under its new id (2026-09-15); a case renamed while another case points at it through `duplicate_of` leaves that pointer dangling. No data is lost, but `pendingItems` stops placing the pair together. *Corrected on 2026-09-15:* this entry originally also claimed that renaming onto an id another item already uses "loses no data". That was wrong and was measured wrong — a duplicated id made one decision resolve over its twin and destroyed the twin's content. Creating such a collision by editing is now refused by `applyDecision`. What remains open is the file that already contains one: `pendingItems` places one entry per id, so the twin is not offered for review at all, and nothing reports that it was passed over.
- **Nothing refuses editing a scenario's `oracle`** — the mirror of the case-level `scenario` refusal added on 2026-09-15, with a wider blast radius: changing a scenario from `label` to `fields` invalidates the `expected` of every case in that scenario at once, and nothing notices until each case is next reviewed. Decide whether to refuse it, or to accept it and re-open every affected case. *Narrowed on 2026-09-15:* `selectCases` now blocks such a pair, so `estimate`, `emit` and `report` all name the case and its mismatch instead of emitting a vacuous assert. What stays open is the review prompt itself, which still lets the edit through without re-opening the cases.
- **The review pass works from a snapshot** — the feature and the scenario list are read once, before the loop. So editing the feature's `output.labels` to add a label and then, in the same pass, approving a case that uses it is wrongly refused by the oracle check. It is a refusal, not corruption, and re-running `forge review` clears it; a person who does not know that will read it as a bug.
- **Bare `dedupe` validates inside its loop** — it refuses a scenario with no cases as it reaches it, so a legacy empty cases file in the middle of the list leaves the scenarios before it already written and exits non-zero. Validating every id before the first write would make the verb all-or-nothing, which is what the rest of the pipeline promises.

## Carried out of the run-half execution ledger (2026-09-16)

- **Token counting is characters ÷ 4 for every provider** — `approxTokens` is the same division whatever the model is, so `estimate` is an order of magnitude, not a number to budget against. A real tokenizer (Anthropic's `count_tokens` endpoint, or a local BPE) is still a follow-up; nothing here replaces the division. `JUDGE_OUTPUT_TOKENS` itself is no longer a guess: calibrated on 2026-09-16 to 174, the mean of `completion + completionDetails.reasoning` over the six `llm-rubric` components of the committed example run (321 over the two of the report-fixture file), because a rejecting judge writes a long explanation and a passing one does not.
- **The example's shim reports zero target tokens** — `examples/moonlighter-classify-email/.forge/forge_target.py` returns `input_tokens: 0`/`output_tokens: 0` because moonlighter's `make_api_caller()` does not expose usage, so the target line of `report.md`'s cost table is empty and says so. Wiring moonlighter's own `record_call` through would make that line real.
- **The example's feature carries one `email` string where moonlighter wants three fields** — the shim's `parse_email` reconstructs `{from_, subject, body}` from that one string (fixed in 732c140, after the split-on-blank-line version dropped bodies), so it stays a parser written against the case shapes seen so far; giving the feature `from`, `subject` and `body` as three inputs would remove the guessing.

## Carried out of the whole-branch review of the run half (2026-09-15)

Findings triaged as too small to fix in the review's own fix wave. Each is one line because each is one place to look.

- The flaky and failing tables filter `report.cases` by the display string `"<case> @ <model>"` rather than by the pair, so a case id containing ` @ ` could collide.
- The ragged padding in the judge-disagreement table (a row with fewer judges than the widest) is untested.
- Unmatched rows are labelled by `testIdx`, which repeats across repeats: with `repeat > 1` two unmatched rows can carry the same number.
- One `Disagreement` per row means a case that disagrees on every repeat inflates the headline count by `repeat`.
- The coverage sentence says "approved scenarios" while the count includes edited ones.
- The zero-matched `ForgeError` names no observed case id, so it cannot be told from a typo in one id.
- The README's command block exports one `FORGE_MODEL` while the example's `describe` and `scenarios` ran on others; the table above it says so, the block does not.
- `biome.json` negates one level deep only, so a nested ignore pattern would not apply.
- A case that passes one repeat and errors on another reads as `flaky`, although nothing about it was unstable.
- A judge provider with no label comes back named by its model string rather than by the forge's name.
- `examples/moonlighter-classify-email/.forge/usage.jsonl` is gitignored on purpose (it carries per-call figures from the run machine); the example README now says so.
- The `promptfoo-validate` CI job downloads 2 GB of promptfoo on every push and has not yet been observed passing on `ubuntu-latest`. Suggestion: `actions/cache` on `~/.bun/install/cache`.
- `ReportSchema.failing` is required, so `forge report --baseline` refuses a `report.json` written before 126ba62 with `failing: Required`; `.default([])` would keep old baselines readable at no cost (the diff never reads the field). Parked at the end of plan 2 because the project is pre-release and the only committed report was regenerated.
