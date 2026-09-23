# passman — design

*Approved 2026-09-23. Built step by step, the same way as the homelab.*

A command-line password manager for one user on one laptop. You type a name.
The password lands on the clipboard. You paste it once. It clears itself.

The rule that makes this acceptable: **the script never does cryptography.**
`age` does it. The script is only the interface.

## 1. Where things live

| Item | Path |
|---|---|
| Script | `~/.local/bin/passman` (`~/.local/bin` is on PATH) |
| Source | `~/Work/Projects/passman/` — this repo |
| Store | `~/.local/share/passman/store.age` — one file, mode 600 |
| Old store | `~/.local/share/passman/store.age.bak` — the previous version, kept after each edit |

No daemon. No cache. No key file. The store is encrypted by `age` with a
passphrase. That passphrase is the one password to remember.

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
| `passman NAME` | Ask passphrase. Copy the password to the clipboard. Clear it after 30 s. |
| `passman -n NAME` | Print the note only. |
| `passman -l` | List the names only. |
| `passman -a NAME` | Add an entry. Asks for the password two times, hidden. |
| `passman -e` | Open the whole store in the editor. |
| `passman -i` | Make a new empty store. |

Exit status: 0 on success, 1 on any error. On error nothing is copied.

## 4. What happens on `passman NAME`

1. `age -d store.age` asks the passphrase on the terminal. It writes the plain
   text to a **pipe**, not a file.
2. `awk` reads the pipe and finds the line whose first field is NAME. It keeps
   field 2.
3. `wl-copy --sensitive` puts the password on the clipboard.
4. A background timer waits 30 s. If the clipboard still holds the password,
   it clears it.
5. The script prints `Copied NAME. Clears in 30 s.` It never prints the
   password.

The plain password exists in three places only: the pipe, one shell variable,
and the clipboard. Never in a file.

## 5. Why `--sensitive` is the key line

omarchy's clipboard watcher (`/usr/share/omarchy/shell/plugins/clipboard/capture.sh`)
checks each copy for the `x-kde-passwordManagerHint` MIME type. If the type is
present, the watcher exits and records nothing. `wl-copy --sensitive`
(wl-clipboard ≥ 2.3.0) sets that type.

Without this line, every password goes into
`~/.local/state/omarchy/clipboard-history.json` in plain text. That is what
happened on 2026-09-23 with the new lab passwords, and why that file was purged.

## 6. Editing without a leak

`passman -e`:

1. Decrypt to a file under `$XDG_RUNTIME_DIR` (`/run/user/1000`, a tmpfs —
   RAM, not disk). Mode 600.
2. Open `nvim -n -i NONE` on it: no swap file, no shada. Without these flags
   the editor writes a plain copy to disk. `$EDITOR` on this laptop is an
   omarchy wrapper, so the script calls `nvim` directly.
3. After save, encrypt again. Write to a temp name, then `mv` over the store.
   Keep the old store as `store.age.bak`.
4. Delete the plain file.

`passman -a` does the same without an editor.

**Cost:** `age` asks the passphrase three times on add or edit — one to
decrypt, two to encrypt. These are rare operations. If it annoys, the upgrade
is an `age` identity file with one prompt. Not now.

## 7. Migration

1. `passman -i`
2. `sudo cat ~/Work/Projects/Secrets/lab-passwords`, then `passman -a` for
   each entry.
3. `sudo rm ~/Work/Projects/Secrets/lab-passwords`. Not `shred`: the disk is
   btrfs (copy-on-write, so shred does not overwrite in place) inside LUKS
   (so the data at rest is already encrypted).

## 8. Backup

The store is one file. Lose it and every password is gone. Put it in the
backup, or make it a sync pair to the NAS later. It is safe to copy anywhere,
because it is encrypted.

## 9. Tests

| Test | Expected |
|---|---|
| init → add → lookup → paste | password appears |
| wait 30 s → paste | clipboard is empty |
| lookup, then count hex strings in the history file | count does not change |
| wrong passphrase | error message, nothing copied, exit 1 |
| unknown name | error message, nothing copied, exit 1 |
