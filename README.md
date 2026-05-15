# ai-gateway

A single-file Python CLI for the [Vercel AI Gateway](https://vercel.com/docs/ai-gateway/models-and-providers) public REST endpoints. No auth required, no dependencies, no install — just `python3`.

Wraps two endpoints:

| Endpoint | What it returns |
| --- | --- |
| `GET https://ai-gateway.vercel.sh/v1/models` | All available models with type, context window, pricing, tags. |
| `GET https://ai-gateway.vercel.sh/v1/models/{creator}/{model}/endpoints` | Per-provider endpoints, pricing, and supported parameters for one model. |

## Requirements

Python 3.9+. Standard library only (`urllib`, `json`, `argparse`).

## Quick start

```bash
./ai-gateway info
./ai-gateway models --type language --provider anthropic
./ai-gateway providers anthropic/claude-opus-4.7
./ai-gateway                          # drops into interactive REPL
```

Optional: drop it on your `PATH`.

```bash
ln -s "$PWD/ai-gateway" /usr/local/bin/ai-gateway
```

## Subcommands

### `info` — high-level summary of the gateway

```
$ ai-gateway info

Total models:     270

By type:
  language     186
  image        30
  video        25
  embedding    24
  reranking    5

Top owners (by model count):
  openai      48
  alibaba     31
  google      26
  ...

Pricing — language models ($/M tokens):
  input:   min $0.02/M   median $0.53/M   max $30.00/M
  output:  min $0.04/M   median $2.00/M   max $180.00/M
```

### `models` — list / filter / search models

```
$ ai-gateway models --help

  --type {language,embedding,reranking,image,video}
  --provider <owner>     filter by owned_by (openai, anthropic, google, ...)
  --tag <tag>            filter by tag (reasoning, tool-use, vision, ...)
  --search <substring>   match across id, name, description, owner, tags
  --sort {id,name,context,input_price,output_price}
  --json                 raw JSON output
```

Examples:

```bash
# All language models with reasoning, cheapest input first
ai-gateway models --type language --tag reasoning --sort input_price

# Everything OpenAI publishes
ai-gateway models --provider openai

# Find anything that looks like "claude opus"
ai-gateway models --search "claude opus"

# Pipe to jq
ai-gateway models --type embedding --json | jq '.[].id'
```

### `providers` — provider endpoints for a single model

```bash
ai-gateway providers anthropic/claude-opus-4.7
ai-gateway providers google/gemini-3.1-pro-preview --params
ai-gateway providers openai/gpt-5.5 --json
```

Output: one row per provider hosting the model with context length, max output, prompt/completion pricing, cache-read pricing, implicit-caching support, and live status. Pass `--params` to also print each provider's `supported_parameters`.

### `repl` — interactive shell

Drops you into a prompt where the same subcommands work without re-typing `ai-gateway`.

```
ai-gateway> models --tag tool-use --type language
ai-gateway> providers anthropic/claude-opus-4.7
ai-gateway> info
ai-gateway> quit
```

Running `./ai-gateway` with no arguments launches the REPL.

## Output

- Default: colorized aligned tables (auto-detects TTY, respects `NO_COLOR`).
- `--json` on any subcommand: raw JSON pass-through suitable for `jq` or scripts.
- Pricing displayed as `$/M tokens` for readability (gateway returns per-token rates).

## Notes

- These endpoints require **no authentication** — they're the same data that powers the [public model index](https://vercel.com/ai-gateway/models).
- Pricing and capability data are sourced from the gateway's own model registry. For actual inference billing, use a gateway API key via the AI SDK (`@ai-sdk/gateway`).
- See the full schema in the [Vercel docs](https://vercel.com/docs/ai-gateway/models-and-providers#dynamic-model-discovery).
