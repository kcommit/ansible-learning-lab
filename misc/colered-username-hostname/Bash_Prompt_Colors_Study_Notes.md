# Bash Prompt Colors — Upgraded Study Notes

## Objective

Customize the Bash prompt so that:

- The **username** appears in green.
- The **hostname** appears in red.
- The **current working directory** appears in blue.

Example prompt:

```text
khalid@rocky-server:~/project$
```

## Temporary Configuration

Run this command in the terminal:

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

This change applies only to the current shell session. Closing the terminal removes it.

## Permanent Configuration

Open the current user's Bash configuration file:

```bash
nano ~/.bashrc
```

Add this line at the bottom:

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

Save the file and reload the configuration:

```bash
source ~/.bashrc
```

You can also open a new terminal session instead of running `source ~/.bashrc`.

## System-Wide Configuration for All Bash Users

The per-user method above changes only one user's prompt. For an administrator-managed default for existing and future Bash users, create a script under `/etc/profile.d/`.

### 1. Create the system-wide prompt file

If the file already exists, back it up first:

```bash
sudo cp -a /etc/profile.d/colored-prompt.sh \
    /etc/profile.d/colored-prompt.sh.backup
```

If it does not exist yet, there is nothing to back up. Open the file:

```bash
sudoedit /etc/profile.d/colored-prompt.sh
```

Add:

```bash
# Apply only to Bash.
[ -n "${BASH_VERSION:-}" ] || return 0

# Apply only to interactive shells that display a prompt.
case $- in
    *i*) ;;
      *) return 0 ;;
esac

PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
export PS1
```
[Explanation in English Click here](./explanation-English.md)

[Explanation in roman-Urdu Click here](./explanation-roman-urdu.md)


Set safe ownership and permissions:

```bash
sudo chown root:root /etc/profile.d/colored-prompt.sh
sudo chmod 644 /etc/profile.d/colored-prompt.sh
```

Check the Bash syntax:

```bash
bash -n /etc/profile.d/colored-prompt.sh
```

No output normally means that no syntax error was found. Test it in the current interactive shell:

```bash
source /etc/profile.d/colored-prompt.sh
```

### 2. Rocky Linux vs. Ubuntu/WSL

| System | System-wide interactive Bash file | Login configuration |
| --- | --- | --- |
| Rocky/RHEL | `/etc/bashrc` | `/etc/profile` and `/etc/profile.d/*.sh` |
| Ubuntu/WSL | `/etc/bash.bashrc` | `/etc/profile` and `/etc/profile.d/*.sh` |

Rocky/RHEL's standard Bash startup configuration normally loads `/etc/profile.d/*.sh` for login and interactive Bash sessions.

Ubuntu/WSL loads `/etc/profile.d/*.sh` for login shells. To also load this prompt from system-wide interactive non-login Bash, back up and edit `/etc/bash.bashrc`:

```bash
sudo cp -a /etc/bash.bashrc /etc/bash.bashrc.backup
sudoedit /etc/bash.bashrc
```

Add this block at the bottom:

```bash
if [ -r /etc/profile.d/colored-prompt.sh ]; then
    . /etc/profile.d/colored-prompt.sh
fi
```

Validate it:

```bash
bash -n /etc/bash.bashrc
```

> A user's own `~/.bashrc` can define `PS1` later and override the system default. This is normal Bash configuration precedence.

## Ensure Future Users Receive the Prompt

`/etc/skel` contains template files that are copied into a new user's home directory when the account is created with a home directory.

Back up and edit the skeleton `.bashrc`:

```bash
sudo cp -a /etc/skel/.bashrc /etc/skel/.bashrc.backup
sudoedit /etc/skel/.bashrc
```

Add this block at the bottom:

```bash
if [ -r /etc/profile.d/colored-prompt.sh ]; then
    . /etc/profile.d/colored-prompt.sh
fi
```

This helps make the administrator-managed prompt the final prompt setting for users created later.

> Updating `/etc/skel/.bashrc` affects only users created **after** the change. It does not modify existing users' home directories.

### Test with a new user on Rocky/RHEL

```bash
sudo useradd -m prompttest
sudo passwd prompttest
sudo -iu prompttest
```

### Test with a new user on Ubuntu/WSL

```bash
sudo adduser prompttest
sudo -iu prompttest
```

The new user should see a green username, red hostname, and blue working directory.

## Existing Users and Configuration Precedence

An existing user may already have another `PS1` assignment in `~/.bashrc`. Find prompt definitions with:

```bash
grep -n 'PS1=' ~/.bashrc /etc/bashrc /etc/bash.bashrc \
    /etc/profile /etc/profile.d/*.sh 2>/dev/null
```

The last applicable `PS1` assignment normally wins. To make an existing user load the administrator-managed prompt last, add this block at the bottom of that user's `~/.bashrc`:

```bash
if [ -r /etc/profile.d/colored-prompt.sh ]; then
    . /etc/profile.d/colored-prompt.sh
fi
```

