# cbstats

> 离线统计 [CodeBuddy CLI](https://cnb.cool/codebuddy/codebuddy-code) 的本地用量：按天输出 **输入 Token / 输出 Token / 缓存读 / 缓存写 / Token 总量 / 使用模型**。

定位类比 [`ccusage`](https://github.com/ryoppippi/ccusage)，但不需要 ccusage、不联网、**不消耗任何额度** —— 直接解析 CodeBuddy 自己写在硬盘上的会话日志。

English → [README.en.md](README.en.md)

---

## 它补的是哪一格

CodeBuddy 内置了 `/stats`（跨项目历史聚合：活动热力图、模型排行、连续活跃天数）和 `/cost`（当前会话），但**没有 CLI 子命令**能直接给出「某月每天的用量明细表」。

cbstats 就是那个子命令：一条命令出表，可进 cron、可导 JSON。

## 安装

```bash
git clone https://github.com/caozeal/cbstats.git
ln -sf "$PWD/cbstats/cbstats" ~/.local/bin/cbstats   # 换成任意在 PATH 里的目录
cbstats --help
```

只需要 **Python 3**（仅标准库，未用新语法；本机实测 3.14）。

## 用法

```bash
cbstats                        # 最近 14 天
cbstats 2026-09                # 指定月份（自动算月末）
cbstats 2026-09-01 2026-09-30  # 指定区间
cbstats --days 30              # 最近 30 天
cbstats --all                  # 全部历史
cbstats 2026-09 --by-model     # 每天按模型拆行
cbstats 2026-09 --json         # JSON 输出（精确整数 + 按模型嵌套 + totals）
cbstats --exact                # 表格打印完整整数（默认 M/k 紧凑）
cbstats --no-subagents         # 只统计主会话，排除子代理
```

### 输出示例

（示例数字为构造值，非真实用量）

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

## 字段口径（重要，别误读表）

数据取自每条模型请求的 `providerData.rawUsage`：

| 字段 | 含义 |
|---|---|
| `prompt_tokens` | 总输入，**已包含**缓存命中 |
| `prompt_cache_hit_tokens` | 缓存读（命中） |
| `prompt_cache_miss_tokens` | 缓存未命中 = 真实新增输入 |
| `completion_tokens` | 输出（含 thinking） |
| `completion_thinking_tokens` | 其中 thinking |
| `total_tokens` | 输入 + 输出 |
| `cache_creation_input_tokens` / `prompt_cache_write_tokens` | 缓存写 |
| `credit` | 计费积分 |

四条容易踩的坑：

1. **别把「缓存读」加到「总量」上** —— 输入已含缓存读（`prompt_tokens = 命中 + 未命中`），`total_tokens = 输入 + 输出` 才是完整口径。表格底部单独给出「缓存未命中」，那才是真实新处理的量。
2. **「缓存写」在有隐式自动缓存的网关上恒为 0** —— 不是漏统计，是数据源本身不计量。只有显式缓存（如 Anthropic 的 `cache_creation`）才有值。
3. **`credit` 不是完整账单** —— 部分模型不记积分（读到 0 属正常）。要做成本核算请以官方账单为准。
4. **子代理默认计入** —— CodeBuddy 把子代理的调用写在 `<会话>/subagents/*.jsonl` 里，漏掉它会少算约 13% 的输出量（并看不到子代理专用的轻量模型）。`--no-subagents` 可关掉做对照。

另外：

- **某天没有记录 ≠ 那天没用过** —— `codebuddy cleanup` 会删除旧会话，统计完全依赖本地日志还在。
- 数据源目录：`~/.codebuddy/projects/<项目>/<会话>.jsonl` 与 `~/.codebuddy/projects/<项目>/<会话>/subagents/*.jsonl`（只读）。
- `timestamp` 兼容 epoch 秒与毫秒。

## JSON 输出

`--json` 给出精确整数和按模型嵌套的明细，方便再加工（管道给 `jq` 等）：

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

## 兼容性

- 面向 CodeBuddy 会话日志的 `providerData.rawUsage`（实测 v2.163+）。
- 若某行没有 `rawUsage`，会回退到归一化 `message.usage`（Anthropic 风格 `input_tokens` / `output_tokens` / `cache_read_input_tokens` / `cache_creation_input_tokens`），此时 thinking 与 credit 不可得。
- 日志格式由上游决定，未来版本若改变字段，脚本可能需同步更新。

## License

[MIT](LICENSE)
