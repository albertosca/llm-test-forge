# llm-test-forge core pipeline — Implementation Plan (plan 1 of 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `forge` CLI that takes a free-text description of an LLM feature and produces a reviewed set of test cases under `.forge/` — `describe`, `scenarios`, `cases`, `dedupe`, `import`, `review` — with schema-validated YAML files, a model layer that never fails silently, and a test suite that runs without network.

**Architecture:** One pure function per verb in `src/core/` operating on parsed YAML objects; a single `src/llm/generate.ts` entry for every model call (Vercel AI SDK `generateObject` with zod schemas, usage logged to `.forge/usage.jsonl`, raw text saved to `.forge/failures/` on validation failure); a thin `src/cli/` that reads YAML, calls core, writes YAML. Plan 2 adds `estimate`, `emit`, `report` and the moonlighter example on top of the same files.

**Tech Stack:** bun 1.4.2, TypeScript strict, `ai` 7.0.100 with `@ai-sdk/anthropic` 4.0.53, `@ai-sdk/google` 4.0.69, `@ai-sdk/openai` 4.0.66, `zod` 4.6.5, `yaml` 2.9.1, Biome 2.5.13, `bun:test` with coverage threshold in `bunfig.toml`, `node:util` `parseArgs` for the CLI (no CLI framework).

**Spec:** `docs/superpowers/specs/2026-09-14-llm-test-forge-design.md`

**Status:** Task 1 ✓ · Task 2 ✓ · Task 3 ✓ · Task 4 ✓ · Task 5 ✓ · Task 6 ✓ · Task 7 ✓ · Task 8 ✓ · Task 9 ✓ · Task 10 ✓ · Task 11 ✓ — executed 2026-09-15 by subagent-driven development, merged as cce512d; the whole-branch review and its six fixes are recorded in PROJECT-LOG.md (checkboxes marked after the fact, on 2026-09-15)

## Global Constraints

- All artefacts in English: code, comments, commit messages, docs. README is bilingual (`README.md` English, `README.pt.md` Portuguese) with flag links at the top of each.
- TypeScript `strict: true`. Biome for lint and format (tabs, double quotes — the `biome init` defaults). No `any` except in test mocks.
- Every model call goes through `src/llm/generate.ts`. No verb imports `ai` directly.
- No verb exits 0 having produced zero items where items were expected. Every failure names the file and the item id. Raw model output that fails validation is saved under `.forge/failures/<verb>-<timestamp>.txt` and the error message contains that path.
- `cases` never overwrites a case whose status is `approved` or `edited`. `scenarios --more` never touches reviewed scenarios.
- Ids are human-readable: scenario ids are kebab-case from the generator; case ids are `<scenario-id>-<nn>` with two-digit zero padding.
- Tests never call the network. Model calls in tests use `MockLanguageModelV4` from `ai/test` (verified present in ai 7.0.100) or the `fake/<file>` provider defined in Task 3. Live calls live only in `tests/live/` behind `FORGE_LIVE=1`.
- Coverage gate: `bunfig.toml` `[test] coverage = true, coverageThreshold = { lines = 0.95, functions = 0.95 }` (verified 2026-09-14: bun 1.4.2 exits 1 when below the threshold). Raise to 1.0 in Plan 2 once the surface stabilises.
- Commit after every task with a message in the imperative, no emoji, ending with the `Co-Authored-By` line the session provides.

## Verified facts the plan relies on (2026-09-14, scratch probe)

- `generateObject({ model, schema, prompt })` from `ai` returns `{ object, usage }`; `usage.inputTokens` and `usage.outputTokens` are numbers (may be `undefined`).
- On unparsable or schema-invalid output it throws `NoObjectGeneratedError` (exported from `ai`) with `.text` (the raw string) and `.usage`. Code-fenced JSON is **not** parsed by `generateObject` — it throws — so the forge's own tolerant parsing is not needed on this path; fences appear only in the promptfoo target path (Plan 2).
- `MockLanguageModelV4` from `ai/test` takes `{ doGenerate: async () => ({ content: [{ type: "text", text }], finishReason: "stop", usage: { inputTokens: { total, noCache, cacheRead, cacheWrite }, outputTokens: { total, text, reasoning } }, warnings: [] }) }`.
- Provider factories: `createAnthropic({ apiKey })` from `@ai-sdk/anthropic`, `createGoogleGenerativeAI({ apiKey })` from `@ai-sdk/google`, `createOpenAI({ apiKey, baseURL })` from `@ai-sdk/openai`. Each returns a function `(modelId: string) => LanguageModel`.
- `parseArgs` from `node:util` works under bun with `allowPositionals: true`.
- **Not verified:** Ollama through `createOpenAI({ baseURL: "http://localhost:11434/v1", apiKey: "ollama" })` — documented by Ollama as OpenAI-compatible, not exercised. Marked in the registry and tested only in `tests/live/`.

## File Structure

```
package.json                 name, bin { forge: "src/cli/main.ts" }, scripts (test, lint, format, check)
tsconfig.json                strict, bundler resolution, types ["bun-types"]
biome.json                   biome init defaults + ignore .forge/ and coverage/
bunfig.toml                  [test] coverage + threshold
.gitignore                   node_modules, coverage, .forge/failures, .forge/usage.jsonl (of the example, Plan 2)
src/core/schemas.ts          zod schemas + TS types for Feature, Scenario, Case, Suite, Expected; the enums Kind, Oracle, Status
src/core/files.ts            forgePaths(dir), readYamlFile/writeYamlFile with schema validation, readFeature/writeFeature, readScenarios/writeScenarios, readCases/writeCases/listCaseScenarios
src/core/errors.ts           ForgeError { message, file?, id?, rawPath? }
src/core/ids.ts              slugify(text), nextCaseId(scenarioId, existing)
src/core/describe.ts         describe(): text (+ prompt) -> Feature (pending)
src/core/scenarios.ts        enumerateScenarios(): Feature (+ existing) -> Scenario[] with kind coverage check and merge
src/core/cases.ts            generateCases(): Feature + Scenario (+ existing) -> Case[] with oracle-shaped expected and merge
src/core/dedupe.ts           dedupeCases(): Case[] -> Case[] with duplicate_of
src/core/import.ts           importCases(): Feature + JSONL text -> Case[] pending without expected, plus the `imported` scenario
src/core/review.ts           applyDecision(item, decision, edited?) pure transitions; pendingItems(feature, scenarios, cases) ordering
src/llm/models.ts            resolveModel("provider/model") -> LanguageModel, reads keys from env; supports fake/<file>
src/llm/generate.ts          createLlm({ forgeDir, resolve? }) -> { generate({ schema, prompt, model, verb }) }; usage log; failures
src/llm/templates.ts         loadTemplate(name), render(template, vars)
templates/describe.md, scenarios.md, cases.md, dedupe.md   prompts, versioned
src/cli/main.ts              parseArgs dispatch: describe | scenarios | cases | dedupe | import | review
src/cli/commands/*.ts        one file per verb: read files -> call core -> write files -> print summary
src/cli/review-loop.ts       interactive loop with injectable ask() and openEditor()
tests/                       mirrors src/; tests/fixtures/ holds recorded model outputs; tests/live/ behind FORGE_LIVE
.github/workflows/ci.yml     jobs: test (with coverage), lint
README.md, README.pt.md      bilingual, flag links, badges (CI, lint, coverage)
```

---

### Task 1: Scaffold, toolchain and the coverage canary

**Files:**
- Create: `package.json`, `tsconfig.json`, `biome.json`, `bunfig.toml`, `.gitignore`, `src/core/errors.ts`, `tests/core/errors.test.ts`

**Interfaces:**
- Produces: `ForgeError` class in `src/core/errors.ts`: `new ForgeError(message: string, details?: { file?: string; id?: string; rawPath?: string })`; `.details` is public; `.message` is the message with the details appended as ` (file: …, id: …, raw: …)` when present.

- [x] **Step 1: Initialise the package and install dependencies**

Run in `~/Programming/llm-test-forge`:

```bash
bun init -y
bun add ai@7.0.100 @ai-sdk/anthropic@4.0.53 @ai-sdk/google@4.0.69 @ai-sdk/openai@4.0.66 zod@4.6.5 yaml@2.9.1
bun add -d @biomejs/biome@2.5.13 @types/bun
```

Then replace `package.json` with:

```json
{
  "name": "llm-test-forge",
  "version": "0.1.0",
  "description": "Forge reviewed regression suites for your LLM features and emit them for promptfoo",
  "license": "MIT",
  "type": "module",
  "bin": { "forge": "src/cli/main.ts" },
  "scripts": {
    "test": "bun test",
    "lint": "biome check .",
    "format": "biome format --write .",
    "check": "bun run lint && bun test"
  },
  "dependencies": {
    "@ai-sdk/anthropic": "4.0.53",
    "@ai-sdk/google": "4.0.69",
    "@ai-sdk/openai": "4.0.66",
    "ai": "7.0.100",
    "yaml": "2.9.1",
    "zod": "4.6.5"
  },
  "devDependencies": {
    "@biomejs/biome": "2.5.13",
    "@types/bun": "latest"
  }
}
```

- [x] **Step 2: Write tsconfig, biome, bunfig and .gitignore**

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "types": ["bun-types"],
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src", "tests"]
}
```

`biome.json` — run `bunx biome init` and then add the ignore list so it becomes:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.5.13/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": false, "includes": ["**", "!**/.forge/**", "!**/coverage/**"] },
  "formatter": { "enabled": true, "indentStyle": "tab" },
  "linter": { "enabled": true, "rules": { "preset": "recommended" } },
  "javascript": { "formatter": { "quoteStyle": "double" } },
  "assist": { "enabled": true, "actions": { "source": { "organizeImports": "on" } } }
}
```

If `biome check .` rejects the `files.includes` key, run `bunx biome migrate --write` and keep whatever key the migration produces for ignoring `.forge/` and `coverage/`. Record the final key in the task note.

`bunfig.toml`:

```toml
[test]
coverage = true
coverageThreshold = { lines = 0.95, functions = 0.95 }
coverageSkipTestFiles = true
```

`.gitignore`:

```
node_modules/
coverage/
.forge/failures/
.forge/usage.jsonl
```

- [x] **Step 3: Write the failing test for ForgeError**

`tests/core/errors.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { ForgeError } from "../../src/core/errors";

describe("ForgeError", () => {
	test("message alone", () => {
		const e = new ForgeError("boom");
		expect(e.message).toBe("boom");
		expect(e.details).toEqual({});
		expect(e).toBeInstanceOf(Error);
	});

	test("appends file, id and raw path to the message", () => {
		const e = new ForgeError("bad label", { file: ".forge/cases/x.yaml", id: "x-01", rawPath: ".forge/failures/cases-1.txt" });
		expect(e.message).toBe("bad label (file: .forge/cases/x.yaml, id: x-01, raw: .forge/failures/cases-1.txt)");
		expect(e.details.id).toBe("x-01");
	});
});
```

- [x] **Step 4: Run the test to verify it fails**

Run: `bun test tests/core/errors.test.ts`
Expected: FAIL — cannot resolve `../../src/core/errors`.

- [x] **Step 5: Implement ForgeError**

`src/core/errors.ts`:

```ts
export interface ForgeErrorDetails {
	file?: string;
	id?: string;
	rawPath?: string;
}

export class ForgeError extends Error {
	readonly details: ForgeErrorDetails;

	constructor(message: string, details: ForgeErrorDetails = {}) {
		const parts: string[] = [];
		if (details.file) parts.push(`file: ${details.file}`);
		if (details.id) parts.push(`id: ${details.id}`);
		if (details.rawPath) parts.push(`raw: ${details.rawPath}`);
		super(parts.length > 0 ? `${message} (${parts.join(", ")})` : message);
		this.name = "ForgeError";
		this.details = details;
	}
}
```

- [x] **Step 6: Run the test, lint, and the coverage canary**

Run: `bun test` — Expected: 2 pass, coverage table printed, exit 0.
Run: `bun run lint` — Expected: no errors (fix formatting with `bun run format` if it complains).
Canary: temporarily add `export function never(): number { return 1; }` to `src/core/errors.ts`, run `bun test`, expect exit 1 with the functions percentage below 95. Remove the function, run again, expect exit 0. Note the two exit codes in the task note — the gate is only trusted after it has been seen red.

- [x] **Step 7: Commit**

```bash
git add package.json bun.lock tsconfig.json biome.json bunfig.toml .gitignore src/core/errors.ts tests/core/errors.test.ts
git commit -m "Scaffold bun/TypeScript project with Biome, coverage gate and ForgeError"
```

---

### Task 2: Schemas and file I/O for the four `.forge/` files

**Files:**
- Create: `src/core/schemas.ts`, `src/core/files.ts`, `tests/core/schemas.test.ts`, `tests/core/files.test.ts`, `tests/fixtures/feature.yaml`

**Interfaces:**
- Produces (schemas.ts): `Kind`, `Oracle`, `Status`, `FeatureStatus` zod enums; `FeatureSchema`, `ScenarioSchema`, `CaseSchema`, `ExpectedSchema`, `SuiteSchema`; types `Feature`, `Scenario`, `Case`, `Expected`, `Suite` via `z.infer`.
- Produces (files.ts): `forgePaths(forgeDir: string)` → `{ root, feature, scenarios, casesDir, suite, usage, failuresDir }`; `readFeature(forgeDir): Promise<Feature>`; `writeFeature(forgeDir, f): Promise<void>`; `readScenarios(forgeDir): Promise<Scenario[]>` (empty array when the file is absent); `writeScenarios(forgeDir, s)`; `readCases(forgeDir, scenarioId): Promise<Case[]>` (empty when absent); `writeCases(forgeDir, scenarioId, cases)`; `listCaseScenarios(forgeDir): Promise<string[]>` (scenario ids that have a cases file, sorted). Every read validates with the schema and throws `ForgeError` naming the file on failure.

- [x] **Step 1: Write the failing schema tests**

`tests/core/schemas.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { CaseSchema, FeatureSchema, ScenarioSchema, SuiteSchema } from "../../src/core/schemas";

const feature = {
	id: "classify-email",
	purpose: "Classify a hiring-process email",
	inputs: [{ name: "email", kind: "text", notes: "from, subject and body" }],
	output: { kind: "json", fields: ["type", "summary"], label_field: "type", labels: ["rejection", "unrelated"] },
	invariants: ["Answer is JSON only"],
	status: "pending",
};

describe("FeatureSchema", () => {
	test("accepts a full feature", () => {
		expect(FeatureSchema.parse(feature)).toEqual(feature);
	});
	test("defaults invariants to [] and rejects unknown status", () => {
		const { invariants: _i, ...noInv } = feature;
		expect(FeatureSchema.parse(noInv).invariants).toEqual([]);
		expect(FeatureSchema.safeParse({ ...feature, status: "rejected" }).success).toBe(false);
	});
	test("label output requires labels", () => {
		expect(FeatureSchema.safeParse({ ...feature, output: { kind: "label" } }).success).toBe(false);
		expect(FeatureSchema.safeParse({ ...feature, output: { kind: "label", labels: ["a", "b"] } }).success).toBe(true);
	});
});

describe("ScenarioSchema", () => {
	test("requires kind and oracle from the enums", () => {
		const ok = { id: "polite-rejection", kind: "happy", oracle: "label", description: "a polite no", status: "pending" };
		expect(ScenarioSchema.parse(ok)).toEqual(ok);
		expect(ScenarioSchema.safeParse({ ...ok, kind: "weird" }).success).toBe(false);
		expect(ScenarioSchema.safeParse({ ...ok, oracle: "exact" }).success).toBe(false);
	});
});

describe("CaseSchema", () => {
	test("expected is optional (imported cases) and duplicate_of is optional", () => {
		const c = { id: "polite-rejection-01", scenario: "polite-rejection", input: { email: "..." }, status: "pending", generated_by: "google/gemini-3.5-flash" };
		expect(CaseSchema.parse(c)).toEqual(c);
		const withExpected = { ...c, expected: { label: "rejection" }, duplicate_of: "polite-rejection-02" };
		expect(CaseSchema.parse(withExpected)).toEqual(withExpected);
	});
	test("expected must carry exactly one of label, fields, rubric", () => {
		const base = { id: "x-01", scenario: "x", input: { a: "b" }, status: "pending", generated_by: "t" };
		expect(CaseSchema.safeParse({ ...base, expected: {} }).success).toBe(false);
		expect(CaseSchema.safeParse({ ...base, expected: { label: "a", rubric: "b" } }).success).toBe(false);
		expect(CaseSchema.safeParse({ ...base, expected: { fields: { company: "Acme" } } }).success).toBe(true);
	});
});

describe("SuiteSchema", () => {
	test("defaults repeat to 1 and include to []", () => {
		const s = SuiteSchema.parse({ target: { kind: "promptfoo-python", entry: "forge_target.py", models: ["anthropic/claude-haiku-4-5"] }, judges: ["anthropic/claude-sonnet-5"] });
		expect(s.repeat).toBe(1);
		expect(s.include).toEqual([]);
	});
});
```

