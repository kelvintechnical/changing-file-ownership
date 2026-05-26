# Lab: Changing File Ownership — `chown` and `chgrp`

- **Series:** linux-ops-mastery — RHCSA Permissions, Special Bits & ACLs
- **Subjects covered:** POSIX owner and group fields, `chown user:group`, `chown user.` / `chown :group`, recursive `-R`, `chgrp` as a focused group changer, impact on quota and backup ACLs, verifying with `ls -l` and `stat`
- **Career arcs covered:** RHCSA (EX200 ownership normalization tasks), RHCE (`ansible.builtin.file` `owner`/`group`), SRE (service account migrations), DevOps (fixing UID drift in bind mounts), AI/MLOps (shared cache directories across training users)
- **Prerequisite:** Lab 40 (Standard File Permissions) — you can read `ls -l` triplets
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 inventory · 2 user-only `chown` · 3 group with `chgrp` and `chown :grp` · 4 combined `user:group` · 5 recursive tree · 6 capstone + cleanup

---

## Objective

Own the **ownership columns** of `ls -l` — the user and group names (or numeric IDs) that decide *which* `rwx` triplet applies to a process. By the end of this lab you can transfer files between users safely, set only the group without touching the user, and apply ownership across a directory tree the way an RHCSA task expects.

The capstone is an exam-style brief: *"Under `/tmp/own-lab`, ensure all files in `project/` are owned by `devuser:projectgrp`, with the project directory itself matching the same ownership."*

> **Lab safety note:** Practice users and groups live under `/tmp` with disposable names. If your environment forbids `useradd`, adapt Task 6 to two pre-existing accounts shown by `getent passwd`.

---

## Concept: Ownership Selects Which Permission Triplet Runs

Permissions (`chmod`) answer **what** can happen. Ownership (`chown` / `chgrp`) answers **as whom** the kernel evaluates those bits for an unprivileged process.

```
   Process tries to open /tmp/own-lab/report.txt
                 │
                 ▼
        Is my euid the file owner?
           /        \
         yes        no
          │          └── Is my egid (or supplementary group) the file group?
          │                    /              \
          │                  yes               no
          ▼                  ▼                 ▼
      apply u bits     apply g bits      apply o bits

   chown alice:finance report.txt
      └── owner alice sees user triplet
      └── any user in group finance sees group triplet first (unless user match)
```

> **Why this matters:** A file mode of `640` is useless if the owning group is still `root` while collaborators live in `wheel`. Ownership and mode are a **pair**. RHCSA tasks often give you both requirements in one paragraph.

---

## 📜 Why `chown` Exists — The Story

Research Unix kept metadata per inode: who created the file, which UNIX "group" should share it, and how everyone else is treated. `chown` entered the toolkit so administrators could **re-home** files after migrations, user renames, or service account changes without copying bytes around the disk.

Historically, only root could change ownership — giving arbitrary users `chown` would be a trivial quota and accounting bypass. Linux preserves that rule for user changes. Group-only adjustments are slightly more flexible on some historical systems, but **modern practice** is still "use `chgrp` or `chown :group` as root unless policy adds capabilities."

Today, containers, NFS user ID mapping, and configuration management all assume you can set owner/group declaratively (`chown` in shell, `owner` in Ansible, `runAsUser` in Kubernetes). The mental model stayed tiny: **two fields**, both visible in `ls -l`.

> **The point of the story:** Every permission bug is either wrong bits, wrong owner, wrong group, or an LSM/ACL layered on top. This lab removes the middle two as unknowns.

---

## 👪 The Ownership Family — Who Lives There

### Primary commands

| Command | Typical use |
|---|---|
| `chown user file` | Set owner only (root) |
| `chown user:group file` | Set both atomically |
| `chown :group file` | Set group only (`chgrp` twin) |
| `chgrp group file` | Expressive group-only change |
| `chown -R user:group dir` | Tree-wide fix (watch symlinks) |

### Discovery helpers

| Command | Output |
|---|---|
| `id` | Current uid/gid + supplementary groups |
| `getent passwd user` | passwd entry |
| `getent group grp` | group entry |
| `ls -n` | Numeric uid/gid columns |

