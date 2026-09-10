# Bash Global PS1 Prompt — English Study Notes

## Goal

These study notes explain a Bash configuration that applies a custom colored prompt **only to Bash** and **only to interactive shells**.

## Code

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

---

# Logical Block 1 — Apply Only to Bash

```bash
[ -n "${BASH_VERSION:-}" ] || return 0
```

The purpose of this block is:

> Check whether the current shell is Bash.

### `BASH_VERSION`

`BASH_VERSION` is a built-in Bash variable.

Check it with:

```bash
echo "$BASH_VERSION"
```

Example output:

```text
5.2.21(1)-release
```

If this variable has a value, the current shell is Bash.

### `${BASH_VERSION:-}`

This is a safe parameter expansion.

It means:

> Use the value of `BASH_VERSION` if it is set; otherwise use an empty string.

General syntax:

```text
${variable:-default}
```

### `-n`

```bash
-n "${BASH_VERSION:-}"
```

`-n` means:

> Check whether the string is not empty.

So:

```bash
[ -n "${BASH_VERSION:-}" ]
```

asks:

> Does `BASH_VERSION` contain a value?

### `|| return 0`

```bash
|| return 0
```

`||` means:

> Run the command on the right if the command on the left fails.

Flow:

```text
Is BASH_VERSION available?
        │
     ┌──┴──┐
    YES    NO
     │      │
 continue  return 0
```

If the shell is not Bash, the configuration stops here.

---

# Logical Block 2 — Apply Only to Interactive Shells

```bash
case $- in
    *i*) ;;
      *) return 0 ;;
esac
```

The purpose of this block is:

> Check whether the current shell is interactive.

An interactive shell is a shell where a prompt is displayed and the user types commands.

Example:

```text
khalid@server:~$
```

### `$-`

Run:

```bash
echo "$-"
```

`$-` shows the current shell option flags.

Example:

```text
himBHs
```

The important character here is:

```text
i
```

`i` means:

> Interactive shell

---

# Logical Block 3 — The `case` Statement

```bash
case $- in
```

This checks the value of `$-` against different patterns.

### `*i*) ;;`

```bash
*i*) ;;
```

Meaning:

> If `$-` contains `i` anywhere, the pattern matches.

The `*` is a wildcard.

### `;;`

Ends the current `case` pattern.

### `*) return 0 ;;`

```bash
*) return 0 ;;
```

This is the default case.

Meaning:

> If `i` is not present, the shell is not interactive, so stop processing this configuration.

Flow:

```text
Is the shell interactive?
        │
     ┌──┴──┐
    YES    NO
     │      │
 continue  return 0
```

---

# Logical Block 4 — Custom `PS1` Prompt

```bash
PS1='\[\e[32m\]\u\[\e[0m\]@\[\e[31m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

`PS1` is Bash's **Primary Prompt String** variable.

It controls how the command prompt appears.

Conceptual example:

```text
khalid@server:~/projects$
```

The colors are:

```text
Username   = Green
Hostname   = Red
Directory  = Blue
$ or #     = Normal terminal color
```

---

# Logical Block 5 — Green Username

```bash
\[\e[32m\]\u
```

### `\e[32m`

Starts green text.

### `\u`

Displays the current username.

Example:

```text
khalid
```

will appear in green.

---

# Logical Block 6 — Reset the Color

```bash
\[\e[0m\]
```

This resets the current formatting.

Meaning:

> Return to the terminal's normal color.

This is important so the next part of the prompt does not inherit the previous color.

---

# Logical Block 7 — Red Hostname

```bash
\[\e[31m\]\h
```

### `\e[31m`

Starts red text.

### `\h`

Displays the short hostname.

Example:

```text
rocky-node1
```

will appear in red.

---

# Logical Block 8 — Blue Working Directory

```bash
\[\e[34m\]\w
```

### `\e[34m`

Starts blue text.

### `\w`

Displays the current working directory.

Example:

```text
~/nit/shell-scripting
```

will appear in blue.

---

# Logical Block 9 — `\$`

```bash
\$
```

This displays the prompt symbol.

For a normal user:

```text
$
```

For root:

```text
#
```

So the same PS1 automatically distinguishes a normal user from root.

---

# Logical Block 10 — Why `\[` and `\]` Matter

Example:

```bash
\[\e[32m\]
```

`\[` and `\]` tell Bash:

> The characters inside are non-printing control sequences.

Color escape codes do not occupy visible screen width.

Without `\[` and `\]`, Bash may calculate prompt length incorrectly, causing problems such as:

- incorrect cursor positioning
- strange wrapping of long commands
- awkward command-line editing

For colored Bash prompts, this is an important best practice.

---

# Logical Block 11 — `export PS1`

```bash
export PS1
```

This marks `PS1` as an exported shell variable.

Simple meaning:

> Export the PS1 value into the shell environment.

The line that actually defines the prompt is:

```bash
PS1='...'
```

For an interactive Bash prompt, assigning `PS1` is the essential step. Exporting it is optional in many normal Bash setups, but it may be included when the configuration is intended to pass the variable through the environment.

---

# Complete Flow

```text
Configuration loads
        ↓
Is this Bash?
   ├── NO → return 0
   └── YES
        ↓
Is the shell interactive?
   ├── NO → return 0
   └── YES
        ↓
Set PS1
        ↓
Username = Green
Hostname = Red
Directory = Blue
        ↓
Prompt is ready
```

---

# Final Prompt Structure

Conceptually:

```text
Green     Red             Blue
  ↓        ↓                ↓
khalid@rocky-node1:~/projects$
```

Prompt components:

```text
\u  = username
\h  = short hostname
\w  = current working directory
\$  = $ for normal user, # for root
```

Color codes:

```text
\e[32m = Green
\e[31m = Red
\e[34m = Blue
\e[0m  = Reset
```

---

# Why These Safety Checks Matter

A simple prompt configuration such as:

```bash
PS1='...'
```

sets the prompt directly.

The safer version first checks:

```text
Bash?
  ↓
Interactive shell?
  ↓
Then apply PS1
```

This prevents the prompt configuration from being unnecessarily applied to non-Bash or non-interactive shell sessions.

---

# Important Note About `return 0`

These lines:

```bash
return 0
```

are appropriate when the code is inside a **sourced shell configuration file**.

Examples:

```text
~/.bashrc
/etc/profile.d/custom-prompt.sh
```

If the same code is placed in a standalone executable script and run as:

```bash
./script.sh
```

then a top-level `return` is not appropriate.

A standalone script normally uses:

```bash
exit 0
```

---

# Quick Revision

```text
BASH_VERSION          = Bash version variable
-n                    = string is non-empty
||                    = run next command if previous fails
$-                    = current shell option flags
i                     = interactive-shell flag
case ... esac         = pattern-based conditional
PS1                   = primary Bash prompt variable
\u                    = username
\h                    = short hostname
\w                    = current working directory
\$                    = $ for normal user, # for root
\e[32m                = green
\e[31m                = red
\e[34m                = blue
\e[0m                 = reset
\[ ... \]             = non-printing sequence markers
export PS1            = export the PS1 variable
```

---

# Final Memory Flow

```text
BASH CHECK
   ↓
INTERACTIVE CHECK
   ↓
SET COLORS
   ↓
USERNAME
   ↓
HOSTNAME
   ↓
WORKING DIRECTORY
   ↓
PROMPT SYMBOL
```
