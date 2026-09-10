# Bash Global PS1 Prompt — Roman Urdu Study Notes

## Goal

Ye study notes is Bash configuration ko explain karti hain jo custom colored prompt ko **sirf Bash** aur **sirf interactive shells** mein apply karta hai.

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

# Logical Block 1 — Sirf Bash ke Liye Apply Karna

```bash
[ -n "${BASH_VERSION:-}" ] || return 0
```

Is block ka purpose:

> Check karo ke current shell Bash hai ya nahi.

### `BASH_VERSION`

`BASH_VERSION` Bash ka built-in variable hai.

Check:

```bash
echo "$BASH_VERSION"
```

Example output:

```text
5.2.21(1)-release
```

Agar ye variable available hai, current shell Bash hai.

### `${BASH_VERSION:-}`

Ye safe variable expansion hai.

Meaning:

> Agar `BASH_VERSION` set hai to uski value use karo, warna empty string use karo.

General syntax:

```text
${variable:-default}
```

### `-n`

```bash
-n "${BASH_VERSION:-}"
```

`-n` ka matlab:

> Check karo ke string empty nahi hai.

So:

```bash
[ -n "${BASH_VERSION:-}" ]
```

ka matlab:

> Kya `BASH_VERSION` non-empty hai?

### `|| return 0`

```bash
|| return 0
```

`||` ka matlab:

> Agar left-side command fail kare to right-side command run karo.

Flow:

```text
BASH_VERSION available?
        │
     ┌──┴──┐
    YES    NO
     │      │
 continue  return 0
```

Agar Bash nahi hai, configuration yahin stop ho jati hai.

---

# Logical Block 2 — Sirf Interactive Shell ke Liye

```bash
case $- in
    *i*) ;;
      *) return 0 ;;
esac
```

Is block ka purpose:

> Check karo ke current shell interactive hai ya nahi.

Interactive shell wo hoti hai jahan prompt nazar aata hai aur user commands type karta hai.

Example:

```text
khalid@server:~$
```

### `$-`

```bash
echo "$-"
```

`$-` current shell ke enabled flags/options show karta hai.

Example:

```text
himBHs
```

Yahan important character:

```text
i
```

hai.

`i` means:

> Interactive shell

---

# Logical Block 3 — `case` Statement

```bash
case $- in
```

Ye `$-` ki value ko different patterns ke against check karta hai.

### `*i*) ;;`

```bash
*i*) ;;
```

Meaning:

> Agar `$-` ke andar kahin bhi `i` ho to match ho gaya.

Yahan `*` wildcard hai.

### `;;`

Current `case` option ko finish karta hai.

### `*) return 0 ;;`

```bash
*) return 0 ;;
```

Ye default case hai.

Meaning:

> Agar `i` nahi mila to shell interactive nahi hai, isliye configuration yahin stop karo.

Flow:

```text
Shell interactive hai?
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

`PS1` Bash ka **Primary Prompt String** variable hai.

Ye decide karta hai ke terminal prompt kaisa nazar aayega.

Example concept:

```text
khalid@server:~/projects$
```

Is prompt mein:

```text
Username   = Green
Hostname   = Red
Directory  = Blue
$ or #     = Normal
```

---

# Logical Block 5 — Green Username

```bash
\[\e[32m\]\u
```

### `\e[32m`

Green color start karta hai.

### `\u`

Current username show karta hai.

Example:

```text
khalid
```

green color mein show hoga.

---

# Logical Block 6 — Color Reset

```bash
\[\e[0m\]
```

Ye current color formatting reset karta hai.

Meaning:

> Ab normal terminal color par wapas jao.

Ye important hai taake next text previous color inherit na kare.

---

# Logical Block 7 — Red Hostname

```bash
\[\e[31m\]\h
```

### `\e[31m`

Red color start karta hai.

### `\h`

Short hostname show karta hai.

Example:

```text
rocky-node1
```

red color mein show hoga.

---

# Logical Block 8 — Blue Working Directory

```bash
\[\e[34m\]\w
```

### `\e[34m`

Blue color start karta hai.

### `\w`

Current working directory show karta hai.

Example:

```text
~/nit/shell-scripting
```

blue color mein show hoga.

---

# Logical Block 9 — `\$`

```bash
\$
```

Ye prompt symbol show karta hai.

Normal user:

```text
$
```

Root user:

```text
#
```

Yani same PS1 automatically normal user aur root ko differentiate kar sakta hai.

---

# Logical Block 10 — `\[` aur `\]` Kyun Important Hain?

Example:

```bash
\[\e[32m\]
```

`\[` aur `\]` Bash ko batate hain:

> Ye characters screen par visible text ka part nahi hain.

Color escape sequences actual visible characters nahi hoti.

Agar `\[` aur `\]` na hon to:

- cursor position weird ho sakti hai
- long commands wrap incorrectly ho sakti hain
- command editing awkward ho sakti hai

So ye PS1 best practice hai.

---

# Logical Block 11 — `export PS1`

```bash
export PS1
```

Ye `PS1` variable ko exported shell variable bana deta hai.

Simple meaning:

> PS1 ko shell environment mein export karo.

Prompt ko actually define karne wali main line:

```bash
PS1='...'
```

hai.

---

# Complete Flow

```text
Configuration load
        ↓
Kya Bash shell hai?
   ├── NO → return 0
   └── YES
        ↓
Kya shell interactive hai?
   ├── NO → return 0
   └── YES
        ↓
PS1 set karo
        ↓
Username = Green
Hostname = Red
Directory = Blue
        ↓
Prompt ready
```

---

# Final Prompt Structure

Conceptually:

```text
Green     Red             Blue
  ↓        ↓                ↓
khalid@rocky-node1:~/projects$
```

Prompt breakdown:

```text
\u  = username
\h  = hostname
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

Simple PS1 line:

```bash
PS1='...'
```

directly prompt set kar deti hai.

Polished version pehle verify karti hai:

```text
Bash?
  ↓
Interactive?
  ↓
Then apply PS1
```

Is se configuration scripts ya non-Bash shells par unnecessary apply nahi hoti.

---

# Important Note About `return 0`

Ye lines:

```bash
return 0
```

sourced shell configuration files ke liye suitable hain.

Examples:

```text
~/.bashrc
/etc/profile.d/custom-prompt.sh
```

Agar same code standalone executable script mein ho:

```bash
./script.sh
```

to top-level `return` suitable nahi hota.

Standalone script mein normally:

```bash
exit 0
```

use hota hai.

---

# Quick Revision

```text
BASH_VERSION          = Bash version variable
-n                    = string non-empty check
||                    = run next command if previous fails
$-                    = current shell flags
i                     = interactive shell flag
case ... esac         = pattern-based condition
PS1                   = primary Bash prompt variable
\u                    = username
\h                    = short hostname
\w                    = current working directory
\$                    = $ for normal user, # for root
\e[32m                = green
\e[31m                = red
\e[34m                = blue
\e[0m                 = reset
\[ ... \]             = non-printing escape markers
export PS1            = export PS1 variable
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
