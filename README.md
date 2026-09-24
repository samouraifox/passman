# passman

A simple, modern password manager built on [age](https://age-encryption.org)
and [wl-clipboard](https://github.com/bugaevc/wl-clipboard), with one encrypted
file, self-clearing copies, and no daemon or config.

```
$ passman switch
Copied switch. Clears in 30 s.
```

You type a name. The password lands on the clipboard. You paste it once. It
clears itself.

## Why it exists

Desktop clipboard managers record every copy. On omarchy, that record is a
plain-text file on disk. A password manager that copies to the clipboard and
does nothing else hands every password to that file.

`passman` marks each copy as sensitive (`wl-copy --sensitive`), which the
clipboard watcher honours, and clears the clipboard after 30 seconds.

The script does no cryptography. `age` does it. The kernel keyring caches the
unlocked key. The script is only the interface — about 150 lines of bash.

## Install

Dependencies: `age`, `keyutils`, `wl-clipboard`, `neovim`, `awk`.

```
sudo pacman -S --needed age keyutils wl-clipboard neovim
git clone https://github.com/samouraifox/passman
chmod +x passman/passman
ln -s "$PWD/passman/passman" ~/.local/bin/passman
passman -i
```

`-i` makes a key pair and an empty store. It asks a passphrase two times.
**That passphrase protects the private key. There is no recovery.**

## Use

| Command | Action |
|---|---|
| `passman NAME` | Copy the password to the clipboard. Clear it after 30 s. |
| `passman -n NAME` | Print the note (user name, address). |
| `passman -l` | List the names. |
| `passman -a NAME` | Add an entry. Asks the password two times, hidden. |
| `passman -e` | Edit the whole store in `nvim`. |
| `passman -x` | Forget the cached key now. |
| `passman -i` | Create the key pair and an empty store. |

The first command in ten minutes asks the passphrase. The others ask nothing.

## How it works

Three files in `~/.local/share/passman/`:

| File | Content |
|---|---|
| `key.age` | the private key, encrypted with your passphrase |
| `recipient.txt` | the public key, plain text |
| `store.age` | the entries, encrypted to the public key |

The store is one text file inside the encryption. One entry per line:

```
switch     a1b2c3...   admin  10.10.10.2
opnsense   d4e5f6...   root   https://10.10.10.1
```

Name, password, note. Lines that start with `#` are comments.

**Reading.** `unlock` looks for the private key in the kernel keyring (`keyctl`,
user keyring `@u`). If it is there, it uses it. If not, `age -d key.age` asks
the passphrase once, and the key goes into the keyring with a 600 s timeout.
Then `age -d -i /dev/stdin store.age` decrypts the store into a pipe, and `awk`
picks one field. Nothing plain touches the disk.

**Writing.** `age -r PUBLIC_KEY` encrypts to a temp file, then `mv` replaces the
store. Encryption to a public key needs no secret, so it never asks anything.
The previous store is kept as `store.age.bak`.

**Editing.** The plain text goes to a file under `$XDG_RUNTIME_DIR` — a tmpfs,
RAM — and `nvim -n -i NONE` opens it with the swap file and shada off. A trap
deletes the file when the script ends. If the fingerprint did not change, the
store is not rewritten.

## What it does not protect against

- While the key is cached, any program running as your user can read it from
  the keyring. Same window as `sudo`, same model as `gpg-agent`. `-x` ends it
  early; the timeout ends it anyway.
- A password with a space in it. The store format is space-separated. `-a`
  refuses one.
- Losing `key.age`. Back up the whole `~/.local/share/passman/` directory. All
  three files are safe to copy anywhere.

## Design

[DESIGN.md](DESIGN.md) holds the full design, the reasons for each decision,
and the tests.
