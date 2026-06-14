# NeMo Relay Example

Run `claude` inside an OpenShell sandbox under the
[NeMo Relay](https://github.com/NVIDIA/NeMo-Relay) runtime — observability,
guardrails, and trace export — via `openshell sandbox create -- nemo-relay claude`.

## How it works

`nemo-relay claude` binds a loopback gateway on `127.0.0.1`, points
`ANTHROPIC_BASE_URL` at it, and launches `claude` as a child so it can record
hooks and LLM traffic. This image configures NeMo Relay to forward that LLM
traffic to OpenShell's in-sandbox inference router (`https://inference.local`)
instead of straight to `api.anthropic.com`, so OpenShell keeps owning the
credentials and the egress policy.

```
claude -> nemo-relay (hooks + guardrails + traces) -> inference.local
          (OpenShell: TLS-terminate, policy check, inject real credentials)
          -> upstream provider
```

OpenShell detects the `claude-code` provider even though the command starts with
`nemo-relay`: provider detection looks past known agent wrappers to the inner
agent (see `detect_provider_from_command` in
[crates/openshell-providers/src/lib.rs](../../crates/openshell-providers/src/lib.rs)).

## Files

| File | Description |
|---|---|
| `Dockerfile` | Base sandbox image plus the `nemo-relay` CLI and its config |
| `nemo-relay-config.toml` | Runtime config; forwards the Anthropic upstream to `inference.local` |
| `nemo-relay-plugins.toml` | Observability exporters (OpenInference traces to Phoenix) |
| `sandbox-policy.yaml` | Self-contained policy: writable `/tmp`, runs as `sandbox`, Anthropic control-plane + Phoenix egress |

## Quick start

Requires a running OpenShell gateway with inference configured:

```shell
openshell inference set --provider <name> --model <model>
```

Then create the sandbox:

```shell
openshell sandbox create \
  --from examples/nemo-relay \
  --policy examples/nemo-relay/sandbox-policy.yaml \
  -- nemo-relay claude
```

If your build skips command-based provider detection, attach the provider
explicitly with `--provider claude-code`.

## Notes and troubleshooting

- **Release version.** Set `NEMO_RELAY_VERSION` in the `Dockerfile` to a version
  that has a published NeMo Relay release asset. The target triple is
  auto-detected (amd64/arm64) and the release tag uses the bare version with no
  leading `v`.
- **TLS trust.** NeMo Relay's HTTP client must trust the sandbox CA for
  `inference.local`. The `Dockerfile` sets `SSL_CERT_FILE` to the system trust
  bundle; adjust the path if your base image stores the sandbox CA elsewhere.
- **Proxy.** NeMo Relay's gateway must honor `HTTPS_PROXY` so its forwarded
  request reaches the `inference.local` interceptor.
- **Anthropic control plane.** Claude Code makes a startup connectivity check
  (and telemetry calls) directly to `api.anthropic.com`, independent of the
  inference route. The `anthropic` block in `sandbox-policy.yaml` allows these;
  without it you will see `Unable to connect to Anthropic services`. On a
  gateway with `providers_v2_enabled=true`, attaching `--provider claude-code`
  composes these endpoints automatically and the block is redundant.
- **Phoenix optional.** If you do not run Phoenix on the host, remove the
  `nemo-relay-plugins.toml` `COPY` line from the `Dockerfile` and the `phoenix`
  block from `sandbox-policy.yaml`.
