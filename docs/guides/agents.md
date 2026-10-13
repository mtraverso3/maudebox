# Agents and state

Claude Code, Codex and opencode run in the same container. This guide covers what persists, what is shared from the host, and how each agent is configured.

## State namespaces

Each launch selects one **state namespace**: a Docker volume holding logins and agent state, with isolated subtrees mounted at the paths each tool expects.

| Subtree | Mounted at | Holds |
| --- | --- | --- |
| `claude/` | `/root/.claude` | Claude login, plugin caches |
| `codex/` | `/root/.codex` | Codex login, `config.toml`, sessions, SQLite state DB (`$CODEX_HOME`) |
| `gh/` | `/root/.config/gh` | `gh` auth and config |
| `ssh/` | `/root/.ssh` | SSH keys, `known_hosts` |
| `opencode/config` | `/root/.config/opencode` | config, agents, commands, skills, plugins |
| `opencode/data` | `/root/.local/share/opencode` | login, session database, logs |
| `opencode/state` | `/root/.local/state/opencode` | locks guarding that database |
| `opencode/cache` | `/root/.cache/opencode` | LSP servers downloaded on demand |

Select one with `--state NAME` (also applies to `maudebox new`) or `state = "NAME"` in the config; the flag wins. With neither, the reserved `default` namespace uses the original `maudebox-state` volume. Named namespaces are labelled volumes such as `maudebox-state-work`. Names are 1–32 lowercase letters, digits or single hyphens, starting and ending with a letter or digit.

```sh
maudebox --state work .
maudebox --state personal new feature-x
maudebox state list          # namespaces and the instances using them
maudebox state rm work       # after all its containers stop; forgets every credential and session in it
```

**Log in once per namespace**: `claude`, `codex login`, `opencode auth login` (or `/connect`), `gh auth login`. Project removal and ephemeral cleanup never touch namespaces.

Notes on individual subtrees:

- **SQLite state (Codex, opencode).** Both keep WAL-mode databases. Sharing them across concurrent containers is safe only because a Docker named volume is VM-local ext4; never relocate them onto a host bind mount or overlay upperdir.
- **Credentials stay in the container.** No agent's credential store is mounted from the host, so a container can hold different or more limited credentials. Losing a namespace costs one re-login.
- **`ssh/`** is a container-only `~/.ssh`, starting empty. Generate a key with `ssh-keygen` and every launch in that namespace reuses it; the entrypoint resets the directory to `0700` each launch. The host's `~/.ssh` is never mounted.
- **`~/.claude.json`** lives outside `~/.claude/`, so the entrypoint symlinks it to `~/.claude/state.json` inside the volume.

## Global config from the host

Hand-written config is bind-mounted read-only (only items that exist), so it survives dropping a namespace:

| Agent | Host source |
| --- | --- |
| Claude | `~/.claude/` `CLAUDE.md`, `settings.json`, `agents/`, `commands/`, `skills/`, `plugins/` |
| Codex | `~/.codex/AGENTS.md`, `~/.agents/skills/` |
| opencode | `~/.config/opencode/` `AGENTS.md`, `tui.json`, `agent/`, `command/`, `skill/`, `plugin/` |

Host-path-keyed or session state (`projects/`, `todos/`, `statsig/`, `shell-snapshots/`) is deliberately not mounted. opencode's `opencode.json` is excluded because it holds provider API keys; add a `mounts` entry if you want it anyway. opencode also falls back to `~/.claude/CLAUDE.md` and auto-loads skills from `~/.claude/skills/` and `~/.agents/skills/`, so it needs nothing extra. Per-project `AGENTS.md`/`CLAUDE.md` is already inside the mounted worktree.

**Claude memory** is the one read-write carve-out: `~/.claude/projects/<encoded-cwd>/memory/` is bind-mounted so memories written in the container reach the host and vice versa. `--memory-from PATH` re-keys the host side to another project; `maudebox new` uses it so workspace memories accrue to the parent project.

## Sandbox defaults

