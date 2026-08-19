# App Directories Role

Creates standardized directory structure for Python applications deployed via GitHub Actions.

## Purpose

- Creates consistent directory layout for all applications
- Sets proper ownership and permissions for service-accounts group
- Separates deployment artifacts (repo) from running code (current)

## Directory Structure

```
/opt/{app_name}/
├── repo/           # Git repository (managed by GitHub Actions)
├── current/        # Running application (symlink or copy from repo)
├── .env            # Environment configuration (managed by Ansible)
└── logs/           # Application logs (optional, via app_log_dir)

/app-data/{app_name}/
├── logs/           # Persistent log directory
└── data/           # Application data directory (optional)
```

## Usage

```yaml
-- name: Create app directories
  include_role:
    name: oshvl.infra.app_directories
  vars:
    app_dir_name: myapp
    app_dir_base: /opt/myapp
    app_dir_user: myapp
    app_log_dir: /app-data/myapp/logs
    app_data_dir: /app-data/myapp/data
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `app_dir_name` | Yes | - | Application name (for logging) |
| `app_dir_base` | Yes | - | Base directory path (e.g., `/opt/myapp`) |
| `app_dir_user` | Yes | - | Owner username |
| `app_dir_group` | No | `service-accounts` | Owner group |
| `app_log_dir` | No | - | Separate log directory (e.g., `/app-data/myapp/logs`) |
| `app_data_dir` | No | - | Additional data directory |

## Created Directories

- `{app_dir_base}/repo` - 0775 (group writable for deployments)
- `{app_dir_base}/current` - 0775 (group writable)
- `{app_log_dir}` - 0775 (if specified)
- `{app_data_dir}` - 0775 (if specified)

## Dependencies

- `app-user` role (user must exist)

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.app_user
      vars:
        app_user_name: myapp

    - role: oshvl.infra.app_directories
      vars:
        app_dir_name: myapp
        app_dir_base: /opt/myapp
        app_dir_user: myapp
        app_log_dir: /app-data/myapp/logs
```
