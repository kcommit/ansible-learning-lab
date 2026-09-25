# Lesson 02: Linux Resource and Filesystem Troubleshooting

> Repository-ready study notes based on the practical WSL lab.

## Table of Contents

1. [Lesson goals](#1-lesson-goals)
2. [Troubleshooting method](#2-troubleshooting-method)
3. [`df -hT` — filesystem capacity](#3-df--ht--filesystem-capacity)
4. [Understanding the columns](#4-understanding-the-columns)
5. [Analysis of the lab output](#5-analysis-of-the-lab-output)
6. [Why Snap filesystems show 100%](#6-why-snap-filesystems-show-100)
7. [`df` versus `du`](#7-df-versus-du)
8. [Filesystem investigation workflow](#8-filesystem-investigation-workflow)
9. [Important commands](#9-important-commands)
10. [Common mistakes](#10-common-mistakes)
11. [Practice exercises](#11-practice-exercises)
12. [Interview questions](#12-interview-questions)
13. [Lab evidence template](#13-lab-evidence-template)
14. [Quick-reference summary](#14-quick-reference-summary)
15. [Personal lab investigation](#15-personal-lab-investigation)
16. [Docker storage investigation](#16-docker-storage-investigation)
17. [Deleted-but-open file experiment](#17-deleted-but-open-file-experiment)
18. [Memory and swap investigation](#18-memory-and-swap-investigation)
19. [Load average and CPU capacity](#19-load-average-and-cpu-capacity)
20. [Mistakes and corrections](#20-mistakes-and-corrections)
21. [Real-life troubleshooting scenarios](#21-real-life-troubleshooting-scenarios)
22. [Final lesson conclusions](#22-final-lesson-conclusions)

---

## 1. Lesson goals

By the end of this lesson, I should be able to:

- Check filesystem capacity and type.
- Identify which mounted filesystem is actually becoming full.
- Explain every important column produced by `df`.
- Distinguish real disk-full conditions from expected read-only mounts.
- Use `du` to investigate which directories consume space.
- Check inode consumption when free space appears available.
- Investigate deleted files that are still held open by a process.
- Collect evidence before deleting or changing anything.

---

## 2. Troubleshooting method

Use this sequence during every investigation:

1. **Observe** — What is the reported symptom?
2. **Form a hypothesis** — What might explain it? [Hypothesis is a conditional scientific statement based on logic and facts]
3. **Collect evidence** — Run safe, read-only commands first.
4. **Interpret** — Compare normal and abnormal values.
5. **Narrow the scope** — Identify the filesystem, directory or process.
6. **Act safely** — Make the smallest justified change.
7. **Validate** — Confirm that the symptom is resolved.
8. **Document** — Record commands, output, conclusion and prevention.

### Why do we work this way?

Guessing can cause an administrator to delete the wrong data or hide the real cause. Evidence-first troubleshooting makes the investigation repeatable, safer and easier to explain to another engineer.

---

## 3. `df -hT` — filesystem capacity

### Command used

```bash
df -hT
```

### Meaning

- `df` means **disk free** and reports usage from the filesystem's perspective.
- `-h` displays human-readable units such as MiB and GiB.
- `-T` displays the filesystem type.

To inspect only the filesystem containing the root directory:

```bash
df -hT /
```

This focused command is often better during troubleshooting because it avoids unrelated WSL, Snap and temporary mounts.

### Initial hypothesis

> The root filesystem has enough available capacity and is not close to full.

---

## 4. Understanding the columns

| Column | Meaning | Question it answers |
|---|---|---|
| `Filesystem` | Device or virtual filesystem | Which filesystem supplies the storage? |
| `Type` | Filesystem format or mount technology | Is it `ext4`, `tmpfs`, `9p`, overlay or something else? |
| `Size` | Total capacity | How large is it? |
| `Used` | Consumed capacity | How much space is occupied? |
| `Avail` | Space available for normal use | How much can still be used? |
| `Use%` | Percentage of capacity consumed | Is it approaching a warning threshold? |
| `Mounted on` | Directory where the filesystem is attached | Which path is affected? |

Important: always connect `Use%` with `Mounted on`. A full filesystem affects the paths mounted from that filesystem, not automatically every filesystem on the server.

---

## 5. Analysis of the lab output

The important root-filesystem line was:

```text
/dev/sdd  ext4  1007G  33G  923G  4%  /
```

### Interpretation

| Item | Lab value | Interpretation |
|---|---:|---|
| Device | `/dev/sdd` | Storage device backing the WSL Linux filesystem |
| Type | `ext4` | Standard Linux filesystem |
| Total size | `1007G` | Approximately 1 TB total capacity |
| Used | `33G` | A small portion is occupied |
| Available | `923G` | Plenty of capacity remains |
| Usage | `4%` | Healthy; root is not close to full |
| Mount point | `/` | This is the Linux root filesystem |

### Conclusion

The hypothesis is supported. The root filesystem is healthy and no cleanup is required.

The Windows `C:` drive appears through WSL as a `9p` mount:

```text
C:\  9p  952G  301G  652G  32%  /mnt/c
```

Its 32% usage is also healthy. The `9p` type indicates the Windows filesystem is being exposed to the WSL environment rather than stored as native Linux `ext4` data.

Temporary filesystems such as `tmpfs` use memory-backed storage for runtime data. Their displayed size does not mean that amount of RAM is permanently occupied; usage grows only as data is written.

---

## 6. Why Snap filesystems show 100%

The output contained entries similar to:

```text
snapfuse  fuse.snapfuse  62M  62M  0  100%  /snap/core24/1644
```

Snap packages are distributed as compressed, read-only filesystem images. Each image has a fixed size and is mounted read-only, so `df` commonly reports it as 100% used.

This is normally **expected behavior**, not proof that the writable root filesystem is full.

### Correct decision

- Do not delete files from a mounted Snap image.
- Check `/` or another affected writable mount separately.
- Investigate Snap revisions only if package maintenance or old-revision cleanup is actually required.

### General lesson

Never react to `100%` alone. First identify the filesystem type, mount point and whether the filesystem is writable.

---

## 7. `df` versus `du`

| Command | Viewpoint | Best use |
|---|---|---|
| `df` | Filesystem allocation | Identify which mounted filesystem is full |
| `du` | Visible files and directories | Identify where space is being consumed |

Example:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

- `-x` stays on the same filesystem.
- `-h` uses readable units.
- `-d1` limits the report to one directory level.
- `2>/dev/null` hides permission-denied messages during this learning exercise.
- `sort -h` sorts human-readable sizes.

Why start with `df`? It identifies the affected filesystem. After that, `du` helps narrow the investigation to a directory. Running an unrestricted search everywhere creates noise and may cross unrelated mounts.

---

## 8. Filesystem investigation workflow

### Step 1: Identify the affected filesystem

```bash
df -hT
```

### Step 2: Check inode usage

```bash
df -ih
```

A filesystem can fail to create new files when its inodes reach 100%, even if capacity remains. This commonly happens when millions of small files exist.

### Step 3: Find large top-level directories

For the root filesystem:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Then repeat against the largest directory, for example:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

### Step 4: Find unusually large files

```bash
sudo find /var -xdev -type f -size +500M -printf '%s %p\n' 2>/dev/null | sort -n
```

### Step 5: Look for deleted but still-open files

```bash
sudo lsof +L1
```

A process may keep disk blocks allocated after a file has been deleted. In that case, `df` still counts the space while `du` cannot see the deleted pathname.

### Step 6: Validate after a justified fix

```bash
df -hT /
df -ih /
```

Never remove unfamiliar files simply because they are large. First identify their owner, purpose, retention requirement and whether a service is actively using them.

---

## 9. Important commands

### Filesystem capacity

```bash
df -hT
df -hT /
```

### Inode capacity

```bash
df -ih
df -ih /
```

### Directory consumption

```bash
du -sh PATH
sudo du -xhd1 PATH 2>/dev/null | sort -h
```

### Memory and swap

```bash
free -h
```

Pay special attention to the `available` column. Linux intentionally uses otherwise-idle RAM for cache, so a low `free` value alone does not prove memory pressure.

### Load and uptime

```bash
uptime
nproc
```

Load average should be interpreted in relation to the number of logical CPUs and sustained behavior, not from one isolated number.

### Largest visible files

```bash
sudo find PATH -xdev -type f -size +500M -printf '%s %p\n' 2>/dev/null | sort -n
```

### Deleted-open files

```bash
sudo lsof +L1
```

---

## 10. Common mistakes

1. Confusing `df` and `du`.
2. Treating every `100%` entry as an emergency.
3. Ignoring the `Mounted on` column.
4. Deleting large files before learning what created them.
5. Checking space but forgetting inode exhaustion.
6. Assuming low `free` memory automatically means a RAM shortage.
7. Running broad searches across `/mnt`, network mounts and virtual filesystems.
8. Making changes before collecting evidence.

---

## 11. Practice exercises

### Exercise 1: Focused filesystem check

```bash
df -hT /
```

Record:

- Device:
- Filesystem type:
- Total size:
- Used:
- Available:
- Use percentage:
- Conclusion:

### Exercise 2: Inode check

```bash
df -ih /
```

Answer: Can new files be created safely, based on inode availability?

### Exercise 3: Compare `df` and `du`

```bash
df -hT /
sudo du -xhd1 / 2>/dev/null | sort -h
```

Explain why the totals may not match exactly.

### Exercise 4: Resource baseline

```bash
free -h
uptime
nproc
```

Write one observation and one conclusion for each command.

---

## 12. Interview questions

### Why might `df` report more usage than `du`?

A deleted file may still be open by a running process. `df` counts its allocated blocks, while `du` cannot see the deleted pathname. Filesystem metadata, reserved blocks and mount boundaries may also contribute to differences.

### Can a filesystem fail when it still has free space?

Yes. It may have exhausted its inodes, be mounted read-only, have quota restrictions or encounter another filesystem error.

### What should you check first when a server reports “No space left on device”?

Check both block usage and inode usage on the affected path:

```bash
df -hT AFFECTED_PATH
df -ih AFFECTED_PATH
```

### Is 100% usage always a problem?

No. Read-only fixed-size images such as Snap mounts often display 100% normally. Interpret filesystem type, mount point and writability before deciding.

### Why use `du -x`?

It prevents the scan from crossing onto other mounted filesystems, which keeps the results relevant to the filesystem under investigation.

---

## 13. Lab evidence template

```markdown
## Investigation title

### Symptom

### Hypothesis

### Commands used

### Important output

### Interpretation

### Root cause

### Corrective action

### Validation

### Prevention
```

Suggested evidence filename:

```text
evidence/lesson-02-filesystem-baseline.txt
```

Example capture command:

```bash
{
  echo '===== FILESYSTEM CAPACITY ====='
  df -hT
  echo
  echo '===== INODE CAPACITY ====='
  df -ih
  echo
  echo '===== MEMORY ====='
  free -h
  echo
  echo '===== LOAD ====='
  uptime
  nproc
} | tee evidence/lesson-02-filesystem-baseline.txt
```

Review the output before committing it. Evidence files should never contain passwords, access keys, tokens or other secrets.

---

## 14. Quick-reference summary

| Goal | Command |
|---|---|
| Check all mounted filesystem capacity | `df -hT` |
| Check the filesystem containing `/` | `df -hT /` |
| Check inode usage | `df -ih` |
| Size of one path | `du -sh PATH` |
| Compare top-level directory sizes | `sudo du -xhd1 PATH 2>/dev/null \| sort -h` |
| Check memory and swap | `free -h` |
| Check load averages | `uptime` |
| Count logical CPUs | `nproc` |
| Find deleted-open files | `sudo lsof +L1` |

### Current lab result

The WSL root filesystem `/dev/sdd`, mounted on `/`, is healthy at **4% usage**. No disk cleanup is currently necessary.

---

## 15. Personal lab investigation

This section records the actual investigation performed during Lesson 2.

### 15.1 Filesystem capacity

Command:

```bash
df -hT
```

Important root-filesystem result:

```text
Filesystem  Type  Size   Used  Avail  Use%  Mounted on
/dev/sdd    ext4  1007G  33G   923G   4%    /
```

Interpretation:

- `/dev/sdd` is the storage device that backs the Linux filesystem.
- `ext4` is the filesystem type.
- `/` is the root directory and mount point.
- `33G` is used disk capacity.
- `923G` is available to normal users and applications.
- `4%` is disk usage, not RAM usage.

The values do not add up exactly because `ext4` may reserve blocks for the root user and recovery, the filesystem has internal metadata, and `-h` rounds values.

### 15.2 Storage device versus mount point

```text
Storage device  →  Filesystem  →  Mount point
/dev/sdd        →  ext4        →  /
```

Analogy:

- `/dev/sdd` is the storage cabinet.
- `ext4` is the organization system inside the cabinet.
- `/` is the door through which Linux accesses it.

### 15.3 Inode capacity

Command:

```bash
df -ih /
```

Actual result:

```text
Filesystem  Inodes  IUsed  IFree  IUse%  Mounted on
/dev/sdd       64M   748K    64M     2%  /
```

Conclusion:

> Only 2% of the root filesystem's inodes were used. Inode exhaustion was not a problem.

An inode stores metadata such as file type, owner, permissions, timestamps, size, and pointers to data blocks. Directory entries connect filenames to inode numbers.

A filesystem can return `No space left on device` in at least two major situations:

1. Data blocks are exhausted — investigate with `df -h`.
2. Inodes are exhausted — investigate with `df -i`.

### 15.4 Narrowing directory usage

Initial command:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Important results:

```text
3.2G  /usr
15G   /home
16G   /var
33G   /
```

Interpretation:

- `/` is the total visible usage for the entire root tree.
- `/var` is the largest top-level child directory.
- Do not add the `/` total to its child-directory results.
- `/mnt` appeared very small because `-x` prevented `du` from crossing into the separate Windows `9p` mount at `/mnt/c`.

Next command:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Important results:

```text
195M  /var/cache
373M  /var/log
16G   /var/lib
16G   /var
```

Conclusion:

> `/var/lib`, not `/var/log`, accounted for almost all `/var` usage. Logs were not the main storage consumer.

Next command:

```bash
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
```

Important results:

```text
186M  /var/lib/snapd
208M  /var/lib/apt
6.1G  /var/lib/docker
8.7G  /var/lib/containerd
16G   /var/lib
```

Conclusion:

> Container-related data accounted for almost all `/var/lib` usage. `/var/lib/containerd` was larger than `/var/lib/docker`. This identified the largest consumer, but it did not identify a disk-full failure because `/` was only 4% used.

### Command-option correction

```text
-d1 = show only one directory level
-x  = remain on the same filesystem
-h  = use human-readable units
```

---

## 16. Docker storage investigation

Command:

```bash
docker system df
```

Actual result:

```text
TYPE            TOTAL  ACTIVE  SIZE      RECLAIMABLE
Images          15     7       7.529GB   2.938GB (39%)
Containers      9      0       6.283MB   6.283MB (100%)
Local Volumes   3      2       5.941GB   205.5MB (3%)
Build Cache     115    0       3.494GB   783MB
```

### Interpretation

- Images were the largest Docker-reported category at `7.529GB`.
- Seven images were referenced by containers.
- Nine stopped containers used only about `6.283MB`; removing them would recover insignificant space.
- Local volumes used `5.941GB` and could contain important persistent application or database data.
- Build cache speeds future builds and should not be removed without a reason.
- `RECLAIMABLE` does not mean `safe to delete without investigation`.

### Why `docker system df` and `du` may differ

Docker understands images, shared layers, writable container layers, volumes, and build cache. `du` measures physical blocks reachable through directory paths. Shared layers and different accounting methods mean their totals should not be added or compared as exact equivalents.

### Safe decision

The root filesystem was only 4% used, so there was no operational reason to prune Docker data.

Do not run cleanup merely because reclaimable storage exists. First confirm:

1. Actual filesystem pressure
2. Resource ownership and purpose
3. Application dependencies
4. Backup and recovery requirements
5. Change approval, if required

Plain `docker system prune` is destructive. It can remove stopped containers, unused networks, unused images, and build cache. Volumes require special attention because they may contain persistent data.

---

## 17. Deleted-but-open file experiment

### 17.1 Purpose

This controlled experiment demonstrated why `df` can report used space that `du` cannot find.

### 17.2 Baseline

Initial command:

```bash
sudo lsof +L1
```

Initial result: no output. No deleted-but-open files were detected at that moment.

### 17.3 Create a controlled 50 MiB file

```bash
dd if=/dev/zero \
  of=projects/linux-resource-lab/deleted-open-demo.log \
  bs=1M count=50 status=progress
```

Actual result:

```text
50+0 records in
50+0 records out
52428800 bytes (52 MB, 50 MiB) copied
```

Unit lesson:

- `50 MiB` uses binary units: 50 × 1,048,576 bytes.
- `52 MB` is the approximate decimal representation.

Verification:

```bash
ls -lh projects/linux-resource-lab/deleted-open-demo.log
```

```text
-rw-r--r-- 1 khalid khalid 50M ... deleted-open-demo.log
```

### 17.4 Open the file with a background process

```bash
tail -f projects/linux-resource-lab/deleted-open-demo.log >/dev/null &
```

Actual background process:

```text
[1] 2104
```

Verification:

```bash
lsof -p 2104 | grep 'deleted-open-demo.log'
```

```text
tail  2104 khalid  3r  REG  8,48  52428800  1154  .../deleted-open-demo.log
```

Field meanings:

- `tail` — process name
- `2104` — process ID
- `3r` — file descriptor 3, opened for reading
- `REG` — regular file
- `52428800` — file size in bytes
- `1154` — inode number

### 17.5 Delete the pathname while the process holds it open

```bash
rm -- projects/linux-resource-lab/deleted-open-demo.log
```

Path verification:

```bash
ls -lh projects/linux-resource-lab/deleted-open-demo.log
```

```text
ls: cannot access '...deleted-open-demo.log': No such file or directory
```

The pathname disappeared, but the process still held the inode and blocks open.

### 17.6 Detect the hidden allocation

```bash
sudo lsof +L1
```

Actual relevant result:

```text
COMMAND  PID   USER    FD  TYPE  SIZE/OFF  NLINK  NAME
tail     2104  khalid  3r  REG   52428800  0      .../deleted-open-demo.log (deleted)
```

Evidence:

- `NLINK=0` means no directory entry points to the inode.
- `(deleted)` confirms the pathname was removed.
- The open file descriptor prevented immediate block reclamation.
- `du` could no longer find the pathname.
- `df` continued counting the allocated blocks.

A temporary deleted journal entry was also observed for `systemd-journald`. It disappeared naturally later. A system service should not be killed without investigating impact.

### 17.7 Release the blocks safely

Because PID `2104` was a controlled lab process, it was terminated with normal `SIGTERM`:

```bash
kill 2104
```

Bash confirmed:

```text
[1]+  Terminated  tail -f ... > /dev/null
```

Why normal `kill` first?

- `kill PID` normally sends `SIGTERM`.
- It allows a process to perform a graceful shutdown.
- `kill -9` sends `SIGKILL` and should not be the first response.

### 17.8 Validate cleanup

```bash
sudo lsof +L1
```

Final result: empty output. The controlled deleted-open file was released, and the 50 MiB allocation was reclaimed.

### Exit-status timing lesson

```bash
echo $?
```

`$?` reports only the immediately preceding command's exit status. If another command is run between `kill` and `echo $?`, it no longer reports the status of `kill`.

The command:

```bash
echo $0
```

returned `-bash`. This identifies Bash; the leading hyphen commonly indicates a login shell.

---

## 18. Memory and swap investigation

Command:

```bash
free -h
```

Actual result:

```text
               total  used   free   shared  buff/cache  available
Mem:           7.5Gi  699Mi  6.1Gi  3.9Mi   948Mi       6.9Gi
Swap:          2.0Gi  0B     2.0Gi
```

### Interpretation

- Total RAM: `7.5Gi`
- Actively used: approximately `699Mi`
- Completely free: `6.1Gi`
- Buffers/cache: `948Mi`
- Estimated available for applications: `6.9Gi`
- Swap used: `0B`

The `available` column is generally more useful than `free` for estimating usable memory. Linux uses otherwise-idle RAM to cache files for faster access. The kernel can reclaim much of that cache when applications need memory, so `available` can be greater than `free`.

Conclusion:

> The system had approximately 6.9 GiB available RAM and no swap usage. There was no evidence of current memory pressure.

Important nuance: some swap usage alone does not prove a problem. Investigate active swapping, available RAM, application latency, memory growth, and OOM events.

---

## 19. Load average and CPU capacity

Commands:

```bash
uptime
nproc
```

Actual results:

```text
12:45:20 up 2:43, 1 user, load average: 0.01, 0.01, 0.00
```

```text
12
```

Interpretation:

- WSL uptime: 2 hours and 43 minutes
- Logged-in users: 1
- Load average: 1, 5, and 15 minutes
- Logical CPUs: 12
- A load of `0.01` is extremely low for 12 CPUs.

A sustained load near `12` means roughly one runnable or uninterruptible task per logical CPU. A sustained load above `12`, such as `15`, deserves investigation, but does not automatically prove CPU saturation.

Linux load includes:

- Runnable tasks competing for CPU
- Tasks in uninterruptible sleep, commonly waiting for I/O

Therefore, investigate high load with CPU utilization, I/O wait, process state, and workload context.

Trend guidance:

- 1-minute greater than 15-minute: load may be rising.
- 1-minute lower than 15-minute: load may be falling.
- Similar values: load is relatively stable.

---

## 20. Mistakes and corrections

Mistakes are part of the learning evidence. These corrections show how the technical model improved during the lesson.

| Initial understanding | Correct understanding | Why it matters |
|---|---|---|
| `/dev/sdd` is the root directory | `/dev/sdd` is the storage device; `/` is the root directory and mount point | Device and mount point are different layers |
| `4%` means memory used | `4%` means filesystem disk capacity used | `df` reports filesystems; `free` reports RAM |
| `Size=1007G` means available storage | `Size` is total capacity; `Avail=923G` is normally usable remaining space | Prevents incorrect capacity conclusions |
| `/` is the largest directory in `du -d1 /` | `/` is the total; `/var` was the largest top-level child | Parent totals should not be compared as sibling directories |
| `-x` limits `du` to one directory level | `-d1` limits depth; `-x` stays on one filesystem | Correct option knowledge produces accurate investigations |
| Docker was immediately the confirmed cause | Docker/container storage was initially a hypothesis | Evidence must confirm a hypothesis before action |
| `/var/lib/docker` was the largest container path | `/var/lib/containerd` was larger: 8.7G versus 6.1G | Read the actual output rather than assume |
| A large directory proves a disk problem | Size identifies a consumer; capacity and impact determine whether it is a problem | Avoid unnecessary cleanup |
| `echo $?` after `echo $0` verifies `kill` | It verifies only the immediately preceding `echo $0` | Exit status is time-sensitive |
| In the final scenario, inodes were exhausted | Disk blocks were nearly exhausted; inodes were only 12% used | Always distinguish `df -h` from `df -i` |
| `df` and `du` differed because of depth | A 60G deleted-but-open file was invisible to `du` but allocated in `df` | Recognizes a classic production issue |
| Recreating the deleted log would release space | The original process must close the old inode, usually by documented reload or graceful restart | A new path receives a different inode |

---

## 21. Real-life troubleshooting scenarios

### Scenario 1: Hands-on deleted log file

#### Symptom

A pathname has been removed, but disk blocks remain allocated.

#### Evidence

```bash
ls -l PATH              # pathname is missing
sudo lsof +L1           # process still holds the deleted inode
df -h FILESYSTEM_PATH   # filesystem still counts the blocks
```

#### Root cause

A process retains an open file descriptor for a file whose link count is zero.

#### Corrective action

Use the application's documented log-reopen mechanism or gracefully restart/reload the responsible process in an approved window. Avoid immediate `kill -9`.

#### Validation

```bash
sudo lsof +L1
df -h FILESYSTEM_PATH
```

### Scenario 2: Java application fills `/var`

#### Symptom

```text
No space left on device
```

#### Evidence

```text
$ df -h /var
Filesystem  Size  Used  Avail  Use%  Mounted on
/dev/sdb1   100G  95G   5G     95%   /var
```

```text
$ df -i /var
Filesystem  Inodes  IUsed  IFree  IUse%  Mounted on
/dev/sdb1   10M     1.2M   8.8M   12%    /var
```

```text
$ sudo du -xsh /var
35G /var
```

```text
$ sudo lsof +L1
COMMAND  PID  USER  FD  TYPE  SIZE/OFF  NLINK  NAME
java     815  app   7w  REG   60G       0      /var/log/app.log (deleted)
```

#### Diagnosis

- Disk-block capacity is nearly exhausted: `df -h` shows 95%.
- Inodes are healthy: `df -i` shows only 12% used.
- `du` reports 35G because it totals visible directory entries.
- `df` reports 95G because it counts all allocated filesystem blocks.
- The missing 60G is the deleted log still held open by Java PID `815` through file descriptor `7w`.

```text
35G visible + 60G deleted-but-open ≈ 95G allocated
```

#### Root cause statement

> Java process `815` still holds the deleted 60 GB `/var/log/app.log` inode open through file descriptor `7w`, preventing the filesystem from reclaiming its blocks.

#### Production-safe corrective action

1. Confirm the application owner and operational impact.
2. Check service health, redundancy, and maintenance/change requirements.
3. Use the application's documented log-reopen function if available.
4. Otherwise gracefully reload or restart the Java application in an approved window.
5. Do not begin with `kill -9`; it can interrupt transactions and prevent cleanup.
6. Confirm that the application returns healthy.
7. Validate block reclamation and deleted-open files.

```bash
sudo lsof +L1
df -h /var
```

#### Why recreating the filename does not fix it

A newly created `/var/log/app.log` receives a new inode. Java PID `815` would still hold the old deleted inode until it closes file descriptor `7w`.

### Why `df` and `du` can differ — interview answer

> `df` reports allocated blocks from the filesystem's perspective, whereas `du` scans visible files and totals their disk usage. A large difference can be caused by deleted-but-open files, mounted filesystems, hidden data beneath mount points, reserved blocks, or filesystem metadata. I would compare the same filesystem using `du -x`, inspect mounts with `findmnt`, and use `lsof +L1` to identify deleted files still held open by processes.

---

## 22. Final lesson conclusions

### Health of the training system

- Root disk capacity: healthy at 4% used
- Root inode capacity: healthy at 2% used
- Main visible consumers: `/var/lib/containerd`, `/var/lib/docker`, and `/home`
- RAM: healthy with approximately 6.9 GiB available
- Swap: 0B used
- Load: extremely low at approximately 0.01 on 12 logical CPUs
- Deleted-but-open files: none after lab cleanup

### Investigation model learned

```text
Observe symptom
→ Form hypothesis
→ Collect read-only evidence
→ Compare filesystem and process views
→ Narrow the scope
→ Identify root cause
→ Choose the safest corrective action
→ Validate service health and resource recovery
→ Document findings
```

### Lesson status

**Lesson 2 completed.**

The most important lesson is not memorizing a cleanup command. It is learning to distinguish disk blocks, inodes, visible directory usage, open file descriptors, memory availability, swap, CPU capacity, and load before changing the system.
