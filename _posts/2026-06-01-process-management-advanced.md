---
layout: post
title: "Process Management Advanced Deep Dive - Zombie, Signals, Cgroups, Namespaces"
date: 2026-06-01
categories: [linux]
tags: [processes, signals, cgroups, namespaces, containers]
---

# Process Management Advanced Deep Dive - Zombie, Signals, Cgroups, Namespaces

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng công ty của bạn có nhiều nhân viên (processes). Mỗi nhân viên:
- Có **mã nhân viên** (PID — Process ID)
- Được **tuyển bởi ai đó** (Parent Process)
- Ở một trong các **trạng thái**: Đang làm việc, Đang chờ, Bị tạm đình chỉ, Đã nghỉ việc

**Các tình huống đặc biệt:**
- **Zombie Process:** Nhân viên đã nghỉ nhưng hồ sơ chưa được thu dọn — tên vẫn trong danh sách nhưng không làm gì
- **Orphan Process:** Sếp bị sa thải, nhân viên còn lại được CEO (init/PID 1) "nhận nuôi"
- **Signal:** Tin nhắn từ quản lý: "Dọn dẹp bàn rồi đi về" (SIGTERM) hoặc "Ra khỏi tòa nhà NGAY" (SIGKILL)
- **Cgroups:** Quota cho phòng ban: "Phòng IT chỉ được dùng tối đa 4 máy tính và 16GB RAM"
- **Namespaces:** Mỗi phòng ban nghĩ mình là CÔNG TY RIÊNG — có CEO riêng, nhân viên riêng (container isolation)

---

## 2. Process States — Trạng Thái Của Process

### 2.1 Các Trạng Thái

```
         ┌──────────────────────────────────────────┐
         │              PROCESS STATES               │
         ├──────────┬───────────────────────────────┤
         │  State   │  Ý nghĩa                      │
         ├──────────┼───────────────────────────────┤
    ┌──→ │ R (Run)  │ Đang chạy hoặc sẵn sàng chạy │ ──┐
    │    ├──────────┼───────────────────────────────┤   │
    │    │ S (Sleep)│ Đang chờ (interruptible)      │ ←─┘
    │    ├──────────┼───────────────────────────────┤
    │    │ D (Disk) │ Đang chờ I/O (uninterruptible)│
    │    ├──────────┼───────────────────────────────┤
    │    │ T (Stop) │ Bị dừng (SIGSTOP/debug)       │
    │    ├──────────┼───────────────────────────────┤
    └────│ Z (Zombie)│ Đã chết, chờ parent thu dọn  │
         └──────────┴───────────────────────────────┘
```

**R (Running/Runnable):**
```
Process đang được CPU thực thi HOẶC đang trong run queue chờ CPU
- "Đang làm việc" hoặc "Xếp hàng chờ đến lượt"
- Ví dụ: CPU-intensive tasks (compression, rendering)
$ ps aux | awk '$8 ~ /R/'
```

**S (Sleeping — Interruptible):**
```
Process đang chờ event (keyboard input, network data, timer)
- CÓ THỂ bị đánh thức bởi signal
- Trạng thái PHỔBIẾN NHẤT (90% processes trên system)
- Ví dụ: shell chờ bạn gõ lệnh, web server chờ request mới
```

**D (Disk Sleep — Uninterruptible):**
```
Process đang chờ I/O hoàn thành
- KHÔNG THỂ bị kill (kể cả SIGKILL!) cho đến khi I/O xong
- Ví dụ: Đang đọc/ghi disk, NFS stuck
- Nếu thấy nhiều D state → I/O bottleneck hoặc storage issue!

⚠️ Process D state = KHÔNG kill được!
   Chỉ cách: fix I/O problem (unmount NFS, fix disk) hoặc reboot
```

**T (Stopped):**
```
Process bị tạm dừng hoàn toàn
- SIGSTOP (Ctrl+Z) → pause process
- SIGCONT → resume process  
- Debugger (gdb, strace) cũng dùng T state
```

**Z (Zombie):**
```
Process ĐÃ CHẾT nhưng parent chưa đọc exit status (wait())
- Không dùng CPU, không dùng memory (chỉ chiếm 1 entry trong process table)
- Hiện trong ps: <defunct>
- Nếu quá nhiều zombies: process table có thể hết slot!
```

### 2.2 Xem Process States

