# llm-test-forge — design

**Date:** 2026-09-14. **Status:** approved in brainstorming, awaiting implementation plan.
**Seeded by:** `~/Programming/bancada-de-prototipos/investigations/2026-09-13-llm-test-forge/` (verdict: viable — build the forge, adopt promptfoo as runner and evaluator).

## Problem

A developer who uses an LLM inside an application has no equivalent of a regression suite for that usage. Changing the prompt, the model, or a config value can silently change behaviour, and the only way to know is to try inputs by hand. The market (17 tools surveyed, 3 executed) covers the *running and judging* half well — promptfoo runs a matrix of prompts × models, judges with a swappable LLM, reports cost per result, and exposes an MCP server — but nothing covers the *forge* half: going from "what my feature does" to an explicit list of scenarios of mixed kinds, to many cases per scenario **with an expected output**, through a human review loop in files, with a cost estimate before the run and judge-disagreement and stability statistics after it.

llm-test-forge is that missing half. It produces reviewed, runnable suites and reads the results back. It does not run models against the target, does not judge, and does not track cost during a run — promptfoo does those, through a config the forge emits.

## Decisions taken (with the alternative that lost)

| Decision | Chosen | Rejected |
|---|---|---|
| Output format | Layered: a cases file (the portable "generic") plus a suite file that adds target, judges and repeats; promptfoo config generated from both | Cases-only export (no execution semantics); suite-only "spec of everything" |
| Language | TypeScript on bun | Python with uv (Alberto's Python quality standard exists, but promptfoo's schema and the MCP reference SDK are TS); Elixir |
| Surface | Library first, CLI as the first shell, MCP as a later thin shell over the same verbs | MCP first; both at once |
| Model access | Vercel AI SDK (`ai` + provider packages), `generateObject` with zod schemas | Official SDK per provider behind a home-made interface; reusing promptfoo's providers as a library |
| Input to `describe` | Free text plus an optional real prompt file | Prompt file only (PromptPex style); pointer to code |
| Review | Files are the truth, an interactive CLI walks pending items and writes back | File-only editing; approve whole scenarios in bulk |
| MVP scope | The whole pipeline, thin: describe → scenarios → cases → dedupe → import → review → estimate → emit → report | Stop at emit; cases-only |
| State between verbs | A pipeline of YAML files under `.forge/`, stateless commands | SQLite per project; promptfoo project as the state with a sidecar |
| Repository name | `llm-test-forge`, CLI binary `forge` | `forja-de-testes-de-llm` |

## The files (the generic format)

Everything lives in `.forge/` inside the repository of the application under test, and is meant to be committed there. Four kinds of file, produced in this order.

### `feature.yaml` — written by `describe`, reviewed by the person

```yaml
id: classify-email
purpose: Classify a hiring-process email received by a job candidate
inputs:                      # what one call receives
  - name: email
    kind: text
    notes: from, subject and body of one email, capped at 3000 chars
output:                      # what one call returns
  kind: json                 # text | json | label
  fields: [type, stage, new_stage, company, job_title, summary]
  label_field: type          # when one field is a closed set
  labels: [rejection, acknowledgement, interview, screening, offer, info_request, unrelated]
invariants:                  # rules the prompt states or the person adds
  - Automated confirmations are acknowledgement, never screening
  - Answer is JSON only, no prose
prompt_file: packages/email/moonlighter/tracking/classification.py   # optional
status: pending              # pending | approved | edited
```

Nothing downstream runs while `feature.yaml` is `pending`.

### `scenarios.yaml` — written by `scenarios`

```yaml
- id: ack-with-optional-quiz
  kind: ambiguous            # happy | edge | ambiguous | out_of_scope | adversarial | language
  oracle: label              # label | fields | rubric
  description: automated acknowledgement that also offers an optional assessment
  status: pending            # pending | approved | rejected | edited
```

The generator must cover every `kind` at least once unless `--kinds` restricts it. `--more` appends new pending scenarios without touching reviewed ones. A scenario with id `imported` is created by `import` and holds real inputs; its `oracle` follows `feature.output.kind` (`label` → `label`, `json` → `fields`, `text` → `rubric`) unless `--oracle` overrides it.

### `cases/<scenario-id>.yaml` — written by `cases` and `import`

```yaml
- id: ack-with-optional-quiz-03
  scenario: ack-with-optional-quiz
  input:
    email: "From: noreply@greenhouse.io\nSubject: ...\n\n..."
  expected:                  # shape follows the scenario's oracle
    label: acknowledgement                       # oracle: label — one of feature.output.labels
    # fields: {company: Biotech Solutions}       # oracle: fields — subset match on parsed JSON
    # rubric: "summary mentions the optional quiz"  # oracle: rubric — one testable sentence
  status: pending            # pending | approved | rejected | edited
  generated_by: google/gemini-3.5-flash          # or "import:<file>"
  duplicate_of: ack-with-optional-quiz-01        # optional, set by dedupe
```

