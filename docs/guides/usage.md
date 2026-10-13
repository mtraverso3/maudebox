# Usage

Day-to-day commands. Run `maudebox --help` for the full option list.

```sh
maudebox                          # interactive shell, current directory
maudebox /path/to/project         # interactive shell, specific project
maudebox . mvnd verify            # one-shot Maven build
maudebox . claude                 # launch Claude Code
maudebox --state work . claude    # use an independent login/session namespace
maudebox --tag my-tag . bash      # use a non-default image tag
maudebox list                     # list projects and running instances
maudebox state list               # list persistent state namespaces
maudebox keep feature-x           # don't tear down on exit (running ephemeral)
maudebox rm /path/to/project      # remove volumes + state dir (and the workspace, if maudebox created it)
```

The project is bind-mounted at its **original host path** (e.g. `/Users/martin/projects/trino/workspaces/trino.lateral`). The entrypoint also creates a `/root/<basename>` symlink to it and starts the shell there, so `$PWD` is short while the filesystem is still the host-path mount.

If the project is a **jj workspace** or **git worktree**, maudebox reads its metadata (`.jj/repo` or `.git`), finds the base repo, and mounts that at its host path too, so the absolute paths recorded in the worktree resolve and `jj`/`git` work normally.

## Instance names

The project basename is exposed as `$MAUDEBOX_INSTANCE` and used as the container name and hostname, so `docker ps` shows it and `docker exec trino bash` works. Aliases such as `claude --remote-control "$MAUDEBOX_INSTANCE"` use the same handle.

For a second concurrent session on one project (say, one to code and one to review), add a discriminator:

```sh
maudebox . claude                         # MAUDEBOX_INSTANCE=trino
maudebox --instance review . claude       # MAUDEBOX_INSTANCE=trino-review
```

The discriminator is appended to the basename, so the basename stays a stable prefix. It also feeds the container name and `maudebox.instance` label. Overlay volumes and the state dir are keyed on the project path only, so both sessions share overlay state; for isolation use `maudebox new` to get a second worktree.

## Referring to projects by name

`maudebox <name>` and `maudebox new --source <name>` accept a bare name instead of a path, resolved in order:

1. **Workspaces created by `maudebox new`**, matched by the name passed to `new` (see [Reattaching](#reattaching-to-a-non-ephemeral-workspace)).
2. **`project-roots`** from the [user config](configuration.md#user-config): the first `<root>/<name>` that exists.

```toml
project-roots = ["~/Projects", "~/work"]
```

With `~/Projects/trino` on disk, `maudebox trino` launches it. A name matching more than one root or workspace is reported as ambiguous. Anything containing `/` or starting with `/`, `.` or `~` is always a literal path.

## Spawning a workspace: `maudebox new`

`maudebox new <name>` creates a fresh jj workspace or git worktree off a project and launches a container on it. The workspace persists after exit unless you pass `--ephemeral`. Anything after `<name>` is the command to run.

```sh
maudebox new feature-x                       # new workspace off the default branch
maudebox new feature-x --source trino        # ...from another project (path or name)
maudebox new feature-x --from main           # start from a specific revision
maudebox new feature-x --fetch               # fetch first, then branch off the default
maudebox new feature-x --path /tmp/scratch   # custom target path
maudebox new feature-x --ephemeral           # tear down workspace + volumes on exit
maudebox new feature-x claude                # spawn the workspace, run a command in it
```

Defaults:

- **Source:** the current directory; override with `--source`.
- **Target path:** `<project>/../<basename>.<name>`; override with `--path`.
- **Starting revision:** the project's default branch — jj's `trunk()`, or the branch `origin/HEAD` points at for git (falling back to probing `origin/main`/`origin/master`). `--from @-` (jj) or `--from HEAD` (git) branches from the current checkout instead. For git a new branch named `<name>` is created. `--fetch` refreshes remote-tracking refs first.
- **VCS:** jj wins when both `.jj/` and `.git` exist.

Memory written by Claude in a workspace is attributed to the source project, so short-lived workspaces don't scatter memories (see `--memory-from`).

### Reattaching to a non-ephemeral workspace

```sh
maudebox new feature-x         # creates <project>/../<basename>.feature-x
exit
maudebox feature-x             # re-enters the same workspace
maudebox feature-x mvnd verify # ...or run a one-shot command in it
```

The name must match exactly one workspace whose directory still exists; ambiguity bails out asking for the full path, and a miss falls through to normal path handling. Workspaces created by older versions (before the path was recorded in the manifest) can't be reattached by name.

### Ephemeral mode

With `--ephemeral`, the workspace is torn down when the container exits — cleanly, on error, or on Ctrl-C — by delegating to `maudebox rm`:

- **jj:** `jj workspace forget <name>` and `rm -rf <target>`. The change at `@` stays recoverable via `jj op log` / `jj op restore`.
- **git:** `git worktree remove --force <target>` and `git branch -D <name>`. Branch tip commits remain in the reflog (default 90 days).
- **Volumes:** the project's `maudebox-overlay-…` volumes.
- **State dir:** `$XDG_STATE_HOME/maudebox/instances/…`, removed last.

Uncommitted working-copy changes are **not** preserved. Commit first.

To change your mind mid-session, run `maudebox-keep` inside the container, or `maudebox keep <id-or-name>` on the host (container ID from `maudebox list`, instance basename, or the name passed to `new`). This drops a `keep` flag; on exit only the flag is removed, leaving the workspace, volumes and manifest intact so a later `maudebox rm <name>` can still do the full teardown. Both forms are no-ops for non-ephemeral instances. Local Maven installs still vanish with the container.

## Listing and removing

`maudebox list` shows one row per **project** — the worktree plus whatever maudebox keeps for it. A project appears if it is running, still owns a volume, or was created by `maudebox new`. The `INSTANCES` column lists the containers running against it right now (`-` when none), annotated with their state namespace, e.g. `trino [default], trino-review [work]`. Those handles are the container names that `docker exec` and `maudebox keep` take.

`maudebox rm <project>` removes every volume labelled with that project and its state dir, and — only when a manifest from `maudebox new` exists — the jj workspace / git worktree. A path launched directly has no manifest, so its worktree is left alone. Everything `rm` removes is shared by all instances of the project, so it is refused while any container is running against it. The argument is a project path or basename; a running container's ID or handle also works.

State namespaces have their own lifecycle and are never removed by project cleanup:

```sh
maudebox rm /path/to/project
maudebox state rm default     # forget that namespace's credentials and sessions
docker rmi maudebox           # remove the image
```