- [x] **Step 2: Run to verify it fails**

Run: `bun test tests/core/schemas.test.ts` — Expected: FAIL, module not found.

- [x] **Step 3: Implement the schemas**

`src/core/schemas.ts`:

```ts
import { z } from "zod";

export const Kind = z.enum(["happy", "edge", "ambiguous", "out_of_scope", "adversarial", "language"]);
export const Oracle = z.enum(["label", "fields", "rubric"]);
export const Status = z.enum(["pending", "approved", "rejected", "edited"]);
export const FeatureStatus = z.enum(["pending", "approved", "edited"]);

export const FeatureInputSchema = z.object({
	name: z.string().min(1),
	kind: z.enum(["text", "json"]),
	notes: z.string().optional(),
});

export const FeatureOutputSchema = z
	.object({
		kind: z.enum(["text", "json", "label"]),
		fields: z.array(z.string()).optional(),
		label_field: z.string().optional(),
		labels: z.array(z.string()).optional(),
	})
	.refine((o) => o.kind !== "label" || (o.labels !== undefined && o.labels.length > 0), {
		message: "output.kind 'label' requires a non-empty labels list",
	});

export const FeatureSchema = z.object({
	id: z.string().min(1),
	purpose: z.string().min(1),
	inputs: z.array(FeatureInputSchema).min(1),
	output: FeatureOutputSchema,
	invariants: z.array(z.string()).default([]),
	prompt_file: z.string().optional(),
	status: FeatureStatus,
});

export const ScenarioSchema = z.object({
	id: z.string().min(1),
	kind: Kind,
	oracle: Oracle,
	description: z.string().min(1),
	status: Status,
});

export const ExpectedSchema = z
	.object({
		label: z.string().optional(),
		fields: z.record(z.string(), z.unknown()).optional(),
		rubric: z.string().optional(),
	})
	.refine((e) => [e.label, e.fields, e.rubric].filter((v) => v !== undefined).length === 1, {
		message: "expected must carry exactly one of label, fields, rubric",
	});

export const CaseSchema = z.object({
	id: z.string().min(1),
	scenario: z.string().min(1),
	input: z.record(z.string(), z.string()),
	expected: ExpectedSchema.optional(),
	status: Status,
	generated_by: z.string().min(1),
	duplicate_of: z.string().optional(),
});

export const SuiteSchema = z.object({
	target: z.object({
		kind: z.literal("promptfoo-python"),
		entry: z.string().min(1),
		models: z.array(z.string()).min(1),
	}),
	judges: z.array(z.string()).min(1),
	repeat: z.number().int().min(1).default(1),
	include: z.array(z.string()).default([]),
});

export type Kind = z.infer<typeof Kind>;
export type Oracle = z.infer<typeof Oracle>;
export type Status = z.infer<typeof Status>;
export type Feature = z.infer<typeof FeatureSchema>;
export type Scenario = z.infer<typeof ScenarioSchema>;
export type Expected = z.infer<typeof ExpectedSchema>;
export type Case = z.infer<typeof CaseSchema>;
export type Suite = z.infer<typeof SuiteSchema>;
```

- [x] **Step 4: Run the schema tests**

Run: `bun test tests/core/schemas.test.ts` — Expected: all pass.

- [x] **Step 5: Write the failing file I/O tests**

`tests/fixtures/feature.yaml`:

```yaml
id: classify-email
purpose: Classify a hiring-process email received by a job candidate
inputs:
  - name: email
    kind: text
    notes: from, subject and body of one email
output:
  kind: json
  fields: [type, stage, new_stage, company, job_title, summary]
  label_field: type
  labels: [rejection, acknowledgement, interview, screening, offer, info_request, unrelated]
invariants:
  - Automated confirmations are acknowledgement, never screening
  - Answer is JSON only, no prose
status: approved
```

`tests/core/files.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp, readFile, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { ForgeError } from "../../src/core/errors";
import { forgePaths, listCaseScenarios, readCases, readFeature, readScenarios, writeCases, writeFeature, writeScenarios } from "../../src/core/files";
import type { Case, Scenario } from "../../src/core/schemas";

async function tmpForge(): Promise<string> {
	const dir = await mkdtemp(join(tmpdir(), "forge-"));
	return join(dir, ".forge");
}

describe("forgePaths", () => {
	test("derives every path from the forge dir", () => {
		const p = forgePaths("/x/.forge");
		expect(p.feature).toBe("/x/.forge/feature.yaml");
		expect(p.scenarios).toBe("/x/.forge/scenarios.yaml");
		expect(p.casesDir).toBe("/x/.forge/cases");
		expect(p.suite).toBe("/x/.forge/suite.yaml");
		expect(p.usage).toBe("/x/.forge/usage.jsonl");
		expect(p.failuresDir).toBe("/x/.forge/failures");
	});
});

describe("feature round trip", () => {
	test("write then read returns the same object and creates the directory", async () => {
		const dir = await tmpForge();
		const fixture = await readFile(join(import.meta.dir, "../fixtures/feature.yaml"), "utf8");
		const { parse } = await import("yaml");
		const feature = parse(fixture);
		await writeFeature(dir, feature);
		expect(await readFeature(dir)).toEqual(feature);
	});

	test("invalid file throws ForgeError naming the file", async () => {
		const dir = await tmpForge();
		await writeFeature(dir, { id: "a", purpose: "b", inputs: [{ name: "x", kind: "text" }], output: { kind: "text" }, invariants: [], status: "pending" });
		await writeFile(forgePaths(dir).feature, "id: only-an-id\n");
		const err = await readFeature(dir).catch((e: unknown) => e);
		expect(err).toBeInstanceOf(ForgeError);
		expect((err as ForgeError).message).toContain("feature.yaml");
	});
});

describe("scenarios and cases", () => {
	test("missing files read as empty lists", async () => {
		const dir = await tmpForge();
		expect(await readScenarios(dir)).toEqual([]);
		expect(await readCases(dir, "nope")).toEqual([]);
		expect(await listCaseScenarios(dir)).toEqual([]);
	});

	test("round trip and listing sorted by scenario id", async () => {
		const dir = await tmpForge();
		const scenarios: Scenario[] = [{ id: "b-scn", kind: "happy", oracle: "label", description: "b", status: "pending" }];
		await writeScenarios(dir, scenarios);
		expect(await readScenarios(dir)).toEqual(scenarios);
		const cases: Case[] = [{ id: "b-scn-01", scenario: "b-scn", input: { email: "hi" }, expected: { label: "x" }, status: "pending", generated_by: "t" }];
		await writeCases(dir, "b-scn", cases);
		await writeCases(dir, "a-scn", cases.map((c) => ({ ...c, id: "a-scn-01", scenario: "a-scn" })));
		expect(await readCases(dir, "b-scn")).toEqual(cases);
		expect(await listCaseScenarios(dir)).toEqual(["a-scn", "b-scn"]);
	});
});
```

- [x] **Step 6: Run to verify it fails**

Run: `bun test tests/core/files.test.ts` — Expected: FAIL, module not found.

- [x] **Step 7: Implement files.ts**

`src/core/files.ts`:

```ts
import { mkdir, readdir, readFile, writeFile } from "node:fs/promises";
import { basename, join } from "node:path";
import { parse, stringify } from "yaml";
import type { ZodType } from "zod";
import { ForgeError } from "./errors";
import { type Case, CaseSchema, type Feature, FeatureSchema, type Scenario, ScenarioSchema } from "./schemas";
import { z } from "zod";

export interface ForgePaths {
	root: string;
	feature: string;
	scenarios: string;
	casesDir: string;
	suite: string;
	usage: string;
	failuresDir: string;
}

export function forgePaths(forgeDir: string): ForgePaths {
	return {
		root: forgeDir,
		feature: join(forgeDir, "feature.yaml"),
		scenarios: join(forgeDir, "scenarios.yaml"),
		casesDir: join(forgeDir, "cases"),
		suite: join(forgeDir, "suite.yaml"),
		usage: join(forgeDir, "usage.jsonl"),
		failuresDir: join(forgeDir, "failures"),
	};
}

async function exists(path: string): Promise<boolean> {
	return readFile(path).then(() => true, () => false);
}

export async function readYamlFile<T>(path: string, schema: ZodType<T>): Promise<T> {
	const text = await readFile(path, "utf8").catch(() => {
		throw new ForgeError("file not found", { file: path });
	});
	let data: unknown;
	try {
		data = parse(text);
	} catch (e) {
		throw new ForgeError(`invalid YAML: ${(e as Error).message}`, { file: path });
	}
	const result = schema.safeParse(data);
	if (!result.success) {
		throw new ForgeError(`does not match schema: ${result.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`).join("; ")}`, { file: path });
	}
	return result.data;
}

export async function writeYamlFile(path: string, data: unknown): Promise<void> {
	await mkdir(join(path, ".."), { recursive: true });
	await writeFile(path, stringify(data, { lineWidth: 0 }), "utf8");
}

export async function readFeature(forgeDir: string): Promise<Feature> {
	return readYamlFile(forgePaths(forgeDir).feature, FeatureSchema);
}

export async function writeFeature(forgeDir: string, feature: Feature): Promise<void> {
	await writeYamlFile(forgePaths(forgeDir).feature, feature);
}

export async function readScenarios(forgeDir: string): Promise<Scenario[]> {
	const path = forgePaths(forgeDir).scenarios;
	if (!(await exists(path))) return [];
	return readYamlFile(path, z.array(ScenarioSchema));
}

export async function writeScenarios(forgeDir: string, scenarios: Scenario[]): Promise<void> {
	await writeYamlFile(forgePaths(forgeDir).scenarios, scenarios);
}

function casesPath(forgeDir: string, scenarioId: string): string {
	return join(forgePaths(forgeDir).casesDir, `${scenarioId}.yaml`);
}

export async function readCases(forgeDir: string, scenarioId: string): Promise<Case[]> {
	const path = casesPath(forgeDir, scenarioId);
	if (!(await exists(path))) return [];
	return readYamlFile(path, z.array(CaseSchema));
}

export async function writeCases(forgeDir: string, scenarioId: string, cases: Case[]): Promise<void> {
	await writeYamlFile(casesPath(forgeDir, scenarioId), cases);
}

export async function listCaseScenarios(forgeDir: string): Promise<string[]> {
	const dir = forgePaths(forgeDir).casesDir;
	const names = await readdir(dir).catch(() => [] as string[]);
	return names
		.filter((n) => n.endsWith(".yaml"))
		.map((n) => basename(n, ".yaml"))
		.sort();
}
```

- [x] **Step 8: Run all tests and lint**

Run: `bun test` — Expected: all pass, coverage above threshold.
Run: `bun run lint` — Expected: clean (Biome may ask to reorder the two `zod` imports in files.ts; apply `bun run format`).

- [x] **Step 9: Commit**

```bash
git add src/core/schemas.ts src/core/files.ts tests/core/schemas.test.ts tests/core/files.test.ts tests/fixtures/feature.yaml
git commit -m "Add zod schemas and YAML I/O for feature, scenarios, cases and suite"
```

---

### Task 3: The model layer — registry, `generate`, usage log, failures, fake provider

**Files:**
- Create: `src/llm/models.ts`, `src/llm/generate.ts`, `src/llm/templates.ts`, `tests/llm/models.test.ts`, `tests/llm/generate.test.ts`, `tests/llm/templates.test.ts`, `tests/fixtures/fake-responses.json`, `templates/.gitkeep`

**Interfaces:**
- Produces (models.ts): `parseModelSpec(spec: string): { provider: "anthropic" | "google" | "openai" | "ollama" | "fake"; model: string }` — throws `ForgeError` on anything else; `resolveModel(spec: string, env?: Record<string, string | undefined>): LanguageModel` — builds the provider with the key from env (`ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `OPENAI_API_KEY`; ollama needs none), throws `ForgeError` naming the missing variable; for `fake/<path>` returns a `MockLanguageModelV4` fed by `<path>` (see below).
- Produces (generate.ts): `interface Llm { generate<T>(args: { schema: ZodType<T>; prompt: string; model: string; verb: string }): Promise<{ object: T; usage: { inputTokens: number; outputTokens: number } }> }`; `createLlm(opts: { forgeDir: string; resolve?: (spec: string) => LanguageModel; now?: () => Date }): Llm`. Appends one JSON line to `.forge/usage.jsonl` per call: `{ ts, verb, model, inputTokens, outputTokens }`. On `NoObjectGeneratedError` writes the raw text to `.forge/failures/<verb>-<ISO timestamp with colons replaced by dashes>.txt` and throws `ForgeError("model output did not match the <verb> schema", { rawPath })`.
- Produces (templates.ts): `loadTemplate(name: string): Promise<string>` reads `templates/<name>.md` relative to the package root (`new URL("../../templates/", import.meta.url)`); `render(template: string, vars: Record<string, string>): string` replaces every `{{name}}` and throws `ForgeError` if a placeholder has no value.
- Fake provider contract: `<path>` is a JSON file `{ "<verb>": string | string[] }` keyed by the verb; `generate` passes the verb through `providerOptions` is not available on the mock, so `createLlm` handles `fake/` itself: it reads the file, picks the entry for `args.verb`, and if it is an array returns entries in order across calls within the process (a module-level cursor per file+verb). Missing verb → `ForgeError`.

- [x] **Step 1: Write the failing tests for models.ts**

`tests/llm/models.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { ForgeError } from "../../src/core/errors";
import { parseModelSpec, resolveModel } from "../../src/llm/models";

describe("parseModelSpec", () => {
	test("splits provider and model on the first slash", () => {
		expect(parseModelSpec("anthropic/claude-haiku-4-5")).toEqual({ provider: "anthropic", model: "claude-haiku-4-5" });
		expect(parseModelSpec("fake/tests/fixtures/fake-responses.json")).toEqual({ provider: "fake", model: "tests/fixtures/fake-responses.json" });
	});
	test("rejects unknown providers and missing slash", () => {
		expect(() => parseModelSpec("mistral/x")).toThrow(ForgeError);
		expect(() => parseModelSpec("claude-haiku-4-5")).toThrow(ForgeError);
	});
});

describe("resolveModel", () => {
	test("builds a real provider model when the key is present", () => {
		const m = resolveModel("anthropic/claude-haiku-4-5", { ANTHROPIC_API_KEY: "sk-test" });
		expect(m.modelId).toBe("claude-haiku-4-5");
		expect(m.provider).toContain("anthropic");
	});
	test("names the missing environment variable", () => {
		expect(() => resolveModel("google/gemini-3.5-flash", {})).toThrow("GOOGLE_API_KEY");
		expect(() => resolveModel("openai/gpt-5-mini", {})).toThrow("OPENAI_API_KEY");
	});
	test("ollama needs no key", () => {
		const m = resolveModel("ollama/llama3.2", {});
		expect(m.modelId).toBe("llama3.2");
	});
});
```

- [x] **Step 2: Run to verify it fails**

Run: `bun test tests/llm/models.test.ts` — Expected: FAIL, module not found.

- [x] **Step 3: Implement models.ts**

`src/llm/models.ts`:

