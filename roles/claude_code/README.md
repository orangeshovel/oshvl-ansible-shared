# Claude Code Role

Installs the Claude Code CLI for one existing user using the native installer,
pinned to an exact version.

## Usage

```yaml
- role: oshvl.infra.app_user
  vars:
    app_user_name: oshvl-developer-agent
- role: oshvl.infra.claude_code
  vars:
    claude_code_user: oshvl-developer-agent
    claude_code_version: "2.1.322"
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `claude_code_user` | Yes | none | User to install for; must already exist |
| `claude_code_home` | No | `/home/{{ claude_code_user }}` | That user's home |
| `claude_code_version` | No | `2.1.322` | Exact CLI version; callers should pin explicitly |
| `claude_code_installer_url` | No | `https://claude.ai/install.sh` | Installer script |

## Not handled here

Authentication. Run `claude setup-token` once by hand and put the token in the
consuming app's `.env` as `CLAUDE_CODE_OAUTH_TOKEN`.