> **The point of the family tree:** If you can navigate `chown` and `getent`, you can answer "which triplet applies to nginx?" without guessing.

---

## 🔬 The Anatomy of `chown alice:developers file.txt` — In One Diagram

```
$ chown alice:developers file.txt
  │     │      │            │
  │     │      │            └─ target path (file or directory)
  │     │      └─ new owning group (name or numeric GID)
  │     └─ new owning user (name or numeric UID)
  └─ command

Kernel result on inode:
  st_uid  ← alice's UID
  st_gid  ← developers GID

ls -l afterwards:
-rw-r-----. 1 alice developers ... file.txt
```

> **Reading rule:** The colon separates **user** (left) from **group** (right). Either side may be omitted, but not both at once: `chown :grp` (group only) and `chown usr` (user only) are valid patterns.

---

## 📚 chown / chgrp Reference Table

| Task | Command | Notes |
|---|---|---|
| User only | `chown alice FILE` | Root required for arbitrary users |
| Group only | `chgrp devs FILE` | Same as `chown :devs FILE` |
| Both | `chown alice:devs FILE` | Preferred one-shot |
| Numeric | `chown 1001:1002 FILE` | Useful when NIS/LDAP names offline |
| Recursive | `chown -R alice:devs DIR` | Follows symlinks to targets by default |
| Preserve root | `chown -R --preserve-root ...` | Safer guard when scripting `/` (never practice on `/`) |
| Verify | `stat -c '%U %G %n' FILE` | Human names |
| Verify numeric | `stat -c '%u %g %n' FILE` | Raw ids |

> **Rule one of recursive chown:** Type the path twice in your head before pressing Enter. Typos become irreversible tree walks.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Tasks often say "correct ownership" alongside mode. Missing `chown` loses points even when `chmod` is perfect. |
| **RHCE candidate** | `ansible.builtin.file` splits `owner`, `group`, and `mode` — same three fields as `ls -l`. |
| **SRE / Platform** | Service cutovers change the runtime user — rotate logs and configs with `chown -R` in maintenance windows. |
| **DevOps** | Volume mounts show host numeric IDs inside containers; fix with `chown` on the host or fsGroup patterns. |
| **AI / MLOps** | Shared HF cache dirs (`~/.cache/huggingface`) break when scheduler users disagree on uid/gid — normalize ownership. |

---

## 🔧 The 6 Tasks

> Six phases building **inventory → single file → group tricks → combined → recursive → capstone**.

---

### Task 1 — Sandbox users and baseline `ls -l`

**Purpose:** Create disposable users/groups if policy allows, then create files to inspect default ownership.

```bash
sudo -i
mkdir -p /tmp/own-lab && cd /tmp/own-lab

useradd -m devuser 2>/dev/null || true
groupadd -f projectgrp 2>/dev/null || true

echo "seed" > notes.txt
ls -l notes.txt
stat -c '%u %g %U %G' notes.txt
```

**Human-Readable Breakdown:** Become root, create lab directory, ensure `devuser` and `projectgrp` exist (ignore errors if already present), create a file, print classic and numeric ownership.

**Reading it left to right:** New files inherit creator **uid** and primary **gid** (often `root:root` here). `stat` prints numeric and resolved names.

**The story:** You need two distinct identities to see ownership changes matter. If `useradd` is blocked, substitute two real accounts from `getent passwd`.

**Expected output:**

```text
-rw-r--r--. 1 root root 5 May 26 11:00 notes.txt
0 0 root root
```

**Switches**

| Token | Meaning |
|---|---|
| `useradd -m` | Create home directory |
| `groupadd -f` | Idempotent group create |
| `stat -c` | Custom format output |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `useradd: command not found` | Install `shadow-utils` or use existing users |
| IDs differ | Expected across systems — track **names** for the lab narrative |

---

### Task 2 — Core operation A: change user owner

**Purpose:** Transfer user ownership from `root` to `devuser` while leaving group temporarily unchanged.