```ts
import { createAnthropic } from "@ai-sdk/anthropic";
import { createGoogleGenerativeAI } from "@ai-sdk/google";
import { createOpenAI } from "@ai-sdk/openai";
import type { LanguageModel } from "ai";
import { ForgeError } from "../core/errors";

export const PROVIDERS = ["anthropic", "google", "openai", "ollama", "fake"] as const;
export type Provider = (typeof PROVIDERS)[number];

export interface ModelSpec {
	provider: Provider;
	model: string;
}

export function parseModelSpec(spec: string): ModelSpec {
	const slash = spec.indexOf("/");
	if (slash <= 0) throw new ForgeError(`model spec must be provider/model, got "${spec}"`);
	const provider = spec.slice(0, slash);
	const model = spec.slice(slash + 1);
	if (!(PROVIDERS as readonly string[]).includes(provider)) {
		throw new ForgeError(`unknown provider "${provider}" in "${spec}"; known: ${PROVIDERS.join(", ")}`);
	}
	return { provider: provider as Provider, model };
}

function requireKey(env: Record<string, string | undefined>, name: string, spec: string): string {
	const value = env[name];
	if (!value) throw new ForgeError(`${name} is not set; needed for "${spec}"`);
	return value;
}

export function resolveModel(spec: string, env: Record<string, string | undefined> = process.env): LanguageModel {
	const { provider, model } = parseModelSpec(spec);
	switch (provider) {
		case "anthropic":
			return createAnthropic({ apiKey: requireKey(env, "ANTHROPIC_API_KEY", spec) })(model);
		case "google":
			return createGoogleGenerativeAI({ apiKey: requireKey(env, "GOOGLE_API_KEY", spec) })(model);
		case "openai":
			return createOpenAI({ apiKey: requireKey(env, "OPENAI_API_KEY", spec) })(model);
		case "ollama":
			// Ollama's OpenAI-compatible endpoint; not exercised outside tests/live (see plan header).
			return createOpenAI({ apiKey: "ollama", baseURL: "http://localhost:11434/v1" })(model);
		case "fake":
			throw new ForgeError(`"fake/" models are handled by createLlm, not resolveModel ("${spec}")`);
	}
}
```

- [x] **Step 4: Run the models tests**

Run: `bun test tests/llm/models.test.ts` — Expected: pass. If `m.provider` is not a string containing "anthropic" on this SDK version, print it once with `console.log` and change the assertion to the real value; record in the task note.

- [x] **Step 5: Write the failing tests for templates.ts**

`tests/llm/templates.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { ForgeError } from "../../src/core/errors";
import { loadTemplate, render } from "../../src/llm/templates";

describe("render", () => {
	test("replaces every placeholder", () => {
		expect(render("Hi {{name}}, {{name}} again: {{thing}}", { name: "A", thing: "B" })).toBe("Hi A, A again: B");
	});
	test("throws on a placeholder without a value", () => {
		expect(() => render("{{missing}}", {})).toThrow(ForgeError);
	});
});

describe("loadTemplate", () => {
	test("reads templates/<name>.md from the package root", async () => {
		const text = await loadTemplate("describe");
		expect(text).toContain("{{description}}");
	});
	test("throws ForgeError naming the file when absent", async () => {
		const err = await loadTemplate("nope").catch((e: unknown) => e);
		expect(err).toBeInstanceOf(ForgeError);
		expect((err as ForgeError).message).toContain("nope.md");
	});
});
```

- [x] **Step 6: Implement templates.ts and the first template**

`src/llm/templates.ts`:

```ts
import { readFile } from "node:fs/promises";
import { fileURLToPath } from "node:url";
import { ForgeError } from "../core/errors";

const TEMPLATES_DIR = fileURLToPath(new URL("../../templates/", import.meta.url));

export async function loadTemplate(name: string): Promise<string> {
	const path = `${TEMPLATES_DIR}${name}.md`;
	return readFile(path, "utf8").catch(() => {
		throw new ForgeError("template not found", { file: path });
	});
}

export function render(template: string, vars: Record<string, string>): string {
	return template.replace(/\{\{(\w+)\}\}/g, (_, key: string) => {
		const value = vars[key];
		if (value === undefined) throw new ForgeError(`template placeholder "{{${key}}}" has no value`);
		return value;
	});
}
```

`templates/describe.md` (the real prompt for Task 4; written now so the template test has a file):

```
You are helping a developer write a regression test suite for one feature of their application that calls an LLM.

Below is the developer's own description of the feature, and optionally the real prompt the application sends. Normalise them into the structured form requested by the schema. Do not invent capabilities the description does not state. When the description names a closed set of possible answers (labels, categories, decisions), list them exactly as written, in the order given. When the real prompt states rules ("answer only with JSON", "never do X"), copy each rule as one invariant in the developer's words. Keep `purpose` to one sentence.

The `id` must be kebab-case, at most five words, derived from the purpose.

Developer's description:
<description>
{{description}}
</description>

Real prompt sent by the application (may be empty):
<prompt>
{{prompt}}
</prompt>
```

- [x] **Step 7: Run the template tests**

Run: `bun test tests/llm/templates.test.ts` — Expected: pass.

- [x] **Step 8: Write the failing tests for generate.ts**

`tests/fixtures/fake-responses.json`:

```json
{
  "probe": "{\"answer\": 42}",
  "sequence": ["{\"answer\": 1}", "{\"answer\": 2}"],
  "broken": "```json\n{\"answer\": 42}\n```"
}
```

`tests/llm/generate.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp, readdir, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { MockLanguageModelV4 } from "ai/test";
import { z } from "zod";
import { ForgeError } from "../../src/core/errors";
import { forgePaths } from "../../src/core/files";
import { createLlm } from "../../src/llm/generate";

const schema = z.object({ answer: z.number() });
const FAKE = "fake/tests/fixtures/fake-responses.json";

function mock(text: string) {
	return new MockLanguageModelV4({
		doGenerate: async () => ({
			content: [{ type: "text", text }],
			finishReason: "stop",
			usage: { inputTokens: { total: 11, noCache: 11, cacheRead: 0, cacheWrite: 0 }, outputTokens: { total: 7, text: 7, reasoning: 0 } },
			warnings: [],
		}),
	});
}

async function tmpForge(): Promise<string> {
	return join(await mkdtemp(join(tmpdir(), "forge-")), ".forge");
}

describe("createLlm.generate", () => {
	test("returns the validated object and usage, and appends to usage.jsonl", async () => {
		const dir = await tmpForge();
		const llm = createLlm({ forgeDir: dir, resolve: () => mock('{"answer": 42}'), now: () => new Date("2026-09-14T10:00:00Z") });
		const r = await llm.generate({ schema, prompt: "p", model: "anthropic/x", verb: "probe" });
		expect(r.object).toEqual({ answer: 42 });
		expect(r.usage).toEqual({ inputTokens: 11, outputTokens: 7 });
		const log = await readFile(forgePaths(dir).usage, "utf8");
		expect(JSON.parse(log.trim())).toEqual({ ts: "2026-09-14T10:00:00.000Z", verb: "probe", model: "anthropic/x", inputTokens: 11, outputTokens: 7 });
	});

	test("saves raw text under failures/ and throws ForgeError with the path when output does not validate", async () => {
		const dir = await tmpForge();
		const llm = createLlm({ forgeDir: dir, resolve: () => mock('{"answer": "not a number"}'), now: () => new Date("2026-09-14T10:00:00Z") });
		const err = await llm.generate({ schema, prompt: "p", model: "anthropic/x", verb: "probe" }).catch((e: unknown) => e);
		expect(err).toBeInstanceOf(ForgeError);
		const rawPath = (err as ForgeError).details.rawPath;
		expect(rawPath).toBe(join(forgePaths(dir).failuresDir, "probe-2026-09-14T10-00-00.000Z.txt"));
		expect(await readFile(rawPath as string, "utf8")).toBe('{"answer": "not a number"}');
		expect(await readdir(forgePaths(dir).failuresDir)).toHaveLength(1);
	});

	test("fake provider picks the response by verb, in sequence for arrays", async () => {
		const dir = await tmpForge();
		const llm = createLlm({ forgeDir: dir });
		expect((await llm.generate({ schema, prompt: "p", model: FAKE, verb: "probe" })).object).toEqual({ answer: 42 });
		expect((await llm.generate({ schema, prompt: "p", model: FAKE, verb: "sequence" })).object).toEqual({ answer: 1 });
		expect((await llm.generate({ schema, prompt: "p", model: FAKE, verb: "sequence" })).object).toEqual({ answer: 2 });
	});

	test("fake provider with a fenced response still fails loudly (generateObject does not strip fences)", async () => {
		const dir = await tmpForge();
		const llm = createLlm({ forgeDir: dir });
		await expect(llm.generate({ schema, prompt: "p", model: FAKE, verb: "broken" })).rejects.toBeInstanceOf(ForgeError);
	});

	test("fake provider with an unknown verb throws ForgeError", async () => {
		const dir = await tmpForge();
		const llm = createLlm({ forgeDir: dir });
		await expect(llm.generate({ schema, prompt: "p", model: FAKE, verb: "nope" })).rejects.toThrow(ForgeError);
	});
});
```

- [x] **Step 9: Run to verify it fails**

Run: `bun test tests/llm/generate.test.ts` — Expected: FAIL, module not found.

- [x] **Step 10: Implement generate.ts**

`src/llm/generate.ts`:

```ts
import { appendFile, mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";
import { generateObject, type LanguageModel, NoObjectGeneratedError } from "ai";
import { MockLanguageModelV4 } from "ai/test";
import type { ZodType } from "zod";
import { ForgeError } from "../core/errors";
import { forgePaths } from "../core/files";
import { parseModelSpec, resolveModel } from "./models";

export interface Usage {
	inputTokens: number;
	outputTokens: number;
}

export interface GenerateArgs<T> {
	schema: ZodType<T>;
	prompt: string;
	model: string;
	verb: string;
}

export interface Llm {
	generate<T>(args: GenerateArgs<T>): Promise<{ object: T; usage: Usage }>;
}

export interface CreateLlmOptions {
	forgeDir: string;
	resolve?: (spec: string) => LanguageModel;
	now?: () => Date;
}

const fakeCursors = new Map<string, number>();

async function fakeModel(path: string, verb: string): Promise<LanguageModel> {
	const text = await readFile(path, "utf8").catch(() => {
		throw new ForgeError("fake responses file not found", { file: path });
	});
	const responses = JSON.parse(text) as Record<string, string | string[]>;
	const entry = responses[verb];
	if (entry === undefined) throw new ForgeError(`fake responses file has no entry for verb "${verb}"`, { file: path });
	let reply: string;
	if (Array.isArray(entry)) {
		const key = `${path}::${verb}`;
		const i = fakeCursors.get(key) ?? 0;
		const picked = entry[Math.min(i, entry.length - 1)];
		if (picked === undefined) throw new ForgeError(`fake responses entry for "${verb}" is empty`, { file: path });
		reply = picked;
		fakeCursors.set(key, i + 1);
	} else {
		reply = entry;
	}
	return new MockLanguageModelV4({
		doGenerate: async () => ({
			content: [{ type: "text", text: reply }],
			finishReason: "stop",
			usage: { inputTokens: { total: 0, noCache: 0, cacheRead: 0, cacheWrite: 0 }, outputTokens: { total: 0, text: 0, reasoning: 0 } },
			warnings: [],
		}),
	});
}

export function createLlm(opts: CreateLlmOptions): Llm {
	const paths = forgePaths(opts.forgeDir);
	const now = opts.now ?? (() => new Date());
	const resolve = opts.resolve ?? resolveModel;

	async function pickModel(spec: string, verb: string): Promise<LanguageModel> {
		const { provider, model } = parseModelSpec(spec);
		return provider === "fake" ? fakeModel(model, verb) : resolve(spec);
	}

	return {
		async generate<T>(args: GenerateArgs<T>) {
			const model = await pickModel(args.model, args.verb);
			const ts = now().toISOString();
			try {
				const result = await generateObject({ model, schema: args.schema, prompt: args.prompt });
				const usage: Usage = { inputTokens: result.usage.inputTokens ?? 0, outputTokens: result.usage.outputTokens ?? 0 };
				await mkdir(paths.root, { recursive: true });
				await appendFile(paths.usage, `${JSON.stringify({ ts, verb: args.verb, model: args.model, ...usage })}\n`);
				return { object: result.object, usage };
			} catch (e) {
				if (NoObjectGeneratedError.isInstance(e)) {
					await mkdir(paths.failuresDir, { recursive: true });
					const rawPath = join(paths.failuresDir, `${args.verb}-${ts.replace(/:/g, "-")}.txt`);
					await writeFile(rawPath, e.text ?? "", "utf8");
					throw new ForgeError(`model output did not match the ${args.verb} schema: ${e.message}`, { rawPath });
				}
				throw e;
			}
		},
	};
}
```

- [x] **Step 11: Run all tests and lint**

Run: `bun test` — Expected: pass. If `NoObjectGeneratedError.isInstance` does not exist on this version, use `e instanceof NoObjectGeneratedError` and note it.
Run: `bun run lint` — Expected: clean.

- [x] **Step 12: Commit**

```bash
git add src/llm tests/llm tests/fixtures/fake-responses.json templates/describe.md
git commit -m "Add the model layer: provider registry, generate with usage log and failure capture, fake provider"
```

---

### Task 4: `describe` — free text (plus optional prompt) to a pending feature

**Files:**
- Create: `src/core/describe.ts`, `tests/core/describe.test.ts`, `tests/fixtures/describe-response.json`
- Modify: `templates/describe.md` (already written in Task 3; unchanged)

**Interfaces:**
- Consumes: `Llm` from `src/llm/generate.ts`; `loadTemplate`, `render` from `src/llm/templates.ts`; `FeatureSchema`, `Feature` from `src/core/schemas.ts`.
- Produces: `describeFeature(args: { text: string; promptText?: string; model: string; llm: Llm }): Promise<Feature>` — returns a feature with `status: "pending"` regardless of what the model said; if `promptText` is given, `prompt_file` is **not** set here (the CLI sets it from the flag); throws `ForgeError` if `text` is blank.

- [x] **Step 1: Write the failing test**

`tests/fixtures/describe-response.json` — a recorded shape the model returns (the generator schema is the feature without `status`):

```json
{
  "describe": "{\"id\":\"classify-email\",\"purpose\":\"Classify a hiring-process email received by a job candidate\",\"inputs\":[{\"name\":\"email\",\"kind\":\"text\",\"notes\":\"from, subject and body\"}],\"output\":{\"kind\":\"json\",\"fields\":[\"type\",\"summary\"],\"label_field\":\"type\",\"labels\":[\"rejection\",\"acknowledgement\",\"unrelated\"]},\"invariants\":[\"Answer is JSON only\"]}"
}
```

`tests/core/describe.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describeFeature } from "../../src/core/describe";
import { ForgeError } from "../../src/core/errors";
import { createLlm, type Llm } from "../../src/llm/generate";

const FAKE = "fake/tests/fixtures/describe-response.json";

async function llm(): Promise<Llm> {
	return createLlm({ forgeDir: join(await mkdtemp(join(tmpdir(), "forge-")), ".forge") });
}

describe("describeFeature", () => {
	test("returns a pending feature from the model's answer", async () => {
		const f = await describeFeature({ text: "The bot classifies hiring emails", model: FAKE, llm: await llm() });
		expect(f.status).toBe("pending");
		expect(f.id).toBe("classify-email");
		expect(f.output.labels).toEqual(["rejection", "acknowledgement", "unrelated"]);
		expect(f.prompt_file).toBeUndefined();
	});

	test("renders description and prompt into the template", async () => {
		const seen: string[] = [];
		const spy: Llm = {
			async generate(args) {
				seen.push(args.prompt);
				return { object: JSON.parse((await import("../fixtures/describe-response.json")).default.describe), usage: { inputTokens: 0, outputTokens: 0 } } as never;
			},
		};
		await describeFeature({ text: "DESC-TEXT", promptText: "PROMPT-TEXT", model: "anthropic/x", llm: spy });
		expect(seen[0]).toContain("DESC-TEXT");
		expect(seen[0]).toContain("PROMPT-TEXT");
	});

	test("rejects blank text", async () => {
		await expect(describeFeature({ text: "   ", model: FAKE, llm: await llm() })).rejects.toThrow(ForgeError);
	});
});
```

- [x] **Step 2: Run to verify it fails**

Run: `bun test tests/core/describe.test.ts` — Expected: FAIL, module not found.

- [x] **Step 3: Implement describe.ts**

`src/core/describe.ts`:

```ts
import type { Llm } from "../llm/generate";
import { loadTemplate, render } from "../llm/templates";
import { ForgeError } from "./errors";
import { type Feature, FeatureSchema } from "./schemas";

export const DescribeOutputSchema = FeatureSchema.omit({ status: true, prompt_file: true });

export interface DescribeArgs {
	text: string;
	promptText?: string;
	model: string;
	llm: Llm;
}

