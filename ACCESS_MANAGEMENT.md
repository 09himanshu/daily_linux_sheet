# 🐧 Linux File Permission Management

A daily-use reference for managing file permissions in Linux.

---

## 📋 Table of Contents
- [File Permissions Basics](#-file-permissions-basics)
- [Changing Permissions (chmod)](#-changing-permissions-chmod)
- [Changing Ownership (chown/chgrp)](#-changing-ownership-chownchgrp)
- [Special Permissions](#-special-permissions)
- [Finding Files by Permission/Owner](#-finding-files-by-permissionowner)
- [Useful One-Liners](#-useful-one-liners)

---

## 🔐 File Permissions Basics

Every file/directory has three permission sets: **owner**, **group**, **others**.

```
-rwxr-xr--  1 himanshu devs  1024 Oct  2 10:00 script.sh
 │└┬┘└┬┘└┬┘
 │ │  │  └── others: r--
 │ │  └───── group:  r-x
 │ └──────── owner:  rwx
 └────────── file type (- = file, d = directory, l = symlink)
```

| Symbol | Meaning (file) | Meaning (directory) |
|---|---|---|
| `r` | Read contents | List contents |
| `w` | Modify contents | Create/delete files inside |
| `x` | Execute file | Enter directory (`cd`) |

### Octal Values
| Octal | Permission |
|---|---|
| `0` | `---` |
| `1` | `--x` |
| `2` | `-w-` |
| `3` | `-wx` |
| `4` | `r--` |
| `5` | `r-x` |
| `6` | `rw-` |
| `7` | `rwx` |

---

## 🛠 Changing Permissions (chmod)

**Octal (numeric) mode:**
```bash
chmod 755 file     # rwxr-xr-x
chmod 644 file     # rw-r--r--
chmod -R 755 dir/   # recursive
```

**Symbolic mode:**
```bash
chmod u+x file      # add execute for owner
chmod g-w file      # remove write for group
chmod o=r file       # set others to read-only
chmod a+r file       # add read for all (owner+group+others)
chmod u+x,g+x file   # multiple changes at once
```

| Symbol | Target |
|---|---|
| `u` | user/owner |
| `g` | group |
| `o` | others |
| `a` | all |

---

## 👑 Changing Ownership (chown/chgrp)

```bash
chown user file              # change owner
chown user:group file        # change owner + group
chown :group file             # change group only
chown -R user:group dir/      # recursive
chgrp group file               # change group only (alt)
```

---

## ⭐ Special Permissions

| Permission | Octal prefix | Effect |
|---|---|---|
| SUID | `4xxx` | Run file as file's owner |
| SGID | `2xxx` | Run file as file's group / new files inherit dir's group |
| Sticky bit | `1xxx` | Only file owner can delete, even if others have write (e.g. `/tmp`) |

```bash
chmod 4755 file      # set SUID
chmod 2755 dir/       # set SGID on directory
chmod 1777 dir/       # set sticky bit (like /tmp)
chmod u+s file         # SUID symbolic
chmod g+s dir/          # SGID symbolic
chmod +t dir/            # sticky bit symbolic
```

---

## 🔍 Finding Files by Permission/Owner

```bash
find / -perm 755 2>/dev/null           # exact permission match
find / -perm -u+s 2>/dev/null          # files with SUID set
find / -user username 2>/dev/null       # files owned by user
find / -group groupname 2>/dev/null     # files owned by group
find / -writable 2>/dev/null            # files writable by current user
```

---

## ⚡ Useful One-Liners

```bash
cut -d: -f1 /etc/passwd              # list all usernames
awk -F: '{print $1}' /etc/group      # list all group names
getent passwd username               # check if user exists
lslogins                              # detailed overview of all users
namei -l /path/to/file                # show permissions of every path component
stat file                              # detailed permission/owner info
```

---

*💡 Tip: Keep this as `README.md` in a personal "linux-cheatsheets" repo for quick reference.*
