# Backlog

Deferred on purpose during the design of 2026-09-14 (see `specs/2026-09-14-llm-test-forge-design.md`).

- **MCP server** — a second thin shell over the same core verbs, so the review loop can run inside a conversation. Comes after the CLI proves the library.
- **`forge run`** — invoke `promptfoo eval` from inside the forge and produce the report in one step. Costs a dependency on the installed promptfoo binary and handling its errors; the MVP has the person run promptfoo themselves.
- **Multi-turn with a simulated user** — cases where the application's own reply shapes the next turn. Per-turn testing (`history` + `message` as input fields) is already covered; simulation is a different product.
- **Multi-run history** — keep `report.json` across runs in one place and plot trends. The MVP compares two runs with `--baseline`.
- **More providers** — OpenRouter, Mistral and others in the registry. MVP ships Anthropic, Google, OpenAI, Ollama.
- **Apply the public-repo showcase standard** — bilingual README with flag links, CI/lint/coverage badges, English description, MIT license. Part of the first plan, listed here so it is not forgotten if the plan is cut.
