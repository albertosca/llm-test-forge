# Backlog sweep, group A — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the 33 backlog items that need no decision from Alberto (`docs/superpowers/BACKLOG.md`, sections "Carried out of the core-pipeline execution ledger", "Carried out of the run-half execution ledger" and "Carried out of the whole-branch review of the run half"), smallest first, each with a test that fails when the fix is reverted, and remove each closed item from `BACKLOG.md` in the same commit that closes it.

**Architecture:** No new module except two small extractions (`src/core/compare.ts` for `sameInput`, `src/cli/flags.ts` for `parsePositiveIntFlag`). Five tasks, grouped by the files they touch so that one task's review sees one coherent diff: (1) errors, file I/O, ids and the generator's messages; (2) prices and `estimate`; (3) `report` and its Markdown, then the example report regenerated offline; (4) the review family (`review`, `import`, `dedupe`, the fake provider's cursor); (5) the example shim's token reporting with one paid eval, the README block, the Biome pattern, and the records.

**Tech Stack:** bun 1.4.2, TypeScript strict, zod 4.6.5, yaml 2.9.1, Biome 2.5.13; coverage gate `lines = 1.0, functions = 1.0` per file (`bunfig.toml`); promptfoo 0.123.0 only through `bunx` in the example (Task 5) and in CI.

**Spec:** `docs/superpowers/specs/2026-09-14-llm-test-forge-design.md` (unchanged; this plan tightens the implementation, it adds no verb). Every item below cites the backlog line it closes by its number in the list Alberto approved on 2026-09-16.

**Status:** Task 1 ✓ (5dda4f7, f40623c) · Task 2 ✓ (af2286a, 764974e) · Task 3 em andamento

## Global Constraints

- All artefacts in English: code, comments, commit messages, docs.
- TypeScript `strict: true` with `noUncheckedIndexedAccess`. Biome (tabs, double quotes). No `any` outside test mocks; no `as` casts to silence the compiler — narrow with zod or a type guard.
- Every model call goes through `src/llm/generate.ts`; nothing under `src/core/`, `src/emit/`, `src/report/` imports `ai`.
- Every failure names the file and, when there is one, the item id (`ForgeError`); a bad command line is a `UsageError` (exit 2). No verb exits 0 having produced zero items where items were expected.
- Tests never call the network and never run promptfoo. Coverage gate is 1.0 lines and functions per file; bun measures **no branch coverage**, so every guard arm gets its own named assertion; assert content (which id, which line), not counts.
- **Every item ends with a mutation:** after the suite is green, revert the item's guard (one edit), confirm with `grep` that the mutant landed, run `bun run typecheck`, run the named test, read it red, restore from a copy taken before the mutation — never `git checkout --` on a file with uncommitted work. Record the red line in the report.
- Each task removes the backlog lines it closed from `docs/superpowers/BACKLOG.md` in the same commit (rule: an item leaves the backlog in the commit that resolves it). A task that closes an item only partly rewrites the line to what remains.
- Commit after every task with a one-sentence imperative message, no emoji, ending with `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`. Stage files by name.

## How to read the items

Each item has: **#n** (its number in Alberto's list), the backlog sentence in short, the change, the test that pins it, and the mutation. The code fragments are hypotheses about shape; the existing files are the conventions. Where an item said "decide", the decision is written here and is final for this plan.

---

### Task 1: Errors, file I/O, ids, the generator's messages, two extractions

**Files:**
- Modify: `src/core/errors.ts`, `src/core/files.ts`, `src/core/ids.ts`, `src/core/cases.ts`, `src/core/import.ts`, `src/cli/commands/cases.ts`, `src/cli/commands/scenarios.ts`, `src/cli/commands/estimate.ts`
- Create: `src/core/compare.ts`, `src/cli/flags.ts`
- Test: `tests/core/errors.test.ts`, `tests/core/files.test.ts`, `tests/core/ids.test.ts`, `tests/core/cases.test.ts`, `tests/core/import.test.ts`, `tests/core/compare.test.ts`, `tests/cli/flags.test.ts`, `tests/cli/main.test.ts`

**Items:**

- [x] **#6 `ForgeError` reads detail fields with truthiness.** `src/core/errors.ts:12-14`: `if (details.file !== undefined)` (same for `id`, `rawPath`). Test in `tests/core/errors.test.ts`: `new ForgeError("m", { id: "" }).message` equals `m (id: )`; `new ForgeError("m", {}).message` equals `m`. Mutation: revert to truthiness → the first test goes red.

- [x] **#7 Array schema errors report the element index, not the item id.**
  → review added the test for an element without an id (f40623c). `src/core/files.ts` `readYamlFile`: when `schema.safeParse` fails and `data` is an array, for each issue whose `path[0]` is a number `i` and `data[i]` is an object with a string `id`, render the issue as `item "<id>" (index i).<rest of path>: <message>`; other issues unchanged. Implement as a small `describeIssue(issue, data)` helper in `files.ts`. Test in `tests/core/files.test.ts`: a `scenarios.yaml` with three entries where the third has `oracle: bogus` → error message contains `item "third-id" (index 2).oracle:` and the file path; a non-array file with a bad field keeps the old `field: message` form. Mutation: drop the array branch → first test red.

- [x] **#26 `readFailure` casts a caught unknown.** `src/core/files.ts`: replace `(e as { code?: string }).code` with a type guard `function hasCode(e: unknown): e is { code: string } { return typeof e === "object" && e !== null && "code" in e && typeof (e as { code: unknown }).code === "string"; }` — note the guard itself may use the one cast on a property read, which is the narrowing pattern the constraints allow; alternatively `Reflect.get`. Export `readFailure` (Task 1 #28 needs it). The existing ENOENT/EISDIR/EACCES/ENOTDIR tests already pin behaviour; add one: an `Error` with no `code` → `cannot read: <message>`. Mutation: make the guard always return false → the ENOENT test reads `cannot read:` instead of `file not found` → red.

- [x] **#28 `promptTokensFor` discards the fs cause and mangles an absolute `prompt_file`.** `src/cli/commands/estimate.ts` `promptTokensFor`: path is `resolve(forgeDir, "..", promptFile)` (absolute `promptFile` wins), and the catch throws `new ForgeError(\`prompt_file ${readFailure(e)}\`, { file: path })`. Tests in `tests/cli/main.test.ts` (estimate block): an absolute `prompt_file` pointing at a real temp file is counted (input tokens reflect it); a `prompt_file` that is a directory → exit 1 and the message contains `is a directory`. Mutation: `resolve` → `join` makes the absolute test red.

- [x] **#11 `slugify` drops accents.** `src/core/ids.ts`: `.normalize("NFD").replace(/\p{M}/gu, "")` before lowercasing. Test in `tests/core/ids.test.ts`: `slugify("Retorno do processo seletivo — confirmação")` equals `retorno-do-processo-seletivo-confirmacao`; `slugify("ação")` equals `acao`. Mutation: remove the normalize → red.

- [x] **#10 `(0 dropped: )` reads as a bug.** `src/core/cases.ts:150`: when `reasons.length === 0` the message is `no usable case generated for scenario: the model returned no candidates`; otherwise unchanged. Test in `tests/core/cases.test.ts`: a fake response with `{"cases": []}` → message equals the new sentence and contains no `dropped`; the existing all-dropped test keeps its `(2 dropped: …)`. Mutation: revert → red.

- [x] **#9 `sameInput` and `parsePositiveIntFlag` duplicated.** Create `src/core/compare.ts` exporting `sameInput(a: Record<string, string>, b: Record<string, string>): boolean` (move the body from `cases.ts`, delete the copy in `import.ts`, import in both) and `src/cli/flags.ts` exporting `parsePositiveIntFlag(flag, raw)` (move from `commands/cases.ts`, delete from `commands/scenarios.ts`, import in both). Tests: `tests/core/compare.test.ts` (equal, different value, different key set, key order irrelevant) and `tests/cli/flags.test.ts` (undefined → undefined; "3" → 3; "0", "-1", "x", "1.5" → `UsageError` naming the flag and the value). Existing tests keep passing. Mutation: make `sameInput` ignore key sets → the "different key set" test red.

- [x] **Step: run, mutate, backlog, commit.** `bun run check` green. Remove the closed lines from `BACKLOG.md` (#6, #7, #9, #10, #11, #26, #28 — seven lines). Commit: `Name the item in array schema errors, keep accents readable in ids, and stop duplicating sameInput and the int flag parser`.

---

### Task 2: Prices and `estimate`

**Files:**
- Modify: `prices.yaml`, `src/llm/prices.ts`, `src/core/estimate.ts`, `src/cli/commands/estimate.ts`, `src/cli/commands/emit.ts`, `src/core/report.ts` (only where `priceFor`'s new field is read)
- Test: `tests/llm/prices.test.ts`, `tests/core/estimate.test.ts`, `tests/cli/main.test.ts`, `tests/core/report.test.ts`

**Items:**

- [x] **#27 `priceFor` prices an unknown provider as the table's first key; ollama comes out as Opus.** (a) `prices.yaml`: add `ollama/llama3: { input: 0, output: 0 }` and `ollama/qwen3: { input: 0, output: 0 }` with a comment that local models cost nothing per token. (b) `ModelPrice` gains `priced: boolean`; `priceFor` returns `{ input: 0, output: 0, pricedAs: model, approximate: true, priced: false }` when the provider has no row at all, and drops the two dead `??` fallbacks (the same-provider longest-prefix search stays). (c) `renderEstimate` prints `not priced` in the dollar column for a line with `priced: false` and adds the note `"not priced: <model> has no row in prices.yaml"`; `estimateSuite` carries `priced` on `EstimateLine` and excludes unpriced lines from `totalDollars`. (d) `report`'s `costsOf` treats `priced: false` like an unreported cost: `realDollars: null` for that line. Tests: `tests/llm/prices.test.ts` "an unknown provider is not priced" (`priced: false`, `pricedAs` equals the model); the old "priced as the first model" test is replaced; `tests/core/estimate.test.ts`: an `ollama/x` target renders `not priced` and the total excludes it; `tests/core/report.test.ts`: a judge with an unknown provider gets `realDollars: null`. Mutation: make `priceFor` fall back to `names[0]` again → the prices test red.

- [x] **#23 `JUDGE_OUTPUT_TOKENS` misses by 64% when judges reject.**
  → measured: example run mean 174 (85, 68, 79, 92, 359, 359), spike fixture mean 321; constant set to 174. Measure first: from `examples/moonlighter-classify-email/results.json`, sum `completion + completionDetails.reasoning` over the six `llm-rubric` components and divide by six; do the same on `tests/fixtures/promptfoo-results-0.123.0.json` (two components). Set `JUDGE_OUTPUT_TOKENS` to the rounded mean of the example run (expected ≈ 180; write the measured figure) and rewrite its doc comment with both measurements and the reason (a rejecting judge writes a long explanation). Update the estimate tests' literals (`JUDGE_OUTPUT_TOKENS` is imported there, so most already follow) and the example README's estimate block is regenerated in Task 5. Test: none new beyond the literals; the mutation is the constant itself — not applicable; record the measurement in the report instead.

- [x] **#42 `renderEstimate` hard-codes column 80.** Build the data rows first, compute `dollarCol = Math.max(...rows.map((r) => r.indexOf("$")))`, then `total` is padded to that column. Test in `tests/core/estimate.test.ts`: a model name of 39 characters and `calls: 120000` → the `$` of the total line sits at the same index as the `$` of that data row (assert `indexOf("$")` equality), and the existing three-row literal test still passes. Mutation: revert to `" ".repeat(73)` → red.

- [x] **#41 `emit:`/`estimate:` do not pluralise cases and scenarios.**
  → `plural()` lives in `src/core/text.ts`, not `src/cli/flags.ts`: core never imports cli (ruling). A tiny `plural(n, word)` in `src/cli/flags.ts` (already created in Task 1): `1 case`, `2 cases`, `1 scenario`. Apply in `emit.ts` and `estimate.ts` lines. Tests in `tests/cli/main.test.ts`: one selected case → `estimate: 1 case, 0 with a rubric; …` and `emit: wrote … (1 case, 1 scenario, 1 target model, 1 judge)`. Mutation: revert one → red.

- [x] **#29 `guessOutputTokens` with `fields: []`.** Test only: `guessOutputTokens({ kind: "json", fields: [] })` equals 24. Mutation: change the check to `output.fields?.length` → red.

- [x] **#32 e2e judge estimate asserts only `> 0`.** Test only: compute the expected figure from the real `prices.yaml` row and the case in `forgeWithApprovedCases` (rubric text length known) and assert `costs.find(role === "judge").estimatedDollars` equals it. Mutation: change `JUDGE_PROMPT_OVERHEAD` by one → red.

- [x] **Step: run, mutate, backlog, commit.** `bun run check` green. Remove #27, #29, #32, #41, #42 from `BACKLOG.md`; rewrite #23 to what remains (a real tokenizer stays deferred; the constant is calibrated). Commit: `Price unknown providers as not priced, calibrate the judge output constant from the example run, and align the estimate total`.

---
  → review caught a Critical in the `report.ts` half of #27: a promptfoo-reported cost was discarded for an unpriced target, and `~` marked unpriced models; fixed in 764974e.

---

### Task 3: `report` and its Markdown, then the example report regenerated

**Files:**
- Modify: `src/core/report.ts`, `src/report/markdown.ts`, `src/cli/commands/report.ts`, `examples/moonlighter-classify-email/.forge/report.md`, `examples/moonlighter-classify-email/.forge/report.json`, `examples/moonlighter-classify-email/README.md` (numbers and section names only)
- Test: `tests/core/report.test.ts`, `tests/report/markdown.test.ts`, `tests/cli/main.test.ts`

**Items:**

- [ ] **#45 `ReportSchema.failing` required breaks old baselines.** `failing: z.array(z.string()).default([])`. Test: `ReportSchema.parse` of a report object without `failing` yields `failing: []`; an e2e `--baseline` with a `report.json` from which `failing` was deleted exits 0. Mutation: drop `.default([])` → red.

- [ ] **#30 Flaky/failing tables filter by display string.** `caseSection` takes `pick: (c: CaseRun) => boolean` and filters `report.cases` by stability; `failing`/`flaky` arrays stay as they are (they are the JSON summary). Test in `tests/report/markdown.test.ts`: a case whose id contains ` @ ` appears exactly once in the right table. Mutation: revert to `keys.includes` → red.

- [ ] **#35 Coverage sentence says "approved" while counting edited.** Rename the field `coverage.approvedScenarios` → `coverage.reviewedScenarios` (schema, builder, renderer, tests, the real-file guard) and the sentence to `Coverage: N of M reviewed scenarios ran`. Test: literal line. Mutation: revert the word in the renderer → red.

- [ ] **#33 Unmatched rows labelled by `testIdx`, which repeats.** Label becomes `row <position in results.results> (testIdx <n>): <model>: <case…>` where position is the 0-based index in the array. Update the two wording tests and the real-file guard if it asserts unmatched. Test: two unmatched rows with the same `testIdx` produce two distinct labels. Mutation: revert to `testIdx` only → red.

- [ ] **#36 Zero-matched error names no observed id.** Message: `no result matches a case in .forge/cases (first metadata.case seen: "<id>", <n> rows); was this results.json produced from a config forge emitted?` — with `none` when no row has a string `metadata.case`. Test: both wordings. Mutation: drop the observed id → red.

- [ ] **#40 An unlabelled provider becomes `file://forge_target.py`.** `modelOf(row)`: `row.provider.label ?? fromPromptfooProvider(row.provider.id) ?? row.provider.id`. Test: a row with `provider: { id: "anthropic:messages:claude-haiku-4-5" }` and no label → model `anthropic/claude-haiku-4-5`; the existing `file://` test keeps its raw id. Mutation: drop the middle term → red.

- [ ] **#39 A case that passes once and errors once reads as flaky.** `StabilitySchema` gains `"partly-errored"`; `stabilityOf`: `errored === runs` → errored; `errored > 0 && failed === 0 && passed > 0` → partly-errored; `passed === runs` → stable; `passed > 0` → flaky; else failing. `report.failing` (the JSON list) includes `failing`, `errored` and `partly-errored`; the Markdown section is titled `## Failing or errored cases` and its empty sentence `No failing or errored case.`; the headline label becomes `**failing or errored:** N`. Update every test literal that carried the old title/label, the example README's quoted headline (Step below) and the CLI summary line (`N failing or errored`). Tests: a pass+error pair → `partly-errored`, listed in that section, absent from the flaky one. Mutation: revert `stabilityOf` → red.

- [ ] **#34 One `Disagreement` per row inflates the headline.** `DisagreementSchema` gains `occurrences: z.number()`; `disagreementsOf` groups by `(model, case)`, keeps the verdicts of the first disagreeing run and counts occurrences; the Markdown table gains a `runs` column with the count. Tests: two disagreeing repeats of one case → one entry with `occurrences: 2`; the headline counts 1; the real-file guard (`disagreements: []`) unchanged. Mutation: remove the grouping → red.

- [ ] **#31 Ragged judge padding untested.** Test only: one disagreement with three judges beside one with two → the two-judge row ends with an empty cell (`| … | judge: pass — ok | judge: fail — no |  |`). Mutation: remove the `while (cells.length < columns)` loop → red.

- [ ] **Step: regenerate the example report offline.** From `examples/moonlighter-classify-email/`: `bun ../../src/cli/bin.ts report results.json` (no key, no eval). Read the new `report.md`; update the README's quoted headline and section names (`Failing or errored cases`, `reviewed scenarios`) and nothing else. Both files are committed.

- [ ] **Step: run, mutate, backlog, commit.** `bun run check` green; the real-file guard in `tests/core/report.test.ts` updated where field names changed. Remove #30, #31, #33, #34, #35, #36, #39, #40, #45 from `BACKLOG.md`. Commit: `Report partly-errored cases apart from flaky ones, count judge disagreements per case, and label unmatched rows uniquely`.

---

### Task 4: The review family — `review`, `import`, `dedupe`, and the fake provider's cursor

**Files:**
- Modify: `src/cli/commands/review.ts`, `src/cli/review-loop.ts`, `src/core/review.ts`, `src/cli/commands/import.ts`, `src/cli/commands/dedupe.ts`, `src/llm/generate.ts`
- Test: `tests/cli/main.test.ts`, `tests/cli/review-loop.test.ts`, `tests/core/review.test.ts`, `tests/llm/generate.test.ts`

**Items:**

- [ ] **#15 `review --all` parses `--only` then ignores it.** Decision: refuse the combination. In `reviewCommand`, after `parseOnlyFlag`, `if (values.all && only !== undefined) throw new UsageError("--all cannot be combined with --only; --all approves one scenario's cases")`. Test: exit 2, message names both flags. Mutation: remove the guard → red.

- [ ] **#14 `import --oracle` ignored when the `imported` scenario exists.** Decision: refuse the flag. In `importCommand`, read scenarios before importing; if `imported` exists and `oracle !== undefined` and differs from the existing oracle → `ForgeError(\`scenario "imported" already exists with oracle ${existing.oracle}; --oracle cannot change it — edit .forge/scenarios.yaml\`, { file: paths.scenarios, id: "imported" })`; the same oracle is accepted silently. Tests: second import with a different `--oracle` → exit 1 with that message and no cases written; the same oracle → exit 0. Mutation: remove the guard → red.

- [ ] **#16 `oracleOf` falls back to rubric for a case whose scenario is missing.** `oracleOf` throws `ForgeError(\`case ${caseId} names scenario "${scenarioId}", which is not in scenarios.yaml\`, { file: paths.scenarios, id: caseId })` — change its signature to `(c: Case) => Oracle` in `ReviewLoopArgs` and the command. Test: a cases file whose `scenario` names nothing, interactive review answering `approve` → exit 1 with that message. Mutation: revert the fallback → red.

- [ ] **#21 The review pass works from a snapshot.** `runReviewLoop` gains `onDecided?: (kind, item) => void`, called after every decision; `reviewCommand` keeps `let feature` and `let scenarios` and updates them in that callback, and `checkExpected`/`askExpected`/`oracleOf` read the variables, not captured copies. Test in `tests/cli/main.test.ts`: in one pass, edit the feature to add a label (scripted editor), then approve a case using that label → approved, exit 0 (today it is refused). Mutation: capture `feature` by value again → red.

- [ ] **#20 Editing a scenario's oracle leaves its approved cases mismatched.** Decision: re-open. After a scenario decision whose `oracle` changed, every `approved`/`edited` case of that scenario whose `expected` no longer matches the new oracle is set back to `pending` (its `expected` kept) and the command prints `review: scenario <id> changed oracle <old> → <new>; <n> case(s) re-opened`. Test: scenario edited from `label` to `rubric` with two approved label cases → both pending afterwards, line printed with `2`. Mutation: skip the re-open → red.

- [ ] **#19 A rename leaves `duplicate_of` dangling; a twin id is never offered.** (a) After writing decisions, any case in the same file whose `duplicate_of` equals a renamed case's `originalId` is rewritten to the new id (in the same write). (b) `reviewCommand` counts ids that appear more than once in a cases file and prints `review: <file> holds <n> entries under id "<id>"; only the first is offered — fix the file` before the loop. Tests: rename with a pointer → pointer updated; a file with a twin id → the line printed naming id and file. Mutation: remove (a) → red; remove (b) → red.

- [ ] **#22 Bare `dedupe` validates inside its loop.** In `dedupeCommand`, before the first write: read every cases file, and if any is empty throw the existing `ForgeError` naming that scenario — no file is written when one is bad. Test: two scenarios, the second with an empty cases file → exit 1 and the first file's mtime unchanged (use `LONG_AGO` + `utimes` as the emit tests do). Mutation: move the check back into the loop → red.

- [ ] **#8 `fakeCursors` is process-lifetime module state.** Move the `Map` into `createLlm`'s closure so each `Llm` instance starts its sequence at 0. Tests in `tests/llm/generate.test.ts`: two `createLlm` instances on the same fixture both get entry 0 first; one instance still advances across calls. Check `tests/cli/main.test.ts` for any test that relied on continuation across `ctxIn` calls and fix its fixture if so (say which in the report). Mutation: move the map back to module scope → the two-instance test red.

- [ ] **Step: run, mutate, backlog, commit.** `bun run check` green. Remove #8, #14, #15, #16, #19, #20, #21, #22 from `BACKLOG.md`. Commit: `Refuse review --all with --only and a second import --oracle, re-open cases when a scenario's oracle changes, and keep the fake provider's cursor per instance`.

---

### Task 5: The example shim reports tokens, the README block, the Biome pattern, and the records

**Files:**
- Modify: `examples/moonlighter-classify-email/.forge/forge_target.py`, `examples/moonlighter-classify-email/results.json`, `examples/moonlighter-classify-email/.forge/report.md`, `examples/moonlighter-classify-email/.forge/report.json`, `examples/moonlighter-classify-email/README.md`, `biome.json`, `docs/superpowers/BACKLOG.md`
- Test: none new (the example is not under `bun test`); `bun run check` must stay green.

**Items:**

- [ ] **#24 The example's shim reports zero target tokens.** moonlighter's `make_api_caller` calls `record_call(seconds, input_tokens=…, output_tokens=…)` imported into `moonlighter.core.llm` from `moonlighter.core.metrics`. In `run_application`, before calling `classify_response`, install a capturing wrapper: `import moonlighter.core.llm as llm_module`; save `llm_module.record_call`; set `llm_module.record_call = lambda seconds, input_tokens=0, output_tokens=0: captured.update(...)` (call the original too); restore in `finally`; return the captured counts as `input_tokens`/`output_tokens`. Keep it inside `run_application` so the shim template stays generic. Verify with a throwaway `python3` run that one call through the venv reports non-zero counts (Anthropic key sourced with `set -a; source ~/.config/anthropic/vim-ai-autocomplete.env; set +a`; never printed).

- [ ] **Step: one eval, one report.** From the example directory with the key sourced, `MOONLIGHTER_HOME=$(mktemp -d) PROMPTFOO_DISABLE_TELEMETRY=1 PROMPTFOO_DISABLE_UPDATE=1`, `target.python` temporarily at `/Users/albertosca/Programming/moonlighter/.venv/bin/python`: `bunx promptfoo@0.123.0 eval -c .forge/promptfooconfig.yaml -o results.json --no-cache --no-progress-bar -j 2`, then `bun ../../src/cli/bin.ts report results.json`. Set `target.python` back to `.venv/bin/python`. Read `report.md`: the target cost line must now carry real tokens and a dollar figure with an error percent against the estimate; `Target usage: reported by the shim.` Cases may pass or fail differently from the previous run (the models are nondeterministic); whatever happens stays and the README says so. Cost: cents.

- [ ] **#37 The README command block exports one model while `describe`/`scenarios` ran on others.** Put a trailing comment on those two lines (`# ran with FORGE_MODEL=google/gemini-3.6-flash on the first day`, `# ran with anthropic/claude-haiku-4-5`) and refresh every number the new run changed (estimate block, eval line, report headline, cost table, the three failure write-ups if they changed). The README's account of the first run's shim defect stays.

- [ ] **#38 `biome.json` negation is one level deep.** `"!examples/**/results.json"`. Verify: `bun run lint` still passes and `bunx biome check examples/moonlighter-classify-email/results.json` reports the file ignored.

- [ ] **Step: records.** Remove #24, #37, #38 and #43 (`usage.jsonl`, a note with no work) from `BACKLOG.md`. Leave every group-B item. The `PROJECT-LOG.md` entry is written by the controller after the whole-branch review.

- [ ] **Step: run, commit.** `bun run check` green; `git grep -n "sk-ant\|AIza"` empty. Commit: `Make the example's shim report moonlighter's real token usage and re-run its eval`.

---

## Self-review against the list (done while writing)

- **Coverage:** group A had 33 items (6–11, 14–16, 19–24 minus 23b, 26–42 minus 37/38 placement, 45). Task 1: 6, 7, 9, 10, 11, 26, 28 (7). Task 2: 23, 27, 29, 32, 41, 42 (6). Task 3: 30, 31, 33, 34, 35, 36, 39, 40, 45 (9). Task 4: 8, 14, 15, 16, 19, 20, 21, 22 (8). Task 5: 24, 37, 38, 43 (4 — 43 is a closure, no work). Total 34 lines closed; every group-A number appears exactly once.
- **Placeholders:** none; each item names the change, the test and the mutation. Task 5's eval step names the exact command.
- **Type consistency:** `readFailure` exported in Task 1 and consumed by Task 1's own `promptTokensFor`; `plural` created in Task 1's `src/cli/flags.ts` and used in Task 2; `ModelPrice.priced` added in Task 2 and read by Task 2's edit of `report.ts`; `StabilitySchema` and `coverage.reviewedScenarios` changed in Task 3 with the real-file guard and the example regenerated in the same task; Task 5's regeneration happens after Task 3's renderer changes, so the committed example reflects both.
- **Order of risk:** Task 3 changes `report.json`'s schema (`reviewedScenarios`, `partly-errored`, `occurrences`) — `--baseline` against a report from before this plan will fail on the renamed coverage field; acceptable pre-release, and #45's `.default([])` is the only backward-compatibility promise made.
