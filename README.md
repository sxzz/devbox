# devbox

A declaratively maintained development environment for coding agents, built with Docker: a paseo daemon plus a shared Chromium, running on the local OrbStack.

- **The repo holds only declarations**: the image, tool versions, agent config and compose file all live here. Changes take effect after `scripts/rebuild`.
- **State lives outside the repo**: `$DATA_DIR` (default `~/app/paseo`) holds credentials, the browser profile and workspaces. Rebuilds don't touch it.
- **Isolation**: the container mounts only directories under `$DATA_DIR`. It does not mount `~/.ssh`, `~/.config/gh` or `docker.sock`. Other services may run on the same Docker daemon, so never give the container Docker access.

## Layout

| Path | Purpose |
| --- | --- |
| `image/Dockerfile` | Based on the official paseo image (pinned by digest); installs mise and apt packages |
| `image/mise.toml` | Tool list: gh, pnpm, claude-code, codex, pi, playwright-mcp, all at latest |
| `image/apt.txt` | Extra Debian packages |
| `config/AGENTS.md` | Instructions for agents, synced on start to the global instruction files of claude, codex and pi |
| `config/mcp.json` | MCP servers, merged into `mcpServers` in `~/.claude/.claude.json` and `~/.pi/agent/mcp.json` |
| `config/codex.toml` | Managed block written into `~/.codex/config.toml` |
| `config/gitconfig` | Included from `~/.gitconfig`: git defaults for agents and the gh credential helper |
| `config/claude-settings.json` | Deep-merged into `~/.claude/settings.json`; keys here override existing values |
| `image/rootfs/usr/local/bin/devbox-activate` | Runs on every container start and syncs `config/` into home |
| `compose.yml` | Two services, `browser` (Chromium) and `paseo`; paseo shares the browser's network |

Config falls into two kinds:
- Files and blocks managed by `config/` are overwritten on every start. Change them in the repo, not inside the container.
- Everything else (login credentials, state agents write at runtime) is kept in `$DATA_DIR/paseo-home`.

## Daily use

```sh
scripts/rebuild           # build from the current tree, tag devbox:<commit>, then restart
scripts/rollback          # list existing tags
scripts/rollback <tag>    # switch back to an older image
scripts/shell             # open a shell in the container as the paseo user
scripts/shell gh auth status
```

Upgrading tools: everything in `image/mise.toml` is latest, so every `scripts/rebuild` reinstalls the newest versions. If an upgrade breaks something, use `scripts/rollback` to return to an older image. Upgrading paseo: replace the digest in `image/Dockerfile` with a new one (look it up with `docker buildx imagetools inspect ghcr.io/getpaseo/paseo:latest`), then run `scripts/rebuild`.

## Access

| Service | Address |
| --- | --- |
| paseo daemon | `127.0.0.1:6767`, or `$TAILSCALE_IP:6767` over Tailscale; password is `PASEO_PASSWORD`. There is no bundled web UI; use the hosted app.paseo.sh, the desktop app or the mobile app |
| Browser desktop | `https://127.0.0.1:6901`, self-signed certificate, username `devbox`, password is `BROWSER_PASSWORD` |

app.paseo.sh, the mobile app and the desktop app can all connect directly with the address and password above (from inside the tailnet). For devices outside Tailscale, run `scripts/shell paseo daemon pair` to enable the relay, then paste the pairing link.

For everyday chats that don't involve code, use the `/workspace/chat` project (a plain directory, not a git repo).

## First-time setup

1. `cp .env.example .env`, generate the two passwords and fill them in along with `TAILSCALE_IP`, run `chmod 600 .env`, then run `scripts/rebuild`.
2. Agent login (credentials are stored in `$DATA_DIR/paseo-home`): run `scripts/shell`, then `claude`, `codex` and `pi` in turn.
3. GitHub bot:
   - Register a bot account and add it as a collaborator with Write access to the target repos.
   - Fill in `DEVBOX_GIT_NAME`/`DEVBOX_GIT_EMAIL` in `.env`, then run `scripts/rebuild`.
   - Run `scripts/shell gh auth login` (with a fine-grained PAT). The credential helper is already set up by `config/gitconfig`; there's no need to run `gh auth setup-git`.
4. Browser: open the browser desktop and sign in to the sites you need by hand. Agents use the same browser session through the playwright MCP (CDP `127.0.0.1:9222`); claude, codex and pi are all configured for it.

## Known limitations

- There is no Docker in the container; agents can't run `docker`.
- paseo's built-in browser tools need a connected browser host (the desktop app). This deployment doesn't enable it; browser automation goes through the playwright MCP.

## License

[MIT](./LICENSE)
