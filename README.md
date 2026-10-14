# nullclaw-image

Container image of [NullClaw](https://github.com/nullclaw/nullclaw) `main`, built with upstream's own
`Dockerfile` (`release` target) plus local patches (`patches/`). Upstream does not publish container images.

## Patches

| Patch | Effect |
|---|---|
| `0001-streaming-native-tool-calls.patch` | OpenAI-compatible providers send `tools` on streaming requests and accumulate `delta.tool_calls`, so streaming channels keep native tool calling (previously streaming fell back to a text tool protocol). |
| `0002-configurable-reasoning-default.patch` | `agent.reasoning_mode` (`off`/`on`/`stream`, default `off`) sets the default reasoning display for new and reset sessions. |

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
