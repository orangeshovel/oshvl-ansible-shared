# App Environment Role

Manages .env files for applications using Ansible Vault secrets.

## Purpose

- Deploys `.env` files from Jinja2 templates
- Integrates with Ansible Vault for secret management
- Sets proper permissions (0640 - readable by group)
- Prevents secrets from being logged

## Usage

```yaml
-- name: Deploy app environment
  include_role:
    name: oshvl.infra.app_env
  vars:
    app_env_name: myapp
    app_env_dest: /opt/myapp/.env
    app_env_template: myapp-env.j2
    app_env_user: myapp
    app_env_vars:
      PG_HOST: "{{ db_host }}"
      PG_PASSWORD: "{{ vault_db_password }}"
      SMTP_USERNAME: "{{ vault_smtp_username }}"
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `app_env_name` | Yes | - | Application name (for logging) |
| `app_env_dest` | Yes | - | Destination path for .env file |
| `app_env_template` | Yes | - | Template file name (in templates/) |
| `app_env_user` | Yes | - | File owner |
| `app_env_group` | No | `service-accounts` | File group |
| `app_env_mode` | No | `0640` | File permissions |
| `app_env_vars` | No | `{}` | Variables to pass to template |

## Template Example

Create `templates/myapp-env.j2`:

```jinja
# MyApp Environment Configuration
# Managed by Ansible - DO NOT EDIT MANUALLY

# Database
PG_HOST={{ app_env_vars.PG_HOST }}
PG_PASSWORD={{ app_env_vars.PG_PASSWORD }}

# SMTP
SMTP_USERNAME={{ app_env_vars.SMTP_USERNAME }}
SMTP_PASSWORD={{ app_env_vars.SMTP_PASSWORD }}
```

## Security

- No logging of secrets (uses `no_log: true`)
- File mode 0640 (owner read/write, group read only)
- service-accounts group can read for deployments

## Dependencies

- `app-user` role (user and group must exist)
- Template file in calling role's `templates/` directory

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.app_user
      vars:
        app_user_name: myapp
    
    - role: oshvl.infra.app_env
      vars:
        app_env_name: myapp
        app_env_dest: /opt/myapp/.env
        app_env_template: myapp-env.j2
        app_env_user: myapp
        app_env_vars:
          DB_PASSWORD: "{{ vault_db_password }}"
```
