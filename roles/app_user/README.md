# App User Role

Creates a dedicated service user for running Python applications with proper group membership.

## Purpose

- Creates application service users with consistent configuration
- Adds users to the `service-accounts` group for operational access
- Isolates application processes from CI/CD and admin users

## Features

- Creates user with home directory
- Adds to `service-accounts` group (created if needed)
- Non-system user (regular UID range)
- Configurable shell (default: `/bin/bash`)

## Usage

```yaml
-- name: Create an application service user
  include_role:
    name: oshvl.infra.app_user
  vars:
    app_user_name: myapp
    app_user_comment: "MyApp Service"
    app_user_home: /home/myapp
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `app_user_name` | Yes | - | Username for the service account |
| `app_user_comment` | No | `"Application Service User"` | GECOS comment field |
| `app_user_home` | No | `/home/{{ app_user_name }}` | Home directory path |
| `app_user_shell` | No | `/bin/bash` | Login shell |
| `app_user_groups` | No | `['service-accounts']` | Additional groups |

## Dependencies

None

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.app_user
      vars:
        app_user_name: myapp
        app_user_comment: "MyApp Service"
```
