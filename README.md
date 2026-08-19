# oshvl.infra

Shared Ansible collection consumed by three independently-owned repos
(`orangeshovel`, `lonebirchlab`, `projectunmute`), each deploying to its own
infrastructure via its own self-hosted GitHub Actions runner. Public so
cross-owner `ansible-galaxy collection install` works with no access grant.

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
- `ssh_keys` — generate SSH keys for a service user, manage known_hosts
  (currently used by orangeshovel and lonebirchlab only)
- `github_deploy` — grant a GitHub Actions runner user sudo permissions to
  deploy an app

**Deliberately excluded**, not yet shareable:

- `systemd-service` — behavior diverged across repos (see each consumer
  repo's own copy); needs all three repos aligned on identical handler
  behavior before it's safe to extract. Tracked as a follow-up.
- `oshvl-github-runner` / `unmute-github-runner` (and equivalents) — GitHub
  Actions runner registration is inherently repo-specific (each points at
  its own repo for registration); not a sharing candidate.

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