export async function describeFeature(args: DescribeArgs): Promise<Feature> {
	if (args.text.trim() === "") throw new ForgeError("describe needs a non-empty description");
	const template = await loadTemplate("describe");
	const prompt = render(template, { description: args.text.trim(), prompt: args.promptText ?? "" });
	const { object } = await args.llm.generate({ schema: DescribeOutputSchema, prompt, model: args.model, verb: "describe" });
	return { ...object, status: "pending" };
}
```

- [x] **Step 4: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 5: Commit**

```bash
git add src/core/describe.ts tests/core/describe.test.ts tests/fixtures/describe-response.json
git commit -m "Add describe: normalise a free-text description into a pending feature"
```

---

### Task 5: `scenarios` — enumerate with mandatory kind coverage, merge with reviewed ones

**Files:**
- Create: `src/core/scenarios.ts`, `src/core/ids.ts`, `templates/scenarios.md`, `tests/core/scenarios.test.ts`, `tests/core/ids.test.ts`, `tests/fixtures/scenarios-response.json`

**Interfaces:**
- Consumes: `Llm`, templates, `Feature`, `Scenario`, `Kind`, `ScenarioSchema`.
- Produces (ids.ts): `slugify(text: string): string` — lowercase, non-alphanumerics to `-`, collapse, trim, at most 40 chars; `nextCaseId(scenarioId: string, existing: Case[]): string` — `<scenarioId>-<nn>` where nn is one more than the highest existing suffix for that scenario, two digits.
- Produces (scenarios.ts): `enumerateScenarios(args: { feature: Feature; existing: Scenario[]; kinds?: Kind[]; more?: number; model: string; llm: Llm }): Promise<Scenario[]>` — returns `existing` (untouched, in order) followed by new scenarios with `status: "pending"`, ids slugified from the model's id and made unique against existing ids by appending `-2`, `-3`…; throws `ForgeError` when `kinds` is not given and the model's list lacks any kind, naming the missing kinds; when `more` is given, asks for that many additional scenarios and skips the coverage check; throws `ForgeError` when the model returns zero scenarios.

- [x] **Step 1: Write the failing ids tests**

`tests/core/ids.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { nextCaseId, slugify } from "../../src/core/ids";
import type { Case } from "../../src/core/schemas";

describe("slugify", () => {
	test("kebab-cases and trims", () => {
		expect(slugify("Ack with Optional Quiz!")).toBe("ack-with-optional-quiz");
		expect(slugify("  --Já--  ")).toBe("j");
		expect(slugify("a".repeat(60))).toHaveLength(40);
	});
});

describe("nextCaseId", () => {
	const c = (id: string): Case => ({ id, scenario: "s", input: {}, status: "pending", generated_by: "t" });
	test("starts at 01 and continues after the highest suffix", () => {
		expect(nextCaseId("s", [])).toBe("s-01");
		expect(nextCaseId("s", [c("s-01"), c("s-07"), c("other-99")])).toBe("s-08");
	});
});
```

- [x] **Step 2: Implement ids.ts**

`src/core/ids.ts`:

```ts
import type { Case } from "./schemas";

export function slugify(text: string): string {
	return text
		.toLowerCase()
		.replace(/[^a-z0-9]+/g, "-")
		.replace(/^-+|-+$/g, "")
		.slice(0, 40)
		.replace(/-+$/g, "");
}

export function nextCaseId(scenarioId: string, existing: Case[]): string {
	const prefix = `${scenarioId}-`;
	let max = 0;
	for (const c of existing) {
		if (!c.id.startsWith(prefix)) continue;
		const n = Number.parseInt(c.id.slice(prefix.length), 10);
		if (Number.isFinite(n) && n > max) max = n;
	}
	return `${prefix}${String(max + 1).padStart(2, "0")}`;
}
```

Run: `bun test tests/core/ids.test.ts` — Expected: pass (the `"  --Já--  "` case: `á` is not `[a-z0-9]` so it becomes `j`; keep the test as the documented behaviour — accents are dropped, not transliterated).

- [x] **Step 3: Write the scenarios template**

`templates/scenarios.md`:

```
You are enumerating test scenarios for one feature of an application that calls an LLM. A scenario is a class of situations the feature must handle, not a single input.

Feature:
<feature>
{{feature}}
</feature>

Produce {{count}} scenarios{{coverage}}. Each scenario has:
- `id`: kebab-case, at most five words, unique.
- `kind`: one of happy, edge, ambiguous, out_of_scope, adversarial, language.
  - happy: the common, well-formed situations the feature exists for.
  - edge: well-formed but unusual — very long, very short, empty fields, unusual formatting.
  - ambiguous: situations where two answers are defensible and the invariants decide.
  - out_of_scope: inputs the feature is not meant to handle and must answer conservatively.
  - adversarial: inputs that try to make the feature ignore its instructions or leak them.
  - language: the same situations in a language other than the one the description assumes.
- `oracle`: how a case in this scenario is checked — `label` when the feature's output has a closed set of labels and this scenario has one correct label; `fields` when specific output fields have a checkable value; `rubric` when only a judgement sentence can check it.
- `description`: one sentence describing the class of situation, concrete enough that another person could write inputs for it.

Already existing scenarios (do not repeat them, complement them):
<existing>
{{existing}}
</existing>
```

- [x] **Step 4: Write the failing scenarios tests**

`tests/fixtures/scenarios-response.json`:

```json
{
  "scenarios": "{\"scenarios\":[{\"id\":\"polite-rejection\",\"kind\":\"happy\",\"oracle\":\"label\",\"description\":\"A polite rejection after an interview\"},{\"id\":\"huge-signature\",\"kind\":\"edge\",\"oracle\":\"label\",\"description\":\"A short message under a 40-line signature\"},{\"id\":\"ack-with-optional-quiz\",\"kind\":\"ambiguous\",\"oracle\":\"label\",\"description\":\"An automated acknowledgement that offers an optional assessment\"},{\"id\":\"newsletter\",\"kind\":\"out_of_scope\",\"oracle\":\"label\",\"description\":\"A job-board digest\"},{\"id\":\"prompt-injection-in-body\",\"kind\":\"adversarial\",\"oracle\":\"label\",\"description\":\"An email whose body instructs the classifier to answer offer\"},{\"id\":\"portuguese-rejection\",\"kind\":\"language\",\"oracle\":\"label\",\"description\":\"A rejection written in Portuguese\"}]}",
  "missing-kinds": "{\"scenarios\":[{\"id\":\"polite-rejection\",\"kind\":\"happy\",\"oracle\":\"label\",\"description\":\"A polite rejection\"}]}",
  "empty": "{\"scenarios\":[]}",
  "clashing-id": "{\"scenarios\":[{\"id\":\"Polite Rejection\",\"kind\":\"happy\",\"oracle\":\"label\",\"description\":\"again\"}]}"
}
```

`tests/core/scenarios.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { parse } from "yaml";
import { ForgeError } from "../../src/core/errors";
import { enumerateScenarios } from "../../src/core/scenarios";
import type { Feature, Scenario } from "../../src/core/schemas";
import { createLlm, type Llm } from "../../src/llm/generate";

const FIXTURE = "tests/fixtures/scenarios-response.json";

async function llmFor(verb: string): Promise<{ llm: Llm; model: string }> {
	// The fake provider picks by verb; enumerateScenarios always uses verb "scenarios",
	// so route through a tiny adapter that rewrites the verb to the fixture key we want.
	const base = createLlm({ forgeDir: join(await mkdtemp(join(tmpdir(), "forge-")), ".forge") });
	return { model: `fake/${FIXTURE}`, llm: { generate: (args) => base.generate({ ...args, verb }) } };
}

async function feature(): Promise<Feature> {
	return parse(await readFile(join(import.meta.dir, "../fixtures/feature.yaml"), "utf8"));
}

describe("enumerateScenarios", () => {
	test("returns pending scenarios covering every kind", async () => {
		const { llm, model } = await llmFor("scenarios");
		const out = await enumerateScenarios({ feature: await feature(), existing: [], model, llm });
		expect(out).toHaveLength(6);
		expect(new Set(out.map((s) => s.kind)).size).toBe(6);
		expect(out.every((s) => s.status === "pending")).toBe(true);
	});

	test("names the missing kinds when coverage fails", async () => {
		const { llm, model } = await llmFor("missing-kinds");
		const err = await enumerateScenarios({ feature: await feature(), existing: [], model, llm }).catch((e: unknown) => e);
		expect(err).toBeInstanceOf(ForgeError);
		expect((err as ForgeError).message).toContain("edge");
		expect((err as ForgeError).message).toContain("language");
	});

	test("--kinds restricts coverage to the listed kinds", async () => {
		const { llm, model } = await llmFor("missing-kinds");
		const out = await enumerateScenarios({ feature: await feature(), existing: [], kinds: ["happy"], model, llm });
		expect(out).toHaveLength(1);
	});

	test("zero scenarios is an error, never a silent empty file", async () => {
		const { llm, model } = await llmFor("empty");
		await expect(enumerateScenarios({ feature: await feature(), existing: [], model, llm })).rejects.toThrow(ForgeError);
	});

	test("--more keeps reviewed scenarios untouched and makes ids unique", async () => {
		const { llm, model } = await llmFor("clashing-id");
		const existing: Scenario[] = [{ id: "polite-rejection", kind: "happy", oracle: "label", description: "kept", status: "approved" }];
		const out = await enumerateScenarios({ feature: await feature(), existing, more: 1, model, llm });
		expect(out[0]).toEqual(existing[0]);
		expect(out[1]?.id).toBe("polite-rejection-2");
		expect(out[1]?.status).toBe("pending");
	});
});
```

- [x] **Step 5: Run to verify it fails**

Run: `bun test tests/core/scenarios.test.ts` — Expected: FAIL, module not found.

- [x] **Step 6: Implement scenarios.ts**

`src/core/scenarios.ts`:

```ts
import { stringify } from "yaml";
import { z } from "zod";
import type { Llm } from "../llm/generate";
import { loadTemplate, render } from "../llm/templates";
import { ForgeError } from "./errors";
import { slugify } from "./ids";
import { type Feature, Kind, type Scenario, ScenarioSchema } from "./schemas";

const DEFAULT_COUNT = 12;

export const ScenariosOutputSchema = z.object({
	scenarios: z.array(ScenarioSchema.omit({ status: true })),
});

export interface EnumerateScenariosArgs {
	feature: Feature;
	existing: Scenario[];
	kinds?: Kind[];
	more?: number;
	model: string;
	llm: Llm;
}

function uniqueId(base: string, taken: Set<string>): string {
	let id = base;
	let n = 2;
	while (taken.has(id)) {
		id = `${base}-${n}`;
		n += 1;
	}
	taken.add(id);
	return id;
}

export async function enumerateScenarios(args: EnumerateScenariosArgs): Promise<Scenario[]> {
	const wantedKinds = args.kinds ?? [...Kind.options];
	const count = args.more ?? DEFAULT_COUNT;
	const coverage = args.more === undefined ? `, with at least one of each kind: ${wantedKinds.join(", ")}` : "";
	const template = await loadTemplate("scenarios");
	const prompt = render(template, {
		feature: stringify(args.feature, { lineWidth: 0 }),
		count: String(count),
		coverage,
		existing: args.existing.length > 0 ? stringify(args.existing.map((s) => ({ id: s.id, kind: s.kind, description: s.description })), { lineWidth: 0 }) : "(none)",
	});
	const { object } = await args.llm.generate({ schema: ScenariosOutputSchema, prompt, model: args.model, verb: "scenarios" });
	if (object.scenarios.length === 0) throw new ForgeError("the model returned zero scenarios");

	if (args.more === undefined) {
		const present = new Set(object.scenarios.map((s) => s.kind));
		const missing = wantedKinds.filter((k) => !present.has(k));
		if (missing.length > 0) throw new ForgeError(`generated scenarios miss these kinds: ${missing.join(", ")}; rerun, or restrict with --kinds`);
	}

	const taken = new Set(args.existing.map((s) => s.id));
	const fresh: Scenario[] = object.scenarios.map((s) => ({
		...s,
		id: uniqueId(slugify(s.id) || "scenario", taken),
		status: "pending",
	}));
	return [...args.existing, ...fresh];
}
```

- [x] **Step 7: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 8: Commit**

```bash
git add src/core/scenarios.ts src/core/ids.ts templates/scenarios.md tests/core/scenarios.test.ts tests/core/ids.test.ts tests/fixtures/scenarios-response.json
git commit -m "Add scenarios: enumerate with mandatory kind coverage and merge with reviewed ones"
```

---

### Task 6: `cases` — N cases per scenario with oracle-shaped expected, never overwriting reviewed cases

**Files:**
- Create: `src/core/cases.ts`, `templates/cases.md`, `tests/core/cases.test.ts`, `tests/fixtures/cases-response.json`

**Interfaces:**
- Consumes: `Llm`, templates, `Feature`, `Scenario`, `Case`, `ExpectedSchema`, `nextCaseId`.
- Produces: `generateCases(args: { feature: Feature; scenario: Scenario; existing: Case[]; n: number; model: string; llm: Llm }): Promise<Case[]>` — returns `existing` followed by new cases; new cases get ids from `nextCaseId`, `status: "pending"`, `generated_by: args.model`; the model's output schema is `{ cases: [{ input: Record<string,string>, expected: Expected }] }` where `expected` must match the scenario's oracle (label → `label` present and inside `feature.output.labels`; fields → `fields` present; rubric → `rubric` present) — a case violating that is dropped and counted, and if **all** are dropped the function throws `ForgeError` explaining why; a new case whose `input` deep-equals an existing case's input is dropped (exact dedupe); input keys must match `feature.inputs` names exactly or the case is dropped; throws `ForgeError` if the scenario is not `approved`/`edited`.

- [x] **Step 1: Write the cases template**

`templates/cases.md`:

```
You are writing concrete test cases for one scenario of a feature that calls an LLM.

Feature:
<feature>
{{feature}}
</feature>

Scenario:
<scenario>
{{scenario}}
</scenario>

Write {{n}} cases. Each case is an `input` object whose keys are exactly the feature's input names ({{input_names}}) with realistic, varied, complete values — real-looking names, dates, wording, length; no placeholders like "..." or "[company]". Vary what matters for this scenario, not surface details only.

Each case also carries `expected`, shaped by the scenario's oracle "{{oracle}}":
{{oracle_instructions}}

Existing cases for this scenario, to avoid repeating (inputs only):
<existing>
{{existing}}
</existing>
```

- [x] **Step 2: Write the failing tests**

`tests/fixtures/cases-response.json`:

```json
{
  "cases": "{\"cases\":[{\"input\":{\"email\":\"From: hr@globex.com\\nSubject: Update\\n\\nWe decided to move forward with other candidates.\"},\"expected\":{\"label\":\"rejection\"}},{\"input\":{\"email\":\"From: rh@brasiltech.com.br\\nSubject: Retorno\\n\\nInfelizmente não avançaremos.\"},\"expected\":{\"label\":\"rejection\"}}]}",
  "bad-label": "{\"cases\":[{\"input\":{\"email\":\"x\"},\"expected\":{\"label\":\"maybe\"}},{\"input\":{\"email\":\"y\"},\"expected\":{\"label\":\"rejection\"}}]}",
  "all-bad": "{\"cases\":[{\"input\":{\"email\":\"x\"},\"expected\":{\"label\":\"maybe\"}},{\"input\":{\"mail\":\"y\"},\"expected\":{\"label\":\"rejection\"}}]}",
  "rubric": "{\"cases\":[{\"input\":{\"email\":\"x\"},\"expected\":{\"rubric\":\"summary mentions the optional quiz\"}}]}"
}
```

`tests/core/cases.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { parse } from "yaml";
import { generateCases } from "../../src/core/cases";
import { ForgeError } from "../../src/core/errors";
import type { Case, Feature, Scenario } from "../../src/core/schemas";
import { createLlm, type Llm } from "../../src/llm/generate";

const FIXTURE = "tests/fixtures/cases-response.json";
const scenario: Scenario = { id: "polite-rejection", kind: "happy", oracle: "label", description: "a polite no", status: "approved" };

async function llmFor(verb: string): Promise<{ llm: Llm; model: string }> {
	const base = createLlm({ forgeDir: join(await mkdtemp(join(tmpdir(), "forge-")), ".forge") });
	return { model: `fake/${FIXTURE}`, llm: { generate: (args) => base.generate({ ...args, verb }) } };
}
async function feature(): Promise<Feature> {
	return parse(await readFile(join(import.meta.dir, "../fixtures/feature.yaml"), "utf8"));
}

