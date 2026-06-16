# Routing hermes-agent through cc-gateway

This guide explains how to make sure Claude Code sessions spawned by
[hermes-agent](https://github.com/) (and, transitively,
claw-orchestrator) flow through cc-gateway instead of talking to
`api.anthropic.com` directly. The goal is a single fingerprint for
every CC session a hermes operator launches, regardless of which
workstation or container the agent runs on.

> Status: alpha. Tested against cc-gateway 0.2.x. The hermes-agent
> launcher API is still in flux; treat the snippets below as templates,
> not contracts.

## TL;DR

Hermes spawns Claude Code as a child process. Claude Code reads its
upstream URL and auth token from environment variables, so the
integration is just:

1. Point `ANTHROPIC_BASE_URL` (or the equivalent CC env var) at the
   gateway.
2. Set `ANTHROPIC_API_KEY` to the **client token** you minted with
   `scripts/add-client.sh`, *not* the real OAuth token.
3. Trust the gateway's TLS cert from the hermes host (or run the
   gateway on `127.0.0.1` and skip TLS for local-only setups).

Nothing in hermes-agent itself needs to know that the gateway exists.

## 1. Mint a client token for hermes

On the cc-gateway host:

```bash
bash scripts/add-client.sh hermes-agent
```

This appends a new entry under `auth.tokens` in `config.yaml`. Copy
the generated token; it is the value hermes will pass as its
`x-api-key` / `ANTHROPIC_API_KEY`.

If hermes runs in multiple environments (laptop, dev VM, CI), mint a
separate token per environment so you can revoke them independently:

```bash
bash scripts/add-client.sh hermes-agent-laptop
bash scripts/add-client.sh hermes-agent-ci
```

## 2. Configure the hermes-agent launcher

Hermes-agent launches CC by exec'ing the `claude` binary in a child
process. Inject the gateway env vars into that child's environment.

### Option A: shell wrapper

The lowest-friction integration: replace the `claude` binary on
`$PATH` with a wrapper that hermes will pick up unchanged.

```bash
# ~/.local/bin/claude
#!/usr/bin/env bash
export ANTHROPIC_BASE_URL="https://cc-gateway.internal:8443"
export ANTHROPIC_API_KEY="$(cat ~/.config/hermes/cc-gateway-token)"
exec /usr/local/bin/claude.real "$@"
```

Make executable, then move the real binary aside:

```bash
chmod +x ~/.local/bin/claude
sudo mv /usr/local/bin/claude /usr/local/bin/claude.real
```

Pros: works with every hermes version that shells out to `claude`.
Cons: global; affects interactive CC use too. Use Option B for
per-process isolation.

### Option B: hermes-agent config block

If you control the hermes-agent invocation, set the env directly in
its config (exact key depends on hermes version):

```yaml
# hermes-agent.yaml
claude_code:
  env:
    ANTHROPIC_BASE_URL: https://cc-gateway.internal:8443
    ANTHROPIC_API_KEY: ${HERMES_CC_GATEWAY_TOKEN}
    # Optional: skip TLS verification for self-signed gateway certs.
    # Only do this if the gateway is on a private network.
    NODE_TLS_REJECT_UNAUTHORIZED: "0"
```

Then export `HERMES_CC_GATEWAY_TOKEN` in the hermes systemd unit or
shell rc file.

## 3. Verify the integration

From the hermes host, hit the gateway's diagnostic endpoints:

```bash
curl -sk https://cc-gateway.internal:8443/_health | jq
curl -sk -H "x-api-key: $HERMES_CC_GATEWAY_TOKEN" \
    https://cc-gateway.internal:8443/_verify | jq
```

`/_health` should report `status: ok` and a non-empty
`canonical_device`. `/_verify` returns a synthetic before/after
payload showing how the rewriter normalizes a request from this
client; the `after` section is what Anthropic will actually see.

Then spawn a real hermes task and confirm the gateway logs an
incoming request tagged `hermes-agent`:

```
← POST /v1/messages from 10.0.0.42
Client "hermes-agent" → POST /v1/messages
```

## 4. claw-orchestrator note

`claw-orchestrator` invokes hermes-agent as a sub-process. As long as
the hermes invocation inherits the env vars above, claw-spawned CC
sessions will also route through the gateway with no extra wiring.
Mint a distinct client token per orchestrator instance if you need
per-instance audit logs.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `401 Unauthorized - provide client token via x-api-key header` | Hermes is sending the real OAuth token, not the gateway client token. Re-check `ANTHROPIC_API_KEY`. |
| `503 OAuth token not available - gateway is refreshing` | Gateway's stored refresh token expired. Re-run `scripts/quick-setup.sh` on the gateway host. |
| `502 Bad gateway` with `ECONNREFUSED` upstream | Gateway can't reach `api.anthropic.com`. Check `clash-rules.yaml` and any outbound proxy on the gateway host. |
| Streaming responses hang or arrive all at once | Some reverse proxies between hermes and the gateway buffer SSE. Either terminate TLS at the gateway directly or set `proxy_buffering off` upstream. |

## See also

- [`README.md`](../../README.md) — full gateway reference.
- [`CLAUDE.md`](../../CLAUDE.md) — internal architecture notes.
- `config.example.yaml` — every gateway knob in one place.