```bash
# ps aux — state column ($8)
ps aux
# USER  PID  %CPU %MEM  VSZ   RSS   TTY  STAT  START  TIME  COMMAND
# root    1   0.0  0.1  168   11456  ?    Ss    May15  0:15  /sbin/init
# user  500   0.5  2.0  1.5G  250M   ?    Sl    10:00  1:20  firefox
# user  501   0.0  0.0  0     0      ?    Z     10:01  0:00  [child] <defunct>

# STAT column codes:
# S = sleeping (interruptible)
# R = running
# D = disk sleep (uninterruptible)
# T = stopped
# Z = zombie
# Additional modifiers:
# s = session leader
# l = multi-threaded
# + = foreground process group
# < = high priority (nice < 0)
# N = low priority (nice > 0)

# Đếm processes theo state
ps -eo state | sort | uniq -c | sort -rn
# 250 S
#  10 R
#   3 D
#   2 Z  ← Có 2 zombies!
#   1 T
```

---

## 3. Zombie và Orphan Processes

### 3.1 Zombie Process — "Xác Sống"

**Ví dụ đời thường:** Nhân viên nộp đơn nghỉ việc rồi ra về, nhưng quản lý chưa ký duyệt → hồ sơ nhân viên vẫn nằm trong hệ thống, chiếm 1 slot.

```
Lifecycle dẫn đến Zombie:
1. Parent fork() → tạo Child process
2. Child thực hiện công việc
3. Child exit() → thành ZOMBIE (chờ parent gọi wait())
4. Parent gọi wait() → đọc exit status → Zombie BIẾN MẤT
5. Nếu Parent KHÔNG BAO GIỜ wait() → Zombie TỒN TẠI MÃI MÃI!

Parent ─fork()→ Child (running) ─exit()→ Child (zombie) ─parent wait()→ GONE!
                                                         ↑
                                        Nếu parent không wait() = zombie forever!
```

**Tìm và xử lý Zombies:**
```bash
# Tìm zombie processes
ps aux | grep 'Z'
ps -eo pid,ppid,state,cmd | grep '^.*Z'

# Zombie có PPID (Parent PID) → kill PARENT để giải phóng zombie
# (Khi parent chết, orphaned zombies được init adopt → init auto wait())
kill -SIGCHLD <PPID>    # Nhắc parent gọi wait()
kill <PPID>             # Kill parent → init adopt zombie → auto reap

# Không thể kill zombie trực tiếp (nó đã chết rồi!)
kill -9 <zombie_PID>    # VÔ ÍCH! Zombie không có gì để kill!
```

### 3.2 Orphan Process — "Trẻ Mồ Côi"

```
Scenario:
1. Parent fork() → Child
2. Parent exit() TRƯỚC Child
3. Child còn sống nhưng parent đã chết → ORPHAN!
4. Kernel tự động "nhận nuôi" (reparent) orphan cho PID 1 (init/systemd)
5. Khi orphan exit() → init/systemd sẽ wait() → clean up

Parent (PID 100) ─fork()→ Child (PID 101, PPID=100)
Parent exit()              Child (PID 101, PPID=1) ← reparented to init!

Orphan KHÔNG nguy hiểm: init luôn reap chúng properly
Zombie MỚI nguy hiểm: chiếm process table slots
```

---

## 4. Signals — Tín Hiệu Liên Lạc Giữa Processes

### 4.1 Signal Là Gì?

**Ví dụ đời thường:** Signal = tin nhắn gửi cho process:
- "Xin hãy dọn dẹp và đi về" (SIGTERM — lịch sự)
- "RA KHỎI TÒA NHÀ NGAY!" (SIGKILL — cưỡng chế, không thể từ chối)
- "Tạm dừng!" (SIGSTOP — pause)
- "Tiếp tục!" (SIGCONT — resume)

### 4.2 Signals Quan Trọng

| Signal | Số | Default Action | Catchable? | Ý nghĩa |
|--------|-----|---------------|-----------|----------|
| SIGHUP | 1 | Terminate | ✅ | Terminal closed / reload config |
| SIGINT | 2 | Terminate | ✅ | Ctrl+C (interrupt) |
| SIGQUIT | 3 | Core dump | ✅ | Ctrl+\ (quit + dump) |
| SIGKILL | 9 | Terminate | ❌ | Force kill (KHÔNG catch được!) |
| SIGTERM | 15 | Terminate | ✅ | Graceful shutdown (DEFAULT of kill) |
| SIGSTOP | 19 | Stop | ❌ | Pause (KHÔNG catch được!) |
| SIGCONT | 18 | Continue | ✅ | Resume stopped process |
| SIGUSR1 | 10 | Terminate | ✅ | User-defined (app custom) |
| SIGUSR2 | 12 | Terminate | ✅ | User-defined (app custom) |
| SIGCHLD | 17 | Ignore | ✅ | Child terminated |
| SIGPIPE | 13 | Terminate | ✅ | Broken pipe (reader closed) |
| SIGALRM | 14 | Terminate | ✅ | Timer expired |

