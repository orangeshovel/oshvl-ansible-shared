# GitHub Deploy Role

Configures sudoers permissions for GitHub Actions runner to deploy applications.

## Purpose

- Grants GitHub Actions runner permission to deploy as service users
- Enables file ownership and permission management
- Follows principle of least privilege

## Usage

```yaml
-- name: Configure GitHub deployment for an app
  include_role:
    name: oshvl.infra.github_deploy
  vars:
    github_deploy_app_name: myapp
    github_deploy_app_user: myapp
    github_deploy_runner_user: oshvl-github-runner
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `github_deploy_app_name` | Yes | - | Application name (for sudoers file) |
| `github_deploy_app_user` | Yes | - | Service user to run commands as |
| `github_deploy_runner_user` | No | `oshvl-github-runner` | GitHub runner username |
| `github_deploy_allowed_commands` | No | See defaults | Additional sudo commands to allow |

## Default Permissions

The role grants the following sudo permissions:

1. **Run as service user**: `sudo -u {app_user} <any command>`
2. **Change group ownership**: `sudo chgrp ...`
3. **Change permissions**: `sudo chmod ...`
4. **Copy files**: `sudo cp ...`

All commands are **NOPASSWD** for automated deployments.

## Security

- Scoped to specific users (not ALL)
- Validated with `visudo -cf` before deployment
- Separate sudoers file per application (`/etc/sudoers.d/github-runner-{app}`)

## Dependencies

- GitHub runner user must exist

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.github_deploy
      vars:
        github_deploy_app_name: myapp
        github_deploy_app_user: myapp
```

## Sudoers File Output

```
# Allow github-runner to run commands as the application service user (example)
oshvl-github-runner ALL=(myapp) NOPASSWD: ALL

# Allow changing group ownership and copying files
oshvl-github-runner ALL=(root) NOPASSWD: /usr/bin/chgrp, /usr/bin/chmod, /usr/bin/cp
```

## GitHub Actions Workflow Example

```yaml
- name: Deploy code
  run: |
    sudo -u myapp cp -a $GITHUB_WORKSPACE /opt/myapp/repo
    sudo chgrp -R service-accounts /opt/myapp/repo
    sudo chmod -R g+w /opt/myapp/repo
```
