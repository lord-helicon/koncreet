# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Koncreet is a plain-Bash, first-hour hardening tool for a fresh Linux VPS. It targets **Debian 12/13 and Ubuntu 22.04/24.04 only**, and every mutating command refuses to run anywhere else (`koncreet_require_supported_os` exits 2). There is no build step and there are no dependencies beyond bash 4+ and coreutils.

## Commands

```bash
bash tests/run.sh                 # unit tests (plain assert runner, no bats)
bash tests/smoke-dry-run.sh       # version + doctor + dry-run apply of share/koncreet.conf.example
docker run --rm -v "$PWD":/k -w /k -e KONCREET_INTEGRATION_OK=1 ubuntu:24.04 bash tests/integration.sh   # real sshd/sudo/ufw scenarios
shellcheck -S error koncreet install.sh lib/*.sh modules/*.sh   # what CI enforces
```

- **Bash 4+ is required.** macOS `/bin/bash` is 3.2 and fails on `declare -A` and `mapfile` (e.g. `modules/firewall.sh: http: unbound variable`). Run with Homebrew bash or in a container: `docker run --rm -v "$PWD":/k -w /k debian:12 bash tests/run.sh`.
- `tests/run.sh` is one sequential script with no per-test filter. To check a single area, source the relevant `lib/`/`modules/` files in a subshell and call the function directly, the way the test file does.
- CI runs ShellCheck at `-S error` only, because `KONCREET_CFG_*` vars set in `lib/config.sh` and read elsewhere trip SC2034 across files. Don't "fix" those warnings by removing variables.
- `tests/integration.sh` is destructive (creates users, rewrites `/etc/ssh`) and refuses to run without `KONCREET_INTEGRATION_OK=1`; only run it in a throwaway container. It stubs `systemctl`/`timedatectl`/`swapon` because containers have no systemd, so real service reloads and ufw enabling are still untested. When fixing a lockout bug, add a scenario there and check it fails against the old code.
- Smoke CI runs in `debian:12` and `ubuntu:24.04` containers without root, systemd, or ufw, so dry-run paths must not require any of them.

## Releasing

`VERSION` is the single source of truth. Pushing tag `v$(cat VERSION)` triggers `.github/workflows/release.yml`, which checks that the tag matches `VERSION`, runs the tests, builds `koncreet.tar.gz`, and publishes it with `install.sh`. Add a `CHANGELOG.md` entry with each version bump. `tests/run.sh` asserts that `VERSION` is `X.Y.Z` and that `koncreet version` prints `koncreet X.Y.Z`.

## Architecture

`koncreet` is the entrypoint. It resolves `KONCREET_ROOT` through symlinks (it gets installed as `/usr/local/bin/koncreet -> /opt/koncreet/koncreet`), sources every `lib/*.sh` and `modules/*.sh`, then runs `parse_globals` and `main`. Global flags (`-n/-y/-c/-v`) can appear anywhere. Unknown dashed args are passed through to module parsers such as `--user` and `--public`.

**Three entry paths call the same module functions.** If you change a module's signature, update all three in `koncreet`:
1. `dispatch_module`: `koncreet <module> <sub> [args]`
2. `cmd_apply_config`: `koncreet apply -c file`, driven by `modules=` in config
3. `cmd_menu`: the interactive menu, including "Run everything"

**Module contract.** Each `modules/<name>.sh` exposes some subset of:
- `<name>_plan_lines ARGS`: prints human-readable plan lines to stdout, with no side effects. Callers collect them with `mapfile` and pass them to `maybe_confirm_plan`, which prints the plan, stops there on dry-run, and otherwise asks for confirmation.
- `<name>_apply ARGS`: does the work. It must route every mutation through `run_cmd` / `write_file` / `backup_file` (in `lib/common.sh`) so that `KONCREET_DRY_RUN=1` turns it into `PLAN:` lines. Guard any direct commands with `if [[ "$KONCREET_DRY_RUN" -eq 1 ]]; then plan ...; return 0; fi`.
- `<name>_status`, `<name>_undo`: undo removes only koncreet-owned files. It never deletes users, swap, or the timezone.

**Reversibility conventions.** Changes go into koncreet-named drop-ins (e.g. `/etc/ssh/sshd_config.d/00-koncreet.conf`, sysctl/journald drop-ins) rather than edits to distro files. `write_file` backs up anything it overwrites to `*.koncreet.bak`. The firewall snapshots `/etc/ufw` to `/var/lib/koncreet/ufw.rules.before.tgz` before it enables ufw.

**Lockout-safety gates are core behaviour, not incidental checks:**
- `ssh apply` refuses unless `koncreet_ssh_harden_gate` (`lib/sshd.sh`) finds a non-root user with a real `authorized_keys` entry **and** sudo. If baseline generated that user's sudo password (`/root/USER.koncreet-password`), it asks for confirmation that the password was saved.
- sshd keeps the *first* value it reads, and `sshd_config.d/*.conf` is read in lexical order, so the SSH drop-in is `00-` prefixed to beat e.g. `50-cloud-init.conf`. After `sshd -t`, `ssh apply` checks the effective values with `sshd -T` (`koncreet_sshd_mismatches`) and rolls back if anything overrides them. `ssh status` reports effective values, not the drop-in's contents.
- The firewall always allows the SSH ports sshd actually listens on (`koncreet_ssh_listen_ports`) before it enables ufw. Sensitive services (`KONCREET_FW_SENSITIVE`: mysql, vsftpd) need `--public` / `firewall_public=true`. Custom ports must be `N/tcp` or `N/udp`, and a bare `8080` is rejected.
- Config and menu flows plan and run SSH hardening only when the gate already passes. Otherwise they print a SKIP line.

**lib/ roles:**
- `common.sh`: logging, `die`, `ask`/`confirm`, `run_cmd`/`write_file`/`backup_file`. `ask`/`confirm` honour `KONCREET_YES` and fall back to defaults or "no" when stdin isn't a TTY. Logs go to `/var/log/koncreet.log` as root, or to `$KONCREET_ROOT/koncreet.log` on dry-run or non-root runs.
- `ui.sh`: all terminal output (`ui_*`, spinners via `ui_run_quiet`, change-plan rendering). It respects `NO_COLOR` and `TERM=dumb`. Modules should print through `ui_*`/`log_*` and never through raw `echo`. `ui_*` helpers write to stderr, so never call them with `2>/dev/null` (`_log_show` in `common.sh` handles `ui.sh` not being loaded).
- `os.sh`: OS detection and gating, `pkg_install`, the SSH unit name (`ssh` vs `sshd`), and the distro-specific unattended-upgrades origins: Debian uses `Origins-Pattern`, Ubuntu uses `Allowed-Origins`.
- `config.sh`: a strict `key=value` parser. It never `eval`s or sources the file, and it rejects unknown keys. Values land in `KONCREET_CFG_*`. To add a config key, update `koncreet_config_defaults`, the `case` in `koncreet_config_load`, `share/koncreet.conf.example`, and whichever entry path consumes the key.
- `doctor.sh`: `koncreet doctor`, a read-only check that is not gated on root or OS. It exits 1 only on FAIL; warnings alone exit 0.

Everything runs under `set -euo pipefail`. Guard optional variables with `${VAR:-}` (see the 0.2.3 fix for unset `SSH_CONNECTION`), and expand possibly-empty arrays as `"${arr[@]+"${arr[@]}"}"`. `die` exits the whole shell, so tests wrap failing calls in `( ... )` subshells.