### 4.3 Gửi Signals

```bash
# kill command (tên gây hiểu lầm — không chỉ kill!)
kill <PID>              # Gửi SIGTERM (default) — graceful
kill -15 <PID>          # Same as above
kill -SIGTERM <PID>     # Same as above

kill -9 <PID>           # SIGKILL — force kill (last resort!)
kill -SIGKILL <PID>     # Same

kill -HUP <PID>         # SIGHUP — reload config (nginx, apache)
kill -USR1 <PID>        # SIGUSR1 — application-defined action

# killall — kill by name
killall nginx           # SIGTERM to all nginx processes
killall -9 python       # SIGKILL to all python processes

# pkill — kill by pattern
pkill -f "python app.py"    # Kill process matching "python app.py"
pkill -u alice              # Kill all processes of user alice

# Keyboard signals:
# Ctrl+C = SIGINT (interrupt)
# Ctrl+Z = SIGTSTP (terminal stop — foreground → stopped)
# Ctrl+\ = SIGQUIT (quit + core dump)
```

### 4.4 Graceful Shutdown Pattern

```bash
# ĐÚNG: Graceful shutdown
kill <PID>          # Gửi SIGTERM
sleep 10            # Chờ process cleanup (close connections, flush data)
kill -0 <PID> 2>/dev/null && kill -9 <PID>  # Nếu vẫn sống → force kill

# Application nên handle SIGTERM:
# 1. Nhận SIGTERM
# 2. Stop accepting new requests
# 3. Finish current requests (connection draining)
# 4. Flush buffers, close files
# 5. Exit gracefully

# SAI: kill -9 ngay lập tức
kill -9 <PID>       # Process bị "bắn chết" không kịp cleanup!
                    # → Corrupt data, orphaned connections, lost work!
```

### 4.5 SIGHUP — Reload Configuration

```bash
# Nhiều daemon dùng SIGHUP để reload config MÀ KHÔNG restart:
kill -HUP $(cat /var/run/nginx.pid)     # Nginx reload config
kill -HUP $(pidof rsyslogd)             # Rsyslog reload
systemctl reload nginx                   # Same effect, nicer syntax

# Ưu điểm reload vs restart:
# - Không downtime (existing connections tiếp tục)
# - Faster (không cần re-initialize)
# - Zero-downtime config changes
```

---

## 5. Process Priority — nice/renice

### 5.1 Nice Values

**Ví dụ đời thường:** nice value = mức "nhường nhịn" của process:
- nice cao (process lịch sự, nhường CPU cho người khác)
- nice thấp (process ích kỷ, ưu tiên chiếm CPU)

```
Nice range: -20 (highest priority) → +19 (lowest priority)
Default: 0

-20 ←────────── 0 ──────────→ +19
Ưu tiên CAO    Default     Ưu tiên THẤP
(ít "nice")               (rất "nice", nhường)
```

```bash
# Chạy process với nice value
nice -n 10 ./heavy_backup.sh      # Low priority (nice to others)
nice -n -5 ./important_task       # High priority (cần root cho negative)

# Thay đổi nice của process đang chạy
renice 15 -p <PID>                # Reduce priority
sudo renice -10 -p <PID>         # Increase priority (cần root)
renice 5 -u alice                 # Tất cả processes của alice → nice 5

# Xem nice value
ps -eo pid,ni,comm | head
# PID  NI  COMMAND
#   1   0  systemd
# 500  10  backup
# 501  -5  database
```

---

## 6. Cgroups (Control Groups) — Giới Hạn Resources

### 6.1 Cgroups Là Gì?

**Ví dụ đời thường:** Cgroups = **quota cho phòng ban trong công ty**:
- "Phòng Marketing chỉ được dùng tối đa 4 máy tính" (CPU limit)
- "Phòng IT budget tối đa 16GB RAM" (Memory limit)
- "Phòng Sales chỉ được dùng 100 Mbps internet" (I/O limit)

