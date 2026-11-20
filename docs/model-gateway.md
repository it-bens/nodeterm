# Model gateway and model switcher

`settings.modelGateway` and the per-node model switch: how one gateway root maps to launch routes, env vars and models for each base harness, and how a running node moves to another model. Main code: `shared/agents/model-gateway.ts`, `shared/agents/expansion.ts`, `ModelGatewayCredentialService`, core handler `agent:discover-models`.

Related: [adding an agent](adding-an-agent.md) (items 17 and 18), [permission mode](permission-mode.md).

## Settings and credentials

- `settings.modelGateway` stores one gateway root plus a NON-SECRET credential reference: `${env:VAR}` for environment mode, or `${secret:model-gateway-api-key}` for a literal held by `ModelGatewayCredentialService`.
- Desktop literal keys reuse the GitHub token store's safeStorage encryption / 0600 fallback; Server Edition uses the same generic 0600 atomic store.
- Legacy plaintext settings migrate only after the secret write succeeds.
- Env expansion reuses `shared/agents/expansion.ts` and happens only in core against the host process environment. An unset reference fails closed instead of sending a token or partial credential.

## Mapping and discovery

- `shared/agents/model-gateway.ts` is the ONE mapping from a base harness to derived routes, env vars, compatible models and safely quoted model flags.
- Discovery at `/v1/models` is the **OpenAI Models API convention**, implemented by both LiteLLM and Bifrost. The current `/openai/v1` + `/anthropic` launch-route derivation is Bifrost's layout, not the source of the discovery convention.
- Discovery sends the standard bearer header plus Bifrost's `x-bf-vk` header (needed by legacy, non-`sk-bf-` virtual keys), and runs in core (`agent:discover-models`) so browser CORS cannot block the Server Edition and the key never enters a terminal command.
- Support is a capability (`MODEL_SWITCH_CAPABLE = claude/codex/copilot`) resolved through `capabilityAgentId`, so a custom agent with a supported `baseAgent` inherits it automatically. The settings UI and canvas menu carry no agent allowlist.
- Custom-agent env is still merged last and may override the shared mapping.

## Switching a running node's model

- A model switch SIGTERMs the pane's foreground non-shell process group (never types `/exit`) and RECYCLES the tmux session before cold-resume: an existing shell may predate the gateway setting, and tmux env changes do not retroactively change that shell's environment. Recreating it guarantees the current URL/key applies without typing a secret into the pane.
- On Windows (session host / direct ConPTY, no process groups) the kill is `taskkill /T /F` on the re-proven agent pid only, never the console root, and requires the agent-identity gate (`session-host/terminate-owner.ts`; an older live host answers `'unsupported'`).
- Ordinary Restart stays in-place.

## Surfaces

Desktop and Server Edition use the same core handler. Relay tabs deliberately do not apply this machine's gateway to another core. Mobile needs a settings/model-picker surface before it can expose the feature.
