# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`ssh-connect` is a single-file bash script (`connect`) that wraps SSH with fzf-based fuzzy search for server selection. There is no build system, test suite, or package manager — the entire project is the script itself.

## Installation

```bash
sudo ln -sf /path/to/ssh-connect/connect /usr/local/bin/connect
```

Requires `fzf` (`brew install fzf`).

## How it works

The script reads `~/.ssh/connect.conf` (whitespace-separated: `name host [user] [port] [key]`), builds tab-delimited records for fzf display, then either:
- Presents an interactive fzf picker (no args)
- Does a case-insensitive name lookup and connects directly (`connect <name>`)

Selected entries are parsed and passed to `exec ssh`, with `-p` added only when port ≠ 22.

When an entry has a `key` (per-server column, or the `@key` config-wide fallback), the script pins that single identity with `-i <key> -o IdentitiesOnly=yes`. The `key` value is interpreted two ways:

- **Path** (contains `/`, e.g. `~/.ssh/id_ed25519`): an identity file on disk. `~` is expanded. If the file is missing it warns and falls back to default auth.
- **Bare name** (e.g. `homelab`): the name/comment of a key held by the SSH agent, as listed by `ssh-add -L`. The script greps that line, writes the matching **public** key to a `mktemp` temp file (removed on exit via an `EXIT` trap), and points `ssh -i` at it. The private key never leaves the agent. This is the way to use a Bitwarden/Vaultwarden-stored key while offering only one identity — it prevents the "Too many authentication failures" error that occurs when the agent holds more keys than the server's `MaxAuthTries` (default 6). If the name isn't found in `ssh-add -L`, it warns and falls back to default auth.

Because a bare name is resolved against the live agent, the script no longer `exec`s ssh — it runs ssh as a child so the `EXIT` trap can clean up the temp public key after the session.

With no `key`, ssh uses normal auth — any agent on `$SSH_AUTH_SOCK` (offering *all* its keys) then a password prompt. A `key` of `-` or `none` means **password-only**: it skips the `@key` fallback *and* passes `-o PubkeyAuthentication=no -o IdentitiesOnly=yes` so ssh offers no keys at all and goes straight to the password prompt. Use this when the agent holds more keys than the server's `MaxAuthTries` (default 6) allows and you want to reach that server by password — it prevents the "Too many authentication failures" error. It shows a 🔓 marker in the picker.

## Config format

```
# ~/.ssh/connect.conf
@key            ~/.ssh/id_ed25519
work-server     192.168.1.10
dev-box         10.0.0.5           deploy   2222
staging         staging.example.com ubuntu
backup          192.168.1.10       deploy   22     ~/.ssh/connect_backup
homelab         homelab.example.com deploy   22     homelab
open-box        10.0.0.9           deploy   22     -
```

Fields are whitespace-separated; `user` defaults to `$USER`, `port` defaults to `22`, `key` is optional. The `key` field is a path if it contains `/`, otherwise an agent key name (see "How it works"). The `@key <path>` directive sets a fallback identity file used for any entry without its own `key`. A `key` of `-` or `none` means password-only auth: it opts the entry out of the `@key` fallback and disables pubkey auth entirely (`-o PubkeyAuthentication=no`), so no keys are offered — useful to avoid "Too many authentication failures" when the agent holds many keys. Comments (`#`) and blank lines are ignored.
