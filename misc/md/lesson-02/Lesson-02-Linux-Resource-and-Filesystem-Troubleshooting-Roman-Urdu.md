# Lesson 02: Linux Resource aur Filesystem Troubleshooting — Roman Urdu

> Yeh repository-ready study notes practical WSL lab, asal command outputs, ghaltiyon ki correction aur real-life production scenarios par mabni hain.

## Fehrist (Table of Contents)

1. [Lesson ke goals](#1-lesson-ke-goals)
2. [Troubleshooting ka sahi tareeqa](#2-troubleshooting-ka-sahi-tareeqa)
3. [`df -hT` se filesystem capacity](#3-df--ht-se-filesystem-capacity)
4. [Device, filesystem aur mount point](#4-device-filesystem-aur-mount-point)
5. [Disk blocks aur inodes](#5-disk-blocks-aur-inodes)
6. [`du` se directory investigation](#6-du-se-directory-investigation)
7. [Docker storage investigation](#7-docker-storage-investigation)
8. [`df` aur `du` mein farq](#8-df-aur-du-mein-farq)
9. [Deleted-but-open file practical](#9-deleted-but-open-file-practical)
10. [Memory aur swap](#10-memory-aur-swap)
11. [Load average aur CPU capacity](#11-load-average-aur-cpu-capacity)
12. [Meri ghaltiyan aur corrections](#12-meri-ghaltiyan-aur-corrections)
13. [Real-life Java production scenario](#13-real-life-java-production-scenario)
14. [Interview questions](#14-interview-questions)
15. [Quick command reference](#15-quick-command-reference)
16. [Final conclusion](#16-final-conclusion)

---

## 1. Lesson ke goals

Is lesson ke baad mujhe yeh kaam aane chahiye:

- Filesystem ki total, used aur available capacity check karna.
- Storage device aur mount point ka farq samajhna.
- Disk blocks aur inode exhaustion ko alag pehchanna.
- `du` se bari directory ko step-by-step locate karna.
- Docker aur containerd storage ko samajhna.
- `df` aur `du` ke different results ki wajah investigate karna.
- Deleted-but-open file ko `lsof +L1` se locate karna.
- RAM, cache, available memory aur swap ko samajhna.
- Load average ko logical CPUs ke saath compare karna.
- Production mein destructive action se pehle evidence collect karna.

---

## 2. Troubleshooting ka sahi tareeqa

Har investigation mein yeh sequence use karein:

1. **Observe** — asal symptom kya hai?
2. **Hypothesis** — mumkin wajah kya ho sakti hai?
3. **Evidence collect karein** — pehle read-only commands chalayein.
4. **Interpret karein** — normal aur abnormal values compare karein.
5. **Scope narrow karein** — filesystem se directory aur phir process tak jayein.
6. **Root cause identify karein** — assumption aur proof ko mix na karein.
7. **Safe action lein** — sab se chhota aur justified change.
8. **Validate karein** — resource aur application health dono confirm karein.
9. **Document karein** — commands, outputs, result aur prevention likhein.

### Hum is tarah kyun karte hain?

Guess karne se administrator ghalat data delete kar sakta hai ya asal problem chhup sakti hai. Evidence-first approach safe, repeatable aur interview mein explain karne ke qabil hota hai.

---

## 3. `df -hT` se filesystem capacity

Command:

```bash
df -hT
```

Options:

- `df` = disk free; filesystem ke perspective se capacity report karta hai.
- `-h` = human-readable units, jaise MiB aur GiB.
- `-T` = filesystem type bhi show karta hai.

Sirf root filesystem check karne ke liye:

```bash
df -hT /
```

Lab ka important output:

```text
Filesystem  Type  Size   Used  Avail  Use%  Mounted on
/dev/sdd    ext4  1007G  33G   923G   4%    /
```

### Columns ka matlab

| Column | Roman Urdu explanation |
|---|---|
| `Filesystem` | Kaunsa device ya virtual filesystem storage de raha hai |
| `Type` | Filesystem ka type, jaise `ext4`, `tmpfs`, `9p` |
| `Size` | Total filesystem capacity |
| `Used` | Kitni disk capacity use ho chuki hai |
| `Avail` | Normal users/applications ke liye kitni capacity baqi hai |
| `Use%` | Kitna percentage use hua hai |
| `Mounted on` | Filesystem kis directory par accessible hai |

### Hamare output ka result

- Total capacity: `1007G`
- Used: `33G`
- Available: `923G`
- Usage: sirf `4%`
- Result: root filesystem healthy hai; cleanup ki zaroorat nahi.

### `33G + 923G` exactly `1007G` kyun nahi?

Possible reasons:

- `ext4` root user aur emergency recovery ke liye blocks reserve kar sakta hai.
- Filesystem metadata aur internal structures space lete hain.
- `-h` values ko round karta hai.

Yeh space lost nahi hoti. Reserved space system ko full-disk emergency mein repair ka mauqa deti hai.

### Snap mounts 100% kyun show karte hain?

Example:

```text
snapfuse  fuse.snapfuse  62M  62M  0  100%  /snap/core24/1644
```

Snap packages compressed, fixed-size, read-only filesystem images hoti hain. Is liye `df` mein 100% show hona aam tor par normal hai. Har 100% entry disk-full emergency nahi hoti. Pehle filesystem type, mount point aur writability check karein.

---

## 4. Device, filesystem aur mount point

```text
Storage device  →  Filesystem  →  Mount point
/dev/sdd        →  ext4        →  /
```

- `/dev/sdd` Linux storage device hai.
- `ext4` data organize karne wala filesystem format hai.
- `/` root directory aur mount point hai jahan se Linux files access karta hai.

Analogy:

- `/dev/sdd` = storage cabinet
- `ext4` = cabinet ke andar organization system
- `/` = cabinet ka access door

Important correction:

> `/dev/sdd` root directory nahi hai. Root directory `/` hai.

Aur:

> `4%` disk storage usage hai, RAM usage nahi. RAM `free -h` se check hoti hai.

---

## 5. Disk blocks aur inodes

Command:

```bash
df -ih /
```

Actual output:

```text
Filesystem  Inodes  IUsed  IFree  IUse%  Mounted on
/dev/sdd       64M   748K    64M     2%  /
```

### Inode kya hota hai?

Inode filesystem ka data structure hota hai jo file ya directory ka metadata rakhta hai, jaise:

- Owner aur group
- Permissions
- File type
- Size
- Timestamps
- Data blocks ki location/pointers

Directory entry filename ko inode number ke saath connect karti hai. File ka actual content data blocks mein hota hai.

### Hamare result ka matlab

- Total inode capacity: taqreeban 64 million
- Used: 748K
- Usage: sirf 2%
- Result: inode exhaustion nahi hai.

### `No space left on device` do wajah se aa sakta hai

1. **Disk blocks full** — `df -h` se check karein.
2. **Inodes full** — `df -i` se check karein.

Is liye hamesha dono check karein:

```bash
df -hT AFFECTED_PATH
df -ih AFFECTED_PATH
```

---

## 6. `du` se directory investigation

Top-level directories check karne ka command:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Options:

- `du` = visible files/directories ka disk usage.
- `-x` = same filesystem par rahe; doosre mounts cross na kare.
- `-h` = human-readable sizes.
- `-d1` = sirf one level deep.
- `2>/dev/null` = permission errors hide karta hai.
- `sort -h` = sizes ko smallest se largest order mein sort karta hai.

Important output:

```text
3.2G  /usr
15G   /home
16G   /var
33G   /
```

### Important catch

`33G /` poore root tree ka total hai. Yeh `/var`, `/home` aur `/usr` ka sibling result nahi.

Largest top-level child directory:

```text
16G /var
```

`/mnt` sirf 4K is liye show hua kyun ke `-x` ne separate Windows `9p` filesystem `/mnt/c` ko cross nahi kiya.

### `/var` ko narrow karna

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Important output:

```text
195M  /var/cache
373M  /var/log
16G   /var/lib
16G   /var
```

Result:

> `/var/lib` almost poore `/var` usage ka zimmedar tha. `/var/log` main consumer nahi tha.

### `/var/lib` ko narrow karna

```bash
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
```

Important output:

```text
186M  /var/lib/snapd
208M  /var/lib/apt
6.1G  /var/lib/docker
8.7G  /var/lib/containerd
16G   /var/lib
```

Result:

- `/var/lib/containerd` sab se bara immediate subdirectory tha.
- `/var/lib/docker` second major consumer tha.
- Container-related data almost poora `/var/lib` use kar raha tha.
- Yeh sirf largest consumer identify karta hai; disk-full problem prove nahi karta.

Investigation path:

```text
/ → /var → /var/lib → containerd aur Docker
```

---

## 7. Docker storage investigation

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

- Images sab se bari reported category thi: `7.529GB`.
- 15 images mein se 7 containers ke reference mein thin.
- 9 stopped containers sirf `6.283MB` use kar rahe thay.
- Local volumes `5.941GB` use kar rahe thay aur unmein important persistent data ho sakta hai.
- Build cache future builds ko fast banata hai.
- `RECLAIMABLE` ka matlab yeh nahi ke bina investigation delete karna safe hai.

### Kya `docker system prune` chalana chahiye tha?

Nahi. Root filesystem sirf 4% used tha. Koi capacity problem nahi thi.

Cleanup se pehle confirm karein:

1. Kya asal mein filesystem pressure hai?
2. Resource ka owner aur purpose kya hai?
3. Application dependency kya hai?
4. Backup/recovery available hai?
5. Change approval required hai?

Plain `docker system prune` stopped containers, unused networks, unused images aur build cache remove kar sakta hai. Volumes persistent application/database data rakh sakte hain; unko bohat ehtiyat se handle karein.

---

## 8. `df` aur `du` mein farq

| Command | Kis perspective se dekhta hai? |
|---|---|
| `df` | Filesystem ke allocated blocks |
| `du` | Visible directory entries aur files |

`df` aur `du` different ho sakte hain kyun ke:

- File delete ho chuki ho lekin process ne open rakhi ho.
- Mounted filesystem boundaries different hon.
- Mount point ke neeche hidden data ho.
- Reserved blocks hon.
- Filesystem metadata/overhead ho.
- Sparse ya shared-layer accounting different ho.

Investigation commands:

```bash
df -hT AFFECTED_PATH
df -ih AFFECTED_PATH
sudo du -xhd1 AFFECTED_PATH 2>/dev/null | sort -h
findmnt
sudo lsof +L1
```

Interview answer:

> `df` filesystem ke perspective se allocated blocks report karta hai, jab ke `du` visible files ko scan karke unka disk usage total karta hai. Bara difference deleted-but-open files, mounted filesystems, mount point ke neeche hidden data, reserved blocks ya filesystem metadata ki wajah se ho sakta hai. Main same filesystem ko `du -x` se compare karunga, `findmnt` se mounts check karunga aur `lsof +L1` se deleted files ko locate karunga jo processes ne abhi tak open rakhi hon.

---

## 9. Deleted-but-open file practical

### Maqsad

Practical tor par prove karna ke deleted pathname ke bawajood open process disk blocks ko hold kar sakta hai.

### Step 1: Initial check

```bash
sudo lsof +L1
```

Initial output empty tha: us waqt koi deleted-but-open file detect nahi hui.

### Step 2: 50 MiB controlled file banana

```bash
dd if=/dev/zero \
  of=projects/linux-resource-lab/deleted-open-demo.log \
  bs=1M count=50 status=progress
```

Output:

```text
50+0 records in
50+0 records out
52428800 bytes (52 MB, 50 MiB) copied
```

- `if=/dev/zero` zero bytes provide karta hai.
- `of=` exact output file hai.
- `bs=1M` har block 1 MiB ka hai.
- `count=50` total 50 blocks likhta hai.

Verification:

```bash
ls -lh projects/linux-resource-lab/deleted-open-demo.log
```

```text
-rw-r--r-- 1 khalid khalid 50M ... deleted-open-demo.log
```

### Step 3: File ko background process mein open rakhna

```bash
tail -f projects/linux-resource-lab/deleted-open-demo.log >/dev/null &
```

Output:

```text
[1] 2104
```

- `[1]` shell job number tha.
- `2104` process ID tha.
- `>/dev/null` output discard karta tha.
- `&` process background mein chalata tha.

Verification:

```bash
lsof -p 2104 | grep 'deleted-open-demo.log'
```

```text
tail  2104 khalid  3r  REG  8,48  52428800  1154  .../deleted-open-demo.log
```

- `3r` = file descriptor 3, read mode.
- `REG` = regular file.
- `52428800` = 50 MiB bytes.
- `1154` = inode number.

### Step 4: Pathname delete karna

```bash
rm -- projects/linux-resource-lab/deleted-open-demo.log
```

Verification:

```bash
ls -lh projects/linux-resource-lab/deleted-open-demo.log
```

```text
No such file or directory
```

Path delete ho gaya, magar PID `2104` ne inode open rakha.

### Step 5: Hidden allocation detect karna

```bash
sudo lsof +L1
```

Relevant output:

```text
COMMAND  PID   USER    FD  TYPE  SIZE/OFF  NLINK  NAME
tail     2104  khalid  3r  REG   52428800  0      .../deleted-open-demo.log (deleted)
```

- `NLINK=0` = koi directory entry inode ko point nahi karti.
- `(deleted)` = pathname delete ho chuka hai.
- Open FD ki wajah se blocks abhi allocated hain.
- `du` pathname nahi dekh sakta.
- `df` allocated blocks count karta rehta hai.

### Step 6: Safe cleanup

Controlled lab process ko normal `SIGTERM` diya:

```bash
kill 2104
```

Output:

```text
[1]+ Terminated tail -f ... > /dev/null
```

Normal `kill PID` pehle use karein. `kill -9` first response nahi hona chahiye, kyun ke process ko graceful cleanup ka chance nahi milta.

### Step 7: Validation

```bash
sudo lsof +L1
```

Final output empty tha. File descriptor close hua aur 50 MiB blocks reclaim ho gaye.

### `$?` ka important lesson

```bash
echo $?
```

`$?` sirf immediately previous command ka exit status deta hai. Agar `kill` ke baad `echo $0` chalaya aur phir `echo $?`, to status `echo $0` ka hoga, `kill` ka nahi.

```bash
echo $0
```

Output `-bash` tha. Yeh Bash shell show karta hai; leading hyphen aam tor par login shell indicate karta hai.

---

## 10. Memory aur swap

Command:

```bash
free -h
```

Actual output:

```text
               total  used   free   shared  buff/cache  available
Mem:           7.5Gi  699Mi  6.1Gi  3.9Mi   948Mi       6.9Gi
Swap:          2.0Gi  0B     2.0Gi
```

### Columns

| Column | Matlab |
|---|---|
| `total` | Linux ke liye total RAM |
| `used` | Actively used memory |
| `free` | Bilkul unused RAM |
| `shared` | Shared temporary filesystems ki memory |
| `buff/cache` | Buffers aur filesystem cache |
| `available` | Naye applications ke liye estimated usable RAM without swapping |

### `available`, `free` se zyada kyun ho sakti hai?

Linux idle RAM ko file cache ke liye use karta hai taa-ke access fast ho. Jab application ko RAM chahiye, kernel cache ka bara hissa reclaim kar sakta hai. Is liye:

```text
free RAM ≠ tamam usable RAM
available RAM ≈ free + reclaimable memory
```

Hamare system mein `6.9Gi` available thi aur swap `0B` used tha. Current memory pressure ka evidence nahi tha.

Swap usage akeli problem prove nahi karti. Active swapping, `available`, latency, memory growth aur OOM events bhi check karein.

---

## 11. Load average aur CPU capacity

Commands:

```bash
uptime
nproc
```

Actual outputs:

```text
12:45:20 up 2:43, 1 user, load average: 0.01, 0.01, 0.00
```

```text
12
```

Interpretation:

- System ke paas 12 logical CPUs thay.
- Load averages last 1, 5 aur 15 minutes ke liye thay.
- `0.01` load, 12 CPUs ke liye bohat low hai.
- System almost idle tha.

Rough understanding:

- Load around `1` = approximately aik runnable ya uninterruptible task.
- Load around `12` = 12 logical CPUs par taqreeban aik demanding task per CPU.
- Sustained load `15` = investigation deserve karta hai.

High load hamesha high CPU utilization prove nahi karta. Linux load mein CPU ke liye runnable tasks ke saath uninterruptible I/O wait tasks bhi shamil hote hain.

Trend:

- 1-minute > 15-minute: load barh sakta hai.
- 1-minute < 15-minute: load kam ho sakta hai.
- Values similar: load relatively stable hai.

---

## 12. Meri ghaltiyan aur corrections

| Initial jawab/soch | Correct understanding |
|---|---|
| `/dev/sdd` root directory hai | `/dev/sdd` storage device hai; `/` root directory/mount point hai |
| `4%` memory used hai | `4%` disk filesystem usage hai |
| `Size=1007G` available storage hai | `Size` total capacity; `Avail=923G` remaining usable capacity hai |
| `/` sab se bari child directory hai | `/` overall total hai; `/var` largest top-level child tha |
| `-x` one-level depth deta hai | `-d1` depth limit karta hai; `-x` same filesystem par rakhta hai |
| Docker immediately confirmed cause tha | Pehle Docker sirf hypothesis tha; output ne container storage confirm ki |
| `/var/lib/docker` sab se bara tha | `/var/lib/containerd` 8.7G ke saath bara tha |
| Bara directory automatically problem hai | Bara size sirf consumer identify karta hai; capacity/impact problem decide karte hain |
| `echo $?` hamesha purane `kill` ka status deta hai | `$?` sirf immediately previous command ka status deta hai |
| Final scenario mein inodes exhausted thay | Disk blocks 95% thay; inodes sirf 12% used thay |
| `df` aur `du` ka farq depth ki wajah se tha | 60G deleted-but-open file `du` se invisible magar `df` mein allocated thi |
| Deleted log dobara create karne se old space release ho jayegi | New file ka new inode hoga; old FD close karna zaroori hai |

---

## 13. Real-life Java production scenario

### Symptom

```text
No space left on device
```

### Evidence

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

### Kaunsa resource exhausted hai?

**Disk blocks/capacity nearly exhausted hai.**

- `df -h` = 95% disk used.
- `df -i` = sirf 12% inodes used.
- Inodes healthy hain; 88% available hain.

### `df` 95G aur `du` 35G kyun?

```text
35G visible data + 60G deleted-but-open log ≈ 95G allocated blocks
```

`du` visible path scan karta hai. Deleted file ka pathname nahi, is liye 60G count nahi hoti. `df` filesystem allocated blocks count karta hai, is liye full 95G show karta hai.

### Root cause

> Java process PID `815` ne deleted 60 GB `/var/log/app.log` ko file descriptor `7w` ke through open rakha hua hai. Is wajah se filesystem blocks reclaim nahi kar sakta.

### `kill -9 815` foran kyun nahi?

`SIGKILL` application ko graceful shutdown, transactions complete karne, locks release karne ya state save karne ka chance nahi deta. Production impact ho sakta hai.

### Safe corrective action

1. Application owner aur operational impact confirm karein.
2. Service health, redundancy aur approved change window check karein.
3. Agar application documented log-reopen command/signal support karti hai to woh use karein.
4. Warna approved window mein Java application ko gracefully reload/restart karein.
5. Application health wapas verify karein.
6. Disk blocks aur deleted-open files validate karein.

```bash
sudo lsof +L1
df -h /var
```

### Same filename dobara create karna fix kyun nahi?

New `/var/log/app.log` ko new inode milega. Java PID `815` ab bhi old deleted inode ko FD `7w` se hold karega. Old blocks tab tak release nahi honge jab tak process old FD close na kare.

---

## 14. Interview questions

### Q1. `df` aur `du` mein basic difference kya hai?

`df` filesystem allocated blocks report karta hai. `du` visible files/directories ka usage total karta hai.

### Q2. Disk free ho phir bhi `No space left on device` aa sakta hai?

Haan. Inodes exhausted, quota limit, read-only filesystem ya doosri filesystem problem ho sakti hai.

### Q3. `df -h` 95% aur `df -i` 12% ho to kya exhausted hai?

Disk blocks nearly exhausted hain; inodes healthy hain.

### Q4. Deleted file disk space kyun hold kar sakti hai?

Agar process ka file descriptor open ho, inode aur data blocks tab tak allocated rehte hain jab tak FD close na ho.

### Q5. `lsof +L1` kya check karta hai?

Link count 1 se kam—typically zero—wali open files show karta hai, yani deleted-but-open files.

### Q6. `available` memory, `free` se zyada kyun?

Kyun ke `available` mein reclaimable cache ka estimate bhi shamil hota hai.

### Q7. High load kya hamesha high CPU hai?

Nahi. Load mein runnable tasks ke saath uninterruptible I/O wait tasks bhi shamil hain.

### Q8. Docker reclaimable space ko foran prune karna chahiye?

Nahi. Pehle actual capacity pressure, ownership, dependency, backup aur approval confirm karein.

---

## 15. Quick command reference

| Goal | Command |
|---|---|
| Filesystem capacity aur type | `df -hT` |
| Root filesystem | `df -hT /` |
| Inode usage | `df -ih /` |
| Top-level directory sizes | `sudo du -xhd1 PATH 2>/dev/null \| sort -h` |
| Ek path ka total visible usage | `du -sh PATH` |
| Mounts inspect karna | `findmnt` |
| Deleted-but-open files | `sudo lsof +L1` |
| Process ke open files | `lsof -p PID` |
| Docker storage summary | `docker system df` |
| Memory aur swap | `free -h` |
| Uptime aur load | `uptime` |
| Logical CPUs | `nproc` |
| Normal process termination | `kill PID` |
| Last command exit status | `echo $?` |
| Current shell | `echo $0` |

### Evidence collection example

```bash
{
  echo '===== FILESYSTEM ====='
  df -hT
  echo
  echo '===== INODES ====='
  df -ih
  echo
  echo '===== MEMORY ====='
  free -h
  echo
  echo '===== LOAD ====='
  uptime
  nproc
  echo
  echo '===== DELETED OPEN FILES ====='
  sudo lsof +L1
} | tee evidence/lesson-02-resource-baseline.txt
```

Evidence commit karne se pehle passwords, keys, tokens aur sensitive data check karein.

---

## 16. Final conclusion

### Training system ki health

- Root filesystem: 4% used — healthy
- Root inodes: 2% used — healthy
- Largest visible consumers: containerd, Docker aur `/home`
- Available RAM: taqreeban 6.9 GiB — healthy
- Swap used: 0B
- Load: 0.01 on 12 logical CPUs — bohat low
- Deleted-but-open files: practical cleanup ke baad none

### Final troubleshooting model

```text
Symptom observe karo
→ Hypothesis banao
→ Read-only evidence collect karo
→ Disk blocks aur inodes ko alag check karo
→ Filesystem se directory aur process tak scope narrow karo
→ Root cause prove karo
→ Safe corrective action lo
→ Resource aur application health validate karo
→ Documentation complete karo
```

### Lesson status

**Lesson 2 completed.**

Sab se important learning cleanup command yaad karna nahi, balki yeh samajhna hai ke disk blocks, inodes, visible directory usage, open file descriptors, available memory, swap, logical CPUs aur load average alag signals hain. Change karne se pehle in signals ko evidence ke saath interpret karna professional Linux troubleshooting hai.

