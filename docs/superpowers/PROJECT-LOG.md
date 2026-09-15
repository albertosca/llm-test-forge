# Project Log

## 2026-09-15 — Plano concluído: core-pipeline (describe → scenarios → cases → dedupe → import → review)

O MVP da forja está de pé: biblioteca + CLI em TypeScript/bun, provider via Vercel AI SDK (anthropic, google, openai, ollama e um `fake/` para teste), tudo em arquivos YAML dentro de `.forge/`. Onze tasks executadas por subagente, cada uma revisada contra o próprio brief — e a revisão whole-branch no fim achou seis defeitos que nenhuma revisão por-task podia ver, todos da mesma família: o verbo relatava sucesso sem ter feito o trabalho. Corrigidos nesta mesma branch: o gate de feature pendente valia em um verbo de três; `describe` destruía uma feature já revisada em silêncio; uma edição que mudava o id era descartada e contada como "1 edited"; `review --scenario` desconhecido respondia "nothing pending" e saía 0; `dedupe` escrevia um arquivo de casos vazio; e o contrato de oráculo valia para o modelo mas não para a pessoa (Enter no prompt gravava `expected: {label: ""}` como aprovado). Ficou de fora, por decisão: `emit` para promptfoo, execução da suíte e estimativa de custo — são o plano 2.
→ plan: `docs/superpowers/plans/2026-09-14-core-pipeline.md`

## 2026-09-14 — Spec novo: llm-test-forge (design do MVP)

Nasce do veredito viável da bancada (investigação de 13/09/2026): o mercado cobre executor e avaliador (promptfoo), e ninguém cobre a forja — descrição → cenários de tipos mistos → casos com saída esperada → revisão humana em arquivo → estimativa de custo antes → discordância entre juízes e instabilidade depois. Decidido em brainstorming: formato em camadas (casos + suíte, promptfoo gerado dos dois), TypeScript com bun, lib + CLI primeiro e MCP depois, Vercel AI SDK, entrada em texto livre com prompt opcional, revisão em arquivo com CLI interativa, pipeline inteiro fino com dedupe semântico e `import` de logs por contrato JSONL.
→ spec: `docs/superpowers/specs/2026-09-14-llm-test-forge-design.md`