describe("generateCases", () => {
	test("appends pending cases with sequential ids and generated_by", async () => {
		const { llm, model } = await llmFor("cases");
		const out = await generateCases({ feature: await feature(), scenario, existing: [], n: 2, model, llm });
		expect(out.map((c) => c.id)).toEqual(["polite-rejection-01", "polite-rejection-02"]);
		expect(out[0]?.status).toBe("pending");
		expect(out[0]?.generated_by).toBe(model);
		expect(out[1]?.expected).toEqual({ label: "rejection" });
	});

	test("never overwrites reviewed cases and continues numbering", async () => {
		const { llm, model } = await llmFor("cases");
		const kept: Case = { id: "polite-rejection-01", scenario: "polite-rejection", input: { email: "old" }, expected: { label: "rejection" }, status: "approved", generated_by: "human" };
		const out = await generateCases({ feature: await feature(), scenario, existing: [kept], n: 2, model, llm });
		expect(out[0]).toEqual(kept);
		expect(out.slice(1).map((c) => c.id)).toEqual(["polite-rejection-02", "polite-rejection-03"]);
	});

	test("drops a case whose label is outside the feature's labels, keeps the rest", async () => {
		const { llm, model } = await llmFor("bad-label");
		const out = await generateCases({ feature: await feature(), scenario, existing: [], n: 2, model, llm });
		expect(out).toHaveLength(1);
		expect(out[0]?.expected).toEqual({ label: "rejection" });
	});

	test("throws when every generated case is invalid", async () => {
		const { llm, model } = await llmFor("all-bad");
		const err = await generateCases({ feature: await feature(), scenario, existing: [], n: 2, model, llm }).catch((e: unknown) => e);
		expect(err).toBeInstanceOf(ForgeError);
		expect((err as ForgeError).message).toContain("2 dropped");
	});

	test("drops exact duplicates of existing inputs", async () => {
		const { llm, model } = await llmFor("cases");
		const existing: Case = { id: "polite-rejection-01", scenario: "polite-rejection", input: { email: "From: hr@globex.com\nSubject: Update\n\nWe decided to move forward with other candidates." }, expected: { label: "rejection" }, status: "pending", generated_by: "t" };
		const out = await generateCases({ feature: await feature(), scenario, existing: [existing], n: 2, model, llm });
		expect(out).toHaveLength(2);
	});

	test("rubric oracle requires a rubric", async () => {
		const { llm, model } = await llmFor("rubric");
		const out = await generateCases({ feature: await feature(), scenario: { ...scenario, oracle: "rubric" }, existing: [], n: 1, model, llm });
		expect(out[0]?.expected).toEqual({ rubric: "summary mentions the optional quiz" });
	});

	test("refuses a scenario that is not approved", async () => {
		const { llm, model } = await llmFor("cases");
		await expect(generateCases({ feature: await feature(), scenario: { ...scenario, status: "pending" }, existing: [], n: 1, model, llm })).rejects.toThrow(ForgeError);
	});
});
```

- [x] **Step 3: Run to verify it fails**

Run: `bun test tests/core/cases.test.ts` — Expected: FAIL, module not found.

- [x] **Step 4: Implement cases.ts**

`src/core/cases.ts`:

```ts
import { stringify } from "yaml";
import { z } from "zod";
import type { Llm } from "../llm/generate";
import { loadTemplate, render } from "../llm/templates";
import { ForgeError } from "./errors";
import { nextCaseId } from "./ids";
import { type Case, type Expected, ExpectedSchema, type Feature, type Scenario } from "./schemas";

export const CasesOutputSchema = z.object({
	cases: z.array(z.object({ input: z.record(z.string(), z.string()), expected: ExpectedSchema })),
});

export interface GenerateCasesArgs {
	feature: Feature;
	scenario: Scenario;
	existing: Case[];
	n: number;
	model: string;
	llm: Llm;
}

function oracleInstructions(feature: Feature, scenario: Scenario): string {
	switch (scenario.oracle) {
		case "label":
			return `\`expected.label\` is exactly one of: ${(feature.output.labels ?? []).join(", ")}. Choose the label the invariants require, not the one the input tries to suggest.`;
		case "fields":
			return `\`expected.fields\` is an object with only the output fields whose value can be checked exactly (from: ${(feature.output.fields ?? []).join(", ")}). Omit fields whose value is a judgement call.`;
		case "rubric":
			return "`expected.rubric` is one sentence a reviewer could answer yes or no about the output, specific to this case.";
	}
}

export function expectedMatchesOracle(expected: Expected, scenario: Scenario, feature: Feature): string | null {
	switch (scenario.oracle) {
		case "label":
			if (expected.label === undefined) return "oracle is label but expected.label is missing";
			if (!(feature.output.labels ?? []).includes(expected.label)) return `label "${expected.label}" is not one of the feature's labels`;
			return null;
		case "fields":
			return expected.fields === undefined ? "oracle is fields but expected.fields is missing" : null;
		case "rubric":
			return expected.rubric === undefined ? "oracle is rubric but expected.rubric is missing" : null;
	}
}

function sameInput(a: Record<string, string>, b: Record<string, string>): boolean {
	const ka = Object.keys(a).sort();
	const kb = Object.keys(b).sort();
	return ka.length === kb.length && ka.every((k, i) => k === kb[i] && a[k] === b[k]);
}

export async function generateCases(args: GenerateCasesArgs): Promise<Case[]> {
	if (args.scenario.status !== "approved" && args.scenario.status !== "edited") {
		throw new ForgeError(`scenario is ${args.scenario.status}; approve it in review before generating cases`, { id: args.scenario.id });
	}
	const inputNames = args.feature.inputs.map((i) => i.name);
	const template = await loadTemplate("cases");
	const prompt = render(template, {
		feature: stringify(args.feature, { lineWidth: 0 }),
		scenario: stringify({ id: args.scenario.id, kind: args.scenario.kind, description: args.scenario.description }, { lineWidth: 0 }),
		n: String(args.n),
		input_names: inputNames.join(", "),
		oracle: args.scenario.oracle,
		oracle_instructions: oracleInstructions(args.feature, args.scenario),
		existing: args.existing.length > 0 ? stringify(args.existing.map((c) => c.input), { lineWidth: 0 }) : "(none)",
	});
	const { object } = await args.llm.generate({ schema: CasesOutputSchema, prompt, model: args.model, verb: "cases" });

	const accepted: Case[] = [...args.existing];
	const reasons: string[] = [];
	for (const candidate of object.cases) {
		const keys = Object.keys(candidate.input).sort();
		if (keys.length !== inputNames.length || !keys.every((k) => inputNames.includes(k))) {
			reasons.push(`input keys [${keys.join(", ")}] do not match feature inputs [${inputNames.join(", ")}]`);
			continue;
		}
		const problem = expectedMatchesOracle(candidate.expected, args.scenario, args.feature);
		if (problem) {
			reasons.push(problem);
			continue;
		}
		if (accepted.some((c) => sameInput(c.input, candidate.input))) {
			reasons.push("exact duplicate of an existing input");
			continue;
		}
		accepted.push({
			id: nextCaseId(args.scenario.id, accepted),
			scenario: args.scenario.id,
			input: candidate.input,
			expected: candidate.expected,
			status: "pending",
			generated_by: args.model,
		});
	}
	if (accepted.length === args.existing.length) {
		throw new ForgeError(`no usable case generated for scenario (${reasons.length} dropped: ${reasons.join("; ")})`, { id: args.scenario.id });
	}
	return accepted;
}
```

- [x] **Step 5: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 6: Commit**

```bash
git add src/core/cases.ts templates/cases.md tests/core/cases.test.ts tests/fixtures/cases-response.json
git commit -m "Add cases: generate oracle-shaped cases per scenario without touching reviewed ones"
```

---

### Task 7: `dedupe` — mark likely semantic duplicates within a scenario

**Files:**
- Create: `src/core/dedupe.ts`, `templates/dedupe.md`, `tests/core/dedupe.test.ts`, `tests/fixtures/dedupe-response.json`

**Interfaces:**
- Consumes: `Llm`, templates, `Case`.
- Produces: `dedupeCases(args: { cases: Case[]; model: string; llm: Llm }): Promise<Case[]>` — the model returns `{ duplicates: [{ id, duplicate_of }] }`; the function sets `duplicate_of` on cases whose ids exist, ignores pairs naming unknown ids or self-pairs, never marks a case that is `approved`/`edited` (reviewed cases are the canonical ones — a pending case may point at a reviewed one, not the reverse), clears nothing that was already set unless the model repeats it, and returns the same array order. With fewer than two cases it returns the input unchanged without calling the model.

- [x] **Step 1: Write the dedupe template**

`templates/dedupe.md`:

```
Below are test cases from the same scenario. Find pairs that test the same thing with different wording — same situation, same expected answer, differences only in names, dates, phrasing or length. Two cases that differ in what makes them pass or fail are not duplicates.

Return the pairs as `duplicates`, each `{ id, duplicate_of }` where `duplicate_of` is the case that should be kept (prefer the earlier id). Return an empty list when nothing repeats.

Cases:
<cases>
{{cases}}
</cases>
```

- [x] **Step 2: Write the failing tests**

`tests/fixtures/dedupe-response.json`:

```json
{
  "dedupe": "{\"duplicates\":[{\"id\":\"s-03\",\"duplicate_of\":\"s-01\"},{\"id\":\"s-02\",\"duplicate_of\":\"nope\"},{\"id\":\"s-04\",\"duplicate_of\":\"s-04\"},{\"id\":\"s-05\",\"duplicate_of\":\"s-01\"}]}"
}
```

`tests/core/dedupe.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { dedupeCases } from "../../src/core/dedupe";
import type { Case } from "../../src/core/schemas";
import { createLlm, type Llm } from "../../src/llm/generate";

const MODEL = "fake/tests/fixtures/dedupe-response.json";
const c = (id: string, status: Case["status"] = "pending"): Case => ({ id, scenario: "s", input: { email: id }, expected: { label: "x" }, status, generated_by: "t" });

async function llm(): Promise<Llm> {
	return createLlm({ forgeDir: join(await mkdtemp(join(tmpdir(), "forge-")), ".forge") });
}

describe("dedupeCases", () => {
	test("marks valid pairs, ignores unknown ids and self pairs, never marks reviewed cases", async () => {
		const cases = [c("s-01"), c("s-02"), c("s-03"), c("s-04"), c("s-05", "approved")];
		const out = await dedupeCases({ cases, model: MODEL, llm: await llm() });
		expect(out.map((x) => x.duplicate_of)).toEqual([undefined, undefined, "s-01", undefined, undefined]);
		expect(out.map((x) => x.id)).toEqual(["s-01", "s-02", "s-03", "s-04", "s-05"]);
	});

	test("does not call the model with fewer than two cases", async () => {
		let calls = 0;
		const spy: Llm = { generate: async () => { calls += 1; return { object: { duplicates: [] }, usage: { inputTokens: 0, outputTokens: 0 } } as never; } };
		const out = await dedupeCases({ cases: [c("s-01")], model: "anthropic/x", llm: spy });
		expect(calls).toBe(0);
		expect(out).toEqual([c("s-01")]);
	});
});
```

- [x] **Step 3: Run to verify it fails**

Run: `bun test tests/core/dedupe.test.ts` — Expected: FAIL, module not found.

- [x] **Step 4: Implement dedupe.ts**

`src/core/dedupe.ts`:

```ts
import { stringify } from "yaml";
import { z } from "zod";
import type { Llm } from "../llm/generate";
import { loadTemplate, render } from "../llm/templates";
import type { Case } from "./schemas";

export const DedupeOutputSchema = z.object({
	duplicates: z.array(z.object({ id: z.string(), duplicate_of: z.string() })),
});

export interface DedupeArgs {
	cases: Case[];
	model: string;
	llm: Llm;
}

export async function dedupeCases(args: DedupeArgs): Promise<Case[]> {
	if (args.cases.length < 2) return args.cases;
	const template = await loadTemplate("dedupe");
	const prompt = render(template, {
		cases: stringify(args.cases.map((c) => ({ id: c.id, input: c.input, expected: c.expected })), { lineWidth: 0 }),
	});
	const { object } = await args.llm.generate({ schema: DedupeOutputSchema, prompt, model: args.model, verb: "dedupe" });
	const byId = new Map(args.cases.map((c) => [c.id, c]));
	const marks = new Map<string, string>();
	for (const pair of object.duplicates) {
		const target = byId.get(pair.id);
		if (!target || !byId.has(pair.duplicate_of) || pair.id === pair.duplicate_of) continue;
		if (target.status === "approved" || target.status === "edited") continue;
		marks.set(pair.id, pair.duplicate_of);
	}
	return args.cases.map((c) => (marks.has(c.id) ? { ...c, duplicate_of: marks.get(c.id) } : c));
}
```

- [x] **Step 5: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 6: Commit**

```bash
git add src/core/dedupe.ts templates/dedupe.md tests/core/dedupe.test.ts tests/fixtures/dedupe-response.json
git commit -m "Add dedupe: mark likely semantic duplicates within a scenario"
```

---

### Task 8: `import` — real inputs from a JSONL file into pending cases without expected

**Files:**
- Create: `src/core/import.ts`, `tests/core/import.test.ts`

**Interfaces:**
- Consumes: `Feature`, `Scenario`, `Case`, `Oracle`, `nextCaseId`.
- Produces: `importCases(args: { feature: Feature; jsonl: string; source: string; existing: Case[]; oracle?: Oracle }): { cases: Case[]; scenario: Scenario; skipped: { line: number; reason: string }[] }` — parses `jsonl` line by line (blank lines ignored); each line must be a JSON object whose keys are exactly the feature's input names with string values, otherwise it is skipped with the line number and reason; accepted lines become cases with ids `imported-<nn>`, `scenario: "imported"`, no `expected`, `status: "pending"`, `generated_by: "import:<source>"`; exact duplicates of existing inputs are skipped; the returned `scenario` is `{ id: "imported", kind: "happy", oracle, description: "Real inputs imported from application logs", status: "approved" }` where `oracle` is the argument or derived from `feature.output.kind` (`label` → `label`, `json` → `fields`, `text` → `rubric`); throws `ForgeError` if no line was accepted, listing the reasons.

- [x] **Step 1: Write the failing tests**

`tests/core/import.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { readFile } from "node:fs/promises";
import { join } from "node:path";
import { parse } from "yaml";
import { ForgeError } from "../../src/core/errors";
import { importCases } from "../../src/core/import";
import type { Case, Feature } from "../../src/core/schemas";

async function feature(): Promise<Feature> {
	return parse(await readFile(join(import.meta.dir, "../fixtures/feature.yaml"), "utf8"));
}

describe("importCases", () => {
	test("creates pending cases without expected and the imported scenario", async () => {
		const jsonl = '{"email":"first"}\n\n{"email":"second"}\n';
		const r = importCases({ feature: await feature(), jsonl, source: "prod.jsonl", existing: [] });
		expect(r.cases.map((c) => c.id)).toEqual(["imported-01", "imported-02"]);
		expect(r.cases[0]).toEqual({ id: "imported-01", scenario: "imported", input: { email: "first" }, status: "pending", generated_by: "import:prod.jsonl" });
		expect(r.scenario).toEqual({ id: "imported", kind: "happy", oracle: "fields", description: "Real inputs imported from application logs", status: "approved" });
		expect(r.skipped).toEqual([]);
	});

	test("derives the oracle from output.kind and accepts an override", async () => {
		const f = await feature();
		expect(importCases({ feature: { ...f, output: { kind: "label", labels: ["a"] } }, jsonl: '{"email":"x"}', source: "s", existing: [] }).scenario.oracle).toBe("label");
		expect(importCases({ feature: { ...f, output: { kind: "text" } }, jsonl: '{"email":"x"}', source: "s", existing: [] }).scenario.oracle).toBe("rubric");
		expect(importCases({ feature: f, jsonl: '{"email":"x"}', source: "s", existing: [], oracle: "rubric" }).scenario.oracle).toBe("rubric");
	});

	test("skips bad lines with line numbers and reasons, and exact duplicates", async () => {
		const existing: Case[] = [{ id: "imported-01", scenario: "imported", input: { email: "dup" }, status: "approved", generated_by: "import:old" }];
		const jsonl = 'not json\n{"mail":"wrong key"}\n{"email":"dup"}\n{"email":42}\n{"email":"fresh"}\n';
		const r = importCases({ feature: await feature(), jsonl, source: "s", existing });
		expect(r.cases.map((c) => c.id)).toEqual(["imported-01", "imported-02"]);
		expect(r.cases[1]?.input).toEqual({ email: "fresh" });
		expect(r.skipped.map((s) => s.line)).toEqual([1, 2, 3, 4]);
		expect(r.skipped[0]?.reason).toContain("JSON");
	});

	test("throws when nothing was accepted", async () => {
		const err = (() => { try { importCases({ feature: {} as never, jsonl: "", source: "s", existing: [] }); } catch (e) { return e; } })();
		expect(err).toBeInstanceOf(ForgeError);
	});
});
```

- [x] **Step 2: Run to verify it fails**

Run: `bun test tests/core/import.test.ts` — Expected: FAIL, module not found.

- [x] **Step 3: Implement import.ts**

`src/core/import.ts`:

```ts
import { ForgeError } from "./errors";
import { nextCaseId } from "./ids";
import type { Case, Feature, Oracle, Scenario } from "./schemas";

