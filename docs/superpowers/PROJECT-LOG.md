# Project Log

## 2026-09-14 — Spec novo: llm-test-forge (design do MVP)

Nasce do veredito viável da bancada (investigação de 13/09/2026): o mercado cobre executor e avaliador (promptfoo), e ninguém cobre a forja — descrição → cenários de tipos mistos → casos com saída esperada → revisão humana em arquivo → estimativa de custo antes → discordância entre juízes e instabilidade depois. Decidido em brainstorming: formato em camadas (casos + suíte, promptfoo gerado dos dois), TypeScript com bun, lib + CLI primeiro e MCP depois, Vercel AI SDK, entrada em texto livre com prompt opcional, revisão em arquivo com CLI interativa, pipeline inteiro fino com dedupe semântico e `import` de logs por contrato JSONL.
→ spec: `docs/superpowers/specs/2026-09-14-llm-test-forge-design.md`