The container is the sandbox, so each agent's own approval and sandbox layers are relaxed in the lowest-precedence place that works, leaving projects free to tighten them:

- **Claude:** `IS_SANDBOX=1`.
- **Codex:** `sandbox_mode = "danger-full-access"` and `approval_policy = "never"` in `/etc/codex/config.toml` (the lowest layer; your `~/.codex/config.toml` can override). Auto-update checks are off.
- **opencode:** `{"permission": "allow"}` seeded into the global `~/.config/opencode/opencode.json` on first launch if absent. This chiefly silences `external_directory` prompts, which fire constantly because a jj workspace's base repo lives outside the project. Edit that file or a project `opencode.json` to tighten. `opencode --auto` auto-approves for one session.

Auto-updaters are disabled for all three (`DISABLE_AUTOUPDATER`, `OPENCODE_DISABLE_AUTOUPDATE`, Codex config) so the pinned versions stay pinned.

## Managed MCP servers

Declare servers once in the user config as `[mcp.NAME]` tables and all three agents get them, with no per-container setup:

```toml
[mcp.my_http]
type = "http"
url = "http://host.docker.internal:8080/mcp"

[mcp.my_stdio]
type = "stdio"
command = "python"
args = ["-m", "my_server"]

[mcp.my_stdio.env]
DEBUG = "1"

# Bundled Playwright MCP server, driving the preinstalled Chromium.
# --headless: no display. --no-sandbox: already sandboxed.
[mcp.playwright]
type = "stdio"
command = "playwright-mcp"
args = ["--headless", "--no-sandbox"]
```

Each agent receives a translation written on the host under `$XDG_STATE_HOME/maudebox/` and bind-mounted read-only:

| Agent | File in container | Layer | Translation |
| --- | --- | --- | --- |
| Claude | `/etc/claude-code/managed-mcp.json` | managed scope (highest) | fields passed through verbatim; Claude validates |
| Codex | `/etc/codex/config.toml` → `[mcp_servers.NAME]` | system (lowest) | no `type`; `headers` → `http_headers`; unknown fields warned and dropped |
| opencode | `/etc/opencode/opencode.json` → `mcp` | managed (highest) | `"local"` or `"remote"` chosen by presence of `url`; local `command`+`args` become one `command` array, `env` → `environment`; inapplicable fields warned and dropped |

Escape hatches for agent-specific keys are `[mcp.NAME.codex]` and `[mcp.NAME.opencode]` sub-tables, passed through verbatim and overriding translated fields. Each agent's sub-table is stripped from the other agents' configs, so both can coexist:

```toml
[mcp.github]
type = "http"                       # Claude only; dropped elsewhere
url  = "https://api.example.com/mcp"
headers = { X-Env = "prod" }        # → http_headers for Codex

[mcp.github.codex]
bearer_token_env_var = "GITHUB_TOKEN"

[mcp.github.opencode]
timeout = 30000
```