```bash
cd /tmp/own-lab

chown devuser notes.txt
ls -l notes.txt
```

**Human-Readable Breakdown:** `chown USER` rebinds the inode's uid; group stays until you change it.

**Reading it left to right:** Only root may set arbitrary owners (CAP_CHOWN in namespaces follows policy).

**The story:** Package extracts often create `root:root` trees; application tasks want `apache` or `nginx` — first step is almost always `chown user`.

**Expected output:**

```text
-rw-r--r--. 1 devuser root 5 May 26 11:02 notes.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chown USER FILE` | Owner swap |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Operation not permitted` | You are not root |
| Owner shows numeric | Name service missing — still valid |

---

### Task 3 — Core operation B: change group with `chgrp` and `chown :group`

**Purpose:** Demonstrate two equivalent ways to set only the group.

```bash
cd /tmp/own-lab

chgrp projectgrp notes.txt
ls -l notes.txt

chown :root notes.txt
ls -l notes.txt
```

**Human-Readable Breakdown:** First move group to `projectgrp`, then demonstrate `chown :root` returning group to `root` without altering the now-`devuser` owner.

**Reading it left to right:** `chgrp` is explicit; leading colon form is POSIX `chown` syntax for "group half only."

**The story:** Scripts from Solaris heritage often use `chown :grp`; Linux admins frequently type `chgrp`. Know both — you will read both in the wild.

**Expected output:**

```text
-rw-r--r--. 1 devuser projectgrp 5 May 26 11:04 notes.txt
-rw-r--r--. 1 devuser root 5 May 26 11:04 notes.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chgrp G FILE` | Group-only change |
| `chown :G FILE` | Group-only change via chown |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Invalid argument` | Group does not exist — `groupadd` first |
| Process still cannot access | User triplet may deny — adjust `chmod` (Lab 40) |

---

### Task 4 — Verification: combined `user:group` in one call

**Purpose:** Set both fields atomically — the RHCSA-preferred pattern when a prompt specifies user **and** group.

```bash
cd /tmp/own-lab

install -m 644 /dev/null shared.conf
echo app=yes > shared.conf

chown devuser:projectgrp shared.conf
stat -c '%U:%G %a %n' shared.conf
ls -l shared.conf
```

**Human-Readable Breakdown:** Create a fresh file, assign `devuser:projectgrp`, print concise proof.

**Reading it left to right:** Colon joins user and group; order is always **user then group**.

**The story:** Two separate commands risk a half-updated inode if a script fails midway — combined `chown` is atomic at the syscall layer.

**Expected output:**

```text
devuser:projectgrp 644 shared.conf
-rw-r--r--. 1 devuser projectgrp 8 May 26 11:06 shared.conf
```

**Switches**

| Token | Meaning |
|---|---|
| `install -m 644` | Create file with explicit mode |
| `chown U:G FILE` | Set owner and group together |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cannot access ...` | Wrong path — `pwd` and retry |
| Group wrong | Typo after colon — re-run `chown U:G` |

---

### Task 5 — Recursive ownership on a small tree

**Purpose:** Apply `chown -R` to a directory tree containing nested files.

```bash
cd /tmp/own-lab

mkdir -p project/{src,bin}
echo main > project/src/main.txt
echo run > project/bin/run.sh
chmod 755 project/bin/run.sh

chown -R devuser:projectgrp project
find project -printf '%u:%g %m %p\n' | sort
```

**Human-Readable Breakdown:** Populate a tree, recursively set owner/group, list with `find` showing user:group and mode.

**Reading it left to right:** `-R` walks depth-first; every inode under `project/` receives the same ownership tuple unless a symlink is followed to its target (be aware when practicing on `/srv`).

**The story:** This is the command behind "fix the entire unpacked tarball." Pair with `chmod -R` only when you understand directory execute bits.

**Expected output:**

```text
devuser:projectgrp 755 project/bin
devuser:projectgrp 755 project/bin/run.sh
devuser:projectgrp 775 project
devuser:projectgrp 775 project/src
devuser:projectgrp 644 project/src/main.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chown -R U:G DIR` | Recursive ownership |
| `find ... -printf` | Custom per-file summary |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Some files remain `root` | You ran command outside `project/` parent — rerun with correct path |
| Broken symlinks | Expected — target may live outside tree |

---

### Task 6 — Capstone + cleanup

**Task statement:** *"Ensure `/tmp/own-lab/project` and everything beneath it is owned by `devuser:projectgrp`. Create `/tmp/own-lab/CAPSTONE.txt` stating DONE, owned by `devuser:projectgrp`, mode `640`. Verify with `ls -l` and `stat`."*

**Purpose:** Mirror a two-sentence exam prompt and return the system to a clean state.

```bash
sudo -i
cd /tmp/own-lab

