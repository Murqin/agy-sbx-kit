# Antigravity CLI (`agy`) Sandbox Kit

Docker Sandboxes kit for running [Google's Antigravity CLI](https://antigravity.google/product/antigravity-cli) (`agy`) inside an isolated sandbox.

The kit installs the official `agy` binary and forces its headless OAuth flow so authentication works without a local browser. `agy` stores its OAuth token under `~/.gemini/antigravity-cli/`; on `sbx` v0.30+ sandbox home directories are persistent by default, so a single sign-in survives across `sbx run` invocations.

## Quick start

```bash
sbx run --kit git+https://github.com/shelajev/agy-sbx-kit.git agy .
```

On the first run `agy` prints a Google OAuth URL. Open it in a browser on your laptop, complete the Google sign-in, then paste the callback URL (or code) back into the sandbox terminal. After that the credentials are cached on the sandbox's persistent volume and subsequent runs go straight to the agent.

## Named sandbox

For a sandbox you can reattach to later:

```bash
sbx create --name agy-current \
  --kit git+https://github.com/shelajev/agy-sbx-kit.git agy .

sbx run --kit git+https://github.com/shelajev/agy-sbx-kit.git agy-current
```

For custom agent kits, pass `--kit` again when re-running an existing sandbox if `sbx` does not resolve it automatically.

## How auth works

`agy`'s local auth path tries to open a browser and listen on `localhost:36742` for the OAuth callback — neither of which makes sense from inside a sandbox. The kit sets `SSH_CONNECTION` so `agy` detects a "remote" environment and switches to its copy/paste fallback:

1. CLI prints an `accounts.google.com` authorization URL.
2. You open it in any local browser, sign in, approve.
3. Google redirects you to a `http://localhost:36742/oauth-callback?code=...` URL that won't load — that's expected.
4. Copy the full URL from your browser's address bar and paste it back at the sandbox prompt.

The OAuth token lands at `~/.gemini/antigravity-cli/antigravity-oauth-token` (alongside conversations, settings, and an update lock). The home directory is persistent by default on `sbx` v0.30+, so the token survives across `sbx run` restarts.

To log out from inside the sandbox, run `/logout` at the `agy` prompt.

## How it works

- **Install (once at sandbox creation):**
  - `curl -fsSL https://antigravity.google/cli/install.sh | bash` — downloads the platform-specific `agy` binary into `~/.local/bin/agy` and runs the binary's `install` step for shell wiring.
- **Entrypoint:** `agy` (the shell-docker image puts `~/.local/bin` on PATH).
- **Persistence:** Inherited from `sbx` defaults — the home directory persists across runs, so the OAuth token doesn't need to be re-issued.
- **Self-update:** `agy` self-updates in the background; the updater domain is allowlisted.

## Network policy

The kit allows only:

- `antigravity.google` — installer and docs
- `antigravity-cli-auto-updater-974169037036.us-central1.run.app` — release manifests and binaries (also used for self-update)
- `accounts.google.com`, `oauth2.googleapis.com`, `www.googleapis.com` — Google OAuth
- `cloudaicompanion.googleapis.com`, `cloudcode-pa.googleapis.com`, `generativelanguage.googleapis.com` — Antigravity / Gemini Code Assist APIs

If your workflow needs to reach package registries (npm, PyPI, crates.io, Go modules, etc.) or your own services, fork the kit and extend `network.allowedDomains` in `spec.yaml`.

## Smoke test

For `sbx exec`, close or pipe stdin so the CLI does not block waiting for input:

```bash
sbx exec agy-current -- sh -lc 'agy --help < /dev/null'
```

You should see the standard `agy` help text. Hitting the actual model requires an authenticated session, so the first interactive run still needs the OAuth paste-back.

## Local clone

If you clone this repo, `run.sh` runs a named sandbox using the local kit path:

```bash
./run.sh agy-current
```

Use any sandbox name as the first argument:

```bash
./run.sh my-sandbox
```

## License

Apache 2.0. See [LICENSE](LICENSE).
