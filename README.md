# oshvl.infra

Shared Ansible collection consumed by repos worked on by orangeshovel and deployed via self-hosted GitHub Actions runner on orangeshovel infrastructure. Public so cross-owner `ansible-galaxy collection install` works with no access grant.

Referenced via a floating git ref (`version: main`) in each consumer repo's
`infra/ansible/requirements.yml` — no version pinning. Unlike this org's
shared CI tooling (`oshvl-ci-shared`), a bad change here can restart
services or otherwise mutate live infrastructure on any consumer's next
deploy, not just break a CI pipeline — that tradeoff is accepted
deliberately, consistent with the CI project's "no gate, accept the risk"
stance, not an oversight.

## Contents

Roles present in more than one of the three consumer repos, extracted here
once they were confirmed byte-identical or safely parameterized via
`defaults/main.yml` (see each role's own README for usage):

- `tailscale` — install and configure Tailscale on Debian LXC containers
- `user_management` — OS-level user management
- `python_app` — install Python system packages for an app
- `app_user` — create a service user in the `service-accounts` group
- `app_directories` — create `/opt/{app}/` and `/app-data/{app}/` layout
- `app_env` — deploy an app's `.env` file from Ansible Vault vars
- `log_rotation` — configure logrotate for an app
- `systemd_service` — create a systemd service + optional timer
- `ssh_keys` — generate SSH keys for a service user, manage known_hosts
  (currently used by orangeshovel and lonebirchlab only)
- `github_deploy` — grant a GitHub Actions runner user sudo permissions to
  deploy an app
- `ops_agent_sudo` — grant `oshvl-ops-agent`'s service user sudo access to
  exactly one fixed root-owned helper script, for the handful of
  compute-node housekeeping operations that need to touch paths outside
  `/opt/*`/`/app-data/*` (see the role's own README for why this is shaped
  differently from `github_deploy`'s raw-verb whitelist)

**Deliberately excluded**, not yet shareable:

- `oshvl-github-runner` / `unmute-github-runner` (and equivalents) — GitHub
  Actions runner registration is inherently repo-specific (each points at
  its own repo for registration); not a sharing candidate.

## Playbooks

Full, FQCN-referenced playbooks (not just roles) for logic that's identical
across all three consumer repos and shouldn't be hand-copied into each one:

- `setup_github_runner.yml` (`oshvl.infra.setup_github_runner`) — provisions
  an existing compute node as a self-hosted GitHub Actions runner
- `deploy_ops_agent.yml` (`oshvl.infra.deploy_ops_agent`) — deploys
  `oshvl-ops-agent-shared` (compute-node disk cleanup + the daily
  backup/log-monitor/digest job originally extracted from orangeshovel's
  `shovel.watch`); each consumer repo's own compute-node setup playbook
  imports this with a small per-host `vars:` block (runner username, DB/S3
  connection details, cleanup target list) rather than duplicating the
  deploy logic itself

## Usage from a consumer repo

`infra/ansible/requirements.yml`:

```yaml
collections:
  - name: community.general
  - name: community.proxmox
  - name: https://github.com/orangeshovel/oshvl-ansible-shared.git
    type: git
    version: main
```

Installed the same way existing collections already are — no new CI step,
just `--force` added to the existing `ansible-galaxy collection install -r
requirements.yml` call (required because the self-hosted runners' galaxy
collection cache persists outside the per-run venv and won't reliably
notice a floating git ref has moved without it).

Playbook role references use the collection's fully-qualified name (FQCN)
instead of a relative path:

```yaml
roles:
  - role: oshvl.infra.python_app
  - role: oshvl.infra.app_user
    vars:
      app_user_name: "{{ app_user }}"
```
