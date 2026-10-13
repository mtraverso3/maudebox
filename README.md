# maudebox

A Docker image and host-side wrapper for running coding agents — [Claude Code](https://claude.ai/code), [Codex](https://github.com/openai/codex) and [opencode](https://opencode.ai) — against a project in an isolated, reproducible Linux environment, with Maven, Rust and JavaScript toolchains ready to go.

```sh
maudebox . claude      # Claude Code, sandboxed to this project
maudebox . mvnd verify # one-shot Maven build
maudebox new feature-x # fresh jj workspace / git worktree, in its own container
```

## What's in the image

| Area | Contents |
| --- | --- |
| Java | Eclipse Temurin JDK 26; [`mvnd`](https://github.com/apache/maven-mvnd) (also installed as `mvn`) |
| Rust | rustup-managed toolchain (default 1.98.0) and `build-essential` |
| Python | Python 3 with `pip`, `venv` and headers (`python` → `python3`) |
| JavaScript | [`bun`](https://bun.sh) and [`pnpm`](https://pnpm.io) (pnpm bundles its own Node.js) |
| Browser | [Playwright](https://playwright.dev) CLI with Chromium, plus the [`@playwright/mcp`](https://github.com/microsoft/playwright-mcp) server (`playwright-mcp`), both run under `bun` |
| VCS | `git`, [`jj`](https://github.com/jj-vcs/jj), GitHub CLI (`gh`) |
| Agents | Claude Code, Codex CLI, opencode |
| Utilities | `ripgrep`, `jq`, `vim`, `less`, `sudo`, `rsync`, OpenSSH client |
| Profiling | `perf` (container gets only `CAP_PERFMON`; profiling beyond the host's `perf_event_paranoid` needs `sudo perf`) |
| FUSE | `fusermount3`; mounting needs `--fuse` ([details](docs/guides/configuration.md#fuse-filesystems)) |

Pinned versions live in `xtask/src/main.rs`. The image builds natively on amd64 and arm64.

## Why

- **Sandboxed agents.** The container sees only the project directory (plus the base repo of a jj workspace or git worktree), explicitly configured mounts, and the host's Maven cache. `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.git-credentials`, sibling projects and your shell config are never mounted, and commit signing is forced off so a host signing setup can't fail or be misused. The container has its own `~/.ssh`, persisted but never populated from the host.
- **Conflict-free snapshot installs.** Maven Resolver's split local repository sends third-party downloads to a warm cache shared with the host and `mvn install` output to the container's own layer, so concurrent worktrees never see each other's `-SNAPSHOT` jars. The shared cache is mounted read-write on purpose; this is not a security boundary against a hostile process. See [Maven repository](docs/guides/internals.md#maven-repository).
- **Log in once.** Claude, Codex, opencode, `gh` and SSH state live in a persistent *state namespace*, and your hand-written global agent config is mounted read-only. Agent credentials are never taken from the host. See [Agents and state](docs/guides/agents.md).

## Quick start

Prerequisites: Docker 25+ (tested with OrbStack on macOS; Docker Desktop and native Linux also work) and a Rust toolchain (1.85+) to build the wrapper.

```sh
cargo build --release   # wrapper at target/release/maudebox
cargo xtask image       # docker image, tag "maudebox"
cargo xtask all         # both
```

Put the wrapper on your `$PATH` (`cargo install --path .`, or symlink `target/release/maudebox`), then:

```sh
maudebox                          # interactive shell, current directory
maudebox /path/to/project         # interactive shell, specific project
maudebox . claude                 # launch Claude Code
maudebox --state work . claude    # independent login/session namespace
maudebox list                     # projects and their running instances
maudebox rm /path/to/project      # remove a project's volumes and state
```

`maudebox --help` lists every option; `cargo xtask image --help` lists the version pins.

## Documentation

| Guide | Covers |
| --- | --- |
| [Usage](docs/guides/usage.md) | Launching, instance names, project names, `maudebox new`, ephemeral workspaces, cleanup |
| [Configuration](docs/guides/configuration.md) | `config.toml`, mounts, aliases, default command, networking, FUSE |
| [Agents and state](docs/guides/agents.md) | State namespaces, logins, global config, managed MCP servers, Codex, opencode, terminal integration |
| [Internals](docs/guides/internals.md) | Maven repository, overlays, container user model, repository layout, building the image |
| [Product specification](docs/SPEC.md) | Source of truth for user-visible behavior and deliberate limits |
| [Decision records](docs/decisions/) | Why the non-obvious choices were made |

Contributors: start with [`CLAUDE.md`](CLAUDE.md).
