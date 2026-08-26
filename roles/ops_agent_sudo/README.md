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
| `ops_agent_sudo_claude_keep_newest_n` | `3` | Number of Claude CLI versions to retain. |

## Defense in depth

The helper independently re-checks that the GitHub Actions runner is idle
before touching `_work` tool caches (in addition to the calling Python app's
own check), and treats `--apply` as its own independent dry-run gate — a bug
in the unprivileged app's `APPLY` handling can produce a wrong report, but
never an unwanted deletion, since the privileged surface refuses to delete
anything unless `--apply` is explicitly passed to it directly.
