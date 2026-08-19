# Log Rotation Role

Configures logrotate for application logs.

## Purpose

- Manages log rotation with consistent policies
- Prevents disk space issues
- Supports multiple log patterns per application

## Usage

```yaml
-- name: Configure app log rotation
  include_role:
    name: oshvl.infra.log_rotation
  vars:
    logrotate_name: myapp
    logrotate_configs:
      - logs:
          - /app-data/myapp/logs/*.log
        user: myapp
        group: service-accounts
        rotate: 30
        size: 10M
      
      - logs:
          - /app-data/myapp/data/*.dat
        user: myapp
        group: service-accounts
        rotate: 30
        size: 1M
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `logrotate_name` | Yes | - | Config file name (becomes `/etc/logrotate.d/{name}`) |
| `logrotate_configs` | Yes | - | List of log rotation configurations |

### Log rotation configuration

Each item in `logrotate_configs`:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `logs` | Yes | - | List of log file patterns |
| `user` | Yes | - | File owner after rotation |
| `group` | No | Owner's primary group | File group after rotation |
| `rotate` | No | `30` | Number of rotations to keep |
| `size` | No | `10M` | Max size before rotation |
| `compress` | No | `true` | Compress rotated logs |
| `postrotate` | No | - | Command to run after rotation |

## Features

- Daily rotation check
- Compression with `delaycompress`
- Missing file handling (`missingok`)
- Empty file skipping (`notifempty`)
- Proper file permissions after rotation

## Dependencies

None (logrotate must be installed on system)

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.log_rotation
      vars:
        logrotate_name: myapp
        logrotate_configs:
          - logs:
              - /app-data/myapp/logs/*.log
            user: myapp
            group: service-accounts
            rotate: 30
            size: 10M
```

## Template Output

```
/app-data/myapp/logs/*.log {
    daily
    rotate 30
    size 10M
    compress
    delaycompress
    missingok
    notifempty
    create 0644 myapp service-accounts
}
```