```
Cgroups cho phép:
1. LIMIT: Giới hạn resource (CPU, Memory, I/O, Network)
2. PRIORITIZE: Ưu tiên resource cho nhóm processes
3. ACCOUNT: Đo lường resource usage
4. CONTROL: Freeze/unfreeze nhóm processes

Cgroups v1 vs v2:
- v1: Multiple hierarchies (mỗi controller = 1 tree) — phức tạp, khó consistent
- v2: Single unified hierarchy — 1 tree cho tất cả controllers — RECOMMENDED
  Linux 5.0+ (2019): v2 considered stable
  systemd 244+ (2020): default to v2
```

### 6.2 Cgroups v2 Hierarchy

```
/sys/fs/cgroup/          ← Root cgroup
├── system.slice/        ← System services
│   ├── nginx.service/
│   ├── postgresql.service/
│   └── ssh.service/
├── user.slice/          ← User sessions
│   ├── user-1000.slice/
│   └── user-1001.slice/
└── machine.slice/       ← VMs/Containers
    ├── docker-abc123.scope/
    └── libvirt-vm1.scope/
```

### 6.3 Cgroups Controllers

```
CPU Controller:
- cpu.max: "100000 100000"  # max 100% of 1 CPU (quota/period)
- cpu.max: "50000 100000"   # max 50% of 1 CPU
- cpu.weight: 100           # Relative weight (default 100, range 1-10000)

Memory Controller:
- memory.max: 536870912     # Hard limit 512MB (kill if exceed = OOM!)
- memory.high: 268435456    # Soft limit 256MB (throttle, not kill)
- memory.current: 150000000 # Current usage
- memory.swap.max: 0        # No swap allowed

I/O Controller:
- io.max: "8:0 rbps=1048576 wbps=524288"  # Limit device 8:0 to 1MB/s read, 512KB/s write
- io.weight: "8:0 100"     # Relative I/O weight

PIDs Controller:
- pids.max: 100            # Max 100 processes in this cgroup
- pids.current: 15         # Current process count
```

### 6.4 Thao Tác Cgroups

```bash
# Xem cgroup của process hiện tại
cat /proc/self/cgroup
# 0::/user.slice/user-1000.slice/session-1.scope

# Xem resource usage
cat /sys/fs/cgroup/user.slice/user-1000.slice/memory.current
# 150000000 (bytes)

# Tạo cgroup mới và set limits
sudo mkdir /sys/fs/cgroup/myapp
echo "512M" | sudo tee /sys/fs/cgroup/myapp/memory.max
echo "50000 100000" | sudo tee /sys/fs/cgroup/myapp/cpu.max
echo "100" | sudo tee /sys/fs/cgroup/myapp/pids.max

# Thêm process vào cgroup
echo <PID> | sudo tee /sys/fs/cgroup/myapp/cgroup.procs

# Dùng systemd (recommended way):
# Tạo service với resource limits
systemctl set-property myapp.service MemoryMax=512M
systemctl set-property myapp.service CPUQuota=50%

# Hoặc trong unit file:
# [Service]
# MemoryMax=512M
# CPUQuota=50%
# TasksMax=100
```

### 6.5 Cgroups Trong Docker/Containers

```bash
# Docker dùng cgroups để limit containers:
docker run --memory=512m --cpus=1.5 --pids-limit=100 myapp

# Tương đương cgroups v2:
# memory.max = 536870912 (512MB)
# cpu.max = "150000 100000" (1.5 CPUs)
# pids.max = 100

# Xem container resource usage
docker stats
# CONTAINER  CPU%  MEM USAGE/LIMIT  MEM%  NET I/O  BLOCK I/O  PIDS
# abc123     45%   256MB / 512MB    50%   1.2MB    500KB      15
```

---

## 7. Namespaces — Cô Lập (Isolation)

### 7.1 Namespaces Là Gì?

**Ví dụ đời thường:** Namespaces = **phòng cách ly** — mỗi process group sống trong "thế giới riêng", nghĩ rằng mình là duy nhất:
- Mỗi container nghĩ mình có PID 1 (PID namespace)
- Mỗi container nghĩ mình có eth0 riêng (Network namespace)
- Mỗi container nghĩ mình thấy toàn bộ filesystem (Mount namespace)
- Mỗi container nghĩ hostname là riêng mình (UTS namespace)

### 7.2 Các Loại Namespace

