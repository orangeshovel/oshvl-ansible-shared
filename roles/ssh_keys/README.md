# SSH Keys Role

Generates SSH keys for service users and configures remote host access.

## Purpose

- Generates SSH key pairs for service users (ED25519 by default)
- Adds remote hosts to known_hosts to prevent verification prompts
- Displays public key for manual deployment to remote systems
- Idempotent - won't regenerate existing keys

## Features

- ED25519 key generation (modern, secure, fast)
- Automatic known_hosts management
- Public key display for manual deployment
- Proper file permissions and ownership
- Key-based authentication only (no passwords)

## Usage

```yaml
- name: Setup SSH keys for backup user
  include_role:
    name: oshvl.infra.ssh_keys
  vars:
    ssh_keys_user_name: backup
    ssh_keys_remote_host: nas.example.lan
    ssh_keys_comment: "backup@compute.example.lan"
```

**Or** leverage `app_user` variable from your playbook:

```yaml
vars:
  app_user: backup

roles:
  - role: oshvl.infra.app_user
    vars:
      app_user_name: "{{ app_user }}"
  
  - role: oshvl.infra.ssh_keys  # Automatically uses app_user
    vars:
      ssh_keys_remote_host: nas.example.lan
```

## Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ssh_keys_user_name` | No* | `{{ app_user }}` | Local username for SSH key generation |
| `ssh_keys_user` | No* | - | Alternative to ssh_keys_user_name (backwards compatible) |
| `ssh_keys_remote_host` | Yes | - | Remote hostname to add to known_hosts |
| `ssh_keys_comment` | No | `"{{ ssh_keys_user_name }}@{{ ansible_hostname }}"` | SSH key comment |
| `ssh_keys_type` | No | `ed25519` | Key type (ed25519, rsa, etc.) |
| `ssh_keys_path` | No | `/home/{{ ssh_keys_user_name }}/.ssh/id_{{ ssh_keys_type }}` | SSH key location |
| `ssh_keys_display_public_key` | No | `true` | Display public key after generation |
| `ssh_keys_remote_user` | No | `{{ ssh_keys_user_name }}` | Remote username (if different from local) |

\* Either `ssh_keys_user_name`, `ssh_keys_user`, or `app_user` must be defined

## Dependencies

- `community.crypto` Ansible collection (for `openssh_keypair` module)

Install with:
```bash
ansible-galaxy collection install community.crypto
```

## Example Playbook

```yaml
---
- hosts: compute_servers
  become: yes
  vars:
    app_user: backup
  
  roles:
    # First create the user
    - role: oshvl.infra.app_user
      vars:
        app_user_name: "{{ app_user }}"
        app_user_comment: "Backup Service"
    
    # Then setup SSH keys (automatically uses app_user)
    - role: oshvl.infra.ssh_keys
      vars:
        ssh_keys_remote_host: nas.example.lan
```

**Backwards compatible usage:**
```yaml
- role: oshvl.infra.ssh_keys
  vars:
    ssh_keys_user: backup  # Still works!
    ssh_keys_remote_host: nas.example.lan
```

## Post-Deployment

After running this role, you'll see output with the public key. Deploy it to the remote host:

**Manual Deployment:**
```bash
# Copy the public key displayed in the Ansible output
ssh target-user@remote-host
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3N... user@host" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**Automated Deployment (from control machine):**
```bash
echo "ssh-ed25519 AAAAC3N... user@host" | \
  ssh target-user@remote-host 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

## Security Considerations

- Keys are generated with secure default algorithms (ED25519)
- Private keys readable only by owner (600)
- Public key deployment is intentionally manual for security
- Known_hosts prevents MITM attacks

## Notes

- This role does NOT deploy the public key to remote hosts automatically
- You must manually add the public key to the remote host's authorized_keys
- Re-running the role won't regenerate existing keys (idempotent)
- The role runs `ssh-keyscan` to populate known_hosts