chown -R devuser:projectgrp project
echo DONE > CAPSTONE.txt
chown devuser:projectgrp CAPSTONE.txt
chmod 640 CAPSTONE.txt

ls -l CAPSTONE.txt
stat -c '%U:%G %a' CAPSTONE.txt project project/src/main.txt
```

**Cleanup**

```bash
sudo -i
rm -rf /tmp/own-lab
userdel -r devuser 2>/dev/null || true
groupdel projectgrp 2>/dev/null || true
exit
```

**Expected output:**

```text
-rw-r-----. 1 devuser projectgrp 5 May 26 11:15 CAPSTONE.txt
devuser:projectgrp 640
devuser:projectgrp 644
```

**Switches**

| Token | Meaning |
|---|---|
| `userdel -r` | Remove user and home |
| `groupdel` | Remove group |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `userdel: user is logged in` | Close sessions or use `userdel -f` per policy |
| Cannot delete users | Skip cleanup user removal in locked-down labs |

---

## 🔍 Ownership Decision Guide

```
Need to fix who owns a file?
  │
  ├── "Only the user is wrong"
  │       └── chown newuser path
  │
  ├── "Only the group is wrong"
  │       └── chgrp newgroup path   OR   chown :newgroup path
  │
  ├── "Both are wrong"
  │       └── chown newuser:newgroup path
  │
  ├── "Entire tree unpacked from tarball"
  │       └── chown -R user:group topdir
  │
  └── "Process still denied after chown"
          └── chmod / ACL / SELinux — return to Labs 40 & 44+
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Create lab users/groups and baseline file ownership
- [ ] 02 Move user owner with `chown user`
- [ ] 03 Adjust group using `chgrp` and `chown :group`
- [ ] 04 Apply combined `user:group` on a config file
- [ ] 05 Recursively fix `project/` tree ownership
- [ ] 06 Write `CAPSTONE.txt`, verify, and cleanup `/tmp/own-lab`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Reversed `user:group` | Group column shows username | Swap sides of colon |
| Recursive on wrong path | Mass ownership corruption | Restore from snapshot; practice with `/tmp` only |
| Expect non-root `chown` across users | EPERM | Use `sudo` |
| Fixed owner but not mode | Still cannot execute | `chmod u+x` |
| NFS squash | Ownership snaps back | Fix export map or run on server |
| Forgot supplementary groups | `chgrp` ok but user lacks group membership | `usermod -aG grp user` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the three proofs: `ls -l`, `stat -c '%U %G'`, and `namei -l` for path component ownership.

**RHCE candidate**
- Map `owner`/`group` in tasks to `chown` invocations — handlers often re-run `chown` after unarchive.

**SRE / Platform interview**
- Explain **why** recursive `chown` on a symlinked log directory might traverse into `/var/log` — demonstrate caution.

**DevOps**
- Document numeric UID/GID requirements beside Helm charts to avoid silent permission drift.

**AI / MLOps**
- When schedulers run jobs as different service accounts, centralize artifact ownership with `chown -R` at workflow end.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 40 — Standard File Permissions | `chmod` after `chown` completes the story |
| Lab 42 — SUID Executables | Owner field defines *whose* effective UID SUID uses |
| Lab 43 — SGID and Sticky Bit | Group ownership inheritance on directories |
| Lab 02 — stderr redirection | Silence noisy `find` while auditing ownership |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