| Namespace | Cô lập gì | Flag | Ví dụ |
|-----------|-----------|------|-------|
| PID | Process IDs | CLONE_NEWPID | Container PID 1 ≠ Host PID 1 |
| NET | Network (interfaces, IPs, routes) | CLONE_NEWNET | Container có eth0 riêng |
| MNT | Mount points | CLONE_NEWNS | Container thấy / riêng |
| UTS | Hostname, domain | CLONE_NEWUTS | Container hostname riêng |
| IPC | Shared memory, semaphores | CLONE_NEWIPC | Container IPC riêng |
| USER | User/Group IDs | CLONE_NEWUSER | Root trong container ≠ root ngoài |
| Cgroup | Cgroup root | CLONE_NEWCGROUP | Container thấy cgroup tree riêng |
| Time | System clocks | CLONE_NEWTIME | Container thời gian riêng (Linux 5.6+) |

### 7.3 PID Namespace

```
Host (PID namespace 0):
PID 1: systemd
PID 100: dockerd
PID 200: container init (sh)   ← Trong container, đây là PID 1!
PID 201: nginx (child of 200)  ← Trong container, đây là PID 2!

Container view (PID namespace 1):
PID 1: sh (container init)     ← Nghĩ mình là PID 1!
PID 2: nginx                   ← Chỉ thấy processes trong namespace

# Host thấy TẤT CẢ processes (cả container)
# Container chỉ thấy processes CỦA MÌNH
```

### 7.4 Network Namespace

```bash
# Tạo network namespace
sudo ip netns add myns

# Xem danh sách
ip netns list

# Chạy command trong namespace
sudo ip netns exec myns ip addr
# Chỉ thấy loopback (lo), KHÔNG thấy host interfaces!

# Tạo veth pair (virtual ethernet cable) nối 2 namespaces
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns myns

# Configure
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up
sudo ip netns exec myns ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec myns ip link set veth1 up

# Ping giữa namespaces
ping 10.0.0.2          # From host → myns: works!
sudo ip netns exec myns ping 10.0.0.1  # From myns → host: works!
```

### 7.5 Mount Namespace

```bash
# Mount namespace cho phép mỗi process thấy filesystem tree khác nhau
# Đây là cơ sở của container filesystem isolation

# Container chỉ thấy rootfs của mình:
# / (overlay filesystem from image layers)
# /proc (new procfs showing only container processes)
# /sys (limited sysfs)
# /dev (limited devices)
# /etc/hostname (container hostname)
# /etc/resolv.conf (container DNS config)

# Host vẫn thấy filesystem thật
```

### 7.6 Containers = Cgroups + Namespaces

```
Container = Process(es) chạy với:
├── Namespaces (ISOLATION — container nghĩ mình là 1 machine riêng)
│   ├── PID namespace → PID 1 riêng
│   ├── NET namespace → Network stack riêng
│   ├── MNT namespace → Filesystem riêng
│   ├── UTS namespace → Hostname riêng
│   ├── IPC namespace → IPC riêng
│   └── USER namespace → UID/GID mapping riêng
│
├── Cgroups (RESOURCE LIMITS — container bị giới hạn resources)
│   ├── CPU → max X CPUs
│   ├── Memory → max Y GB
│   ├── I/O → max Z MB/s
│   └── PIDs → max N processes
│
└── Security (thêm layers bảo vệ)
    ├── Seccomp → limit system calls
    ├── AppArmor/SELinux → MAC policies
    └── Capabilities → limit root powers

Docker/containerd/CRI-O chỉ là "tools" thiết lập cgroups + namespaces!
```

---

## 8. Practical Process Management

### 8.1 Monitor Processes

```bash
# Real-time monitoring
top                     # Classic
htop                    # Better UI, tree view
btop                    # Modern, beautiful

# Snapshot
ps aux                  # All processes, BSD syntax
ps -ef                  # All processes, POSIX syntax
ps -eo pid,ppid,state,ni,%cpu,%mem,cmd --sort=-%cpu | head  # Custom columns

# Process tree
pstree -p               # Tree with PIDs
pstree -p <PID>         # Tree of specific process

# Resource usage per process
pidstat 1               # CPU/memory/IO per process every 1 second
pidstat -d 1            # I/O per process
pidstat -r 1            # Memory per process
```

### 8.2 Troubleshooting Scenarios

