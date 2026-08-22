# ssh-key-distributor

A lightweight Bash utility for distributing an SSH public key to one or more remote Linux hosts — idempotently, with logging, and without requiring any extra tooling like Ansible.

## Why this exists

Manually running `ssh-copy-id` against each host in a fleet works fine for a handful of servers, but it doesn't scale:

- No consistent logging of which hosts succeeded or failed
- Easy to miss a host or accidentally duplicate a key entry
- No dry-run to preview changes before touching production
- Repetitive, error-prone typing across dozens of hosts

`distribute_ssh_key.sh` solves this by wrapping the distribution logic in a single script that's safe to re-run, logs every action, and supports both quick interactive use and unattended batch runs.

## Features

- **Idempotent** — checks `~/.ssh/authorized_keys` on each host before appending, so re-running the script never creates duplicate entries
- **Two modes**:
  - **Interactive** (default) — prompts for hostname/IP one at a time, useful for quick ad-hoc pushes
  - **Batch** (`-f hosts.txt`) — reads a pre-built list of hosts, useful for fleet-wide rollouts
- **Dry-run support** (`-n`) — preview exactly what would happen with no changes made
- **Correct permissions enforced** — sets `700` on `~/.ssh` and `600` on `authorized_keys`, since `sshd` silently ignores keys with overly permissive file modes
- **Per-host logging** — every run produces a timestamped log file with a pass/skip/fail summary
- **Flexible targeting** — supports `user@host` per-line overrides, a configurable default user, and a custom SSH port

<img width="888" height="757" alt="Script_screen" src="https://github.com/user-attachments/assets/2a8b7df0-50e0-4ca0-8a4d-3af6b07586bc" />







## Requirements

- Bash 4+
- `ssh` client
- Existing SSH access to each target host (password auth or an existing key) — this script *distributes* keys to hosts you can already reach; it does not bootstrap access from zero

## Usage

### Generate a key pair (if you don't already have one)

```bash
ssh-keygen -t ed25519 -C "you@yourhost"
```

### Interactive mode

```bash
./distribute_ssh_key.sh
```

You'll be prompted for each hostname/IP, with the option to add more or stop.

### Batch mode

```bash
cat > hosts.txt << 'EOF'
web01.internal.example.com
deploy@web02.internal.example.com
10.0.1.15
EOF

./distribute_ssh_key.sh -f hosts.txt
```

### Dry run

```bash
./distribute_ssh_key.sh -f hosts.txt -n
```

## Options

| Flag | Description | Default |
|------|-------------|---------|
| `-f` | Path to a hosts file (one `host` or `user@host` per line). Omit to use interactive mode. | — |
| `-k` | Path to the public key to distribute | `~/.ssh/id_ed25519.pub` |
| `-u` | Default remote user, used when a host entry has no `user@` prefix | current user |
| `-p` | SSH port | `22` |
| `-n` | Dry run — no changes made | off |
| `-v` | Verbose SSH output, useful for debugging connection issues | off |

## Example output

```
=== SSH Key Distribution — Sat Aug 22 19:02:05 UTC 2026 ===
Key file:    /home/david/.ssh/id_ed25519.pub
Hosts file:  hosts.txt
Default user: david
Port:        22
Dry run:     false
----------------------------------------

>> david@web01.internal.example.com
   SUCCESS: key added

>> deploy@web02.internal.example.com
   SKIPPED: key already present

>> david@10.0.1.15
   FAILED: cannot connect (check host/port/user/password auth availability)

=== Summary ===
Added:   1
Skipped: 1 (already present)
Failed:  1

Failed hosts:
  - david@10.0.1.15

Full log: ./ssh_key_distribution_20260822_190205.log
```

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | All hosts succeeded |
| `1` | One or more hosts failed — see summary and log file |
| `2` | Bad arguments or missing files |

## License

MIT