Ids are stable and human-readable (`<scenario>-<nn>`), never UUIDs, so diffs read well. `cases` never overwrites a case that is `approved` or `edited`. Imported cases arrive without `expected`; `review` asks for it. A conversational application is tested per turn: `history` and `message` are two input fields — there is no multi-turn format in the MVP (see Backlog).

### `suite.yaml` — what changes between runs; what `estimate` reads

```yaml
target:
  kind: promptfoo-python     # how the app is called: a shim around the real function
  entry: forge_target.py
  models: [anthropic/claude-haiku-4-5, google/gemini-3.8-flash]
judges: [anthropic/claude-sonnet-5, google/gemini-3.5-flash]
repeat: 2
include: [ack-with-optional-quiz, polite-rejection]   # scenario ids; default: all approved
```

### Other files under `.forge/`

`usage.jsonl` (one line per model call the forge makes: verb, model, tokens, dollars), `failures/<verb>-<timestamp>.txt` (raw model output that failed schema validation), `promptfooconfig.yaml` and `forge_target.py` (emitted), `report.md` and `report.json` (from `report`).

## The verbs

Each verb is one pure function in `src/core/`: it takes the parsed previous file(s) plus options and returns the object of the next file. The CLI reads YAML, calls the function, writes YAML. Model calls go through one function in `src/llm/`: `generate(schema, prompt, model)` on top of the Vercel AI SDK's `generateObject`, returning the validated object and the token usage. A model is a string `provider/model` resolved by a small registry (Anthropic, Google, OpenAI, Ollama in the MVP). The prompts the forge sends live in `templates/` as versioned text files, not as strings in code.

| Verb | Reads | Writes | Model call |
|---|---|---|---|
| `describe <text> [--prompt-file f]` | free text, optional prompt | `feature.yaml` (pending) | yes: normalise to the feature schema |
| `scenarios [--kinds k,k] [--more n] [--model m]` | feature (approved) | `scenarios.yaml` | yes: enumerate, covering every kind |
| `cases [--n 5] [--scenario id] [--model m]` | feature, approved scenarios | `cases/<id>.yaml` | yes: one batch per scenario, expected in the oracle's shape |
| `dedupe [--scenario id] [--model m]` | cases | `duplicate_of` on likely repeats | yes: one call per scenario |
| `import <file.jsonl> [--oracle o]` | JSONL: one object per line whose keys match `feature.inputs` | `cases/imported.yaml` (pending, no expected) + the `imported` scenario if absent | no |
| `review [--scenario id] [--only cases] [--all]` | everything pending | statuses, edits, expected of imported cases | no |
| `estimate` | suite, approved cases, `prices.yaml` | stdout table | no (tokenizer where available, chars/4 otherwise) |
| `emit [--format promptfoo\|jsonl]` | suite, approved cases | `promptfooconfig.yaml` + `forge_target.py`, or `cases.jsonl` | no |
| `report <results.json> [--baseline report.json]` | promptfoo output, cases, scenarios, estimate | `report.md`, `report.json` | no |

### `review`

Walks pending items in order: feature, scenarios, cases grouped by scenario, with `duplicate_of` pairs shown together. Per item: approve, reject, edit, skip. Edit opens the item as YAML in `$EDITOR` and writes it back as `edited`. An imported case without `expected` asks for it in the oracle's shape. `--all` (requires `--scenario`) approves everything pending in that scenario, for people who already read the file in their editor. Ends with a count of approved, rejected and still pending.

### `estimate`

Before any call: input tokens per case (provider tokenizer when available, characters ÷ 4 otherwise) × cases × target models × repeat, plus judge cost on rubric cases only (one call per judge listed in the suite), priced by `prices.yaml` (versioned in this repo, editable by the user, with a dated header). Output in tokens and dollars per model, with the price-table date and an explicit note that output size is a guess derived from `output.kind`. A model missing from `prices.yaml` whose provider has rows is priced as the closest listed model of that provider and the line is marked; a model whose provider has no row at all is not priced: its line says so, the total excludes it and says how many lines it left out.

### `emit`

