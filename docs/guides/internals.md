# Internals

How the pieces work. Contributor-level detail and invariants are in [`CLAUDE.md`](../../CLAUDE.md); user-visible guarantees are in [`SPEC.md`](../SPEC.md).

## Maven repository

| Container path | Source | Lifetime |
| --- | --- | --- |
| `/root/.m2/repository/remote-cache` | host `~/.m2/repository` (RW bind mount) | persistent |
| `/root/.m2/repository/local-installs` | container writable layer | container |

The image enables Resolver's enhanced split local repository (`remotePrefix=remote-cache`, `localPrefix=local-installs`). Downloads land in the host cache at Maven's conventional paths, and `mvn install` output stays in the container and vanishes with it. Resolver's lock directory is `remote-cache/.locks`, the host's conventional location, so host and container writers coordinate.

Conventional Maven and Maven Wrapper get the settings through `MAVEN_OPTS`. The `mvnd`/`mvn` launcher additionally prepends them as Maven user properties to every mvnd request, so project-level `.mvn/mvnd.properties` and `.mvn/jvm.config` can't disable the split. If Maven is launched with the settings explicitly disabled, it falls back to the container's repository root and may download an ephemeral duplicate, but never writes into the host mount.

The host cache is mounted RW on purpose: any container process can modify it directly. Settings and credentials elsewhere in `~/.m2` are not mounted. There is no Maven volume to maintain.

## Overlay mounts

For each mount spec in `overlay` mode, the container gets an overlayfs at the target path:

| Layer | Source | Mode |
| --- | --- | --- |
| lower | host source path | ro |
| upper | per-worktree, per-target volume `maudebox-overlay-<basename>-<projhash>-<targethash>` | rw |
| workdir | sibling subdirectory in the same volume | rw |

Tools see host files as a read-only starting point while writes land in a worktree-scoped volume, so concurrent worktrees don't collide and the host isn't mutated. Declare as many overlays as you like. If a lowerdir is empty or absent, the entrypoint logs a notice and skips it.

Upper and work live in a Docker volume because kernel overlayfs refuses an overlay as upperdir/workdir and Docker's container rootfs is itself an overlay. Mounting needs `SYS_ADMIN` and `apparmor=unconfined`, which maudebox requests only when an overlay or `--fuse` asks for them. Every container also gets the much narrower `PERFMON` capability for `perf`.

Volume names derive from the project basename plus a SHA-256 prefix of its full path, so same-named worktrees don't collide. Volumes carry `maudebox.project` and `maudebox.overlay-target` labels; launch properties such as the instance are deliberately never stored on a volume.

## Container user model

The container starts as **root** (no `USER` directive) for privileged setup — overlay mounts, `/etc/passwd` rewrite, chowns — then diverges:

- **macOS / OrbStack:** stays root. virtiofs root-squashes bind mounts, so lowerdir files appear as `uid=0`, and overlay copy-up preserves that UID; a dropped-privilege user couldn't write the upperdir. Writes from root translate back to your host user.
- **Linux Docker:** drops to the host UID/GID via `setpriv` just before `exec`, because Linux bind mounts pass UIDs through and root writes would land root-owned on the host. The wrapper passes `HOST_UID`/`HOST_GID`; the entrypoint rewrites `/etc/passwd` so that UID's home is `/root` (Java's `user.home` reads `pw_dir`, not `$HOME`) and chowns the writable volume trees on first launch.

Everything lives under `/root` either way: Claude config, the Maven mounts, overlay targets, and a `/root/<basename>` symlink to the worktree. Because of the Linux privilege drop, `apt-get install`, `mount` etc. need `sudo` (passwordless for the runtime user; inert on macOS where you're already root).

## Container labels and instance state

Every container is named after its instance and stamped with `maudebox.instance`, `maudebox.project`, `maudebox.state` and `maudebox.ephemeral`; ephemeral runs also carry `maudebox.ephemeral-name` and `maudebox.state-dir`. The wrapper finds its containers by label, not name. Two concurrent containers with the same instance handle collide at `docker run`; `--instance` resolves that.

The per-instance state dir `${XDG_STATE_HOME:-~/.local/state}/maudebox/instances/<volume-name>/` holds a `manifest.toml` written by `maudebox new` (workspace kind, source repo, name, path). Its presence is how `maudebox rm` knows maudebox created the worktree. For ephemeral runs the dir is mounted at `/run/maudebox/` so `maudebox-keep` can drop the `keep` flag.

## Git and jj config sharing

Mounted read-only: `~/.gitconfig`, `~/.config/git/`, and `~/.config/jj/config.toml` (not the whole jj dir, whose `repos/` is host-path-keyed cache). Never mounted: `~/.ssh`, `~/.git-credentials`, `~/.gnupg`. `GIT_CONFIG_COUNT/KEY/VALUE` in the image force `commit.gpgsign=false` and `tag.gpgSign=false`, overriding mounted config.

The shell prompt (`prompt.sh`) shows the path and jj (change ID, closest bookmark) or git (branch or short hash) state, with `>` as the prompt character since neither `#` nor `$` is accurate after the Linux privilege drop.

## Building the image

```sh
cargo xtask image        # build the image
cargo xtask all          # wrapper + image
cargo xtask image --help # all pins
```

Defaults are the constants in `xtask/src/main.rs`; override with flags such as `--mvnd-version`, `--jj-version`, `--rust-version`, `--bun-version`, `--pnpm-version`, `--claude-version`, `--codex-version`, `--opencode-version`, `--playwright-version`, `--playwright-mcp-version` and `--tag`.

mvnd, jj, rustup, bun, pnpm and opencode downloads are verified against publisher SHA-256/SHA-512 metadata before extraction. The Claude installer is fetched over TLS to a file before running; upstream publishes no separate checksum through this interface.

## Repository layout

- `Cargo.toml`, `src/` — the host-side wrapper (single binary).
- `xtask/` — `cargo xtask` helper that drives `docker build`.
- `.cargo/config.toml` — the `xtask` alias.
- `rust-toolchain.toml` — pins the dev toolchain with `rustfmt` and Clippy.
- `.github/workflows/ci.yml` — fmt, Clippy, tests, ShellCheck, image build.
- `docker/` — everything the image is built from: `Dockerfile`; `entrypoint.sh` (overlays, UID drop, state symlinks, then `exec`); `prompt.sh`; `aliases.sh` (installs `MAUDEBOX_ALIASES`); `maudebox-keep`.
- `docs/` — `SPEC.md`, `decisions/`, and these guides.
