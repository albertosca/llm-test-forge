# Backlog

Deferred on purpose during the design of 2026-09-14 (see `specs/2026-09-14-llm-test-forge-design.md`).

- **MCP server** — a second thin shell over the same core verbs, so the review loop can run inside a conversation. Comes after the CLI proves the library.
- **`forge run`** — invoke `promptfoo eval` from inside the forge and produce the report in one step. Costs a dependency on the installed promptfoo binary and handling its errors; `emit` + `report` ship, and the person runs `bunx promptfoo eval` between them by hand.
- **Multi-turn with a simulated user** — cases where the application's own reply shapes the next turn. Per-turn testing (`history` + `message` as input fields) is already covered; simulation is a different product.
- **Multi-run history** — keep `report.json` across runs in one place and plot trends. `report --baseline` compares exactly two runs and nothing accumulates them.
- **More providers** — OpenRouter, Mistral and others in the registry. MVP ships Anthropic, Google, OpenAI, Ollama.

## Carried out of the core-pipeline execution ledger (2026-09-15)

Decisions taken and deliberately deferred while executing `plans/2026-09-14-core-pipeline.md`. They lived in `.superpowers/`, which is gitignored, so they are written down here before that branch merges and they evaporate. None of them is a known wrong answer; each is a place where the answer was postponed.

- **`pendingItems` places duplicates single-hop only** — a case carrying `duplicate_of` is moved next to its target in one left-to-right pass; chains of duplicates-of-duplicates come out order-dependent. Ruled on, documented in the function's own doc comment, and pinned by a test. Revisit only if real suites grow chains.
- **The coverage gate is blind to branches** — bun measures lines and functions and tracks no branch coverage, so a file at 100% can still have guard arms nobody executed. Five of the six defects the whole-branch review found were exactly that. Either move to a tool that measures branches, or treat the number as a floor and keep hunting by hand.
- **Only one of the three badges the spec asks for is shipped** — the README carries the CI badge. A lint badge is honest, since a real `lint` job exists in `ci.yml`, but GitHub's `badge.svg` is per workflow rather than per job, so it needs the lint job split into its own workflow file. A coverage badge needs a service (Codecov or similar) that the project does not use yet.
- **Test scaffolding is in the production import graph** — `src/llm/generate.ts` imports `MockLanguageModelV4` from `ai/test` to back the documented `fake/<file>` provider. It works and the provider is a real feature, not a test-only hook, but shipping a dependency's test module deserves a deliberate yes or no.

## Carried out of the run-half execution ledger (2026-09-16)

- **Token counting is characters ÷ 4 for every provider** — `approxTokens` is the same division whatever the model is, so `estimate` is an order of magnitude, not a number to budget against. A real tokenizer (Anthropic's `count_tokens` endpoint, or a local BPE) is still a follow-up; nothing here replaces the division. `JUDGE_OUTPUT_TOKENS` itself is no longer a guess: calibrated on 2026-09-16 to 174, the mean of `completion + completionDetails.reasoning` over the six `llm-rubric` components of the example run it was fit to (321 over the two of the report-fixture file), because a rejecting judge writes a long explanation and a passing one does not. The example has been re-run since, with two rejecting judges instead of three: the same six calls wrote 666 output tokens rather than 1042, so the committed report now shows the judge estimate 23.9% high. The error tracks how many rubric cases fail, which no constant can know in advance.
- **The example's feature carries one `email` string where moonlighter wants three fields** — the shim's `parse_email` reconstructs `{from_, subject, body}` from that one string (fixed in 732c140, after the split-on-blank-line version dropped bodies), so it stays a parser written against the case shapes seen so far; giving the feature `from`, `subject` and `body` as three inputs would remove the guessing.

## Carried out of the whole-branch review of the run half (2026-09-15)

Findings triaged as too small to fix in the review's own fix wave. Each is one line because each is one place to look.

- The `promptfoo-validate` CI job downloads 2 GB of promptfoo on every push and has not yet been observed passing on `ubuntu-latest`. Suggestion: `actions/cache` on `~/.bun/install/cache`.

## Carried out of the backlog-sweep execution ledger (2026-09-16)

Minors ruled on and deliberately deferred while executing the backlog sweep. They lived in `.superpowers/`, which is gitignored, so they are written down here before the branch merges and they evaporate. None is a known wrong answer; each is a place where the answer was postponed.

- **A `forge move` verb** — renaming a scenario is refused in review since 55a1cb5, because a scenario's id names `.forge/cases/<id>.yaml`; renaming a case's `scenario` is refused for the same reason. Both refusals send the person to `mv` plus a hand edit. The honest way to support either is a verb that moves the file and rewrites the ids it contains in one write, not a review decision that half-applies across files nobody opened.
- **`report.failing` strings can collide on a case id containing ` @ `** — the list holds `"<case> @ <model>"`, so two different pairs can spell the same entry. The Markdown tables already pick their rows by stability rather than by looking the string up, so nothing reads it ambiguously today. Structured pairs would fix it and would break every `--baseline` written so far.
- **A scenario whose oracle changed but has no cases file prints no "changed oracle" line** — the re-open pass walks the cases files, so a scenario with none is silent about a change that will matter as soon as `cases` runs for it.
- **`review --all` does not check a case's `scenario` field against the file it lives in** — a hand-edited `.forge/cases/<id>.yaml` holding a case that names another scenario is reviewed as if it belonged there.
- **`renamedCases` is global across cases files** — `duplicate_of` pointers are rewritten from one map for the whole pass, so a rename in one file could in principle follow a pointer in another. Case ids are scenario-prefixed today, which makes the collision unlikely rather than impossible.
- **The two `as` narrowings in `onDecided`** — `item as Feature` and `item as Scenario`, which exist because the callback takes a `kind` and a bare item rather than a discriminated union. A `PendingItem`-shaped argument would remove both without a cast.
- **The duplicated `editorReplacing` test helper** — `tests/cli/main.test.ts` defines the same helper four times, once per `describe` block that needs it (lines 820, 1328, 1711, 1896). One definition at the top of the file would do.
- **`src/cli/commands/review.ts` is 358 lines and wants a split** — the `--all` path and the write-back are each a module's worth of work sitting inside the command.
- **`report.json`'s `failing` key means failing, errored or partly-errored** — the name says less than the list holds. Renaming it would break `--baseline` for every report written so far, so the doc comment carries the meaning instead.
