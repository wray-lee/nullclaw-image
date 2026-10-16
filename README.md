# nullclaw-image

Container image of [NullClaw](https://github.com/nullclaw/nullclaw) `main`, built with upstream's own
`Dockerfile` (`release` target) plus local patches (`patches/`). Upstream does not publish container images.

## Patches

Applied in filename order; they are a stack (each is diffed against the previous one).

| Patch | Effect |
|---|---|
| `0001-streaming-native-tool-calls-pr971.patch` | Native tool calls on streaming turns, carried from upstream PR [nullclaw#971](https://github.com/nullclaw/nullclaw/pull/971) (vernonstinebaker) until it is merged; it refuses to run tool calls salvaged from a cut stream. **Drop this patch once #971 lands on main**; the rest of the stack applies on top of #971 unchanged. |
| `0002-configurable-reasoning-default.patch` | `agent.reasoning_mode` (`off`/`on`/`stream`, default `off`) sets the default reasoning display for new and reset sessions, instead of always starting at `off`. |
| `0003-slack-streaming-outbound.patch` | Two commits. (1) Slack outbound streaming: `chat.startStream`/`appendStream`/`stopStream` in threads, `chat.postMessage` + throttled `chat.update` elsewhere and wherever Slack rejects the native methods, at most one API call per second, and a permanent error disables the native path for the account. (2) Channel-agnostic live reasoning for Telegram and Slack drafts (Telegram is served by `channel_loop`, so the daemon-only passthrough never reached it), plus the first-turn-of-a-thread session-key miss that stripped reasoning. |
| `0004-slack-status-reactions.patch` | Slack task lifecycle reactions (`reactions.add`/`reactions.remove`) driven from the bus path, since `status_reactions` previously existed only on Telegram's polling loop. |
| `0005-route-configured-providers-by-prefix.patch` | A `provider/model` reference routes to any provider configured under `models.providers`, not just `reliability.fallback_providers`. Routing and failover stay separate: a bare model keeps its existing failover chain. |
| `0006-telegram-draft-html.patch` | Telegram live drafts are rendered with HTML parse mode like the final message, so reasoning quotes and formatting show while streaming; falls back to plain text if Telegram rejects the entities. |
| `0007-per-agent-autonomy.patch` | `agents.list[].autonomy` gives a named agent its own security policy and runtime, so a guest bound to one chat peer does not inherit the main agent's `yolo`. Tools that would act under the global policy (spawn, schedule, delegate, MCP, git, screenshot, image_info), `/config`, `/export` and `/bind` are withheld, and outbound attachment markers are confined to the agent's workspace. The same policy applies to `nullclaw agent --agent <name>`. Use `"level": "read_only"` for untrusted guests: the `supervised` shell confinement is a best-effort argument check, not a sandbox. |
| `0008-reasoning-field-retry.patch` | When an OpenAI-compatible endpoint rejects history with "Add a supported thinking field to each assistant message" (IFM K2), resend once with `reasoning_content` on assistant messages and keep doing so, instead of failing every later turn until `/new`. Streaming errors now carry the provider's message instead of a generic "An error occurred". |
| `0009-think-levels-compatible.patch` | `/think` levels actually reach OpenAI-compatible non-OpenAI models (DeepSeek, Kimi, K2) as `reasoning_effort`; previously they were dropped. Adds `max`. A rejected level is lowered to `high` (or dropped) per model. Providers with their own thinking field are unchanged. |
| `0010-hide-text-tool-calls-in-streams.patch` | One shared stream filter hides text-form tool calls (tags and JSON body) from every live surface, including streamed reasoning and the CLI `reasoning_mode: stream` path, without dropping visible text; hides calls the dispatcher would execute (including unclosed tool calls and bracket dialects) while releasing plain text. Some models (K2 via a gateway) intermittently return tool calls as plain content. |
| `0011-runaway-turns-and-stop.patch` | `/stop` interrupts a busy session on every channel (Slack before its FIFO queue) and aborts an in-flight provider stream; tool calls from cut or length-truncated output are not executed; at most 24 tool calls per batch; non-blocking logging shrink so a stuck stderr cannot wedge the gateway. |
| `0012-telegram-draft-ownership.patch` | Telegram live drafts stay owned by the running turn, so concurrent inbound messages or button callbacks cannot reset or delete an active draft mid-stream; clears the draft on turn completion (including inline choices and edits). |
| `0013-turn-completion.patch` | Guards against structural turn-completion stalls where a model promises action in prose without calling tools: injects prompt execution bias, captures structural provider stop reasons (`finish_reason: tool_calls`, Anthropic `tool_use`, streamed SSE stop), and nudges or errors on empty tool stops across all languages. |
| `0014-telegram-rich-messages.patch` | Telegram rich messages via Bot API `sendRichMessage` and `editMessageText rich_message` for full GFM formatting (headings, tables, task lists, footnotes, `==mark==`, `||spoiler||`, `$math$`). Opt-in per account via `channels.telegram.accounts.<id>.rich_messages` (default `false`); messages exceeding 32768 codepoints or triggering rich API errors fall back directly to legacy HTML/plain delivery. |
| `0015-slack-slash-command.patch` | Slack Socket Mode `slash_commands` envelopes are handled instead of acked and dropped. One configurable command (OpenClaw-style, e.g. `/openclaw /stop`) passes its text through as a normal message or slash command; replies go to `response_url` as ephemeral. A top-level `/stop` also interrupts turns running in that channel's threads. Opt-in via `channels.slack.accounts.<id>.slash_command` `{enabled, name}`. |
| `0016-test-dense-run-runner-independent.patch` | The 0010 dense-run regression tests count scan steps instead of asserting wall-clock time, so they pass on slow CI runners and still catch an O(N²) regression. |
| `0017-k2-think-tags-runaway-output-cap.patch` | Namespaced and pipe think tags such as `</ifm\|think>` are treated as reasoning, and text before an orphan closer is dropped as leaked reasoning (OpenClaw's rule). Stray or repeated tool-call closers such as `</function>` are hidden. OpenAI-compatible requests carry an output cap (`max_completion_tokens` for reasoning endpoints; default 8192, override with top-level `max_tokens`). |
| `0018-bounded-channel-io-and-supervision.patch` | Every network call on the Telegram (and other) poller paths is time-bounded, including credentialed requests. The supervisor stops a wedged poller with a grace period and respawns it instead of joining it forever. Adds diagnostics: the failing Bot API method, the poller step on stale, and "polling thread started" on respawn. |
| `0019-stop-aborts-streams-and-backpressure.patch` | `/stop` aborts the in-flight provider request on every transport (streaming and non-streaming, including retries). Stream preview chunks never block a turn on a full outbound queue. The Slack stream flush honours its 1 s minimum. A stopped turn's queued Slack and Telegram previews are dropped. |

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
