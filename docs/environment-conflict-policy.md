# Environment Conflict Policy / 环境变量冲突处理策略

This fork treats environment conflicts as a precedence problem, not as a reason to block startup.

本 fork 将环境变量冲突视为“优先级选择”问题，而不是“阻止用户继续使用”的理由。

## Background / 背景

Claude Code supports an `env` field in `~/.claude/settings.json`. Provider switching also needs to pass provider-specific environment variables such as `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, and `ANTHROPIC_MODEL`.

If the same key appears in both places, a tool can either fail fast and ask the user to clean up the file manually, or choose a deterministic precedence rule and continue.

Claude Code 支持在 `~/.claude/settings.json` 中配置 `env`。同时，provider 切换也需要传入 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL` 等 provider 相关环境变量。

当同一个 key 同时出现在两个地方时，可以选择直接失败并要求用户手动清理，也可以选择一个明确、可预测的优先级并继续运行。

## Policy / 策略

ccc uses deterministic precedence:

1. The active provider's `providers.<name>.env` in `~/.claude/ccc.json` is authoritative for keys that provider defines.
2. Before writing `settings.json`, ccc removes only the keys managed by the active provider from `settings.json.env`.
3. Other user-defined environment variables stay in `settings.json.env`, including Anthropic or Claude-prefixed keys that are not managed by the active provider.
4. ccc does not refuse to start merely because `settings.json.env` contains `ANTHROPIC_*`, `CLAUDE_*`, or `CLAUDE_CODE_*` keys.

ccc 使用确定性的优先级：

1. 当前 provider 在 `~/.claude/ccc.json` 的 `providers.<name>.env` 中定义的 key，以当前 provider 配置为准。
2. 写回 `settings.json` 前，ccc 只移除当前 provider 管理的同名 key。
3. 其他用户自定义环境变量继续保留在 `settings.json.env` 中，包括当前 provider 没有管理的 `ANTHROPIC_*`、`CLAUDE_*` 或 `CLAUDE_CODE_*` key。
4. ccc 不会仅仅因为 `settings.json.env` 中存在这些前缀的 key 就拒绝启动。

## Why / 为什么

A provider switcher should make provider selection reliable without taking ownership of the user's entire Claude Code configuration.

Failing on broad prefixes such as `ANTHROPIC_*` or `CLAUDE_CODE_*` is safe, but it forces users to delete useful personal configuration and turns a resolvable merge decision into a startup blocker.

The ccc behavior is intentionally narrower:

- provider-controlled keys are resolved by provider precedence;
- non-provider keys remain user-owned;
- startup continues with predictable behavior.

一个 provider 切换工具应该让 provider 选择变得可靠，但不应该接管用户的整个 Claude Code 配置。

对 `ANTHROPIC_*` 或 `CLAUDE_CODE_*` 这类宽泛前缀直接失败是安全的，但它会迫使用户删除有用的个人配置，并把一个可以通过合并策略解决的问题变成启动阻塞。

ccc 的行为刻意保持更窄的边界：

- provider 管理的 key 由 provider 优先级解决；
- 非 provider 管理的 key 继续归用户所有；
- 启动过程继续，并保持行为可预测。

## Example / 示例

Given this `settings.json.env`:

```json
{
  "ANTHROPIC_BASE_URL": "https://old.example.com/anthropic",
  "ANTHROPIC_API_KEY": "user-key",
  "CLAUDE_CODE_MAX_RETRIES": "5",
  "MY_CUSTOM_VAR": "value"
}
```

And an active provider with:

```json
{
  "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
  "ANTHROPIC_AUTH_TOKEN": "sk-...",
  "ANTHROPIC_MODEL": "glm-4.7"
}
```

ccc will:

- launch Claude with the provider's `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, and `ANTHROPIC_MODEL`;
- remove `ANTHROPIC_BASE_URL` from `settings.json.env` because it is managed by the active provider;
- keep `ANTHROPIC_API_KEY`, `CLAUDE_CODE_MAX_RETRIES`, and `MY_CUSTOM_VAR` because the active provider does not manage them.

ccc 会：

- 使用当前 provider 的 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN` 和 `ANTHROPIC_MODEL` 启动 Claude；
- 从 `settings.json.env` 中移除 `ANTHROPIC_BASE_URL`，因为它由当前 provider 管理；
- 保留 `ANTHROPIC_API_KEY`、`CLAUDE_CODE_MAX_RETRIES` 和 `MY_CUSTOM_VAR`，因为当前 provider 没有管理它们。
