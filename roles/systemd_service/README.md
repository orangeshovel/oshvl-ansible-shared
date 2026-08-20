# Systemd Service Role

Creates and manages systemd services and timers for scheduled application tasks.

## Purpose

- Standardized systemd service creation
- Timer-based scheduling (modern alternative to cron)
- Consistent service configuration
- Dependency management

## Usage

```yaml
- name: Configure a scheduled app service
  include_role:
    name: oshvl.infra.systemd_service
  vars:
    systemd_service_name: myapp
    systemd_service_description: "Scheduled MyApp tasks"
    systemd_service_user: myapp
    systemd_service_group: myapp
    systemd_service_working_dir: /opt/myapp/current
    systemd_service_exec_start: /usr/bin/make run
    systemd_service_environment_file: /opt/myapp/current/.env
    systemd_service_environment:
      PATH: "/opt/myapp/current/venv/bin:/usr/local/bin:/usr/bin:/bin"
      LOG_DIR: "/app-data/myapp/logs"
    systemd_timer_enabled: true
    systemd_timer_on_calendar: "00:00:00"
```

## Variables

### Service Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `systemd_service_name` | Yes | - | Service name (without .service) |
| `systemd_service_description` | Yes | - | Human-readable description |
| `systemd_service_user` | Yes | - | User to run service as |
| `systemd_service_group` | No | User's primary group | Group to run service as |
| `systemd_service_working_dir` | Yes | - | Working directory |
| `systemd_service_exec_start` | Yes | - | Command to execute |
| `systemd_service_type` | No | `oneshot` | Service type (oneshot/simple/forking) |
| `systemd_service_environment` | No | `{}` | Environment variables |
| `systemd_service_environment_file` | No | - | Path to .env file |
| `systemd_service_after` | No | `["network-online.target"]` | Service dependencies |
| `systemd_service_wants` | No | `["network-online.target"]` | Weak dependencies |

### Timer Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `systemd_timer_enabled` | No | `false` | Create and enable timer |
| `systemd_timer_on_calendar` | No | `"*-*-* 00:00:00"` | Schedule (systemd calendar format) |
| `systemd_timer_persistent` | No | `true` | Run missed schedules on boot |

## Features

- **Oneshot services**: Perfect for scheduled tasks that run and exit
- **Environment management**: Load .env files and set environment vars
- **Security hardening**: `PrivateTmp`, `NoNewPrivileges`
- **Logging**: Output to systemd journal
- **Timer persistence**: Catch up missed runs if system was down

## Dependencies

None

## Example Playbook

```yaml
---
- hosts: app_servers
  become: yes
  roles:
    - role: oshvl.infra.systemd_service
      vars:
        systemd_service_name: myapp
        systemd_service_description: "Scheduled MyApp tasks"
        systemd_service_user: myapp
        systemd_service_working_dir: /opt/myapp/current
        systemd_service_exec_start: /usr/bin/make run
        systemd_service_environment_file: /opt/myapp/current/.env
        systemd_timer_enabled: true
        systemd_timer_on_calendar: "00:00:00"
```

## Managing Services

```bash
# Check timer status
sudo systemctl status myapp.timer

# View next scheduled run
sudo systemctl list-timers myapp.timer

# Manually trigger service
sudo systemctl start myapp.service

# View logs
sudo journalctl -u myapp.service -f

# Stop/start timer
sudo systemctl stop myapp.timer
sudo systemctl start myapp.timer
```