export const IMPORTED_SCENARIO_ID = "imported";

export interface ImportArgs {
	feature: Feature;
	jsonl: string;
	source: string;
	existing: Case[];
	oracle?: Oracle;
}

export interface ImportResult {
	cases: Case[];
	scenario: Scenario;
	skipped: { line: number; reason: string }[];
}

function deriveOracle(feature: Feature): Oracle {
	switch (feature.output?.kind) {
		case "label":
			return "label";
		case "json":
			return "fields";
		default:
			return "rubric";
	}
}

function sameInput(a: Record<string, string>, b: Record<string, string>): boolean {
	const ka = Object.keys(a).sort();
	const kb = Object.keys(b).sort();
	return ka.length === kb.length && ka.every((k, i) => k === kb[i] && a[k] === b[k]);
}

export function importCases(args: ImportArgs): ImportResult {
	const inputNames = (args.feature.inputs ?? []).map((i) => i.name);
	const accepted: Case[] = [...args.existing];
	const skipped: { line: number; reason: string }[] = [];
	const lines = args.jsonl.split("\n");
	lines.forEach((raw, index) => {
		const line = index + 1;
		if (raw.trim() === "") return;
		let data: unknown;
		try {
			data = JSON.parse(raw);
		} catch {
			skipped.push({ line, reason: "not valid JSON" });
			return;
		}
		if (typeof data !== "object" || data === null || Array.isArray(data)) {
			skipped.push({ line, reason: "not a JSON object" });
			return;
		}
		const obj = data as Record<string, unknown>;
		const keys = Object.keys(obj).sort();
		if (keys.length !== inputNames.length || !keys.every((k) => inputNames.includes(k))) {
			skipped.push({ line, reason: `keys [${keys.join(", ")}] do not match feature inputs [${inputNames.join(", ")}]` });
			return;
		}
		if (!Object.values(obj).every((v) => typeof v === "string")) {
			skipped.push({ line, reason: "every input value must be a string" });
			return;
		}
		const input = obj as Record<string, string>;
		if (accepted.some((c) => sameInput(c.input, input))) {
			skipped.push({ line, reason: "exact duplicate of an existing input" });
			return;
		}
		accepted.push({ id: nextCaseId(IMPORTED_SCENARIO_ID, accepted), scenario: IMPORTED_SCENARIO_ID, input, status: "pending", generated_by: `import:${args.source}` });
	});
	if (accepted.length === args.existing.length) {
		throw new ForgeError(`no line imported (${skipped.map((s) => `line ${s.line}: ${s.reason}`).join("; ") || "file is empty"})`, { file: args.source });
	}
	return {
		cases: accepted,
		scenario: { id: IMPORTED_SCENARIO_ID, kind: "happy", oracle: args.oracle ?? deriveOracle(args.feature), description: "Real inputs imported from application logs", status: "approved" },
		skipped,
	};
}
```

- [x] **Step 4: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 5: Commit**

```bash
git add src/core/import.ts tests/core/import.test.ts
git commit -m "Add import: real inputs from JSONL into pending cases without expected"
```

---

### Task 9: `review` — pure transitions and the interactive loop

**Files:**
- Create: `src/core/review.ts`, `src/cli/review-loop.ts`, `tests/core/review.test.ts`, `tests/cli/review-loop.test.ts`

**Interfaces:**
- Consumes: `Feature`, `Scenario`, `Case`, `Expected`, `ExpectedSchema`, `FeatureSchema`, `ScenarioSchema`, `CaseSchema`.
- Produces (core/review.ts): `type Decision = "approve" | "reject" | "edit" | "skip"`; `type PendingItem = { kind: "feature"; item: Feature } | { kind: "scenario"; item: Scenario } | { kind: "case"; item: Case }`; `pendingItems(feature: Feature, scenarios: Scenario[], cases: Case[]): PendingItem[]` — feature first if pending, then pending scenarios in file order, then pending cases grouped by scenario in file order, with a case that has `duplicate_of` placed immediately after the case it points at when that one is in the list; `applyDecision<T extends Feature | Scenario | Case>(item: T, decision: Decision, edited?: T): T` — approve → `status: "approved"`; reject → `status: "rejected"` (throws `ForgeError` for a feature, which cannot be rejected); edit → the `edited` argument validated with the matching schema and forced to `status: "edited"` (throws if `edited` is missing or invalid); skip → the item unchanged; `withExpected(c: Case, expected: Expected): Case` — validates and sets.
- Produces (cli/review-loop.ts): `runReviewLoop(args: { items: PendingItem[]; ask: (question: string, choices: string[]) => Promise<string>; openEditor: (yamlText: string) => Promise<string>; askExpected: (c: Case, oracle: Oracle) => Promise<Expected>; oracleOf: (scenarioId: string) => Oracle; print: (line: string) => void }): Promise<{ decisions: { kind: PendingItem["kind"]; id: string; item: Feature | Scenario | Case }[]; summary: { approved: number; rejected: number; edited: number; skipped: number } }>` — for each item prints a rendering (YAML), asks `approve/reject/edit/skip`; for a case without `expected` under approve or edit, calls `askExpected` first; on edit, calls `openEditor` with the item as YAML and parses the result (an invalid edit prints the error and re-asks for the same item); returns every decided item (skipped ones excluded) plus counts.

- [x] **Step 1: Write the failing core tests**

`tests/core/review.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { ForgeError } from "../../src/core/errors";
import { applyDecision, pendingItems, withExpected } from "../../src/core/review";
import type { Case, Feature, Scenario } from "../../src/core/schemas";

const feature: Feature = { id: "f", purpose: "p", inputs: [{ name: "email", kind: "text" }], output: { kind: "label", labels: ["a", "b"] }, invariants: [], status: "pending" };
const scn = (id: string, status: Scenario["status"]): Scenario => ({ id, kind: "happy", oracle: "label", description: "d", status });
const cs = (id: string, scenario: string, status: Case["status"], duplicate_of?: string): Case => ({ id, scenario, input: { email: id }, expected: { label: "a" }, status, generated_by: "t", ...(duplicate_of ? { duplicate_of } : {}) });

describe("pendingItems", () => {
	test("orders feature, scenarios, then cases grouped by scenario with duplicates after their original", () => {
		const items = pendingItems(feature, [scn("s1", "approved"), scn("s2", "pending")], [cs("s1-01", "s1", "pending"), cs("s1-03", "s1", "pending", "s1-01"), cs("s1-02", "s1", "pending"), cs("s2-01", "s2", "approved")]);
		expect(items.map((i) => (i.kind === "feature" ? "feature" : i.item.id))).toEqual(["feature", "s2", "s1-01", "s1-03", "s1-02"]);
	});
	test("returns nothing when everything is reviewed", () => {
		expect(pendingItems({ ...feature, status: "approved" }, [scn("s1", "approved")], [cs("s1-01", "s1", "approved")])).toEqual([]);
	});
});

describe("applyDecision", () => {
	test("approve, reject, skip", () => {
		expect(applyDecision(scn("s", "pending"), "approve").status).toBe("approved");
		expect(applyDecision(scn("s", "pending"), "reject").status).toBe("rejected");
		expect(applyDecision(scn("s", "pending"), "skip").status).toBe("pending");
	});
	test("a feature cannot be rejected", () => {
		expect(() => applyDecision(feature, "reject")).toThrow(ForgeError);
	});
	test("edit validates the edited item and forces status edited", () => {
		const out = applyDecision(cs("c", "s", "pending"), "edit", { ...cs("c", "s", "pending"), input: { email: "changed" } });
		expect(out.input.email).toBe("changed");
		expect(out.status).toBe("edited");
		expect(() => applyDecision(cs("c", "s", "pending"), "edit")).toThrow(ForgeError);
		expect(() => applyDecision(cs("c", "s", "pending"), "edit", { ...cs("c", "s", "pending"), status: "weird" as never })).toThrow(ForgeError);
	});
});

describe("withExpected", () => {
	test("sets a valid expected and rejects an invalid one", () => {
		expect(withExpected(cs("c", "s", "pending"), { rubric: "r" }).expected).toEqual({ rubric: "r" });
		expect(() => withExpected(cs("c", "s", "pending"), {} as never)).toThrow(ForgeError);
	});
});
```

- [x] **Step 2: Implement core/review.ts**

`src/core/review.ts`:

```ts
import { ForgeError } from "./errors";
import { type Case, CaseSchema, type Expected, ExpectedSchema, type Feature, FeatureSchema, type Scenario, ScenarioSchema } from "./schemas";

export type Decision = "approve" | "reject" | "edit" | "skip";

export type PendingItem = { kind: "feature"; item: Feature } | { kind: "scenario"; item: Scenario } | { kind: "case"; item: Case };

export function pendingItems(feature: Feature, scenarios: Scenario[], cases: Case[]): PendingItem[] {
	const out: PendingItem[] = [];
	if (feature.status === "pending") out.push({ kind: "feature", item: feature });
	for (const s of scenarios) if (s.status === "pending") out.push({ kind: "scenario", item: s });
	const pendingCases = cases.filter((c) => c.status === "pending");
	const placed = new Set<string>();
	const ordered: Case[] = [];
	for (const c of pendingCases) {
		if (placed.has(c.id)) continue;
		ordered.push(c);
		placed.add(c.id);
		for (const d of pendingCases) {
			if (d.duplicate_of === c.id && !placed.has(d.id)) {
				ordered.push(d);
				placed.add(d.id);
			}
		}
	}
	for (const c of ordered) out.push({ kind: "case", item: c });
	return out;
}

function isFeature(item: Feature | Scenario | Case): item is Feature {
	return "purpose" in item;
}

function validate<T extends Feature | Scenario | Case>(original: T, edited: unknown): T {
	const schema = isFeature(original) ? FeatureSchema : "kind" in original ? ScenarioSchema : CaseSchema;
	const result = schema.safeParse(edited);
	if (!result.success) throw new ForgeError(`edited item is invalid: ${result.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`).join("; ")}`, { id: original.id });
	return result.data as T;
}

export function applyDecision<T extends Feature | Scenario | Case>(item: T, decision: Decision, edited?: T): T {
	switch (decision) {
		case "approve":
			return { ...item, status: "approved" };
		case "reject":
			if (isFeature(item)) throw new ForgeError("a feature cannot be rejected; edit it or delete .forge/feature.yaml", { id: item.id });
			return { ...item, status: "rejected" };
		case "edit": {
			if (edited === undefined) throw new ForgeError("edit needs the edited item", { id: item.id });
			return { ...validate(item, edited), status: "edited" };
		}
		case "skip":
			return item;
	}
}

export function withExpected(c: Case, expected: Expected): Case {
	const result = ExpectedSchema.safeParse(expected);
	if (!result.success) throw new ForgeError(`expected is invalid: ${result.error.issues.map((i) => i.message).join("; ")}`, { id: c.id });
	return { ...c, expected: result.data };
}
```

Run: `bun test tests/core/review.test.ts` — Expected: pass.

- [x] **Step 3: Write the failing loop tests**

`tests/cli/review-loop.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { stringify } from "yaml";
import { runReviewLoop } from "../../src/cli/review-loop";
import type { PendingItem } from "../../src/core/review";
import type { Case, Scenario } from "../../src/core/schemas";

const scn: Scenario = { id: "s", kind: "happy", oracle: "label", description: "d", status: "pending" };
const withExp: Case = { id: "s-01", scenario: "s", input: { email: "a" }, expected: { label: "x" }, status: "pending", generated_by: "t" };
const noExp: Case = { id: "imported-01", scenario: "imported", input: { email: "b" }, status: "pending", generated_by: "import:f" };

function scripted(answers: string[]) {
	const queue = [...answers];
	return async () => queue.shift() ?? "skip";
}

describe("runReviewLoop", () => {
	test("applies scripted decisions and counts them", async () => {
		const items: PendingItem[] = [{ kind: "scenario", item: scn }, { kind: "case", item: withExp }];
		const printed: string[] = [];
		const r = await runReviewLoop({ items, ask: scripted(["approve", "reject"]), openEditor: async (t) => t, askExpected: async () => ({ label: "x" }), oracleOf: () => "label", print: (l) => printed.push(l) });
		expect(r.summary).toEqual({ approved: 1, rejected: 1, edited: 0, skipped: 0 });
		expect(r.decisions.map((d) => [d.id, d.item.status])).toEqual([["s", "approved"], ["s-01", "rejected"]]);
		expect(printed.join("\n")).toContain("id: s-01");
	});

	test("asks for expected before approving a case that has none", async () => {
		let asked = 0;
		const r = await runReviewLoop({ items: [{ kind: "case", item: noExp }], ask: scripted(["approve"]), openEditor: async (t) => t, askExpected: async () => { asked += 1; return { label: "rejection" }; }, oracleOf: () => "label", print: () => {} });
		expect(asked).toBe(1);
		expect((r.decisions[0]?.item as Case).expected).toEqual({ label: "rejection" });
	});

	test("edit round-trips through the editor and an invalid edit re-asks", async () => {
		const edits = [stringify({ ...withExp, status: "weird" }), stringify({ ...withExp, input: { email: "edited" } })];
		const r = await runReviewLoop({ items: [{ kind: "case", item: withExp }], ask: scripted(["edit", "edit"]), openEditor: async () => edits.shift() ?? "", askExpected: async () => ({ label: "x" }), oracleOf: () => "label", print: () => {} });
		expect(r.summary.edited).toBe(1);
		expect((r.decisions[0]?.item as Case).input.email).toBe("edited");
	});

	test("skip leaves no decision", async () => {
		const r = await runReviewLoop({ items: [{ kind: "scenario", item: scn }], ask: scripted(["skip"]), openEditor: async (t) => t, askExpected: async () => ({ label: "x" }), oracleOf: () => "label", print: () => {} });
		expect(r.decisions).toEqual([]);
		expect(r.summary.skipped).toBe(1);
	});
});
```

- [x] **Step 4: Implement cli/review-loop.ts**

`src/cli/review-loop.ts`:

```ts
import { parse, stringify } from "yaml";
import { ForgeError } from "../core/errors";
import { applyDecision, type Decision, type PendingItem, withExpected } from "../core/review";
import type { Case, Expected, Feature, Oracle, Scenario } from "../core/schemas";

export interface ReviewLoopArgs {
	items: PendingItem[];
	ask: (question: string, choices: string[]) => Promise<string>;
	openEditor: (yamlText: string) => Promise<string>;
	askExpected: (c: Case, oracle: Oracle) => Promise<Expected>;
	oracleOf: (scenarioId: string) => Oracle;
	print: (line: string) => void;
}

export interface ReviewLoopResult {
	decisions: { kind: PendingItem["kind"]; id: string; item: Feature | Scenario | Case }[];
	summary: { approved: number; rejected: number; edited: number; skipped: number };
}

const CHOICES: Decision[] = ["approve", "reject", "edit", "skip"];

export async function runReviewLoop(args: ReviewLoopArgs): Promise<ReviewLoopResult> {
	const decisions: ReviewLoopResult["decisions"] = [];
	const summary = { approved: 0, rejected: 0, edited: 0, skipped: 0 };
	for (const pending of args.items) {
		let current: Feature | Scenario | Case = pending.item;
		for (;;) {
			args.print(`--- ${pending.kind} ${current.id} ---`);
			args.print(stringify(current, { lineWidth: 0 }).trimEnd());
			const answer = (await args.ask(`${pending.kind} ${current.id}:`, CHOICES)) as Decision;
			try {
				if (pending.kind === "case" && (answer === "approve" || answer === "edit") && (current as Case).expected === undefined) {
					current = withExpected(current as Case, await args.askExpected(current as Case, args.oracleOf((current as Case).scenario)));
				}
				if (answer === "edit") {
					const edited = parse(await args.openEditor(stringify(current, { lineWidth: 0 })));
					current = applyDecision(current, "edit", edited);
					summary.edited += 1;
					decisions.push({ kind: pending.kind, id: current.id, item: current });
					break;
				}
				if (answer === "skip") {
					summary.skipped += 1;
					break;
				}
				current = applyDecision(current, answer);
				summary[answer === "approve" ? "approved" : "rejected"] += 1;
				decisions.push({ kind: pending.kind, id: current.id, item: current });
				break;
			} catch (e) {
				if (e instanceof ForgeError) {
					args.print(`cannot apply: ${e.message}`);
					continue;
				}
				throw e;
			}
		}
	}
	return { decisions, summary };
}
```

