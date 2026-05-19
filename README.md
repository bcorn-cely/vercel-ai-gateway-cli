# ai-gateway

A single-file Python CLI for the [Vercel AI Gateway](https://vercel.com/docs/ai-gateway/models-and-providers) public REST endpoints. No auth required, no dependencies, no install — just `python3`.

Wraps two endpoints:

| Endpoint | What it returns |
| --- | --- |
| `GET https://ai-gateway.vercel.sh/v1/models` | All available models with type, context window, pricing, tags. |
| `GET https://ai-gateway.vercel.sh/v1/models/{creator}/{model}/endpoints` | Per-provider endpoints, pricing, and supported parameters for one model. |

You don't need to know the schema or the exact model id — the CLI introspects response shapes (`schema`) and resolves fuzzy queries like `"claude opus"` or `"gpt 5.5"` into concrete model ids.

## Requirements

Python 3.9+. Standard library only (`urllib`, `json`, `argparse`).

## Quick start

```bash
./ai-gateway info
./ai-gateway models --type language --provider anthropic
./ai-gateway providers "claude opus 4.7"          # fuzzy match — no need to know exact id
./ai-gateway schema providers                     # describe the response shape, live
./ai-gateway                                      # drops into interactive REPL
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

The positional argument accepts an exact id **or** a partial name. Resolution:

- Exact id → used directly.
- Single fuzzy match → used, with a note showing what it resolved to.
- Multiple matches in a TTY → interactive numbered picker.
- Multiple matches non-interactively → listed on stderr, exit 1 (or pass `--first` to auto-pick the top score).

```bash
ai-gateway providers anthropic/claude-opus-4.7              # exact id
ai-gateway providers "claude opus 4.7"                      # fuzzy, resolves uniquely
ai-gateway providers "claude opus"                          # ambiguous → picker / list
ai-gateway providers "claude opus" --first                  # auto-pick top match
ai-gateway providers "gemini 3 pro" --params                # adds supported_parameters
ai-gateway providers openai/gpt-5.5 --json                  # raw JSON for jq
```

Output: one row per provider hosting the model with context length, max output, prompt/completion pricing, cache-read pricing, implicit-caching support, and live status. Pass `--params` to also print each provider's `supported_parameters`.

### `schema` — describe a gateway endpoint's response shape

For when you don't know what fields the endpoint returns. Fetches a live sample, walks the JSON tree, and prints every field path with its type and an example value. Surfaces fields that aren't in the published docs (e.g. `latency_last_1h.p95`, `uptime_last_1d`, `throughput_last_1h`).

```bash
ai-gateway schema models                              # describe /v1/models
ai-gateway schema providers                           # describe /v1/models/.../endpoints
ai-gateway schema providers --model "gemini 3 pro"    # use a specific model as the sample
ai-gateway schema providers --raw                     # pretty-printed sample JSON instead
```

Output is three columns — `path`, `type`, `example`:

```
path                                         type                     example
data                                         object
data.id                                      string                   "anthropic/claude-opus-4.7"
data.architecture.modality                   string                   "text+image+file→text"
data.endpoints[]                             array of object (len=3)
data.endpoints[].provider_name               string                   "anthropic"
data.endpoints[].pricing.prompt              string                   "0.000005"
data.endpoints[].uptime_last_1h              number                   99.8574
data.endpoints[].latency_last_1h.p95         number                   13880.6
```

### `repl` — interactive shell

Drops you into a prompt where the same subcommands work without re-typing `ai-gateway`.

```
ai-gateway> models --tag tool-use --type language
ai-gateway> providers "claude opus 4.7"
ai-gateway> schema providers
ai-gateway> info
ai-gateway> quit
```

The REPL uses `shlex.split`, so quoted multi-word arguments like `providers "claude opus 4.7"` work as you'd expect.

Running `./ai-gateway` with no arguments launches the REPL.

## Output

- Default: colorized aligned tables (auto-detects TTY, respects `NO_COLOR`).
- `--json` on any subcommand: raw JSON pass-through suitable for `jq` or scripts.
- Pricing displayed as `$/M tokens` for readability (gateway returns per-token rates).

## Notes

- These endpoints require **no authentication** — they're the same data that powers the [public model index](https://vercel.com/ai-gateway/models).
- Pricing and capability data are sourced from the gateway's own model registry. For actual inference billing, use a gateway API key via the AI SDK (`@ai-sdk/gateway`).
- See the full schema in the [Vercel docs](https://vercel.com/docs/ai-gateway/models-and-providers#dynamic-model-discovery).
