# ✂️ sed (Stream Editor) Cheat Sheet

A daily-use reference for the most frequently used `sed` commands and syntax.

---

## 📋 Table of Contents
- [What is sed?](#-what-is-sed)
- [Basic Syntax](#-basic-syntax)
- [Common Options](#-common-options)
- [Substitute (Find & Replace)](#-substitute-find--replace)
- [Delete Lines](#-delete-lines)
- [Print Lines](#-print-lines)
- [Insert, Append, Change](#-insert-append-change)
- [Addresses (Line Numbers, Ranges, Patterns)](#-addresses-line-numbers-ranges-patterns)
- [Regex Tips](#-regex-tips)
- [Using Shell Variables](#-using-shell-variables)
- [DevOps Real-World Examples](#-devops-real-world-examples)
- [Gotchas](#-gotchas)

---

## 🤔 What is sed?

`sed` reads input **line by line**, applies your command, and prints the result. It does not open the file in an editor, so it is ideal for **scripts, pipelines, and automation**.

```
Input → [ sed: read line → apply command → print ] → Output
```

---

## 🧱 Basic Syntax

```bash
sed [OPTIONS] 'COMMAND' file
command | sed [OPTIONS] 'COMMAND'
```

**Command structure:**
```
[address]command[flags]
```

---

## ⚙️ Common Options

| Option | Meaning |
|---|---|
| `-i` | Edit file **in-place** (saves changes to the file) |
| `-i.bak` | In-place edit + create a backup (`file.bak`) |
| `-n` | Silent mode, print only when told to (`p`) |
| `-e` | Add multiple commands |
| `-E` | Use extended regex (no need to escape `()`, `+`, `?`, `|`) |
| `-f script.sed` | Read commands from a script file |

---

## 🔁 Substitute (Find & Replace)

```bash
sed 's/old/new/' file            # replace 1st match on each line
sed 's/old/new/g' file           # replace ALL matches on each line (g = global)
sed 's/old/new/2' file           # replace only the 2nd match on each line
sed 's/old/new/2g' file          # replace 2nd match onwards
sed 's/old/new/I' file           # case-insensitive match (GNU sed)
sed -i 's/old/new/g' file        # save changes to the file
sed -i.bak 's/old/new/g' file    # save changes + keep backup
sed -i 's/old/new/g' *.conf      # multiple files at once
```

**Different delimiter** (useful when the text contains `/`):
```bash
sed 's|/old/path|/new/path|g' file
sed 's#/old/path#/new/path#g' file
```

**Use matched text with `&`:**
```bash
sed 's/error/[&]/g' file         # error → [error]
```

**Capture groups:**
```bash
sed -E 's/(first) (second)/\2 \1/' file    # swap two words
```

**Multiple commands:**
```bash
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file
sed 's/foo/bar/g; s/baz/qux/g' file
```

---

## 🗑️ Delete Lines

```bash
sed '5d' file                    # delete line 5
sed '5,10d' file                 # delete lines 5 to 10
sed '$d' file                    # delete last line
sed '1d' file                    # delete first line
sed '/pattern/d' file            # delete lines containing pattern
sed '/^$/d' file                 # delete blank lines
sed '/^#/d' file                 # delete comment lines
sed '/^\s*$/d' file              # delete blank/whitespace-only lines
sed -i '/pattern/d' file         # delete and save in the file
```

---

## 🖨️ Print Lines

> Use `-n` together with `p`, otherwise lines print twice.

```bash
sed -n '5p' file                 # print line 5 only
sed -n '10,20p' file             # print lines 10 to 20
sed -n '$p' file                 # print last line
sed -n '/pattern/p' file         # print lines matching pattern
sed -n '/start/,/end/p' file     # print from "start" to "end"
sed -n '$=' file                 # count total lines (like wc -l)
sed -n '0~2p' file               # print every 2nd line (GNU sed)
```

---

## ➕ Insert, Append, Change

```bash
sed '3i\New line before line 3' file       # insert BEFORE line 3
sed '3a\New line after line 3' file        # append AFTER line 3
sed '3c\Replace line 3 with this' file     # change/replace whole line 3
sed '1i\Header line' file                  # add line at top of file
sed '$a\Footer line' file                  # add line at end of file
sed '/pattern/a\Added after match' file    # append after matching lines
sed '/pattern/i\Added before match' file   # insert before matching lines
sed '/pattern/c\Replacement line' file     # replace matching lines entirely
```

---

## 🎯 Addresses (Line Numbers, Ranges, Patterns)

| Address | Meaning |
|---|---|
| `5` | Line 5 |
| `5,10` | Lines 5 to 10 |
| `5,+3` | Line 5 and the next 3 lines (GNU) |
| `$` | Last line |
| `/pattern/` | Lines matching pattern |
| `/start/,/end/` | From first match of `start` to `end` |
| `5!d` | `!` negates (everything except line 5) |
| `0~3` | Every 3rd line (GNU) |

**Apply a command only on specific lines:**
```bash
sed '3s/old/new/' file                   # substitute only on line 3
sed '5,10s/old/new/g' file               # substitute only on lines 5-10
sed '/server/s/old/new/g' file           # substitute only on lines containing "server"
```

---

## 🔤 Regex Tips

| Pattern | Meaning |
|---|---|
| `^` | Start of line |
| `$` | End of line |
| `.` | Any single character |
| `*` | Zero or more of previous |
| `[abc]` | Any one of a, b, c |
| `[0-9]` | Any digit |
| `\s` | Whitespace (GNU) |
| `.*` | Anything (any length) |

**Useful one-liners:**
```bash
sed 's/^/# /' file               # add "# " at start of every line (comment out)
sed 's/^# //' file               # remove "# " from start (uncomment)
sed 's/$/;/' file                # add ";" at end of every line
sed 's/^[ \t]*//' file           # trim leading whitespace
sed 's/[ \t]*$//' file           # trim trailing whitespace
sed 's/[0-9]//g' file            # remove all digits
sed 'y/abc/xyz/' file            # transliterate characters (a→x, b→y, c→z)
```

---

## 💲 Using Shell Variables

Use **double quotes** so the shell expands the variable:

```bash
NEW_TAG="v2.1"
sed -i "s/image: myapp:.*/image: myapp:${NEW_TAG}/" deployment.yaml
```

---

## 🛠️ DevOps Real-World Examples

```bash
# Update image tag in a Kubernetes manifest
sed -i 's|image: myapp:.*|image: myapp:v2.1|' deployment.yaml

# Change replica count
sed -i 's/replicas: [0-9]*/replicas: 5/' deployment.yaml

# Disable root SSH login
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Comment out a specific line in a config
sed -i '/some_setting/s/^/#/' app.conf

# Uncomment a specific line
sed -i '/some_setting/s/^#//' app.conf

# Inject a version in a CI/CD pipeline
sed -i "s/__VERSION__/${BUILD_NUMBER}/g" config.yaml

# Show only ERROR lines from a log (pipe)
cat app.log | sed -n '/ERROR/p'

# Strip comments and blank lines from a config
sed -e '/^#/d' -e '/^$/d' nginx.conf

# Print a specific range from a log by pattern
sed -n '/2026-10-24 10:00/,/2026-10-24 11:00/p' app.log
```

---

## ⚠️ Gotchas

- **`-i` changes the file permanently.** First run without `-i` to preview, or use `-i.bak` for a backup.
- **macOS (BSD sed)** needs an argument after `-i`: `sed -i '' 's/a/b/' file`. Many GNU features (`I` flag, `0~2`, `\s`) may not work there.
- **Single quotes** (`'...'`) stop variable expansion, and **double quotes** (`"..."`) allow it.
- **Special characters** like `/`, `.`, `*`, `[` need escaping (`\/`, `\.`) or a different delimiter.
- `sed` works **line by line**, so matching text across multiple lines needs extra tricks (or use `awk` / `perl`).

---

*💡 Tip: Preview first with `sed 's/old/new/g' file`, then add `-i` once the output looks right.*