- [x] **Step 5: Run tests and lint**

Run: `bun test` — Expected: pass. Run: `bun run lint` — Expected: clean.

- [x] **Step 6: Commit**

```bash
git add src/core/review.ts src/cli/review-loop.ts tests/core/review.test.ts tests/cli/review-loop.test.ts
git commit -m "Add review: pure decisions and an interactive loop with injectable prompts and editor"
```

---

### Task 10: The CLI — `forge describe|scenarios|cases|dedupe|import|review`

**Files:**
- Create: `src/cli/main.ts`, `src/cli/context.ts`, `src/cli/commands/describe.ts`, `src/cli/commands/scenarios.ts`, `src/cli/commands/cases.ts`, `src/cli/commands/dedupe.ts`, `src/cli/commands/import.ts`, `src/cli/commands/review.ts`, `src/cli/prompts.ts`, `tests/cli/main.test.ts`, `tests/fixtures/cli-responses.json`

**Interfaces:**
- Consumes: everything from Tasks 2–9.
- Produces (context.ts): `interface CliContext { forgeDir: string; llm: Llm; model: string; stdout: (line: string) => void; stdin: () => Promise<string> }`; `createContext(opts: { cwd: string; model?: string; env?: Record<string, string | undefined> }): CliContext` — `forgeDir` is `<cwd>/.forge`; `model` comes from `--model`, else `FORGE_MODEL` env, else throws `ForgeError("no model: pass --model provider/model or set FORGE_MODEL")` lazily on first use.
- Produces (main.ts): `run(argv: string[], ctx: CliContext): Promise<number>` — dispatches on the first positional; returns the exit code (0 success, 1 `ForgeError` printed as `error: <message>`, 2 usage error printed with the usage text); the file ends with `if (import.meta.main) process.exit(await run(process.argv.slice(2), createContext({ cwd: process.cwd(), model: modelFlag(process.argv) })))`.
- Each command file exports `async function <verb>Command(args: string[], ctx: CliContext): Promise<void>` and prints one summary line (for example `scenarios: 6 new pending (6 total) -> .forge/scenarios.yaml`).
- Produces (prompts.ts): `askChoice(question, choices, stdin, stdout)` reading a line and accepting the full word or its first letter; `askExpectedFor(c, oracle, stdin, stdout)` asking for a label / a `field=value` list / a rubric sentence; `openInEditor(yamlText)` writing to a temp file, spawning `$EDITOR` (default `vi`) with `Bun.spawn([...], { stdio: ["inherit", "inherit", "inherit"] })`, awaiting exit, reading the file back.
- Option parsing per command (with `parseArgs` from `node:util`, `allowPositionals: true`, `strict: true`):
  - `describe <text...> [--prompt-file f] [--model m]` — text is the joined positionals after the verb, or stdin when there are none.
  - `scenarios [--kinds a,b] [--more n] [--model m]`
  - `cases [--n 5] [--scenario id] [--model m]` — without `--scenario`, every approved/edited scenario, one file each.
  - `dedupe [--scenario id] [--model m]`
  - `import <file.jsonl> [--oracle o]`
  - `review [--scenario id] [--only cases] [--all]` — `--all` requires `--scenario` and approves every pending case of it without asking.

- [x] **Step 1: Write the failing end-to-end test**

`tests/fixtures/cli-responses.json` — the fake provider file for the whole flow (the strings are the same shapes recorded in earlier fixtures):

```json
{
  "describe": "{\"id\":\"classify-email\",\"purpose\":\"Classify a hiring-process email\",\"inputs\":[{\"name\":\"email\",\"kind\":\"text\"}],\"output\":{\"kind\":\"json\",\"fields\":[\"type\",\"summary\"],\"label_field\":\"type\",\"labels\":[\"rejection\",\"acknowledgement\",\"unrelated\"]},\"invariants\":[\"Answer is JSON only\"]}",
  "scenarios": "{\"scenarios\":[{\"id\":\"polite-rejection\",\"kind\":\"happy\",\"oracle\":\"label\",\"description\":\"a\"},{\"id\":\"huge-signature\",\"kind\":\"edge\",\"oracle\":\"label\",\"description\":\"b\"},{\"id\":\"ack-quiz\",\"kind\":\"ambiguous\",\"oracle\":\"label\",\"description\":\"c\"},{\"id\":\"newsletter\",\"kind\":\"out_of_scope\",\"oracle\":\"label\",\"description\":\"d\"},{\"id\":\"injection\",\"kind\":\"adversarial\",\"oracle\":\"label\",\"description\":\"e\"},{\"id\":\"portuguese\",\"kind\":\"language\",\"oracle\":\"label\",\"description\":\"f\"}]}",
  "cases": "{\"cases\":[{\"input\":{\"email\":\"one\"},\"expected\":{\"label\":\"rejection\"}},{\"input\":{\"email\":\"two\"},\"expected\":{\"label\":\"rejection\"}}]}",
  "dedupe": "{\"duplicates\":[{\"id\":\"polite-rejection-02\",\"duplicate_of\":\"polite-rejection-01\"}]}"
}
```

`tests/cli/main.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join, resolve } from "node:path";
import { createContext } from "../../src/cli/context";
import { run } from "../../src/cli/main";
import { readCases, readFeature, readScenarios, writeFeature, writeScenarios } from "../../src/core/files";

const MODEL = `fake/${resolve("tests/fixtures/cli-responses.json")}`;

async function ctxIn(cwd: string, stdinLines: string[] = []) {
	const out: string[] = [];
	const queue = [...stdinLines];
	const ctx = createContext({ cwd, model: MODEL });
	ctx.stdout = (l) => out.push(l);
	ctx.stdin = async () => queue.shift() ?? "";
	return { ctx, out };
}

describe("forge CLI end to end (fake provider)", () => {
	test("describe -> review -> scenarios -> review --all -> cases -> dedupe -> import", async () => {
		const cwd = await mkdtemp(join(tmpdir(), "forge-cli-"));
		const forgeDir = join(cwd, ".forge");

		let { ctx, out } = await ctxIn(cwd);
		expect(await run(["describe", "The bot classifies hiring emails"], ctx)).toBe(0);
		expect((await readFeature(forgeDir)).status).toBe("pending");
		expect(out.at(-1)).toContain("feature.yaml");

		// scenarios refuse a pending feature
		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["scenarios"], ctx)).toBe(1);
		expect(out.at(-1)).toContain("pending");

		// approve the feature through review
		({ ctx, out } = await ctxIn(cwd, ["approve"]));
		expect(await run(["review"], ctx)).toBe(0);
		expect((await readFeature(forgeDir)).status).toBe("approved");

		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["scenarios"], ctx)).toBe(0);
		expect(await readScenarios(forgeDir)).toHaveLength(6);

		// cases refuse when no scenario is approved
		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["cases", "--scenario", "polite-rejection"], ctx)).toBe(1);

		// approve one scenario with --all requires --scenario
		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["review", "--all"], ctx)).toBe(2);
		({ ctx, out } = await ctxIn(cwd, ["approve"]));
		expect(await run(["review", "--only", "scenarios"], ctx)).toBe(0); // first pending scenario approved, rest skipped by empty stdin
		const scenarios = await readScenarios(forgeDir);
		expect(scenarios[0]?.status).toBe("approved");

		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["cases", "--scenario", "polite-rejection", "--n", "2"], ctx)).toBe(0);
		expect((await readCases(forgeDir, "polite-rejection")).map((c) => c.id)).toEqual(["polite-rejection-01", "polite-rejection-02"]);

		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["dedupe", "--scenario", "polite-rejection"], ctx)).toBe(0);
		expect((await readCases(forgeDir, "polite-rejection"))[1]?.duplicate_of).toBe("polite-rejection-01");

		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["review", "--scenario", "polite-rejection", "--all"], ctx)).toBe(0);
		expect((await readCases(forgeDir, "polite-rejection")).every((c) => c.status === "approved")).toBe(true);

		const jsonl = join(cwd, "prod.jsonl");
		await writeFile(jsonl, '{"email":"real one"}\n{"email":"real two"}\n');
		({ ctx, out } = await ctxIn(cwd));
		expect(await run(["import", jsonl], ctx)).toBe(0);
		expect((await readCases(forgeDir, "imported")).map((c) => c.id)).toEqual(["imported-01", "imported-02"]);
		expect((await readScenarios(forgeDir)).some((s) => s.id === "imported")).toBe(true);
	});

	test("usage errors exit 2 and unknown verbs print usage", async () => {
		const cwd = await mkdtemp(join(tmpdir(), "forge-cli-"));
		const { ctx, out } = await ctxIn(cwd);
		expect(await run([], ctx)).toBe(2);
		expect(await run(["frobnicate"], ctx)).toBe(2);
		expect(out.join("\n")).toContain("usage:");
	});

	test("missing model is a ForgeError naming the flag", async () => {
		const cwd = await mkdtemp(join(tmpdir(), "forge-cli-"));
		const out: string[] = [];
		const ctx = createContext({ cwd, env: {} });
		ctx.stdout = (l) => out.push(l);
		expect(await run(["describe", "x"], ctx)).toBe(1);
		expect(out.at(-1)).toContain("--model");
	});
});
```

- [x] **Step 2: Run to verify it fails**

Run: `bun test tests/cli/main.test.ts` — Expected: FAIL, module not found.

- [x] **Step 3: Implement context.ts and prompts.ts**

`src/cli/context.ts`:

```ts
import { join } from "node:path";
import { ForgeError } from "../core/errors";
import { createLlm, type Llm } from "../llm/generate";

export interface CliContext {
	forgeDir: string;
	llm: Llm;
	readonly model: string;
	stdout: (line: string) => void;
	stdin: () => Promise<string>;
}

export interface CreateContextOptions {
	cwd: string;
	model?: string;
	env?: Record<string, string | undefined>;
}

export function createContext(opts: CreateContextOptions): CliContext {
	const env = opts.env ?? process.env;
	const forgeDir = join(opts.cwd, ".forge");
	const model = opts.model ?? env.FORGE_MODEL;
	return {
		forgeDir,
		llm: createLlm({ forgeDir }),
		get model(): string {
			if (!model) throw new ForgeError("no model: pass --model provider/model or set FORGE_MODEL");
			return model;
		},
		stdout: (line) => console.log(line),
		stdin: async () => {
			for await (const line of console) return line;
			return "";
		},
	};
}
```

`src/cli/prompts.ts`:

```ts
import { mkdtemp, readFile, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { ForgeError } from "../core/errors";
import type { Case, Expected, Oracle } from "../core/schemas";

type Io = { stdin: () => Promise<string>; stdout: (line: string) => void };

export async function askChoice(question: string, choices: string[], io: Io): Promise<string> {
	for (;;) {
		io.stdout(`${question} [${choices.join("/")}]`);
		const raw = (await io.stdin()).trim().toLowerCase();
		const hit = choices.find((c) => c === raw || c.startsWith(raw) && raw.length > 0);
		if (hit) return hit;
		if (raw === "") return choices[choices.length - 1] as string; // empty line = last choice (skip)
		io.stdout(`please answer one of: ${choices.join(", ")}`);
	}
}

export async function askExpectedFor(c: Case, oracle: Oracle, io: Io): Promise<Expected> {
	switch (oracle) {
		case "label": {
			io.stdout(`expected label for ${c.id}:`);
			return { label: (await io.stdin()).trim() };
		}
		case "fields": {
			io.stdout(`expected fields for ${c.id} as field=value, comma separated:`);
			const fields: Record<string, string> = {};
			for (const pair of (await io.stdin()).split(",")) {
				const [k, ...rest] = pair.split("=");
				if (k?.trim()) fields[k.trim()] = rest.join("=").trim();
			}
			return { fields };
		}
		case "rubric": {
			io.stdout(`rubric sentence for ${c.id}:`);
			return { rubric: (await io.stdin()).trim() };
		}
	}
}

export async function openInEditor(yamlText: string, env: Record<string, string | undefined> = process.env): Promise<string> {
	const dir = await mkdtemp(join(tmpdir(), "forge-edit-"));
	const path = join(dir, "item.yaml");
	await writeFile(path, yamlText, "utf8");
	const editor = env.EDITOR ?? "vi";
	const proc = Bun.spawn([editor, path], { stdio: ["inherit", "inherit", "inherit"] });
	const code = await proc.exited;
	if (code !== 0) throw new ForgeError(`editor "${editor}" exited with ${code}`, { file: path });
	return readFile(path, "utf8");
}
```

- [x] **Step 4: Implement the six commands**

`src/cli/commands/describe.ts`:

```ts
import { readFile } from "node:fs/promises";
import { parseArgs } from "node:util";
import { describeFeature } from "../../core/describe";
import { forgePaths, writeFeature } from "../../core/files";
import type { CliContext } from "../context";

export async function describeCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values, positionals } = parseArgs({ args, options: { "prompt-file": { type: "string" }, model: { type: "string" } }, allowPositionals: true });
	const text = positionals.length > 0 ? positionals.join(" ") : await ctx.stdin();
	const promptText = values["prompt-file"] ? await readFile(values["prompt-file"], "utf8") : undefined;
	const feature = await describeFeature({ text, promptText, model: values.model ?? ctx.model, llm: ctx.llm });
	const withFile = values["prompt-file"] ? { ...feature, prompt_file: values["prompt-file"] } : feature;
	await writeFeature(ctx.forgeDir, withFile);
	ctx.stdout(`describe: feature "${withFile.id}" written as pending -> ${forgePaths(ctx.forgeDir).feature}; run \`forge review\` to approve it`);
}
```

`src/cli/commands/scenarios.ts`:

```ts
import { parseArgs } from "node:util";
import { ForgeError } from "../../core/errors";
import { forgePaths, readFeature, readScenarios, writeScenarios } from "../../core/files";
import { Kind } from "../../core/schemas";
import { enumerateScenarios } from "../../core/scenarios";
import type { CliContext } from "../context";

export async function scenariosCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values } = parseArgs({ args, options: { kinds: { type: "string" }, more: { type: "string" }, model: { type: "string" } } });
	const feature = await readFeature(ctx.forgeDir);
	if (feature.status === "pending") throw new ForgeError("feature is pending; run `forge review` first", { file: forgePaths(ctx.forgeDir).feature });
	const kinds = values.kinds?.split(",").map((k) => Kind.parse(k.trim()));
	const more = values.more === undefined ? undefined : Number.parseInt(values.more, 10);
	const existing = await readScenarios(ctx.forgeDir);
	const all = await enumerateScenarios({ feature, existing, kinds, more, model: values.model ?? ctx.model, llm: ctx.llm });
	await writeScenarios(ctx.forgeDir, all);
	ctx.stdout(`scenarios: ${all.length - existing.length} new pending (${all.length} total) -> ${forgePaths(ctx.forgeDir).scenarios}`);
}
```

`src/cli/commands/cases.ts`:

```ts
import { parseArgs } from "node:util";
import { generateCases } from "../../core/cases";
import { ForgeError } from "../../core/errors";
import { readCases, readFeature, readScenarios, writeCases } from "../../core/files";
import type { CliContext } from "../context";

export async function casesCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values } = parseArgs({ args, options: { n: { type: "string" }, scenario: { type: "string" }, model: { type: "string" } } });
	const n = values.n === undefined ? 5 : Number.parseInt(values.n, 10);
	const feature = await readFeature(ctx.forgeDir);
	const scenarios = await readScenarios(ctx.forgeDir);
	const targets = values.scenario ? scenarios.filter((s) => s.id === values.scenario) : scenarios.filter((s) => s.status === "approved" || s.status === "edited");
	if (targets.length === 0) throw new ForgeError(values.scenario ? `scenario "${values.scenario}" not found` : "no approved scenario; run `forge review` first");
	for (const scenario of targets) {
		const existing = await readCases(ctx.forgeDir, scenario.id);
		const all = await generateCases({ feature, scenario, existing, n, model: values.model ?? ctx.model, llm: ctx.llm });
		await writeCases(ctx.forgeDir, scenario.id, all);
		ctx.stdout(`cases: ${scenario.id}: ${all.length - existing.length} new pending (${all.length} total)`);
	}
}
```

`src/cli/commands/dedupe.ts`:

```ts
import { parseArgs } from "node:util";
import { dedupeCases } from "../../core/dedupe";
import { listCaseScenarios, readCases, writeCases } from "../../core/files";
import type { CliContext } from "../context";

