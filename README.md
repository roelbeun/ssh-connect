# ssh-connect

A minimal bash script to manage and connect to SSH servers using fuzzy search via [fzf](https://github.com/junegunn/fzf).

## Features

- Fuzzy search across all your servers by name, host, user, or port
- Direct connect by server name: `connect SERVERNAME`
- Simple plain-text config file
- Auto-generates an example config on first run

## Requirements

- `fzf` — install with `brew install fzf`

## Installation

```bash
sudo ln -sf /path/to/ssh-connect/connect /usr/local/bin/connect
```

## Configuration

Servers are defined in `~/.ssh/connect.conf`:

```
# Format: name  host  [user]  [port]  [key]
# Fields are whitespace-separated. user, port and key are optional.

@key            ~/.ssh/id_ed25519
work-server     192.168.1.10
dev-box         10.0.0.5           deploy   2222
staging         staging.example.com ubuntu
backup          192.168.1.10       deploy   22     ~/.ssh/connect_backup
homelab         homelab.example.com deploy   22     homelab
```

If the config file doesn't exist, `connect` will create an example one on first run.

### Authentication

- **No `key` column:** `ssh` uses normal authentication — any running SSH agent (via `$SSH_AUTH_SOCK`, including the [Bitwarden/Vaultwarden SSH agent](#ssh-keys-via-vaultwarden)) offering *all* its keys, and then a password prompt.
- **`key` is a path** (contains `/`, e.g. `~/.ssh/id_ed25519`): `connect` adds `-i <key> -o IdentitiesOnly=yes`, forcing that on-disk identity file (`~` is expanded). A missing file warns and falls back to default auth.
- **`key` is a bare name** (e.g. `homelab`): the name/comment of a key held by your SSH agent, as listed by `ssh-add -L`. `connect` pins that single agent key — it writes only the **public** key to a temp file and uses `-o IdentitiesOnly=yes`, so the private key stays in the agent (e.g. Bitwarden) and only one identity is offered. Use this to fix [`Too many authentication failures`](#too-many-authentication-failures).
- **`@key <path>` directive:** sets a config-wide fallback identity file used by any entry that does not name its own `key`.

## Usage

**Fuzzy search (interactive):**
```bash
connect
```

**Direct connect by name:**
```bash
connect work-server
```

Name matching is case-insensitive.

## SSH keys via Vaultwarden

Keys stored in Vaultwarden are served transparently through the Bitwarden
desktop SSH agent — `connect` needs no special configuration, just leave the
`key` column blank for those servers.

**One-time setup:**

1. Vaultwarden server: enable the experimental flags
   `EXPERIMENTAL_CLIENT_FEATURE_FLAGS=ssh-key-vault-item,ssh-agent`.
2. Bitwarden desktop client: enable the SSH agent in Settings.
3. Point `ssh` at the agent — add to `~/.zshrc`:
   ```bash
   # .dmg build:
   export SSH_AUTH_SOCK="$HOME/.bitwarden-ssh-agent.sock"
   # App Store build:
   # export SSH_AUTH_SOCK="$HOME/Library/Containers/com.bitwarden.desktop/Data/.bitwarden-ssh-agent.sock"
   ```

`connect` calls plain `ssh`, which uses whatever `$SSH_AUTH_SOCK` points at — so the agent setup above is all it needs. Leaving the `key` column blank works until the agent holds many keys (see below), at which point you name the key in the `key` column.

### `Too many authentication failures`

```
Received disconnect from HOST port 22:2: Too many authentication failures
```

This is **not** a server fault. With no `key` column, `ssh` offers the agent's
keys one by one until one matches. Each counts as an attempt, and the server
cuts the connection once it exceeds `MaxAuthTries` (default **6**). When the
Bitwarden/Vaultwarden agent holds more than ~6 keys, hosts whose key happens to
sit late in the list never get reached. `ssh-add -l` shows how many keys are
loaded.

**Fix:** put the key's **name** (its comment in `ssh-add -L`, i.e. the SSH-key
item name in Vaultwarden) in the `key` column. `connect` then pins that single
identity with `-o IdentitiesOnly=yes`, so `ssh` offers exactly one key. The
private key still lives in Bitwarden — only the public key is touched, written
to a temp file that is deleted when the session ends, so nothing for a device
compliance tool (e.g. Kolide) to flag.

```
# before — relies on the agent offering the right key in time
homelab         homelab.example.com deploy   22

# after — pins the agent key named "homelab"
homelab         homelab.example.com deploy   22     homelab
```

### When another SSH agent is already running

Some environments ship their own SSH agent (corporate device-trust tools,
1Password, `gpg-agent`, etc.) that owns `$SSH_AUTH_SOCK`. If that agent wins,
`ssh`/`connect` never reach Bitwarden and you get `Permission denied
(publickey)` **with no approval popup**.

Diagnose by comparing the two agents:
```bash
ssh-add -l                                                    # the active agent
SSH_AUTH_SOCK="$HOME/.bitwarden-ssh-agent.sock" ssh-add -l    # should list your key
```

Fix it per-host with `IdentityAgent` in `~/.ssh/config`, scoped to your own
hosts so the other agent stays the default for everything else (adjust the
patterns to match your servers):
```
Host 10.0.0.* *.example.internal
    IdentityAgent ~/.bitwarden-ssh-agent.sock
```
`IdentityAgent` overrides `$SSH_AUTH_SOCK` for matching hosts, so both `ssh`
and `connect` use Bitwarden for them.

### Adding a new server

```bash
# 1. Generate the SSH key inside Vaultwarden (New item -> SSH key).

# 2. Export the public key locally (KEY_NAME = the key's name/comment in Vaultwarden).
#    If `ssh-add -L` does not list it, copy the "Public key" field from the Bitwarden item instead.
ssh-add -L | grep 'KEY_NAME' > /tmp/bw.pub

# 3. Install it on the target server (prompts for the password one last time).
cat /tmp/bw.pub | ssh USER@HOST 'umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'

# 4. Clean up the temp file.
rm /tmp/bw.pub
```

Then add the server to `~/.ssh/connect.conf` (no `key` column) and connect:
```bash
connect SERVERNAME   # Bitwarden pops up to approve the connection
```

### Harden the server (only after confirming key login works)

Once `connect SERVERNAME` logs in via the key **without** a password, disable
password authentication. Keep your current session open and run this in a
**second** terminal so you can roll back if anything goes wrong:

```bash
ssh -t USER@HOST '
  printf "PasswordAuthentication no\nKbdInteractiveAuthentication no\n" \
    | sudo tee /etc/ssh/sshd_config.d/00-disable-password.conf >/dev/null &&
  sudo sshd -t &&
  sudo systemctl reload ssh'   # use "sshd" instead of "ssh" on non-Ubuntu distros
```

The filename matters: drop-ins are read in lexical order and the **first**
value for a keyword wins, so the `00-` prefix ensures this loads *before*
distro defaults like Ubuntu's `50-cloud-init.conf` (which often sets
`PasswordAuthentication yes`). `sshd -t` validates the config before reload.

Verify it took effect, then confirm password login is actually refused:
```bash
ssh USER@HOST 'sudo sshd -T | grep -iE "passwordauthentication|kbdinteractive"'  # expect: no
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password USER@HOST     # expect: Permission denied (publickey)
```

To roll back, remove the drop-in and reload:
```bash
ssh -t USER@HOST 'sudo rm /etc/ssh/sshd_config.d/00-disable-password.conf && sudo systemctl reload ssh'
```
