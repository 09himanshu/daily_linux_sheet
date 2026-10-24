# 🐧 Linux User & Group Management Cheat Sheet

A quick-reference guide for daily Linux user and group administration tasks.

---

## 📋 Table of Contents
- [User Info](#-user-info)
- [User Files (Reference)](#-user-files-reference)
- [Creating Users](#-creating-users)
- [Setting Passwords](#-setting-passwords)
- [Modifying Users](#-modifying-users)
- [Deleting Users](#-deleting-users)
- [Group Management](#-group-management)
- [Switching Users / Privilege](#-switching-users--privilege)
- [Permissions Quick Reference](#-permissions-quick-reference)
- [Useful One-Liners](#-useful-one-liners)

---

## 👤 User Info

| Command | Description |
|---|---|
| `whoami` | Current user |
| `id` | UID, GID, groups |
| `id username` | Info for a specific user |
| `who` | Logged-in users |
| `w` | Logged-in users + activity |
| `last` | Login history |
| `finger username` | Detailed user info (if installed) |

---

## 📁 User Files (Reference)

| File | Purpose |
|---|---|
| `/etc/passwd` | User account info (`name:x:UID:GID:comment:home:shell`) |
| `/etc/shadow` | Encrypted passwords + expiry info |
| `/etc/group` | Group info |
| `/etc/gshadow` | Secure group info |
| `/etc/login.defs` | Default settings (UID/GID ranges, etc.) |

---

## ➕ Creating Users

```bash
useradd username                     # create user (minimal defaults)
useradd -m username                  # create with home directory
useradd -m -s /bin/bash username     # set shell
useradd -m -d /custom/home username  # custom home dir
useradd -m -g groupname username     # set primary group
useradd -m -G grp1,grp2 username     # set secondary/supplementary groups
useradd -u 1050 username             # set specific UID
adduser username                     # interactive (Debian/Ubuntu)
```

---

## 🔑 Setting Passwords

```
passwd username              # set/change password
passwd -e username           # force change at next login
passwd -l username           # lock account
passwd -u username           # unlock account
passwd -S username           # check password status
```
---

## View Aging Info

```
chage -l username            # view all aging info (last change, expiry, min/max, warn)
chage username               # interactive mode (prompts for each value)
```

## Set Aging Policy (chage)

```
chage -m 7 username          # min days between password changes
chage -M 90 username         # max days before password must be changed
chage -W 14 username         # warn user 14 days before expiry
chage -I 30 username         # inactive days after expiry before account locks
chage -E 2026-12-31 username # account expiry date (YYYY-MM-DD)
chage -E -1 username         # remove account expiry (never expires)
chage -d 0 username          # force password change at next login
chage -d 2026-10-24 username # set "last password change" date manually
chage -M 99999 username      # effectively disable password expiry
```

## All in One Command
```
chage -m 7 -M 90 -W 14 -I 30 username
```

## Same Thing Using passwd
```
passwd -n 7 username         # min days
passwd -x 90 username        # max days
passwd -w 14 username        # warning days
passwd -i 30 username        # inactive days
```
---

## ✏️ Modifying Users

```bash
usermod -l newname oldname       # rename user
usermod -d /new/home -m username # change home dir (move contents)
usermod -s /bin/zsh username     # change shell
usermod -aG groupname username   # ADD to supplementary group
usermod -g groupname username    # change primary group
usermod -L username              # lock account
usermod -U username              # unlock account
usermod -e YYYY-MM-DD username   # set account expiry
```

> ⚠️ **Warning:** Never use `usermod -G` without `-a`. Without `-a`, it **replaces** all supplementary groups instead of adding to them.

---

## 🗑️ Deleting Users

```bash
userdel username           # delete user, keep home dir
userdel -r username        # delete user + home dir + mail spool
userdel -f username        # force delete (even if logged in)
```

---

## 👥 Group Management

```bash
groupadd groupname             # create group
groupadd -g 1050 groupname     # create with specific GID
groupmod -n newname oldname    # rename group
groupdel groupname             # delete group
groups username                 # list groups a user belongs to
getent group groupname          # show group members
gpasswd -a username groupname   # add user to group
gpasswd -d username groupname   # remove user from group
```

---

## 🔄 Switching Users / Privilege

```bash
su username                # switch user (partial env)
su - username               # switch user with full login environment
sudo command                # run command as root (per sudoers rules)
sudo -u username command    # run command as specific user
visudo                      # safely edit /etc/sudoers
```

---

## 🔐 Permissions Quick Reference

```bash
chown user:group file       # change owner + group
chown -R user:group dir/    # recursive
chgrp groupname file        # change group only
chmod 755 file               # rwxr-xr-x
chmod u+x file                # add execute for owner
```

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

## ⚡ Useful One-Liners

```bash
cut -d: -f1 /etc/passwd              # list all usernames
awk -F: '{print $1}' /etc/group      # list all group names
getent passwd username               # check if user exists
lslogins                              # detailed overview of all users (util-linux)
```

---

*💡 Tip: Star or fork this gist for quick access during sysadmin work.*
