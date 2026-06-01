---
layout: post
title: "Linux Performance Tuning - Tối ưu hiệu năng hệ thống Linux"
date: 2026-06-01
categories: [linux]
tags: [performance, vmstat, iostat, perf, sysctl, tuning]
---

## Mục lục
1. [Góc nhìn tổng quan - Bệnh viện hệ thống](#goc-nhin-tong-quan)
2. [vmstat - Đo nhịp tim hệ thống](#vmstat)
3. [iostat - Kiểm tra sức khỏe ổ đĩa](#iostat)
4. [sar - Hồ sơ bệnh án dài hạn](#sar)
5. [perf và Flamegraphs - Chụp X-quang CPU](#perf)
6. [strace/ltrace - Theo dõi cuộc gọi hệ thống](#strace)
7. [/proc/sys và sysctl - Điều chỉnh bộ não kernel](#procsys)
8. [I/O Schedulers - Quản lý hàng đợi đĩa](#io-schedulers)
9. [Memory Tuning - Swap, Swappiness, OOM Killer](#memory-tuning)
10. [Tổng kết và quy trình troubleshooting](#tong-ket)

---

## 1. Góc nhìn tổng quan - Bệnh viện hệ thống {#goc-nhin-tong-quan}

### Ví dụ đời thường

Hãy tưởng tượng hệ thống Linux là một **bệnh viện**:

- **CPU** = đội ngũ bác sĩ (nhiều core = nhiều bác sĩ)
- **RAM** = phòng bệnh (chỗ để bệnh nhân nằm điều trị)
- **Disk I/O** = phòng xét nghiệm (gửi mẫu đi, chờ kết quả)
- **Network** = xe cấp cứu (chở bệnh nhân đến/đi)
- **Swap** = hành lang (khi phòng bệnh hết chỗ, bệnh nhân nhẹ ra nằm hành lang)

**Performance tuning** = tìm ra **bottleneck** (nút thắt cổ chai) và khắc phục:
- Bác sĩ quá tải? → Thêm CPU hoặc tối ưu code
- Hết phòng bệnh? → Thêm RAM hoặc giảm memory leak
- Phòng xét nghiệm chậm? → Nâng cấp SSD hoặc tối ưu I/O pattern
- Xe cấp cứu kẹt? → Tăng bandwidth hoặc tối ưu network stack

### Nguyên tắc vàng của Performance Tuning

```
1. MEASURE FIRST (Đo trước, đoán sau)
   - Không bao giờ tối ưu mà không có baseline
   - "Premature optimization is the root of all evil" - Knuth

2. ONE CHANGE AT A TIME (Thay đổi từng cái một)
   - Thay đổi 1 parameter → đo → so sánh
   - Nếu thay 5 cái cùng lúc, không biết cái nào hiệu quả

3. UNDERSTAND BEFORE TUNING (Hiểu trước khi sửa)
   - Hiểu workload: CPU-bound? I/O-bound? Memory-bound?
   - Hiểu tradeoffs: tăng throughput có thể tăng latency

4. DOCUMENT EVERYTHING (Ghi chép mọi thứ)
   - Baseline trước khi thay đổi
   - Parameter đã thay đổi + giá trị cũ/mới
   - Kết quả sau thay đổi
```

### Tổng quan các công cụ

```
┌─────────────────────────────────────────────────┐
│          Linux Performance Tools                 │
├─────────────────────────────────────────────────┤
│  Real-time:                                      │
│    top/htop   - Overview tổng quan               │
│    vmstat     - CPU, memory, I/O summary         │
│    iostat     - Disk I/O detail                  │
│    mpstat     - Per-CPU statistics               │
│    pidstat    - Per-process resource usage        │
│                                                  │
│  Historical:                                     │
│    sar        - System Activity Reporter          │
│    atop       - Advanced top with history         │
│                                                  │
│  Deep profiling:                                 │
│    perf       - CPU profiling, flamegraphs       │
│    strace     - System call tracing              │
│    ltrace     - Library call tracing             │
│                                                  │
│  Kernel tuning:                                  │
│    sysctl     - Runtime kernel parameters        │
│    /proc/sys  - Virtual filesystem interface     │
│    tuned      - Automated tuning profiles        │
└─────────────────────────────────────────────────┘
```

---

## 2. vmstat - Đo nhịp tim hệ thống {#vmstat}

### vmstat là gì?

**vmstat** (Virtual Memory Statistics) cho bạn cái nhìn tổng quan nhanh nhất về CPU, memory, I/O, và system calls. Giống như đo huyết áp + nhịp tim - nhanh, cho biết tình trạng chung.

### Cú pháp và output

```bash
# vmstat [interval] [count]
vmstat 1 5    # Mỗi 1 giây, 5 lần

# Output mẫu:
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 245678  45632 892340    0    0   120    80  890 1534 15  5 78  2  0
 5  1      0 243200  45632 892600    0    0     0   240 1200 2100 45 10 40  5  0
```

### Đọc hiểu output vmstat

```
procs (processes):
  r = Run queue (số process đang chờ CPU)
      → r > số CPU cores = CPU overloaded
  b = Blocked (số process chờ I/O)
      → b > 0 liên tục = I/O bottleneck

memory:
  swpd  = Swap đang dùng (KB)
  free  = RAM trống
  buff  = Buffer (metadata, directory cache)
  cache = Page cache (file data cached)

swap:
  si = Swap In (KB/s) - đọc từ swap vào RAM
  so = Swap Out (KB/s) - ghi từ RAM ra swap
  → si/so > 0 liên tục = RAM không đủ!

io (block I/O):
  bi = Blocks In (KB/s) - đọc từ disk
  bo = Blocks Out (KB/s) - ghi ra disk

system:
  in = Interrupts/second
  cs = Context Switches/second
  → cs rất cao = quá nhiều thread switching

cpu (% thời gian):
  us = User space (application code)
  sy = System/Kernel (system calls)
  id = Idle (rảnh)
  wa = Wait (chờ I/O)
  st = Stolen (VM bị hypervisor lấy CPU)
```

### Các pattern nhận biết bottleneck

```bash
# CPU-bound (quá tải CPU):
# r > num_cores, us + sy > 80%, id < 20%
procs   cpu
 r  b   us sy id wa
 8  0   85 10  5  0    ← 8 processes chờ CPU, idle chỉ 5%

# I/O-bound (chờ disk):
# b > 0, wa > 20%, bi/bo cao
procs   cpu          io
 r  b   us sy id wa  bi    bo
 1  4    5  3 12 80  8500  2000  ← 4 blocked, 80% waiting

# Memory pressure (thiếu RAM):
# si/so > 0, swpd tăng liên tục
swap         memory
si    so     swpd    free
2048  4096   524288  12000  ← Đang swap liên tục!
```

---

## 3. iostat - Kiểm tra sức khỏe ổ đĩa {#iostat}

### iostat là gì?

**iostat** (I/O Statistics) cho chi tiết về hiệu năng từng disk device. Nó giống như "siêu âm" cho ổ đĩa - thấy rõ device nào bận, queue dài bao nhiêu, throughput bao nhiêu.

Thuộc package **sysstat**.

### Cú pháp và output quan trọng

```bash
# iostat -xz [interval] [count]
iostat -xz 1 5

# Output (extended mode):
Device   r/s     w/s    rkB/s   wkB/s  rrqm/s  wrqm/s  %rrqm %wrqm  r_await w_await  aqu-sz  rareq-sz  wareq-sz  svctm  %util
sda     45.00  120.00  180.00  960.00    5.00   80.00  10.00 40.00    1.20    2.50    0.41     4.00      8.00    0.60  10.00
nvme0n1 500.00 2000.00 8000.00 32000.00  0.00    0.00   0.00  0.00    0.08    0.12    0.30    16.00     16.00    0.04   8.00
```

### Đọc hiểu iostat

```
r/s, w/s     = Reads/Writes per second (IOPS)
rkB/s, wkB/s = Throughput (KB/s)
r_await      = Average read latency (ms)
w_await      = Average write latency (ms)
aqu-sz       = Average queue size
%util        = % thời gian device bận

Ngưỡng cảnh báo (rule of thumb):
┌──────────────┬─────────┬──────────┬───────────┐
│ Metric       │ HDD OK  │ SSD OK   │ Cảnh báo  │
├──────────────┼─────────┼──────────┼───────────┤
│ await (ms)   │ < 10    │ < 2      │ > 20      │
│ %util        │ < 70%   │ < 80%    │ > 90%     │
│ aqu-sz       │ < 2     │ < 4      │ > 8       │
│ IOPS (HDD)   │ 100-200 │ N/A      │ Near max  │
│ IOPS (NVMe)  │ N/A     │ 50K-500K │ Near max  │
└──────────────┴─────────┴──────────┴───────────┘
```

### Ví dụ phân tích thực tế

```bash
# Theo dõi disk cụ thể
iostat -xz -p sda 1

# Kiểm tra xem disk nào đang chịu tải
iostat -xz 1 | awk '$NF > 50 {print}'  # %util > 50%

# So sánh throughput vs IOPS
# IOPS cao + throughput thấp = random I/O (database)
# IOPS thấp + throughput cao = sequential I/O (backup, streaming)
```

---

## 4. sar - Hồ sơ bệnh án dài hạn {#sar}

### sar là gì?

**sar** (System Activity Reporter) thu thập và lưu trữ performance data theo thời gian. Nó giống "bệnh án" - bạn có thể xem lại hệ thống hoạt động thế nào hôm qua, tuần trước.

Thuộc package **sysstat**, daemon **sadc** thu thập data mỗi 10 phút (mặc định).

### Các mode hữu ích

```bash
# CPU usage
sar -u 1 5            # Hiện tại, mỗi 1s, 5 lần
sar -u -f /var/log/sysstat/sa01  # Dữ liệu ngày 01

# Memory usage
sar -r 1 5            # RAM utilization
sar -S 1 5            # Swap usage

# Disk I/O
sar -d 1 5            # Disk activity
sar -b 1 5            # I/O transfer rate

# Network
sar -n DEV 1 5        # Network interface stats
sar -n SOCK 1 5       # Socket statistics
sar -n TCP 1 5        # TCP stats (retransmits!)

# Load average
sar -q 1 5            # Run queue and load

# Xem data historical (ngày cụ thể)
sar -u -f /var/log/sysstat/sa15   # CPU ngày 15
sar -r -s 09:00:00 -e 12:00:00   # RAM 9h-12h hôm nay
```

### Ví dụ: Tìm thời điểm spike

```bash
# Tìm thời điểm CPU spike trong ngày
sar -u | awk '$NF < 20 {print}'  # idle < 20% = busy
# Output:
# 14:20:01     all     78.50   12.30    0.00    5.20    0.00    4.00
# 14:30:01     all     92.10    5.80    0.00    1.50    0.00    0.60

# Correlation: CPU spike + disk I/O cùng thời điểm?
sar -u -s 14:00:00 -e 15:00:00
sar -d -s 14:00:00 -e 15:00:00
```

---

## 5. perf và Flamegraphs - Chụp X-quang CPU {#perf}

### perf là gì?

**perf** là công cụ profiling cấp kernel, cho phép bạn biết chính xác CPU đang làm gì - hàm nào tốn thời gian nhất, cache miss ở đâu, branch misprediction bao nhiêu.

Giống như chụp **X-quang**: thấy bên trong chương trình đang chạy.

### Các lệnh perf cơ bản

```bash
# perf stat - Thống kê tổng quan
perf stat ./my_program
# Output:
#   1,523,456,789 cycles
#     845,678,901 instructions      # 0.55 insn per cycle
#      12,345,678 cache-misses      # 2.5% of cache references
#       1,234,567 branch-misses     # 1.2% of branches

# perf top - Real-time profiling (giống top cho functions)
perf top -g
# Thấy ngay hàm nào đang chiếm CPU nhiều nhất

# perf record - Thu thập data
perf record -g ./my_program       # Profile 1 chương trình
perf record -g -p 1234            # Profile PID cụ thể
perf record -g -a sleep 30        # Profile toàn hệ thống 30s

# perf report - Phân tích data đã thu thập
perf report
perf report --stdio              # Text mode
perf report --sort=dso           # Group by library
```

### Flamegraphs - Biểu đồ ngọn lửa

Flamegraph là cách **trực quan hóa** call stack, cho thấy ngay hàm nào tốn thời gian.

```bash
# Tạo flamegraph:
# 1. Thu thập data
perf record -g -a sleep 30

# 2. Chuyển đổi sang format flamegraph
perf script > out.perf

# 3. Dùng FlameGraph tools (Brendan Gregg)
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flamegraph.svg
```

### Đọc hiểu Flamegraph

```
Flamegraph giống chồng sách:

┌─────────────────────────────────────────┐
│         function_C (chiếm 60% CPU)      │ ← Đỉnh = nơi CPU thực sự làm việc
├────────────────────┬────────────────────┤
│   function_B       │    function_D       │ ← Caller (gọi function_C)
├────────────────────┴────────────────────┤
│              function_A                  │ ← Root caller
├─────────────────────────────────────────┤
│              main()                      │ ← Entry point
└─────────────────────────────────────────┘

Chiều rộng = % thời gian CPU
Chiều cao = call stack depth
Đọc từ dưới lên: main → A → B → C
Hàm ở đỉnh rộng nhất = bottleneck!
```

### Ví dụ perf thực tế

```bash
# Tìm nguyên nhân CPU spike
perf record -g -a -F 99 sleep 10    # Sample 99Hz, 10 giây
perf report --sort=comm,dso,symbol  # Group by process, library, function

# Profile chỉ user-space
perf record --call-graph dwarf -p $(pgrep nginx) sleep 30

# Hardware counters - Tìm cache miss
perf stat -e cache-misses,cache-references,instructions,cycles \
  ./database_query_benchmark

# Lock contention profiling
perf lock record ./multithreaded_app
perf lock report
```

---

## 6. strace/ltrace - Theo dõi cuộc gọi hệ thống {#strace}

### strace là gì?

**strace** theo dõi tất cả system calls (cuộc gọi hệ thống) mà process thực hiện. System calls là cách chương trình "nói chuyện" với kernel - đọc file, gửi network, allocate memory.

Giống như **nghe lén cuộc gọi** giữa chương trình và kernel.

### Cú pháp cơ bản

```bash
# Trace một chương trình
strace ./my_program

# Trace process đang chạy
strace -p 1234

# Chỉ theo dõi loại syscall cụ thể
strace -e trace=file ./program      # File operations
strace -e trace=network ./program   # Network operations
strace -e trace=process ./program   # fork, exec, exit
strace -e trace=memory ./program    # mmap, brk, mprotect

# Đếm syscalls (summary)
strace -c ./program
# Output:
# % time     seconds  usecs/call     calls   syscall
# ------ ----------- ----------- --------- ---------
#  45.23    0.052341          12      4362  read
#  30.12    0.034867           8      4358  write
#  15.67    0.018142          52       349  open

# Với timestamps
strace -t ./program           # HH:MM:SS
strace -T ./program           # Duration mỗi syscall
strace -r ./program           # Relative time
```

### Patterns phổ biến khi debug

```bash
# Tìm file nào chương trình đang tìm (debug "file not found")
strace -e trace=openat ./program 2>&1 | grep -i error
# openat(AT_FDCWD, "/etc/app/config.yaml", O_RDONLY) = -1 ENOENT

# Tìm tại sao chương trình treo (hung)
strace -p $(pgrep hung_process) -e trace=futex,read,write
# Thường thấy: futex(0x..., FUTEX_WAIT, ...) → deadlock

# Tìm network connections
strace -e trace=connect ./program
# connect(3, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.0.1")}, ...)

# Đo file I/O patterns
strace -e trace=read,write -e read=3 -p $(pgrep mydb)
```

### ltrace - Library call tracing

```bash
# ltrace theo dõi library function calls (libc, libssl...)
ltrace ./program

# Hữu ích để thấy:
# - malloc/free patterns (memory leaks)
# - strlen gọi quá nhiều lần (performance)
# - SSL function calls (debugging TLS)
ltrace -e malloc+free ./program 2>&1 | awk '/malloc/{m++}/free/{f++}END{print "malloc:",m,"free:",f}'
```

---

## 7. /proc/sys và sysctl - Điều chỉnh bộ não kernel {#procsys}

### /proc/sys là gì?

**/proc/sys** là virtual filesystem cho phép đọc/ghi kernel parameters tại runtime. **sysctl** là command-line interface cho /proc/sys.

Giống như **bảng điều khiển** của xe: bạn có thể điều chỉnh nhiệt độ AC, âm lượng radio, độ nhạy phanh... mà không cần tắt máy.

### Cấu trúc /proc/sys

```
/proc/sys/
├── kernel/     # Kernel behavior
│   ├── threads-max
│   ├── pid_max
│   └── sched_*
├── net/        # Network stack
│   ├── core/
│   ├── ipv4/
│   └── ipv6/
├── vm/         # Virtual memory
│   ├── swappiness
│   ├── dirty_ratio
│   └── overcommit_memory
├── fs/         # Filesystem
│   ├── file-max
│   └── inotify/
└── dev/        # Device specific
```

### sysctl commands

```bash
# Đọc giá trị
sysctl vm.swappiness
sysctl -a                       # Tất cả parameters
sysctl -a | grep net.core      # Filter

# Ghi giá trị (tạm thời - mất khi reboot)
sysctl -w vm.swappiness=10
sysctl -w net.core.somaxconn=65535

# Ghi vĩnh viễn (persist across reboot)
echo "vm.swappiness=10" >> /etc/sysctl.d/99-tuning.conf
sysctl --system    # Reload tất cả sysctl files
```

### Network Tuning Parameters

```bash
# === HIGH-TRAFFIC WEB SERVER ===

# Tăng connection backlog
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# Tăng buffer sizes
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216

# TCP optimization
net.ipv4.tcp_fastopen = 3           # TCP Fast Open
net.ipv4.tcp_fin_timeout = 10       # Giảm TIME_WAIT
net.ipv4.tcp_tw_reuse = 1           # Reuse TIME_WAIT sockets
net.ipv4.tcp_slow_start_after_idle = 0  # Không reset cwnd

# Congestion control
net.ipv4.tcp_congestion_control = bbr  # BBR (Google)
net.core.default_qdisc = fq           # Fair Queueing

# Connection tracking (for firewalls)
net.netfilter.nf_conntrack_max = 1000000
```

### Filesystem Tuning

```bash
# Tăng số file descriptors
fs.file-max = 2097152          # System-wide limit
# + ulimit -n 65535            # Per-process limit (in /etc/security/limits.conf)

# inotify limits (for file watchers - IDEs, webpack)
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024

# AIO (async I/O)
fs.aio-max-nr = 1048576
```

---

## 8. I/O Schedulers - Quản lý hàng đợi đĩa {#io-schedulers}

### I/O Scheduler là gì?

I/O Scheduler quyết định **thứ tự** các disk requests được thực hiện. Giống nhân viên **sắp xếp hàng đợi** ở ngân hàng - có thể theo thứ tự đến (FIFO), ưu tiên VIP, hay nhóm theo khu vực.

### Các scheduler trong Linux

```
┌──────────────┬──────────────────────────────────────────────┐
│ Scheduler    │ Mô tả                                        │
├──────────────┼──────────────────────────────────────────────┤
│ none/noop    │ FIFO đơn giản, không reorder                  │
│              │ → Tốt cho NVMe (hardware đã optimize)         │
├──────────────┼──────────────────────────────────────────────┤
│ mq-deadline  │ Deadline-based, đảm bảo latency              │
│              │ → Tốt cho database workload                   │
├──────────────┼──────────────────────────────────────────────┤
│ bfq          │ Budget Fair Queueing                          │
│              │ → Tốt cho desktop (interactive)               │
├──────────────┼──────────────────────────────────────────────┤
│ kyber        │ Low-overhead, target latency                  │
│              │ → Tốt cho fast SSD với mixed workload         │
└──────────────┴──────────────────────────────────────────────┘
```

### Thay đổi I/O Scheduler

```bash
# Xem scheduler hiện tại
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Đổi scheduler (runtime)
echo "none" > /sys/block/nvme0n1/queue/scheduler

# Đổi vĩnh viễn (kernel parameter)
# /etc/default/grub:
# GRUB_CMDLINE_LINUX="elevator=none"

# Hoặc udev rule cho từng loại device:
# /etc/udev/rules.d/60-io-scheduler.rules
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
```

### Tuning I/O queue parameters

```bash
# Queue depth (nr_requests)
echo 256 > /sys/block/sda/queue/nr_requests

# Read-ahead (sequential read optimization)
echo 4096 > /sys/block/sda/queue/read_ahead_kb  # 4MB read-ahead

# For databases: giảm read-ahead (random I/O dominant)
echo 16 > /sys/block/sda/queue/read_ahead_kb
```

---

## 9. Memory Tuning - Swap, Swappiness, OOM Killer {#memory-tuning}

### Mô hình bộ nhớ Linux

```
┌──────────────────────────────────────────────┐
│                Physical RAM                   │
├──────────────────────────────────────────────┤
│  ┌─────────┐  ┌──────────┐  ┌────────────┐  │
│  │ Process │  │  Page    │  │  Kernel    │  │
│  │ Memory  │  │  Cache   │  │  (slab,    │  │
│  │ (anon)  │  │ (file    │  │   buffers) │  │
│  │         │  │  cache)  │  │            │  │
│  └─────────┘  └──────────┘  └────────────┘  │
└──────────────────────────────────────────────┘
         ↕ swap                  ↕ drop cache
┌──────────────┐
│  Swap Space  │  (disk partition hoặc file)
└──────────────┘

Giải thích bằng ví dụ:
- Process Memory (anon pages) = sách bạn đang ĐỌC
- Page Cache = sách bạn đã đọc xong, để trên bàn (có thể cần lại)
- Swap = kệ sách phía xa (cất sách ít dùng)
- free RAM = chỗ trống trên bàn

Linux LUÔN cố lấp đầy RAM bằng page cache!
"free" RAM thấp ≠ hết RAM (cache có thể giải phóng ngay)
```

### vm.swappiness

```bash
# swappiness = xu hướng kernel swap anonymous pages vs drop page cache
# Range: 0-200 (mặc định: 60)
#
# swappiness = 0  : Chỉ swap khi cực kỳ cần (gần OOM)
# swappiness = 10 : Tối thiểu swap (recommended for databases)
# swappiness = 60 : Mặc định (cân bằng)
# swappiness = 100: Swap và drop cache có priority ngang nhau
# swappiness = 200: Ưu tiên swap hơn drop cache

# Server database (muốn giữ data trong RAM):
sysctl -w vm.swappiness=10

# Server web (muốn giữ file cache):
sysctl -w vm.swappiness=60

# Kiểm tra swap usage
free -h
swapon --show
vmstat 1 | awk '{print $7, $8}'  # si, so columns
```

### Dirty Pages và Write-back

```bash
# "Dirty pages" = data đã thay đổi trong RAM nhưng chưa ghi xuống disk
# Kernel gom dirty pages và ghi batch (write-back) để tối ưu I/O

# vm.dirty_ratio (% RAM): Khi dirty pages đạt ngưỡng này,
# process PHẢI chờ (synchronous write) = freeze/stall!
vm.dirty_ratio = 10              # Mặc định 20, giảm cho DB server

# vm.dirty_background_ratio: Background flush bắt đầu
vm.dirty_background_ratio = 5    # Mặc định 10

# vm.dirty_expire_centisecs: Dirty page sống tối đa (centiseconds)
vm.dirty_expire_centisecs = 1000  # 10 seconds (mặc định 30s)

# vm.dirty_writeback_centisecs: Tần suất flush daemon chạy
vm.dirty_writeback_centisecs = 100  # 1 second
```

### OOM Killer (Out of Memory Killer)

```bash
# Khi RAM + Swap hết sạch, kernel phải kill process để survive
# OOM Killer chọn process có oom_score cao nhất

# Xem OOM score của process
cat /proc/PID/oom_score       # Score hiện tại (0-1000)
cat /proc/PID/oom_score_adj   # Adjustment (-1000 to 1000)

# Bảo vệ process quan trọng (không bị OOM kill)
echo -1000 > /proc/$(pgrep postgres)/oom_score_adj

# Hy sinh process ít quan trọng trước
echo 1000 > /proc/$(pgrep cache-warmer)/oom_score_adj

# Overcommit settings
# vm.overcommit_memory:
# 0 = Heuristic (mặc định) - kernel ước lượng
# 1 = Always overcommit (malloc luôn thành công)
# 2 = Don't overcommit (strict: CommitLimit = RAM * ratio + Swap)

# Cho database server (không muốn OOM surprises):
vm.overcommit_memory = 2
vm.overcommit_ratio = 80   # CommitLimit = 80% RAM + Swap
```

### Transparent Huge Pages (THP)

```bash
# THP tự động dùng 2MB pages thay vì 4KB
# Tốt cho: applications với memory access pattern lớn, liên tục
# Xấu cho: databases (fragmentation, latency spikes khi compaction)

# Kiểm tra trạng thái
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Tắt THP cho database servers
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Persist (systemd service hoặc grub: transparent_hugepage=never)
```

---

## 10. Tổng kết và quy trình troubleshooting {#tong-ket}

### Quy trình 5 bước khi hệ thống chậm

```
Step 1: QUICK OVERVIEW (30 giây)
   uptime         → Load average tăng?
   dmesg -T | tail → Kernel errors?
   vmstat 1 3     → CPU/IO/Memory quick check

Step 2: IDENTIFY BOTTLENECK TYPE
   CPU?    → us+sy > 80%, r > cores
   Memory? → si/so > 0, free rất thấp
   Disk?   → wa > 20%, %util > 80%
   Network? → sar -n DEV (dropped, errors)

Step 3: DEEP DIVE
   CPU    → perf top, perf record + flamegraph
   Memory → free -h, /proc/meminfo, slabtop
   Disk   → iostat -xz 1, iotop
   Network → ss -s, netstat -s, tcpdump

Step 4: IDENTIFY ROOT CAUSE
   Which process?  → pidstat, top, ps aux --sort=-%cpu
   Which function? → perf report, strace
   Which file?     → lsof, strace -e trace=file
   Which connection? → ss -tnp, strace -e trace=network

Step 5: FIX AND VERIFY
   Apply fix → measure → compare with baseline
   Document change → monitor for regression
```

### Bảng tham chiếu nhanh

| Triệu chứng | Công cụ kiểm tra | Tuning thường gặp |
|---|---|---|
| Load average cao | vmstat, mpstat | Check r queue, per-CPU balance |
| Swap thrashing | vmstat (si/so), free | vm.swappiness, thêm RAM |
| Disk latency cao | iostat, iotop | I/O scheduler, read_ahead |
| Network drops | sar -n DEV, ethtool | Ring buffer, net.core.* |
| Too many connections | ss -s, sysctl | somaxconn, file-max |
| Context switch cao | vmstat (cs), pidstat -w | Giảm threads, CPU affinity |

### Tuning profiles phổ biến

```bash
# === WEB SERVER (nginx/Apache) ===
net.core.somaxconn = 65535
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_tw_reuse = 1
vm.swappiness = 30
fs.file-max = 2097152

# === DATABASE SERVER (PostgreSQL/MySQL) ===
vm.swappiness = 1
vm.dirty_ratio = 10
vm.dirty_background_ratio = 3
vm.overcommit_memory = 2
# + THP disabled
# + I/O scheduler = mq-deadline hoặc none (NVMe)

# === CONTAINER HOST (Docker/K8s) ===
vm.swappiness = 10
fs.inotify.max_user_watches = 524288
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.max_map_count = 262144  # Elasticsearch
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Systems Performance (Brendan Gregg, 2020) | Sách kinh điển về Linux performance |
| Linux Perf Analysis in 60s (Brendan Gregg) | Netflix checklist nổi tiếng |
| Red Hat Performance Tuning Guide | Hướng dẫn RHEL chính thức |
| kernel.org sysctl documentation | Tài liệu tham chiếu kernel parameters |
| USE Method (Brendan Gregg) | Utilization, Saturation, Errors framework |
| perf wiki (perf.wiki.kernel.org) | Hướng dẫn perf chính thức |
| FlameGraph repo (github.com/brendangregg) | Tools tạo flamegraph |

---

*Bài viết tiếp theo: [SSH Advanced Usage](/2026/08/09/ssh-advanced-usage/) - Sử dụng SSH nâng cao*
