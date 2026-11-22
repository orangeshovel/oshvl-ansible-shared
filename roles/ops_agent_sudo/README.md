# ops_agent_sudo

Grants `oshvl-ops-agent`'s unprivileged service user (`ops-agent`) root
access to exactly one fixed, root-owned helper script
(`/opt/oshvl-ops-agent/root-helper/root_cleanup_helper.py`), for the handful
of cleanup operations that need to touch paths outside `/opt/*`/`/app-data/*`
— another user's `/home/*`, `/var/log/journal`, `/root/.cache`.

## Why not shaped like `github_deploy`

`oshvl.infra.github_deploy` whitelists raw verbs (`chgrp`, `chmod`, `cp`)
with caller-controlled arguments — safe there because those args come from
a controlled CI deploy flow. A disk-cleanup tool whitelisting `rm -rf` (or
similar) with any caller-controlled path would be a shell-injection-shaped
foot-gun. Instead, the sudoers grant here names exactly one script, with no
`ALL`, no verb list, and no arguments beyond a closed `--action {enum}` +
`--apply` flag baked into the script itself. The script accepts no path or
shell arguments from its caller at all — every real path (runner home,
journal vacuum size, claude version homes) is templated in by Ansible at
deploy time.

## Why the helper lives outside `app_directories`' tree

Every app's `/opt/{app}/current` is mode `0775`, group `service-accounts` —
by design, so apps can cross-read each other's `/app-data/*/logs/` without
sudo (see `oshvl.infra.app_directories`). Every app's service user is a
member of that same shared group. If the root-executed helper lived inside
that tree, any other app's compromised or buggy service user could read or
tamper with a script that's about to run as root. This role creates its own
`/opt/oshvl-ops-agent/root-helper/` directory, `root:root 0700`, entirely
outside `app_directories`' management.

## Required variables

| Variable | Description |
|---|---|
| `ops_agent_sudo_runner_home` | **Required, no default.** Home directory of the org's self-hosted runner user (e.g. `/home/oshvl-github-runner`). Each org's runner uses a different OS username — a wrong value here points cache-pruning actions at the wrong home directory. |

## Optional variables

| Variable | Default | Description |
|---|---|---|
| `ops_agent_sudo_app_user` | `ops-agent` | The unprivileged service user granted this sudo access. |
| `ops_agent_sudo_journal_vacuum_size` | `500M` | Passed to `journalctl --vacuum-size=`. |
| `ops_agent_sudo_claude_version_homes` | `[ops_agent_sudo_runner_home]` | List of home directories to check for old Claude Code CLI version binaries. |
| `ops_agent_sudo_claude_keep_newest_n` | `3` | Number of Claude CLI versions to retain, newest first by mtime. The version `~/.local/bin/claude` resolves to is never deleted, even if its mtime ranks it outside the newest N. |
| `ops_agent_sudo_work_max_age_days` | `30` | Age in days after which entries in the runner's `_work` directory are pruned (never while a job is running). |

## Defense in depth

The helper independently re-checks that the GitHub Actions runner is idle
before touching `_work` tool caches (in addition to the calling Python app's
own check), and treats `--apply` as its own independent dry-run gate — a bug
in the unprivileged app's `APPLY` handling can produce a wrong report, but
never an unwanted deletion, since the privileged surface refuses to delete
anything unless `--apply` is explicitly passed to it directly.

## `apt-upgrade` action

Runs `apt-get update` and `apt-get -y upgrade --with-new-pkgs` (only with `--apply`;
without it, an `apt-get -s upgrade --with-new-pkgs` simulation). It never removes packages,
but may install new ones: plain `upgrade` holds back any update that needs a new package,
including every new kernel image, so kernels would never be updated. Old kernel packages
are therefore not removed either, so watch `/boot` and the root disk (`apt autoremove` is a
separate decision). It keeps locally modified config files (`--force-confold`; unmodified ones are updated), runs with a minimal explicit environment, no stdin and `LC_ALL=C`, gives apt 55 minutes (then kills it and exits 1), and waits up to
5 minutes for the dpkg lock. It upgrades every enabled apt source, not only security
updates. Hold a package you don't want touched with `apt-mark hold`. Updates apt still holds
back are reported in `kept_back`.

It prints two marker lines the caller parses (`oshvl-ops-agent-shared`,
`ops_agent/daily_ops/os_upgrade.py`):

```
[apt] upgraded=N kept_back=M
[apt] reboot-required=true|false
```

On failure (apt non-zero, or the 55-minute timeout) it prints no marker lines and exits 1;
callers must check the exit code before parsing. A dry-run does not run `apt-get update`, so
it only reflects the current local index and must not be treated as "nothing to upgrade".

The helper only makes the action *available*: it has no opt-in gate of its own, so any caller
allowed to run the helper can request it. Whether a host actually upgrades nightly is the
ops-agent's per-host `APT_UPGRADE=true` setting (a trusted-caller control, unlike `--apply`).

Two behaviours depend on packages this role does not install: service restarts after a
library upgrade need `needrestart` (`NEEDRESTART_MODE=a`; otherwise only packages whose
maintainer scripts restart themselves are restarted, and services using an upgraded library
keep running the old code until restarted), and `reboot-required=true` needs the
`/run/reboot-required` file written by `update-notifier-common` (Ubuntu installs it by
default; on a minimal host without it the marker is always `false`). Upgrading restarts
services of the upgraded packages (for example `sshd`, `tailscaled`, `postgresql`), so
schedule it at a quiet time. It does not reboot.
