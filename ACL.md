# 🔐 Linux ACL (Access Control Lists) Cheat Sheet

A daily-use reference for managing fine-grained file permissions with ACL in Linux.

---

## 📋 Table of Contents
- [Why ACL?](#-why-acl)
- [Check ACL Support](#-check-acl-support)
- [Install ACL Tools](#-install-acl-tools)
- [View ACLs](#-view-acls)
- [Set ACLs](#-set-acls)
- [Remove ACLs](#-remove-acls)
- [Understanding the Mask](#-understanding-the-mask)
- [Copy ACLs Between Files](#-copy-acls-between-files)
- [Identifying ACL on a File](#-identifying-acl-on-a-file)
- [Quick Reference](#-quick-reference)

---

## 🤔 Why ACL?

Standard Linux permissions (`rwx` for owner/group/others) allow only **one owner + one group**. ACL lets you set **different permissions for multiple users/groups** on the same file or directory — without changing ownership.

**Example:** You own a file, but want `user2` to have read-write and `user3` to have read-only, without altering group ownership.

---

## ✅ Check ACL Support

```bash
mount | grep acl
tune2fs -l /dev/sdXN | grep "Default mount options"   # ext4 check
```
Most modern distros (Ubuntu, RHEL, etc.) support ACL by default.

---

## 📦 Install ACL Tools

```bash
sudo apt install acl       # Debian/Ubuntu
sudo yum install acl       # RHEL/CentOS
```

---

## 👀 View ACLs

```bash
getfacl filename
getfacl -R directory/      # recursive
```

**Sample output:**
```
# file: test.txt
# owner: himanshu
# group: himanshu
user::rw-
user:deploy:r--
group::r--
group:devs:rw-
mask::rw-
other::r--
```

---

## ✏️ Set ACLs

**User-specific permission:**
```bash
setfacl -m u:username:rwx filename
```

**Group-specific permission:**
```bash
setfacl -m g:groupname:rx filename
```

**Default ACL on a directory** (inherited by new files/dirs created inside):
```bash
setfacl -d -m u:username:rwx directory/
```

**Apply recursively:**
```bash
setfacl -R -m u:username:rwx directory/
```

**Multiple entries in one command:**
```bash
setfacl -m u:user1:rwx,g:group1:rx filename
```

---

## 🗑️ Remove ACLs

```bash
setfacl -x u:username filename     # remove specific user entry
setfacl -x g:groupname filename    # remove specific group entry
setfacl -b filename                # remove ALL ACL entries
setfacl -k directory/              # remove default ACL from directory
```

---

## 🎭 Understanding the Mask

- `mask` defines the **maximum effective permission** for all named users/groups (not owner, not others).
- Even if a user is granted `rwx`, if `mask` is `r--`, effective permission is only `r--`.

```bash
setfacl -m m::rx filename    # set mask manually
```

---

## 📋 Copy ACLs Between Files

```bash
getfacl file1 | setfacl --set-file=- file2
```

---

## 🔍 Identifying ACL on a File

```bash
ls -l filename
```
A `+` after permission bits indicates extended ACL entries:
```
-rw-rwxr--+ 1 himanshu himanshu 0 Oct  3 12:00 file.txt
```

---

## ⚡ Quick Reference

| Command | Purpose |
|---|---|
| `getfacl file` | View ACL entries |
| `setfacl -m u:user:perm file` | Add/modify user ACL |
| `setfacl -m g:group:perm file` | Add/modify group ACL |
| `setfacl -x u:user file` | Remove user ACL entry |
| `setfacl -b file` | Remove all ACL entries |
| `setfacl -R -m ... dir/` | Apply recursively |
| `setfacl -d -m ... dir/` | Set default (inherited) ACL |

---

*💡 Tip: Keep this alongside your user & permission management README for a complete Linux access-control reference.*
