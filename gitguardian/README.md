# gitguardian — GitGuardian secret scanning (ggshield)

A mixin kit that installs [GitGuardian](https://www.gitguardian.com/)'s [`ggshield`](https://github.com/GitGuardian/ggshield) secret scanner into a coding-agent sandbox and wires it in as the **agent's own AI hook** — so the agent's actions are scanned for hardcoded secrets automatically, and the GitGuardian API key never enters the container.

`ggshield` inside the microVM only ever holds a placeholder value for `GITGUARDIAN_API_KEY`. When it calls the GitGuardian API, the sbx proxy rewrites the `Authorization: Token …` header with the real key (sourced from the host) on the wire, and denies any egress outside the kit's allowlist. The real key never enters the sandbox — not in the environment, shell history, or `ps` output.

## Usage

Store a GitGuardian API key once on the host (a Personal or Service Account key with the `scan` scope):

```console
sbx secret set gitguardian
```

Then create a sandbox with the kit. ggshield's AI hook is agent-specific, so set `gitguardian.agent` to match the agent you run:

```console
sbx run --kit "docker.io/sbx/gitguardian-kit:latest" claude
sbx run --kit "docker.io/sbx/gitguardian-kit:latest" --kit-arg gitguardian.agent=codex codex
```

`gitguardian.agent` defaults to `claude-code`, so you can drop `--kit-arg` when running `claude`.

Or target this repo directly over git, or a local clone:

```console
sbx run --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=gitguardian" claude
sbx run --kit ./gitguardian/ --kit-arg gitguardian.agent=cursor cursor
```

The `agent` arg accepts `claude-code`, `codex`, `copilot`, or `cursor` — the four agents ggshield's AI-hook support covers. Other sbx agents (gemini, droid, kiro, opencode) can still layer the kit on for the `ggshield` CLI and manual scanning, but there is no automatic enforcement hook for them.

## How auth works

The kit declares a `gitguardian` credential with one inject rule. Inside the container `GITGUARDIAN_API_KEY` is a sentinel (`proxy-managed`); on any request to `api.gitguardian.com` the proxy sets `Authorization: Token <your-real-key>`. The real key never reaches the sandbox filesystem or environment.

`gitguardian` is a custom (non-registry) service, so sbx does not auto-export the key sentinel into the container — it only sets `SBX_CRED_GITGUARDIAN_MODE`. ggshield refuses to run when `GITGUARDIAN_API_KEY` is unset, so the kit materializes the placeholder itself via `environment.variables`.

## What it installs

1. **`ggshield`** from a pinned, digest-verified GitHub release (version and per-arch SHA256 pinned in `spec.yaml`, no `curl | sh`). To bump, change `GGSHIELD_VERSION` and both checksums.
2. **The agent's AI hook**, via `ggshield machine setup --agent <agent> --no-git-hooks --no-honeytokens`, run as the agent user. This registers `PreToolUse` / `PostToolUse` / `UserPromptSubmit` handlers that run `ggshield secret scan ai-hook` inside the agent's own tool loop.

A blocked action means a real secret was detected — remove and rotate it, don't retry or bypass. Manual scans remain available as an escape hatch:

```console
ggshield secret scan path -r .      # scan the workspace files
ggshield secret scan repo .         # scan full git history + working tree
```

## EU workspace / self-hosted instances

The EU workspace and self-hosted GitGuardian instances use a different API host. Set `GITGUARDIAN_INSTANCE` for ggshield, and add that host to both `permissions.network.allow` and the credential's `inject` block in `spec.yaml` — the proxy only injects on, and only allows egress to, the hosts listed there.

## Cleanup

```console
sbx secret rm -g --service gitguardian
```