Do not make `PS1` read-only just to enforce colors. Users and tools may legitimately need to customize their prompts.

## Verify the User's Shell

This configuration is specifically for Bash. Check a user's configured login shell:

```bash
getent passwd khalid
```

The output should normally end with:

```text
/bin/bash
```

Check the current shell process:

```bash
ps -p $$ -o comm=
```

If the user runs `zsh`, `fish`, or another shell, Bash `PS1` settings will not apply.

## Understanding the Prompt

| Code | Meaning | Example |
| --- | --- | --- |
| `\u` | Current username | `khalid` |
| `\h` | Short hostname | `rocky-server` |
| `\H` | Full hostname | `rocky-server.example.com` |
| `\w` | Full current working directory | `~/project/scripts` |
| `\W` | Current directory name only | `scripts` |
| `\$` | `$` for a regular user and `#` for root | `$` |
| `\@` | Current time in 12-hour format | `10:30 PM` |

## Understanding the Colors

The basic color format is:

```bash
\[\e[COLOR_CODEm\]
```

Common foreground color codes:

| Code | Color |
| ---: | --- |
| `30` | Black |
| `31` | Red |
| `32` | Green |
| `33` | Yellow |
| `34` | Blue |
| `35` | Purple/Magenta |
| `36` | Cyan |
| `37` | White |
| `90`–`97` | Bright versions of the colors |

Reset the color with:

```bash
\[\e[0m\]
```

Resetting is important; otherwise, text typed after the prompt may keep the previous color.

## Breaking Down the Command

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

| Section | Purpose |
| --- | --- |
| `\[\e[32m\]\u` | Displays the username in green |
| `\[\e[0m\]@` | Resets the color and displays `@` |
| `\[\e[31m\]\h` | Displays the hostname in red |
| `\[\e[0m\]:` | Resets the color and displays `:` |
| `\[\e[34m\]\w` | Displays the working directory in blue |
| `\[\e[0m\]\$` | Resets the color and displays `$` or `#` |

## Why `\[` and `\]` Are Important

Color codes do not take up visible space. Wrapping them in `\[` and `\]` tells Bash that they are non-printing characters.

Without these markers, long commands may wrap incorrectly, and editing command lines with the arrow keys can behave strangely.

## Useful Variations

### Green username and red hostname only

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]\$ '
```

### Bright green username and bright red hostname

```bash
PS1='\[\e[92m\]\u\[\e[0m\]@\[\e[91m\]\h\[\e[0m\]:\[\e[94m\]\w\[\e[0m\]\$ '
```

### Include the current time

```bash
PS1='[\@] \[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

## Regular User vs. Root

Each user has a separate `.bashrc` file:

- Regular user: `~/.bashrc`
- Root user: `/root/.bashrc`

If you customize your regular user's `.bashrc`, the root prompt will not automatically change.

Using `\$` is recommended because Bash automatically displays:

- `$` for a regular user
- `#` for root

## Restore the Default Prompt

Remove or comment out the custom `PS1` line in `~/.bashrc`:

```bash
# PS1='custom prompt was here'
```

Then reload the file:

```bash
source ~/.bashrc
```

### Disable the system-wide prompt

Rename the profile script so it no longer ends in `.sh`:

```bash
sudo mv /etc/profile.d/colored-prompt.sh \
    /etc/profile.d/colored-prompt.sh.disabled
```

Remove the block that sources it from `/etc/bash.bashrc`, `/etc/skel/.bashrc`, and any existing users' `.bashrc` files where you added it. Then open a fresh terminal or log in again.

## Troubleshooting

### The colors work after `source`, but not after login

Check which startup files load the prompt:

```bash
grep -nE 'profile\.d|colored-prompt|PS1=' \
    /etc/profile /etc/bashrc /etc/bash.bashrc \
    ~/.bash_profile ~/.profile ~/.bashrc 2>/dev/null
```

### A personal prompt overrides the server default

Search the user's startup files for a later assignment:

```bash
grep -n 'PS1=' ~/.bash_profile ~/.profile ~/.bashrc 2>/dev/null
```

### Long commands wrap incorrectly

Make sure every non-printing color sequence is enclosed in `\[` and `\]`.

### Security and administration notes

- Keep the system-wide prompt file owned by `root` and non-writable by normal users.
- Test changes in a second terminal before closing the current administrator session.
- Run `bash -n` before sourcing an edited Bash file.
- Do not put passwords, tokens, or other secrets in `PS1`.
- Avoid expensive commands in `PS1`; prompt content is evaluated repeatedly.
- Prompt colors are a visual aid, not an access-control feature.

## Quick Practice

Try changing the hostname from red (`31`) to cyan (`36`):

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[36m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

## Key Point

`PS1` controls the primary interactive Bash prompt. Test it in the terminal first. Use `~/.bashrc` for one user, `/etc/profile.d/colored-prompt.sh` for an administrator-managed system default, and `/etc/skel/.bashrc` to prepare users created later. Existing personal startup files may override the system default.
