# nullclaw-image

Container image of [NullClaw](https://github.com/nullclaw/nullclaw) `main`, built with upstream's own
`Dockerfile` (`release` target) plus local patches (`patches/`). Upstream does not publish container images.

## Patches

Applied in filename order; they are a stack (each is diffed against the previous one).

| Patch | Effect |
|---|---|
| `0001-streaming-native-tool-calls.patch` | OpenAI-compatible providers send `tools` on streaming requests and accumulate `delta.tool_calls`, so streaming channels keep native tool calling (previously streaming fell back to a text tool protocol). The capability is forwarded through `reliable.zig`/`router.zig`, so a failover chain or router no longer hides it. |
| `0002-configurable-reasoning-default.patch` | `agent.reasoning_mode` (`off`/`on`/`stream`, default `off`) sets the default reasoning display for new and reset sessions, instead of always starting at `off`. |
| `0003-slack-streaming-outbound.patch` | Two commits. (1) Slack outbound streaming: `chat.startStream`/`appendStream`/`stopStream` in threads, `chat.postMessage` + throttled `chat.update` elsewhere and wherever Slack rejects the native methods, at most one API call per second, and a permanent error disables the native path for the account. (2) Channel-agnostic live reasoning for Telegram and Slack drafts (Telegram is served by `channel_loop`, so the daemon-only passthrough never reached it), plus the first-turn-of-a-thread session-key miss that stripped reasoning. |
| `0004-slack-status-reactions.patch` | Slack task lifecycle reactions (`reactions.add`/`reactions.remove`) driven from the bus path, since `status_reactions` previously existed only on Telegram's polling loop. |
| `0005-route-configured-providers-by-prefix.patch` | A `provider/model` reference routes to any provider configured under `models.providers`, not just `reliability.fallback_providers`. Routing and failover stay separate: a bare model keeps its existing failover chain. |
| `0006-telegram-draft-html.patch` | Telegram live drafts are rendered with HTML parse mode like the final message, so reasoning quotes and formatting show while streaming; falls back to plain text if Telegram rejects the entities. |
| `0007-per-agent-autonomy.patch` | `agents.list[].autonomy` gives a named agent its own security policy and runtime, so a guest bound to one chat peer does not inherit the main agent's `yolo`. Tools that would act under the global policy (spawn, schedule, delegate, MCP, git, screenshot, image_info), `/config`, `/export` and `/bind` are withheld, and outbound attachment markers are confined to the agent's workspace. The same policy applies to `nullclaw agent --agent <name>`. Use `"level": "read_only"` for untrusted guests: the `supervised` shell confinement is a best-effort argument check, not a sandbox. |

## Builds

- Daily check; builds when upstream `main` moves or the patch set changes (tag = upstream commit + patch hash).
  Also on push to `patches/`, or manually with a ref / `force`.
- Fails instead of pushing if a patch no longer applies or the binary requires x86-64-v3+.

```
ghcr.io/wray-lee/nullclaw:latest                    # latest build
ghcr.io/wray-lee/nullclaw:<upstream-sha>-<patchhash>  # pinned
```

Update on the host: `docker compose pull && docker compose up -d`

Note: GitHub disables scheduled workflows after 60 days without repository activity; re-enable it in the
Actions tab or run it manually.