| In the forge | In promptfoo |
|---|---|
| approved or edited case | one `tests` item: `vars` = inputs, `metadata: {scenario, case}` |
| oracle `label` | `javascript` assert: tolerant-parse the output, compare the label field |
| oracle `fields` | `javascript` assert: tolerant-parse, subset match |
| oracle `rubric` | one `llm-rubric` assert per judge listed in the suite, same text, `provider:` set per assert |
| `target.kind: promptfoo-python` | provider `file://forge_target.py`, a skeleton the person fills by calling the real function |
| `target.models` | one promptfoo provider entry per model, each wrapping the same shim with `FORGE_MODEL=<model>` in its config; the shim reads `FORGE_MODEL` so the app uses the run's model |
| `repeat` | `evaluateOptions.repeat` |

The shim is generated once and never overwritten if present. The tolerant parser (strip code fences, take the first JSON object) runs **inside the assert**, so the suite measures the application and not its output formatting — the spike's eight false failures came from testing the raw prompt. `emit` refuses when any included scenario or case is pending, listing them. `--format jsonl` writes approved cases as flat JSONL (`input`, `expected`, `scenario`, `kind`) for any other consumer; both emitters read the same in-memory object.

### `report`

Matches each promptfoo result to its case and scenario through `metadata` and writes `report.md` (decisive numbers first: pass rate, flaky cases, judge disagreements; tables after) and `report.json`. It computes what no runner reports:

- Per scenario: run, passed, failed, errored; coverage (approved scenarios with at least one run case) and scenarios with no case.
- Stability per case across `repeat`: a case passing some but not all repeats is `flaky` and listed apart.
- Judge disagreement on rubric cases: both reasons side by side; input for the next review round.
- Real cost vs estimated, per model, with the error in percent.
- Target models side by side, per scenario, when the suite has more than one.
- With `--baseline`: what changed; cases that passed before and fail now come first.

The person runs `promptfoo eval` themselves; the forge only reads `results.json`.

## Errors

Every failure names the file and the item id. Model output that fails schema validation is saved raw under `.forge/failures/` and the error points to it. No verb exits 0 having produced zero items where items were expected (the spike caught promptfoo doing exactly that). `emit` refuses on pending items. Provider errors surface with the provider's own message, not a retry loop that hides a `credit_balance_exhausted` behind rate-limit waits.

## Repository

```
src/core/       one pure function per verb + zod schemas of the four files
src/llm/        provider registry (Vercel AI SDK), generate(), usage log, prices loader
src/emit/       promptfoo emitter, jsonl emitter, target shim template
src/cli/        thin command layer: parse args, read YAML, call core, write YAML
templates/      prompts the forge sends to the generator, as versioned text
prices.yaml     per-model input/output prices, dated header
examples/moonlighter-classify-email/   first real target; its .forge/ committed as documentation
docs/superpowers/   specs, plans, BACKLOG.md, PROJECT-LOG.md
```

Quality: TypeScript `strict`; Biome for lint and format; `bun test` with coverage and a gate; GitHub CI with three jobs (test, lint, coverage) and the three badges; bilingual README with flag links; English repository description; MIT license. English for all artefacts (code, comments, commits, docs).

## Testing

- Each `core` verb is tested with recorded model outputs as fixtures (real Gemini and Sonnet responses from the spike) plus broken variants: code-fenced JSON, truncated JSON, a label outside the enum, an empty list.
- `src/llm/` is tested with a fake provider that returns what the test dictates and counts tokens; `usage.jsonl` and `failures/` behaviour are asserted there.
- One integration test runs `emit` on the example and validates the result with `promptfoo validate` (promptfoo as a devDependency).
- Live calls exist only in a `live` suite behind an environment variable, never in CI.
- Before trusting the green: break one test on purpose and confirm the runner reports it (the canary rule).

## First real target

moonlighter's `classify_response` (`~/Programming/moonlighter/packages/email/moonlighter/tracking/classification.py`): one email in, one of seven labels out, small input, an explicit rule about acknowledgements that a prompt tweak can break, and no validation of the label against the enum in the code — a class of regression the suite catches and the unit tests do not. The `examples/` directory carries its `.forge/` as documentation of what a finished forge project looks like.

## Environment facts that shape the MVP

Measured 2026-09-13: Alberto's OpenAI account has zero credits; the Google key is free tier (Flash models only, Pro answers `limit: 0`, and Flash returned `503 high demand` in 2 of 16 calls); the Anthropic key lives in `~/.config/anthropic/vim-ai-autocomplete.env`, not in the shell. The example and the live suite therefore use Anthropic and Google Flash; the provider registry still ships OpenAI and Ollama entries.

## Out of scope (recorded in `docs/superpowers/BACKLOG.md`)

MCP server (second shell over the same verbs); `run` verb that invokes promptfoo from inside the forge; multi-turn with a simulated user; multi-run history; more providers (OpenRouter, Mistral); charts.
