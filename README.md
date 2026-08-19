---
# vim: expandtab:shiftwidth=2:filetype=markdown:foldlevel=3

# 
# 
# ~chewygumxx/systemd-override-gpg-socket.git
# ::: :/README.md
# 
# 

ctime: 2026-08-20
title: README - Systemd Override Generator for GnuPG Socket Resolution
tags:  [  ]
---

# Systemd Override Generator for GnuPG Socket Resolution

Reconciles systemd `--user` socket activation with a non-default (XDG-compliant) `GNUPGHOME`, so `dirmngr`, `keyboxd`, and `gpg-agent` (including its `ssh`, `extra`, and `browser` variants) get socket-activated on the paths GnuPG actually expects.

## The Problem

When `GNUPGHOME` points somewhere other than GnuPG's compiled-in default, `gpgconf --list-dirs` computes socket paths under a homedir-specific hashed subdirectory (e.g. `$XDG_RUNTIME_DIR/gnupg/d.<hash>/S.gpg-agent`) to avoid collisions between multiple homedirs. The systemd user units shipped by the `gnupg` package, however, hardcode the *default* socket paths in their `ListenStream=` directives. With a custom `GNUPGHOME`, those two never match — systemd activates sockets nothing is listening on the other end of, and GnuPG clients look for sockets systemd never created.

This repository closes that gap by generating a per-unit systemd drop-in that overrides `ListenStream=` with the real, currently-computed path — recalculated fresh each time the service runs, since the hash depends on `GNUPGHOME`'s resolved path.

## Contents

| File | Purpose |
|---|---|
| `systemd-override-gpg-socket` | Bash script. Diffs `gpgconf --list-dirs` socket entries against installed `systemd --user` `.socket` units and writes matching override drop-ins. |
| `systemd-override-gpg-socket.service` | Oneshot `systemd --user` unit. Runs the script before the GnuPG socket units it overrides. |

## How It Works

1. `systemd-override-gpg-socket.service` is ordered `Before=` and `Wants=` every affected `.socket` unit, and gated on `ConditionEnvironment=GNUPGHOME` — it's inert unless a custom homedir is actually in play.
2. On activation, the script confirms `systemctl` and `gpgconf` are available and that `GNUPGHOME` is non-empty, then exports it and runs `gpgconf --list-dirs`.
3. Each `*-socket` key is mapped to its corresponding unit name:

   | `gpgconf --list-dirs` key | systemd unit |
   |---|---|
   | `dirmngr-socket` | `dirmngr.socket` |
   | `keyboxd-socket` | `keyboxd.socket` |
   | `agent-socket` | `gpg-agent.socket` |
   | `agent-ssh-socket` | `gpg-agent-ssh.socket` |
   | `agent-extra-socket` | `gpg-agent-extra.socket` |
   | `agent-browser-socket` | `gpg-agent-browser.socket` |

4. For each key with a matching, installed unit, it writes:

   ```
   $XDG_CONFIG_HOME/systemd/user/<unit>.socket.d/override.conf
   ```

   clearing the inherited `ListenStream=` and replacing it with the actual resolved path.
5. `systemctl --user daemon-reload` picks up the new drop-ins immediately.

Keys with no corresponding installed unit are skipped with a warning rather than failing the run.

## Requirements

- `bash` ≥ 4 (associative arrays, `[[ -v ]]`)
- `systemd` (`systemctl --user`)
- GnuPG (`gpgconf`)
- GNU coreutils (`realpath`)
- `GNUPGHOME` exported into the systemd user manager's own environment — most conventionally via a drop-in under `~/.config/environment.d/*.conf`, **not** just exported in your shell's rc files. `ConditionEnvironment=` and the script both read the manager's environment, which shell-only exports never reach.

## Installation

```sh
install -Dm755 systemd-override-gpg-socket "$HOME/.local/bin/systemd-override-gpg-socket"
install -Dm644 systemd-override-gpg-socket.service \
    "$HOME/.config/systemd/user/systemd-override-gpg-socket.service"
```

Confirm `GNUPGHOME` is visible to the systemd user manager (a fresh login is usually required after adding an `environment.d` drop-in):

```sh
systemctl --user show-environment | grep GNUPGHOME
```

Enable the unit:

```sh
systemctl --user enable --now systemd-override-gpg-socket.service
```

## Verifying

```sh
systemctl --user status systemd-override-gpg-socket.service
systemctl --user cat gpg-agent.socket
gpgconf --list-dirs agent-socket
```

The `ListenStream=` shown by `systemctl --user cat gpg-agent.socket` (under the generated `override.conf` drop-in) should match the path `gpgconf --list-dirs agent-socket` reports.

## Notes & Caveats

- **Safe no-op without a custom homedir.** If `GNUPGHOME` is unset in the manager's environment, both the service's `ConditionEnvironment=` and the script's own guard skip execution entirely. Stock GnuPG setups are unaffected.
- **Re-run after `GNUPGHOME` changes.** The hashed socket directory is derived from `GNUPGHOME`'s resolved path. Changing it — including moving the underlying directory — requires re-running the service (`systemctl --user restart systemd-override-gpg-socket.service`) to regenerate the overrides.
- **Don't hand-edit the generated `override.conf` files.** They're regenerated on every run and are marked as such in their header; change the script or the unit file instead.

## License

GPL-3.0-or-later — see the SPDX header in each file.
