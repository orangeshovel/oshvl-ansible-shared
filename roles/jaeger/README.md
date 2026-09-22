# Jaeger Role

Installs and runs Jaeger (OpenTelemetry trace-viewing backend) as a systemd
service, downloaded as a plain release binary — no Docker.

## Purpose

- Downloads the official Jaeger v2 all-in-one binary (OTLP receiver +
  Badger storage + UI in one process) and pins it by version
- Templates a minimal Jaeger v2 config (single Badger storage backend, no
  archive tier, OTLP grpc+http receivers)
- Runs it as a systemd service under a dedicated user

## Usage

```yaml
- role: oshvl.infra.app_user
  vars:
    app_user_name: jaeger
    app_user_comment: "Jaeger trace backend"

- role: oshvl.infra.jaeger
  vars:
    jaeger_version: "2.21.0"       # optional, defaults shown
    jaeger_span_ttl: "168h"        # optional
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `jaeger_user` | No | `jaeger` | OS user the service runs as — must already exist (see Dependencies) |
| `jaeger_base_dir` | No | `/opt/jaeger` | Install location for the binary + config |
| `jaeger_data_dir` | No | `/app-data/jaeger/data` | Badger storage directory |
| `jaeger_version` | No | `2.21.0` | Jaeger release version to install |
| `jaeger_arch` | No | `linux-amd64` | Release tarball architecture |
| `jaeger_span_ttl` | No | `168h` | Badger retention for trace data — bounds local disk growth, since there's no external store to offload to |

## What This Role Does

1. **Directories**: creates `{{ jaeger_base_dir }}/bin` and `{{ jaeger_data_dir }}`.
2. **Binary install**: downloads and checksum-verifies the release tarball for
   `jaeger_version` (only if that version isn't already installed), extracts it, and
   installs the binary as `{{ jaeger_base_dir }}/bin/jaeger-{{ jaeger_version }}`, then
   symlinks `bin/jaeger` to it. Re-running with a new `jaeger_version` downloads the new
   version and re-links, leaving the old versioned binary in place.
3. **Config**: templates `{{ jaeger_base_dir }}/config.yaml` — OTLP receiver
   (grpc/http, loopback-only by default), a single Badger storage backend, and the
   `jaeger_query` UI/query extension.
4. **Systemd**: installs `jaeger.service` (type `simple`, `Restart=always`), enables
   and starts it.

## Why a plain binary, not the generic `systemd_service` role

Jaeger's binary and config aren't fetched by any existing generic role (unlike a
monorepo Python app, whose code is synced by that repo's own CI before
`systemd_service` ever runs) — same situation `github_runner` is in with its own
runner tarball. Doing the download-and-configure sequence inside this role's own task
list, before the systemd unit is created, also avoids a real ordering pitfall: a
playbook that instead called the generic `systemd_service` role and did the binary
fetch in its own `post_tasks` would have `systemd_service`'s own first start attempt
fire *before* those `post_tasks` run (Ansible always runs `roles:` before
`post_tasks:`), hitting a "could not start, binary not found yet" warning on every
fresh deploy. Self-contained task ordering here means that never happens.

## Exposure

This role does not open Jaeger's ports beyond the host's own network interfaces, and
takes no position on whether the host itself is publicly reachable — that's a
per-consumer decision (e.g. orangeshovel keeps it tailnet-only, no public Caddy entry;
see that repo's own `infra/ansible/playbooks/services/jaeger/README.md`). The OTLP
receiver template deliberately has no explicit `endpoint:`, which defaults to
loopback-only — safe even on hosts with a public-facing interface, until a caller
explicitly opens it up for a sender on a different host.

## Dependencies

- **`oshvl.infra.app_user`** (or equivalent): `jaeger_user` must already exist; this
  role does not create it — same pattern as `github_runner`'s `runner_user`.

## Idempotency and Re-runs

- The binary is downloaded only if `{{ jaeger_base_dir }}/bin/jaeger-{{ jaeger_version }}`
  doesn't already exist — re-running with the same version is a no-op for the
  download/extract/install steps.
- Config and systemd unit are re-templated on every run; a real content change
  triggers a restart via handler, a no-op change doesn't.

## Related

- Each consumer repo's own `infra/ansible/playbooks/services/jaeger/deploy_jaeger.yml`
  chains `app_user` → this role, following the same shape as
  `oshvl-github-runner`/`setup_github_actions_runner.yml`.
