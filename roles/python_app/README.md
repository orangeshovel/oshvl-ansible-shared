# Python App Role

Installs Python dependencies and system packages required for Python applications.

## Purpose

- Installs Python 3 and essential development tools
- Installs PostgreSQL client for database connectivity
- Installs build tools (make, git)
- Provides consistent Python environment across applications

## Usage

```yaml
- name: Install Python app dependencies
  include_role:
    name: oshvl.infra.python_app
```

Or with custom packages:

```yaml
- name: Install Python app with additional packages
  include_role:
    name: oshvl.infra.python_app
  vars:
    python_app_extra_packages:
      - libxml2-dev
      - libxslt-dev
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `python_app_version` | No | `python3` | Python package to install |
| `python_app_extra_packages` | No | `[]` | Additional apt packages to install |

## Installed Packages

**Base packages:**
- git
- python3
- python3-pip
- python3-venv
- postgresql-client
- make

**Custom packages:**
- Any packages specified in `python_app_extra_packages`

## Dependencies

None

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.python_app
      vars:
        python_app_extra_packages:
          - python3-dev
          - build-essential
```

## Notes

- Virtual environments should be created by the application's Makefile
- This role only installs system-level dependencies
- APT cache is updated with 1-hour validity
