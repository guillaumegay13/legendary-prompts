# legendary-prompts

> A minimalist collection of prompts from legendary founders, investors, engineers, and researchers — system prompts, agent prompts, and skills. Drop them into your SDK or coding agent and think like they do.

## Why

Great operators have a way of thinking. When they share the prompts they actually use, that thinking becomes a drop-in tool. This repo collects those prompts, verbatim, with sources.

Each file is just the raw prompt. The index below adds light context — what kind of prompt it is, the product it targets, and a one-line use — so you know what you're pasting and where.

## Prompts

| Person | Role | Type | Product | Context | Source | Date |
|---|---|---|---|---|---|---|
| [Marc Andreessen](prompts/openclaw/system-prompts/marc-andreessen.md) | Co-founder, a16z | System prompt | OpenClaw | Brutally honest world-class expert — no flattery, no hedging, leads with the counterargument | [@pmarca](https://x.com/pmarca/status/2051374498994364529) | 2026-05-04 |
| [Suzanne](prompts/claude-code/prompts/suzanne.md) | Anthropic | Prompt | Claude Code | Teaching prompt that makes you deeply understand a coding session before it ends | [gist](https://gist.github.com/ThariqS/1389dcdff9eba4789887a2211370f06b) (shared by Thariq Shihipar) | 2026-06-01 |

## Contributing

Open a PR adding the prompt at `prompts/<product>/<type>/<slug>.md`, containing **only** the verbatim prompt text — no front-matter, no fences. The path encodes the taxonomy:

- `<product>` — the target tool/agent, slugified: `any-sdk`, `claude-code`, `openclaw`, …
- `<type>` — `system-prompts`, `prompts`, or `skills`

e.g. `prompts/claude-code/prompts/suzanne.md`. In the PR description, include:

1. A direct link to the public source (tweet, blog, podcast transcript, etc.).
2. The date the prompt was captured (YYYY-MM-DD).
3. One sentence of **context** (what it does / when to use it).

Attribution and context then go into the table above.

No paraphrasing, no "inspired by" prompts, no leaks. Public, attributable, verbatim.

## License

[MIT](LICENSE). Prompt text remains the property of its respective author and is reproduced here under fair-use quotation for commentary and reference.