The Claude and opencode files are written only when at least one `[mcp.*]` table exists; the Codex file is always written because it also carries the sandbox defaults. Managed scope is read-only in the container (`claude mcp remove` can't drop an entry). Verify with `claude mcp list`.

## Codex

Global instructions go in the host's `~/.codex/AGENTS.md`; per-project ones in the project's `AGENTS.md`.

### Host-side settings: `[codex]`

Settings you want in every container belong in a `[codex]` table. Anything set inside the container dies with the namespace, and there is no way to seed it from the host.

```toml
[codex]
model_provider = "litellm"
model = "my-model"
model_reasoning_effort = "high"

[codex.model_providers.litellm]
name = "LiteLLM"
base_url = "http://host.docker.internal:4000/v1"
wire_api = "responses"                             # Codex 0.149+ rejects "chat"
http_headers = { Authorization = "Bearer sk-..." }
```

It is mounted at `/etc/codex/managed_config.toml`, Codex's enterprise-managed layer, because Codex silently ignores `model_provider`/`model_providers` in the system and project layers. **That layer outranks `~/.codex/config.toml` and `-c`**, so keys here are pinned until you edit the table and relaunch. Keep per-session settings out of it. `codex.mcp_servers` is rejected; use `[mcp.NAME]`. Prefer env-var-based credentials in `[codex.*]` tables over literal tokens, since the file is plain text on the host.

### Remote control

Codex is installed in the official installer's "standalone" layout because `codex remote-control start` requires it. The daemon lets the ChatGPT app (mobile, web, desktop) drive a session in the container over an outbound websocket authenticated by `codex login`; nothing is published.

```sh
codex remote-control pair     # once; enrollment persists in the namespace
codex remote-control start
codex --remote unix://        # optional: attach a local terminal in another pane
```

The container hostname is `$MAUDEBOX_INSTANCE`, so a paired worktree shows up under its handle in your device list. The daemon's runtime directory `~/.codex/app-server-control` (a startup lock and Unix socket) is a per-container tmpfs, so each container starts or attaches to its own daemon rather than reusing another's. Auth, config and history are shared within a namespace.

## opencode

A pinned native binary. Its credentials are deliberately separate from the host's (nothing under its data directory is mounted).

**Remote access.** `opencode serve --hostname 0.0.0.0 --port 4096` starts a headless server; `opencode attach http://<host>:4096` connects a TUI and `opencode web` serves a browser UI. Set `--username`/`--password` (or `OPENCODE_SERVER_USERNAME`/`OPENCODE_SERVER_PASSWORD`) before binding to `0.0.0.0`. Maudebox publishes no ports:

- **Same host:** use the container's IP (routable on Linux; OrbStack also resolves `<instance>.orb.local`; Docker Desktop for Mac doesn't route container IPs). `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <instance>` prints it.
- **Another container:** join a shared network with `network` and use the instance name.
- **Another machine:** not currently supported.

For a second terminal on the same machine, `docker exec -it <instance> opencode` is simpler.

## Terminal integration

The wrapper forwards `TERM`, `TERM_PROGRAM`, `TERM_PROGRAM_VERSION`, `COLORTERM` and iTerm2's `LC_TERMINAL*`.

- **`TERM_PROGRAM`** selects an escape dialect; TUIs use it to decide whether to wrap clipboard writes in tmux's DCS passthrough. It is forced to `tmux` inside tmux, since tmux only exports it itself from 3.2.
- **`TERM`** selects capabilities. `docker run -it` otherwise defaults to a bare `xterm`. The image includes `ncurses-term`, and the entrypoint falls back to `xterm-256color` with a notice if the forwarded value has no terminfo entry, since ncurses programs abort on an unknown `TERM`. Enhanced key reporting is negotiated at runtime and doesn't depend on `TERM`.
- **Window titles.** The terminal is named after the instance: the entrypoint sets it, `prompt.sh` re-asserts it each prompt, and under tmux the wrapper runs `tmux rename-window "$MAUDEBOX_INSTANCE"` host-side (which also disables `automatic-rename`). opencode would overwrite the title with `OpenCode`, notably under iTerm2's `tmux -CC`, so the image sets `OPENCODE_DISABLE_TERMINAL_TITLE=1`; unset it per launch to get opencode's titles back.
- **Clipboard.** The only route out of a container is OSC 52, which needs `set -g set-clipboard on` and `set -g allow-passthrough on` in tmux plus your terminal's permission.

**Known issue: iTerm2 tmux control mode.** Under `tmux -CC`, ctrl-chords can reach a TUI as literal text such as `0x10` or `0x3`. This is an iTerm2 bug ([fix](https://github.com/gnachman/iTerm2/commit/dc0587cb3a68128f15799407a8cb8391ad7ab70e), in 3.7.0 betas): with tmux extended keys on and an application requesting `modifyOtherKeys` mode 1, `send-keys` writes the argument text into the pane. opencode requests mode 1; Claude Code and vim use mode 2 and Codex disables it, so they are unaffected. Workarounds: `set -s extended-keys off` in tmux, attach with plain `tmux attach`, or use iTerm2 3.7.0beta6+.
