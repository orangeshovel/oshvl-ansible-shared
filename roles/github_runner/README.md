# GitHub Runner Role

Installs and configures a self-hosted GitHub Actions runner on a compute
node, registered against a specific GitHub repository.

## Purpose

- Downloads and installs the official GitHub Actions runner
- Registers the runner with the target GitHub repository
- Runs the runner as a systemd service under a dedicated user
- Creates a secrets directory and deployment secrets file for CI tooling on
  that host (e.g. `ci_notify.sh`) to source

## Usage

```yaml
- role: oshvl.infra.github_runner
  vars:
    github_repo: your-owner/your-repo   # required, no default
    runner_user: your-repo-runner-user  # required, no default
    github_token: "{{ vault_github_runner_token }}"  # first run only
    runner_secrets_env:
      SHOVEL_BOT_URL: "https://api.shovel.bot"
      SHOVEL_BOT_API_KEY: "{{ vault_shovel_bot_api_key }}"
      SHOVEL_BOT_CHANNEL: "#all-orangeshovel"
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `github_repo` | **Yes** | - (no default) | Repository in `owner/repo` form for registration. Every caller must set this explicitly — a default here previously caused a real incident (see below). |
| `runner_user` | **Yes** | - (no default) | OS user that runs the runner service. Same reasoning as `github_repo`. |
| `runner_home` | No | `/home/{{ runner_user }}` | Home directory for the runner user |
| `runner_dir` | No | `{{ runner_home }}/actions-runner` | Directory where the runner is installed |
| `runner_version` | No | `2.311.0` | GitHub Actions runner release version |
| `runner_arch` | No | `linux-x64` | Runner architecture (e.g. `linux-x64`, `linux-arm64`) |
| `runner_name` | No | `{{ ansible_hostname }}-runner` | Name the runner registers under in GitHub |
| `github_token` | Yes (first run) | - | GitHub token with `repo` / `admin:actions` scope; used to obtain a registration token |
| `runner_force_reregister` | No | `false` | Force removal + re-registration (e.g. after moving the runner to a new URL) — requires `runner_token` |
| `runner_secrets_env` | No | `{}` | Dict of `{ENV_VAR_NAME: value}` written as `export` lines into `deployment_secrets.yml`. Empty by default — this role makes no assumptions about which secrets a given repo's runner needs; every caller passes its own set. |

## Why `github_repo` and `runner_user` have no default

This role is shared across multiple independently-owned repos, each with
its own runner registered against its own GitHub repo under its own OS
username. A sibling role in this collection (`github_deploy`) used to
default its equivalent identity variable to one specific repo's value —
another consumer repo's playbook never overrode it, silently inherited the
wrong value, and had that repo's real runner's sudo access revoked in
production until manually fixed. This role applies the same fix
proactively: both identity variables are required, and the role asserts
loudly with a clear message if either is missing, rather than silently
applying the wrong repo's identity.

## What This Role Does

1. **Runner install**: Creates `runner_dir`, downloads the runner tarball, extracts it, and runs `installdependencies.sh` (only if not already present).
2. **Registration**: Calls the GitHub API for a registration token and runs `config.sh` to register the runner (only if `.credentials` does not exist); requires `github_token`.
3. **Systemd**: Installs `github-runner.service` (user `runner_user`, working dir `runner_dir`, runs `run.sh`), enables and starts it.
4. **Secrets**: Creates `{{ runner_home }}/.secrets` and `deployment_secrets.yml` from `runner_secrets_env`, if non-empty.

## Dependencies

- **`oshvl.infra.python_app`** (or equivalent): must install packages such as `curl`, `wget`, `tar`, `jq`, `sudo` before this role.
- **`oshvl.infra.app_user`** (or equivalent): the `runner_user` must already exist; this role does not create it.
- **Vault**: for first-time registration, `github_token` must be provided. For deployment secrets, whatever vault variables feed `runner_secrets_env` must be defined by the caller.

## Idempotency and Re-runs

- Runner binary is downloaded and extracted only if `{{ runner_dir }}/config.sh` is missing.
- Runner is registered only if `{{ runner_dir }}/.credentials` is missing and `github_token` is set.
- Systemd unit and secrets are updated on every run. Re-running the playbook is safe.

## Related

- **`oshvl.infra.github_deploy`**: grants the runner user sudo permissions to deploy applications; used per-app, not by this role.
- Each consumer repo's own `infra/ansible/playbooks/services/{oshvl-github-runner,github-runner}/setup_github_actions_runner.yml` chains `python_app` → `app_user` → this role to provision a new compute node end-to-end (a local playbook, not a shared one — see this collection's own README for why).