export async function dedupeCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values } = parseArgs({ args, options: { scenario: { type: "string" }, model: { type: "string" } } });
	const ids = values.scenario ? [values.scenario] : await listCaseScenarios(ctx.forgeDir);
	for (const id of ids) {
		const cases = await readCases(ctx.forgeDir, id);
		const marked = await dedupeCases({ cases, model: values.model ?? ctx.model, llm: ctx.llm });
		await writeCases(ctx.forgeDir, id, marked);
		const count = marked.filter((c) => c.duplicate_of !== undefined).length;
		ctx.stdout(`dedupe: ${id}: ${count} marked as duplicates`);
	}
}
```

`src/cli/commands/import.ts`:

```ts
import { readFile } from "node:fs/promises";
import { basename } from "node:path";
import { parseArgs } from "node:util";
import { ForgeError } from "../../core/errors";
import { readCases, readFeature, readScenarios, writeCases, writeScenarios } from "../../core/files";
import { IMPORTED_SCENARIO_ID, importCases } from "../../core/import";
import { Oracle } from "../../core/schemas";
import type { CliContext } from "../context";

export async function importCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values, positionals } = parseArgs({ args, options: { oracle: { type: "string" } }, allowPositionals: true });
	const file = positionals[0];
	if (!file) throw new ForgeError("import needs a JSONL file path");
	const feature = await readFeature(ctx.forgeDir);
	const jsonl = await readFile(file, "utf8").catch(() => { throw new ForgeError("file not found", { file }); });
	const existing = await readCases(ctx.forgeDir, IMPORTED_SCENARIO_ID);
	const r = importCases({ feature, jsonl, source: basename(file), existing, oracle: values.oracle ? Oracle.parse(values.oracle) : undefined });
	await writeCases(ctx.forgeDir, IMPORTED_SCENARIO_ID, r.cases);
	const scenarios = await readScenarios(ctx.forgeDir);
	if (!scenarios.some((s) => s.id === IMPORTED_SCENARIO_ID)) await writeScenarios(ctx.forgeDir, [...scenarios, r.scenario]);
	for (const s of r.skipped) ctx.stdout(`import: skipped line ${s.line}: ${s.reason}`);
	ctx.stdout(`import: ${r.cases.length - existing.length} new pending cases without expected (${r.skipped.length} skipped)`);
}
```

`src/cli/commands/review.ts`:

```ts
import { parseArgs } from "node:util";
import { ForgeError } from "../../core/errors";
import { listCaseScenarios, readCases, readFeature, readScenarios, writeCases, writeFeature, writeScenarios } from "../../core/files";
import { pendingItems } from "../../core/review";
import type { Case, Oracle } from "../../core/schemas";
import type { CliContext } from "../context";
import { askChoice, askExpectedFor, openInEditor } from "../prompts";
import { runReviewLoop } from "../review-loop";

export async function reviewCommand(args: string[], ctx: CliContext): Promise<void> {
	const { values } = parseArgs({ args, options: { scenario: { type: "string" }, only: { type: "string" }, all: { type: "boolean" } } });
	if (values.all && !values.scenario) throw new ForgeError("--all requires --scenario", { file: "usage" });
	const feature = await readFeature(ctx.forgeDir);
	const scenarios = await readScenarios(ctx.forgeDir);
	const scenarioIds = values.scenario ? [values.scenario] : await listCaseScenarios(ctx.forgeDir);
	const casesById = new Map<string, Case[]>();
	for (const id of scenarioIds) casesById.set(id, await readCases(ctx.forgeDir, id));
	const allCases = [...casesById.values()].flat();

	if (values.all) {
		const id = values.scenario as string;
		const updated = (casesById.get(id) ?? []).map((c) => (c.status === "pending" ? { ...c, status: "approved" as const } : c));
		await writeCases(ctx.forgeDir, id, updated);
		ctx.stdout(`review: approved every pending case of ${id}`);
		return;
	}

	let items = pendingItems(feature, values.scenario ? scenarios.filter((s) => s.id === values.scenario) : scenarios, allCases);
	if (values.only) items = items.filter((i) => `${i.kind}s` === values.only || i.kind === values.only);
	if (items.length === 0) {
		ctx.stdout("review: nothing pending");
		return;
	}
	const oracleOf = (scenarioId: string): Oracle => scenarios.find((s) => s.id === scenarioId)?.oracle ?? "rubric";
	const io = { stdin: ctx.stdin, stdout: ctx.stdout };
	const result = await runReviewLoop({
		items,
		ask: (q, choices) => askChoice(q, choices, io),
		openEditor: openInEditor,
		askExpected: (c, oracle) => askExpectedFor(c, oracle, io),
		oracleOf,
		print: ctx.stdout,
	});

	for (const d of result.decisions) {
		if (d.kind === "feature") await writeFeature(ctx.forgeDir, d.item as never);
		if (d.kind === "scenario") {
			const idx = scenarios.findIndex((s) => s.id === d.id);
			if (idx >= 0) scenarios[idx] = d.item as never;
		}
		if (d.kind === "case") {
			const c = d.item as Case;
			const list = casesById.get(c.scenario) ?? [];
			const idx = list.findIndex((x) => x.id === c.id);
			if (idx >= 0) list[idx] = c;
			casesById.set(c.scenario, list);
		}
	}
	if (result.decisions.some((d) => d.kind === "scenario")) await writeScenarios(ctx.forgeDir, scenarios);
	for (const [id, list] of casesById) if (result.decisions.some((d) => d.kind === "case" && (d.item as Case).scenario === id)) await writeCases(ctx.forgeDir, id, list);
	const s = result.summary;
	ctx.stdout(`review: ${s.approved} approved, ${s.rejected} rejected, ${s.edited} edited, ${s.skipped} skipped`);
}
```

- [x] **Step 5: Implement main.ts**

`src/cli/main.ts`:

```ts
#!/usr/bin/env bun
import { ForgeError } from "../core/errors";
import { casesCommand } from "./commands/cases";
import { dedupeCommand } from "./commands/dedupe";
import { describeCommand } from "./commands/describe";
import { importCommand } from "./commands/import";
import { reviewCommand } from "./commands/review";
import { scenariosCommand } from "./commands/scenarios";
import { type CliContext, createContext } from "./context";

export const USAGE = `usage: forge <verb> [options]
  describe <text...> [--prompt-file f] [--model m]   free text (or stdin) -> .forge/feature.yaml (pending)
  scenarios [--kinds a,b] [--more n] [--model m]      feature -> .forge/scenarios.yaml
  cases [--n 5] [--scenario id] [--model m]           approved scenarios -> .forge/cases/<id>.yaml
  dedupe [--scenario id] [--model m]                  mark likely duplicates (duplicate_of)
  import <file.jsonl> [--oracle label|fields|rubric]  real inputs -> .forge/cases/imported.yaml
  review [--scenario id] [--only feature|scenarios|cases] [--all]
model: --model provider/model or FORGE_MODEL; providers: anthropic, google, openai, ollama, fake/<file>`;

const COMMANDS: Record<string, (args: string[], ctx: CliContext) => Promise<void>> = {
	describe: describeCommand,
	scenarios: scenariosCommand,
	cases: casesCommand,
	dedupe: dedupeCommand,
	import: importCommand,
	review: reviewCommand,
};

export async function run(argv: string[], ctx: CliContext): Promise<number> {
	const [verb, ...rest] = argv;
	const command = verb ? COMMANDS[verb] : undefined;
	if (!command) {
		ctx.stdout(USAGE);
		return 2;
	}
	try {
		await command(rest, ctx);
		return 0;
	} catch (e) {
		if (e instanceof ForgeError) {
			ctx.stdout(`error: ${e.message}`);
			return e.details.file === "usage" ? 2 : 1;
		}
		if (e instanceof TypeError && /option|argument/i.test(e.message)) {
			ctx.stdout(`error: ${e.message}\n${USAGE}`);
			return 2;
		}
		throw e;
	}
}

export function modelFlag(argv: string[]): string | undefined {
	const i = argv.indexOf("--model");
	return i >= 0 ? argv[i + 1] : undefined;
}

if (import.meta.main) {
	process.exit(await run(process.argv.slice(2), createContext({ cwd: process.cwd(), model: modelFlag(process.argv) })));
}
```

- [x] **Step 6: Run the tests, then the CLI by hand with the fake provider**

Run: `bun test` — Expected: pass, coverage above threshold (prompts.ts `openInEditor` and `createContext`'s real stdin are the expected uncovered lines; if the functions threshold fails because of them, move `openInEditor` into its own file `src/cli/editor.ts` and exclude nothing — instead add a test that runs it with `EDITOR=true` (the `true` binary exits 0 and leaves the file unchanged) and asserts the text round-trips).

Then by hand, in a scratch directory:

```bash
cd $(mktemp -d) && FORGE_MODEL=fake/$OLDPWD/tests/fixtures/cli-responses.json bun $OLDPWD/src/cli/main.ts describe "The bot classifies hiring emails" && cat .forge/feature.yaml
```

Expected: the feature YAML printed with `status: pending`.

- [x] **Step 7: Lint and commit**

Run: `bun run lint` — Expected: clean.

```bash
git add src/cli tests/cli/main.test.ts tests/fixtures/cli-responses.json
git commit -m "Add the forge CLI: describe, scenarios, cases, dedupe, import, review"
```

---

### Task 11: Live smoke test, CI, README (showcase standard)

**Files:**
- Create: `tests/live/describe-scenarios.live.test.ts`, `.github/workflows/ci.yml`, `README.md`, `README.pt.md`, `LICENSE`
- Modify: `bunfig.toml` (exclude `tests/live` from the default run), `package.json` (add `test:live` script)

**Interfaces:**
- Consumes: the CLI from Task 10.
- Produces: a green CI on GitHub with jobs `test` and `lint`, and the two READMEs.

- [x] **Step 1: Write the live test, off by default**

`tests/live/describe-scenarios.live.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { mkdtemp } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { createContext } from "../../src/cli/context";
import { run } from "../../src/cli/main";
import { readFeature, readScenarios, writeFeature } from "../../src/core/files";

const live = process.env.FORGE_LIVE === "1";
const model = process.env.FORGE_MODEL ?? "google/gemini-3.5-flash";

describe.skipIf(!live)("live: describe and scenarios against a real model", () => {
	test("produces a pending feature and six kinds of scenarios", async () => {
		const cwd = await mkdtemp(join(tmpdir(), "forge-live-"));
		const ctx = createContext({ cwd, model });
		const out: string[] = [];
		ctx.stdout = (l) => out.push(l);
		expect(await run(["describe", "The bot receives one hiring-process email (from, subject, body) and classifies it as rejection, acknowledgement, interview, screening, offer, info_request or unrelated, answering JSON only. Automated confirmations are acknowledgement, never screening."], ctx)).toBe(0);
		const feature = await readFeature(join(cwd, ".forge"));
		expect(feature.output.labels).toContain("acknowledgement");
		await writeFeature(join(cwd, ".forge"), { ...feature, status: "approved" });
		expect(await run(["scenarios"], ctx)).toBe(0);
		const scenarios = await readScenarios(join(cwd, ".forge"));
		expect(new Set(scenarios.map((s) => s.kind)).size).toBe(6);
		console.log(out.join("\n"));
	}, 120_000);
});
```

Add to `bunfig.toml` under `[test]`: nothing — `describe.skipIf` keeps it green offline. Add to `package.json` scripts: `"test:live": "FORGE_LIVE=1 bun test tests/live"`.

- [x] **Step 2: Run it once for real (costs cents; needs GOOGLE_API_KEY)**

Run: `GOOGLE_API_KEY=… bun run test:live` — Expected: pass; paste the `scenarios:` summary line into the task note along with the token count from `.forge/usage.jsonl` of the temp dir. If Gemini answers 503, retry once; if the coverage check fails because the model skipped a kind, that is a real finding about the template — note it, and either strengthen `templates/scenarios.md` or accept and document.

- [x] **Step 3: Write CI**

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: 1.4.2
      - run: bun install --frozen-lockfile
      - run: bun test --coverage --coverage-reporter=lcov --coverage-reporter=text
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/lcov.info
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: 1.4.2
      - run: bun install --frozen-lockfile
      - run: bun run lint
```

The coverage badge: the `test` job already fails below the threshold (that is the honest badge — it is the CI status of a job that enforces coverage). Do not add a third-party coverage service in this task; note it in the backlog if Alberto wants a percentage badge.

- [x] **Step 4: Write the READMEs and license**

`LICENSE`: the MIT text with `Copyright (c) 2026 Alberto de Sá Cavalcanti de Albuquerque`.

`README.md`:

```markdown
🇺🇸 [English](README.md) · 🇧🇷 [Português](README.pt.md)

[![ci](https://github.com/albertosca/llm-test-forge/actions/workflows/ci.yml/badge.svg)](https://github.com/albertosca/llm-test-forge/actions/workflows/ci.yml)

# llm-test-forge

Forge reviewed regression suites for the LLM features in your application, and emit them for [promptfoo](https://promptfoo.dev) to run.

Describe what your feature does → the forge enumerates scenarios of six kinds (happy, edge, ambiguous, out-of-scope, adversarial, language) → generates cases per scenario **with an expected output** → you review them in files, one by one → (plan 2) estimate the cost, emit a promptfoo suite, and read the results back for judge disagreement and flaky cases.

## Install

Requires [bun](https://bun.sh) 1.4 or newer.

    bun install
    bun link      # exposes `forge` on your PATH

## Use

    cd your-app
    export FORGE_MODEL=google/gemini-3.5-flash      # or anthropic/claude-sonnet-5, openai/…, ollama/…
    forge describe "The bot classifies hiring emails into rejection, acknowledgement, …" --prompt-file src/prompt.txt
    forge review                                     # approve the feature
    forge scenarios
    forge review --only scenarios
    forge cases --n 5
    forge dedupe
    forge import logs/real-inputs.jsonl              # one {"input-name": "value"} object per line
    forge review

Everything lives in `.forge/` in your repository: `feature.yaml`, `scenarios.yaml`, `cases/<scenario>.yaml`. Commit it.

Provider keys come from `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `OPENAI_API_KEY`; `ollama/<model>` needs none. `fake/<file.json>` replays canned answers, for tests.

## Develop

    bun test          # unit tests, no network, coverage gate
    bun run lint      # Biome
    bun run test:live # one real call per verb, needs a key (FORGE_LIVE=1)

Design: `docs/superpowers/specs/2026-09-14-llm-test-forge-design.md`.

## License

MIT.
```

`README.pt.md`: the same content in Portuguese, starting with the same flag line (`🇺🇸 [English](README.md) · 🇧🇷 [Português](README.pt.md)`) and the same badge.

- [x] **Step 5: Run everything, commit**

Run: `bun test && bun run lint` — Expected: green.

```bash
git add tests/live .github/workflows/ci.yml README.md README.pt.md LICENSE bunfig.toml package.json
git commit -m "Add live smoke test, CI workflow, bilingual README and MIT license"
```

---

## Self-review against the spec (done while writing; findings applied)

- **Spec coverage:** `describe` (T4), `scenarios` with kind coverage and `--more` (T5), `cases` with oracle shapes, id rule, never overwriting reviewed (T6), `dedupe` (T7), `import` with the `imported` scenario and derived oracle (T8), `review` with editor, per-item decisions, expected for imported cases, `--all` requiring `--scenario` (T9–T10), `usage.jsonl` and `failures/` (T3), templates as files (T3–T7), errors naming file and id (T1, throughout), "no verb exits 0 with zero items" (T5, T6, T8), quality standard and README (T1, T11). **Not in this plan, by design:** `estimate`, `emit`, `report`, `suite.yaml` consumers, the moonlighter example, `prices.yaml` — plan 2. `SuiteSchema` is defined in T2 so plan 2 starts from a stable type.
- **Placeholders:** none; every step carries its code.
- **Type consistency:** `Llm.generate` signature is the same in T3 and every consumer; `createLlm` takes `{ forgeDir, resolve?, now? }` everywhere; `nextCaseId(scenarioId, existing: Case[])` used identically in T6 and T8; `PendingItem`, `applyDecision`, `withExpected` names match between T9 core, T9 loop and T10 review command; `ForgeError.details.file === "usage"` is the convention `run()` uses for exit code 2, set in `reviewCommand`.
- **Known soft spot to watch during execution:** the `--only` filter in `reviewCommand` accepts `scenarios`/`cases`/`feature`; the CLI test uses `--only scenarios`. The e2e test's `review --only scenarios` with a single `approve` line relies on `askChoice` mapping an empty stdin line to `skip` — that behaviour is implemented in `prompts.ts` and is the reason the remaining five scenarios stay pending.
