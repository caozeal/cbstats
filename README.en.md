# cbstats

> Offline usage statistics for the [CodeBuddy CLI](https://cnb.cool/codebuddy/codebuddy-code): daily **input / output / cache-read / cache-write tokens**, **total tokens**, and **which models** were used.

In spirit it is a [`ccusage`](https://github.com/ryoppippi/ccusage) for CodeBuddy — but it needs neither ccusage nor network access, and **consumes zero quota**: it simply parses the session logs CodeBuddy already writes to disk.

中文 → [README.md](README.md)

---

## Why this exists

CodeBuddy ships `/stats` (cross-project history: activity heatmap, model ranking, streaks) and `/cost` (current session), but there is **no CLI subcommand** that prints a per-day usage table.

`cbstats` is that subcommand: one command for the table, cron-friendly, with a `--json` mode for further processing.

## Install

```bash
git clone https://github.com/caozeal/cbstats.git
ln -sf "$PWD/cbstats/cbstats" ~/.local/bin/cbstats   # or any directory on your PATH
cbstats --help
```

Requires **Python 3** only (standard library, no modern syntax; tested on 3.14).

## Usage

```bash
cbstats                        # last 14 days
cbstats 2026-09                # a whole month (end of month computed)
cbstats 2026-09-01 2026-09-30  # an explicit range
cbstats --days 30              # last 30 days
cbstats --all                  # all history
cbstats 2026-09 --by-model     # one row per day per model
cbstats 2026-09 --json         # JSON (exact integers, per-model nesting, totals)
cbstats --exact                # print full integers (default is compact M/k)
cbstats --no-subagents         # main sessions only
```

### Sample output

(Numbers below are synthetic, not real usage)

```
日期                 输入Token       输出Token      缓存读Token      缓存写Token       Token总量            积分            请求  使用模型
--------------------------------------------------------------------------------------------------------------------------------------------
2026-09-01           25.4M           70k         24.2M             0         25.5M             0           106  model-a-flash
2026-09-02          311.1M          1.0M        305.3M             0        312.1M           411         1,213  model-b(1194),model-lite(19)
--------------------------------------------------------------------------------------------------------------------------------------------
合计                  336.5M          1.1M        329.5M             0        337.6M           411         1,319

缓存未命中(真实新增输入) 7.6M | thinking 输出 657k | 有效天数 2
注：本机网关为隐式自动缓存，缓存写计量恒为 0（非漏统计）。
```

(The table labels are Chinese; the numbers are locale-independent.)

## Field semantics (read this before interpreting the table)

Values come from `providerData.rawUsage` of each model request:

| Field | Meaning |
|---|---|
| `prompt_tokens` | Total input, **already includes** cache hits |
| `prompt_cache_hit_tokens` | Cache read (hits) |
| `prompt_cache_miss_tokens` | Cache misses = genuinely new input |
| `completion_tokens` | Output (includes thinking) |
| `completion_thinking_tokens` | Thinking tokens within the output |
| `total_tokens` | input + output |
| `cache_creation_input_tokens` / `prompt_cache_write_tokens` | Cache write |
| `credit` | Billing credits |

Four caveats:

1. **Do not add "cache read" to "total".** Input already includes cache reads (`prompt_tokens = hits + misses`); `total_tokens = input + output` is the complete figure. The footer prints cache misses separately — that is the genuinely new input.
2. **"Cache write" is always 0 on gateways with implicit caching.** It is not a parsing bug: the source simply does not meter it. Only explicit caching (e.g. Anthropic's `cache_creation`) yields a value.
3. **`credit` is not a complete bill.** Some models are not credited (reading 0 is normal). Use official billing for cost accounting.
4. **Subagents are included by default.** CodeBuddy records subagent calls in `<session>/subagents/*.jsonl`; skipping them undercounts output by roughly 13% (and hides subagent-only lightweight models). Use `--no-subagents` for comparison.

Also:

- **A day with no records does not mean no usage** — `codebuddy cleanup` deletes old sessions; statistics depend entirely on the local logs still being present.
- Data sources: `~/.codebuddy/projects/<project>/<session>.jsonl` and `~/.codebuddy/projects/<project>/<session>/subagents/*.jsonl` (read-only).
- `timestamp` accepts both epoch seconds and milliseconds.

## JSON output

`--json` emits exact integers with per-model breakdowns, ready to pipe into `jq`:

```json
{
  "generatedAt": "2026-09-02T10:00:00",
  "range": { "start": "2026-09-01", "end": "2026-09-30" },
  "includeSubagents": true,
  "days": [
    {
      "date": "2026-09-01",
      "inputTokens": 25400000,
      "outputTokens": 70000,
      "cacheReadTokens": 24200000,
      "cacheWriteTokens": 0,
      "cacheMissTokens": 1200000,
      "thinkingTokens": 30000,
      "totalTokens": 25470000,
      "credit": 0,
      "requests": 106,
      "models": {
        "model-a-flash": { "requests": 106, "inputTokens": 25400000, "outputTokens": 70000, "cacheReadTokens": 24200000, "totalTokens": 25470000, "credit": 0 }
      }
    }
  ],
  "totals": { "inputTokens": 25400000, "outputTokens": 70000, "requests": 106, "credit": 0 }
}
```

## Compatibility

- Targets the `providerData.rawUsage` shape found in CodeBuddy session logs (verified on v2.163+).
- If a line has no `rawUsage`, it falls back to the normalized `message.usage` (Anthropic style: `input_tokens` / `output_tokens` / `cache_read_input_tokens` / `cache_creation_input_tokens`); thinking and credit are unavailable in that path.
- The log format is upstream-owned; a future release may change fields and require an update here.

## License

[MIT](LICENSE)
