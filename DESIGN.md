# passman — design

*Approved 2026-09-23. Revised the same day: key file + kernel keyring cache
instead of a passphrase on the store. Built step by step, the same way as the
homelab.*

A command-line password manager for one user on one laptop. You type a name.
The password lands on the clipboard. You paste it once. It clears itself.

The rule that makes this acceptable: **the script never does cryptography.**
`age` does it. The kernel holds the cached key. The script is only the
interface.

## 1. Where things live

| Item | Path |
|---|---|
| Script | `~/.local/bin/passman` → symlink to the repo copy |
| Source | `~/Work/Projects/passman/` — this repo |
| Store | `~/.local/share/passman/store.age` — the entries, encrypted to the public key |
| Private key | `~/.local/share/passman/key.age` — encrypted with your passphrase |
| Public key | `~/.local/share/passman/recipient.txt` — plain text, not secret |
| Old store | `~/.local/share/passman/store.age.bak` — kept after each change |

Directory mode 700, files mode 600. No daemon. No config file.

## 2. Inside the store

One entry per line. Three fields, separated by spaces.

```
switch     a1b2c3...   admin  10.10.10.2
opnsense   d4e5f6...   root   https://10.10.10.1
core       ...         root   ssh 10.10.10.3
```

| Field | Rule |
|---|---|
| name | no spaces |
| password | no spaces |
| note | free text: user name, address, anything |

Lines that start with `#` are comments. Empty lines are ignored.

## 3. Commands

| Command | Action |
|---|---|
| `passman NAME` | Copy the password to the clipboard. Clear it after 30 s. |
| `passman -n NAME` | Print the note only. |
| `passman -l` | List the names only. |
| `passman -a NAME` | Add an entry. Asks for the password two times, hidden. |
| `passman -e` | Open the whole store in the editor. |
| `passman -i` | Make the key pair and an empty store. |
| `passman -x` | Forget the cached key now. |

Exit status: 0 on success, 1 on any error. On error nothing is copied.

## 4. Keys and the passphrase

`passman -i` runs `age-keygen`, which makes a key pair.

| Half | File | Protected by |
|---|---|---|
| public | `recipient.txt` | nothing — it is not secret |
| private | `key.age` | your passphrase, through `age -p` |

The store is encrypted **to the public key**. Encryption never asks for
anything. Decryption needs the private key, and the private key needs the
passphrase. So the passphrase protects one small file, and only decryption
ever needs it.

Why not a passphrase on the store itself (the first design):

- `age -p` reads the passphrase from the terminal only. Not from a file, a
  variable, or a pipe. So it cannot be asked once and reused.
- Every add or edit re-encrypts the store, so it asked three times.
- If you typed the *same wrong* passphrase twice at the encrypt prompt, the
  store would be locked with a passphrase you do not know. The confirm prompt
  catches a typo, not a consistent mistake. With a key file, a mistyped
  passphrase can only fail to unlock. It can never change what the store is
  locked with.

## 5. The keyring cache

Linux keeps a key store inside the kernel (`keyctl`, package `keyutils`).
`passman` uses the user keyring, `@u`, which all your processes share.

`unlock()`:

1. Look for a key named `passman` in `@u`. If it is there, print it and stop.
2. If not, `age -d key.age` asks the passphrase once and prints the private key.
3. Put the private key in `@u` with `keyctl padd` (payload on stdin, never on a
   command line) and set a timeout of `CACHE_SECONDS` (600).
4. Print the private key.

Every command that reads the store calls `unlock`. So the first call in ten
minutes asks the passphrase; the others ask nothing. `passman -x` unlinks the
key from `@u` now. When the timeout passes, the kernel drops it on its own.

**Cost, stated plainly:** while the key is cached, any program that runs as
your user can read it from `@u`. This is the same window `sudo` gives for 15
minutes, and the same model as `gpg-agent`. The timeout bounds it; `-x` ends it
early.

Checked on the laptop 2026-09-23: `padd` with a timeout, a read from a new
process, expiry, and a re-add after expiry all behave as described.

## 6. What happens on `passman NAME`

1. `unlock` gives the private key — from the keyring, or after one passphrase.
2. `age -d -i /dev/stdin store.age` reads the key from stdin and writes the
   plain text to a **pipe**, not a file.
3. `awk` reads the pipe and finds the line whose first field is NAME. It keeps
   field 2.
4. `wl-copy --sensitive` puts the password on the clipboard.
5. A background timer waits 30 s. If the clipboard still holds the password,
   it clears it.
6. The script prints `Copied NAME. Clears in 30 s.` It never prints the
   password.

The plain password exists in the pipe, in one shell variable, and on the
clipboard. Never in a file.

## 7. Why `--sensitive` is the key line

omarchy's clipboard watcher (`/usr/share/omarchy/shell/plugins/clipboard/capture.sh`)
checks each copy for the `x-kde-passwordManagerHint` MIME type. If the type is
present, the watcher exits and records nothing. `wl-copy --sensitive`
(wl-clipboard ≥ 2.3.0) sets that type.

Without this line, every password goes into
`~/.local/state/omarchy/clipboard-history.json` in plain text. That is what
happened on 2026-09-23 with the new lab passwords, and why that file was purged.

## 8. Editing without a leak

`passman -e`:

1. Decrypt to a file under `$XDG_RUNTIME_DIR` (`/run/user/1000`, a tmpfs —
   RAM, not disk). Mode 600.
2. Open `nvim -n -i NONE` on it: no swap file, no shada. Without these flags
   the editor writes a plain copy to disk. `$EDITOR` on this laptop is an
   omarchy wrapper, so the script calls `nvim` directly.
3. After save, encrypt to the public key. Write to a temp name, then `mv` over
   the store. Keep the old store as `store.age.bak`.
4. Delete the plain file.

`passman -a` does the same without an editor. Neither asks a passphrase unless
the cache is empty, and then only once.

## 9. Migration

1. `passman -i`
2. `sudo cat ~/Work/Projects/Secrets/lab-passwords`, then `passman -a` for
   each entry.
3. `sudo rm ~/Work/Projects/Secrets/lab-passwords`. Not `shred`: the disk is
   btrfs (copy-on-write, so shred does not overwrite in place) inside LUKS
   (so the data at rest is already encrypted).

## 10. Backup

Back up the whole directory `~/.local/share/passman/`. `key.age` is the file
that matters: without it the store cannot be read, and no passphrase brings it
back. `recipient.txt` can be made again from the key. All three files are safe
to copy anywhere.

## 11. Tests

| Test | Expected |
|---|---|
| `-i` | asks the passphrase two times; three files appear, modes 600, dir 700 |
| `-l` right after | asks the passphrase once, prints nothing |
| `-l` again | asks nothing |
| `-a test` | asks nothing; `added test` |
| lookup → paste | password appears |
| wait 30 s → paste | clipboard is empty |
| lookup, then count hex strings in the history file | count does not change |
| `-x`, then `-l` | asks the passphrase once |
| wrong passphrase | error message, nothing copied, exit 1 |
| unknown name | error message, nothing copied, exit 1 |
