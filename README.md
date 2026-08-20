# systemd-override-gpg-socket

Generates systemd `--user` socket drop-ins so `dirmngr`, `keyboxd`, and `gpg-agent` (plus its `ssh`, `extra`, and `browser` variants) socket-activate on the paths GnuPG actually computes for a non-default, XDG-compliant `GNUPGHOME` — instead of the compiled-in defaults hardcoded in the `.socket` units GnuPG ships.

## The Problem

When `GNUPGHOME` points somewhere other than GnuPG's compiled-in default, `gpgconf --list-dirs` computes socket paths under a homedir-specific hashed subdirectory (e.g. `$XDG_RUNTIME_DIR/gnupg/d.<hash>/S.gpg-agent`) to avoid collisions between multiple homedirs. The systemd user units shipped by the `gnupg` package hardcode the *default* socket paths in their `ListenStream=` directives, so with a custom `GNUPGHOME` the two never match — systemd activates sockets nothing is listening on, and GnuPG clients look for sockets systemd never created.

This repository closes that gap: a oneshot service runs before the affected socket units, diffs GnuPG's current socket paths against installed systemd units, and writes a drop-in overriding `ListenStream=` with the real, freshly-computed path — recalculated on every run, since the hash depends on `GNUPGHOME`'s resolved path.

## Contents

| File | Purpose |
|---|---|
| `systemd-override-gpg-socket` | Bash script. Diffs `gpgconf --list-dirs` socket entries against installed `systemd --user` `.socket` units, writes matching override drop-ins, prunes stale ones, and reloads the manager. |
| `systemd-override-gpg-socket.service` | Oneshot `systemd --user` unit. Runs the script before the GnuPG socket units it overrides. |

## How It Works

1. `systemd-override-gpg-socket.service` is ordered `Before=` and `Wants=` every affected `.socket` unit, and gated on `ConditionEnvironment=GNUPGHOME` — it's inert unless a custom homedir is actually in play.
2. On activation, the script confirms `systemctl` and `gpgconf` are on `$PATH` (exit `127` otherwise), confirms `GNUPGHOME` is set and non-empty (exit `1` otherwise), and confirms the systemd user config directory already exists (exit `1` otherwise — see [Requirements](#requirements)).
3. It runs `gpgconf --list-dirs` and maps each `*-socket` key to a systemd unit name:

    | `gpgconf --list-dirs` key | systemd unit |
    |---|---|
    | `dirmngr-socket` | `dirmngr.socket` |
    | `keyboxd-socket` | `keyboxd.socket` |
    | `agent-socket` | `gpg-agent.socket` |
    | `agent-ssh-socket` | `gpg-agent-ssh.socket` |
    | `agent-extra-socket` | `gpg-agent-extra.socket` |
    | `agent-browser-socket` | `gpg-agent-browser.socket` |

    A key with no matching *installed* unit is logged and skipped rather than failing the run.
4. For each remaining key it writes:

    ```
    $XDG_CONFIG_HOME/systemd/user/<unit>.socket.d/90-systemd-override-gpg-socket.conf
    ```

    clearing the inherited `ListenStream=` and replacing it with the resolved path, via a tempfile + atomic `mv` so an interrupted run can't leave a truncated drop-in. The drop-in carries its own `ConditionEnvironment=GNUPGHOME`, so it's harmless even if read outside this service's control.
5. Any previously-generated `90-systemd-override-gpg-socket.conf` whose unit is no longer in the current set is removed, along with its `*.socket.d/` directory if that leaves it empty (a directory still holding other drop-ins is left alone).
6. `systemctl --user daemon-reload` picks up the changes.

The numeric `90-` prefix follows the same drop-in ordering convention used by `sysctl.d(5)`, `tmpfiles.d(5)`, and `udev` rules: files in a `.d/` directory are applied in lexical order, so a high prefix wins over lower-numbered drop-ins touching the same directive.

## Requirements

- `bash` ≥ 4 (associative arrays, `[[ -v ]]`)
- `systemd` (`systemctl --user`)
- GnuPG (`gpgconf`)
- GNU coreutils (`realpath`, `mktemp`)
- `$XDG_CONFIG_HOME/systemd/user` (or `~/.config/systemd/user`) must already exist — `install -D` below creates it as a side effect of installing the `.service` unit, so run the install steps in order.
- `GNUPGHOME` exported into the **systemd user manager's** own environment, most conventionally via a drop-in under `~/.config/environment.d/*.conf` — not just exported in your shell's rc files. Both `ConditionEnvironment=` and the script itself read the manager's environment, which shell-only exports never reach.

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

The `ListenStream=` shown under the `90-systemd-override-gpg-socket.conf` drop-in in `systemctl --user cat gpg-agent.socket` should match the path `gpgconf --list-dirs agent-socket` reports. Run-by-run detail (`INFO`/`ERROR`/`FATAL` lines) is available via:

```sh
journalctl --user -u systemd-override-gpg-socket.service
```

## Notes & Caveats

- **Safe no-op without a custom homedir.** If `GNUPGHOME` is unset in the manager's environment, both the service's `ConditionEnvironment=` and the script's own guard skip execution entirely. Stock GnuPG setups are unaffected.
- **Re-run after `GNUPGHOME` changes.** The hashed socket directory is derived from `GNUPGHOME`'s resolved path. Changing it — including moving the underlying directory — requires re-running the service (`systemctl --user restart systemd-override-gpg-socket.service`) to regenerate the overrides.
- **Stale overrides are pruned automatically.** If a socket unit that previously got an override disappears from the current run's mapping (e.g. the `gnupg` package no longer ships it), the script deletes the corresponding `90-systemd-override-gpg-socket.conf` and, if nothing else is left in it, the `.socket.d/` directory too.
- **Don't hand-edit the generated `.conf` files.** They're regenerated on every run and are marked as such in their own header; change the script or the unit file instead.

## License

GPL-3.0-or-later — see the SPDX header in each file.
