# Configuration

## User config

Persistent settings live in `$XDG_CONFIG_HOME/maudebox/config.toml` (default `~/.config/maudebox/config.toml`) and apply to every launch.

```sh
maudebox config edit    # open in $VISUAL, then $EDITOR, then vi (run via `sh -c`, so "code --wait" works)
maudebox config path    # print the path
```

Values are type-checked on read, and an invalid one stops the launch naming the key and array index:

| Key | Type |
| --- | --- |
| `mounts`, `project-roots`, `default-command` | array of strings |
| `network` | string or array of strings |
| `state` | string |
| `fuse` | boolean |
| `[aliases]` | table of strings |
| `[mcp.NAME]`, `[codex]` | tables, see [Agents and state](agents.md) |

Changes affect only **new** containers; a running one keeps the snapshot it launched with.

## Extra bind mounts

By default maudebox mounts only the project, the Maven repository and the selected state namespace. Anything else (`~/.aws`, a notes directory, a shared cache) is declared explicitly, either per launch with the repeatable `--mount` flag or in the config:

```sh
maudebox --mount ~/.aws:~/.aws:ro --mount ~/Documents/notes:~/notes
```

```toml
mounts = [
    "~/.aws:~/.aws:ro",
    "~/Documents/notes:~/notes",
    "~/.gradle:~/.gradle:overlay",
]
```

Syntax is `HOST:CONTAINER[:ro|rw|overlay]`. A leading `~` is `$HOME` on the host side and `/root` on the container side. Mode defaults to `rw`. CLI mounts are forwarded to the inner container by `maudebox new`.

**Overlay mode** layers a per-worktree writable Docker volume over the read-only host source: each worktree gets isolated writes while sharing the host content as warm starting state. It suits caches such as `~/.cargo`, `~/.gradle` and `~/.npm`. See [Internals](internals.md#overlay-mounts). Maven doesn't need one; an existing `~/.m2:~/.m2:overlay` entry conflicts with the built-in mounts and must be removed:

```sh
maudebox mount rm ~/.m2:~/.m2:overlay
```

Manage the `mounts` array from the CLI:

```sh
maudebox mount add ~/.gradle:~/.gradle:overlay  # append (idempotent)
maudebox mount list
maudebox mount rm  ~/.gradle:~/.gradle:overlay  # exact match, copy from `list`
```

Only the `mounts` array is rewritten; comments and other content are preserved.

## Aliases

Bash aliases installed inside the container, from the `[aliases]` table:

```toml
[aliases]
cl = "claude --dangerously-skip-permissions --remote-control $MAUDEBOX_INSTANCE"
build = "mvnd verify"
```

```sh
maudebox alias add cl 'claude --dangerously-skip-permissions --remote-control $MAUDEBOX_INSTANCE'
maudebox alias list
maudebox alias rm cl
```

Single-quote the value on the host so `$VAR` references stay literal and expand at invocation, picking up the container's environment. `add` overwrites an existing name. Only the `[aliases]` table is rewritten.

Aliases also work non-interactively: `maudebox <path> <name> <args>` detects the alias as the first argument and runs `bash -c '<value> "$@"' <name> <args>`.

## Default command

With no command, `maudebox` and `maudebox new` open a shell. To launch something else:

```toml
default-command = ["claude", "--dangerously-skip-permissions"]
```

An explicit command always wins, so `maudebox . bash` still gives a shell. The default applies to plain `maudebox`, `maudebox <name>` and `maudebox new <name>`.

## Reaching other services

### Host services

The container reaches host services at `host.docker.internal` (wired up with `--add-host=host.docker.internal:host-gateway`, so it works on Linux too). `localhost` is the container itself. For an HTTP MCP server, either put the URL in a project-level `.mcp.json` or run `claude mcp add --transport http <name> http://host.docker.internal:<port>` once; it persists in the state namespace.

### Services in other containers

`host.docker.internal` only reaches ports published to the host. To reach a service (typically a docker-compose project) without publishing it, join its network:

```sh
maudebox --network myservice_default . claude
maudebox --network app_default --network db_default . claude   # several
```

Compose names its network `<compose-project>_default` (check `docker network ls`). The service is then reachable by compose service name and container port, e.g. `http://app:8080`. The networks must already exist.

```toml
network = "myservice_default"            # single
network = ["app_default", "db_default"]  # several
```

Any `--network` flag replaces the config key entirely; they don't merge.

## FUSE filesystems

Mounting `sshfs`, `rclone`, `gocryptfs` and similar from inside the container is off by default; enable it per launch or in the config:

```sh
maudebox --fuse . bash
```

```toml
fuse = true
```

This exposes the host's `/dev/fuse` and adds `SYS_ADMIN` plus `--security-opt apparmor=unconfined` — the same privileges overlays need, and why it isn't the default. `fusermount3` is setuid root, but a setuid binary only gains capabilities in the container's bounding set, and Docker's default seccomp and AppArmor profiles block mounting without them.

The image ships `fuse3` (with `user_allow_other`) but no clients:

```sh
sudo apt-get update && sudo apt-get install -y sshfs
mkdir -p ~/remote && sshfs user@host:/srv ~/remote
```

Mounts live and die with the container, and `--fuse` fails at `docker run` if the host has no `/dev/fuse` (load the `fuse` module first).