```bash
# Scenario 1: System slow, CPU high
top → sort by CPU (P key)
# Tìm process ăn CPU → kill hoặc renice
renice 19 -p <PID>     # Lower priority

# Scenario 2: System slow, high I/O wait
top → check %wa (I/O wait) in header
iotop                   # Xem process nào đang I/O nhiều
# Process ở D state = đang chờ I/O

# Scenario 3: Out of memory, OOM killer
dmesg | grep -i "out of memory"
dmesg | grep -i "killed process"
# OOM killer chọn process có oom_score cao nhất để kill
cat /proc/<PID>/oom_score        # Xem OOM score
echo -1000 > /proc/<PID>/oom_score_adj  # Protect khỏi OOM killer (-1000 = never kill)

# Scenario 4: Too many zombies
ps -eo state | grep Z | wc -l   # Đếm zombies
ps -eo pid,ppid,state,cmd | grep Z
# Kill parent process → init reap zombies
kill <PPID of zombie>

# Scenario 5: Process cannot be killed
kill -9 <PID>           # Nếu vẫn không chết:
cat /proc/<PID>/status | grep State
# State: D → uninterruptible sleep (I/O stuck)
# Phải fix I/O problem (NFS mount, disk error) hoặc reboot
```

---

## 9. OOM Killer — Khi Hệ Thống Hết RAM

### 9.1 OOM Killer Là Gì?

**Ví dụ đời thường:** Tàu sắp chìm vì quá tải. Thuyền trưởng phải "hy sinh" ai đó để cứu số đông. OOM Killer = Linux kernel's "thuyền trưởng" — chọn process ít quan trọng nhất để kill khi hết RAM.

```bash
# Khi total memory (RAM + Swap) cạn kiệt:
# 1. Kernel triggers OOM Killer
# 2. OOM Killer tính oom_score cho mỗi process
# 3. Kill process có oom_score CAO NHẤT
# 4. Log in dmesg

# Check dmesg cho OOM events
dmesg | grep -i "oom\|killed"
# [12345.678] Out of memory: Killed process 1234 (java) total-vm:8GB

# oom_score: 0 (never kill) → 1000 (first to kill)
# Factors: memory usage, process age, nice value, oom_score_adj

# Protect critical processes:
echo -1000 > /proc/$(pidof sshd)/oom_score_adj    # SSHD: never kill
echo -500 > /proc/$(pidof mysqld)/oom_score_adj   # MySQL: unlikely to kill
echo 1000 > /proc/$(pidof backup)/oom_score_adj   # Backup: kill first

# System-wide OOM behavior
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic overcommit (default)
# 1 = always overcommit (never fail malloc)
# 2 = never overcommit (strict, process gets ENOMEM if no real RAM)
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Quick Reference

```
Process States: R(run) S(sleep) D(disk IO) T(stopped) Z(zombie)
Zombie: process chết chưa được reap → kill parent để fix
Orphan: parent chết trước → init adopts (harmless)

Signals:
  kill <PID>        = SIGTERM (graceful)
  kill -9 <PID>     = SIGKILL (force, last resort)
  kill -HUP <PID>   = SIGHUP (reload config)
  Ctrl+C            = SIGINT
  Ctrl+Z            = SIGTSTP (background)

Priority: nice -20 (highest) → +19 (lowest), default 0

Cgroups: Limit resources (CPU, Memory, I/O, PIDs)
  memory.max, cpu.max, pids.max

Namespaces: Isolate (PID, NET, MNT, UTS, IPC, USER)
  = Foundation of containers!
```

### 10.2 Key Takeaways

1. **Process states** chỉ ra process đang làm gì — D state = I/O problem
2. **Zombie** = process chết chưa reap → kill parent để fix, không kill zombie trực tiếp
3. **SIGTERM trước, SIGKILL sau** — luôn cho process cơ hội cleanup
4. **SIGHUP** = reload config without restart (nginx, apache, rsyslog)
5. **Cgroups v2** limit resources — basis of container resource management
6. **Namespaces** provide isolation — basis of container security
7. **Container = Namespaces + Cgroups** + security layers
8. **OOM Killer** kills processes when RAM exhausted — protect critical services

### 10.3 Tài Liệu Tham Khảo

- man pages: signal(7), cgroups(7), namespaces(7), clone(2), fork(2), wait(2)
- Linux kernel documentation: Documentation/admin-guide/cgroup-v2.rst
- Linux kernel documentation: Documentation/process/
- "The Linux Programming Interface" by Michael Kerrisk — Chapters 20-28
- Red Hat Resource Management Guide
- Docker documentation: Resource constraints
- man capabilities(7) — Linux capabilities

---

*Bài viết tiếp theo: Systemd Deep Dive — Quản lý services trên Linux hiện đại*
