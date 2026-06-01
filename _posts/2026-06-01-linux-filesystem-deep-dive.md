---
layout: post
title: "Linux Filesystem Deep Dive - Hệ Thống File Trên Linux"
date: 2026-06-01
categories: [linux]
tags: [filesystem, inodes, lvm, ext4, xfs, btrfs]
---

# Linux Filesystem Deep Dive - Hệ Thống File Trên Linux

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng ổ cứng máy tính như một **kho hàng rất lớn**. Filesystem (Hệ thống file) là cách bạn TỔ CHỨC kho hàng đó:
- **Không có filesystem:** Hàng hóa vứt lung tung khắp kho — muốn tìm 1 món phải lục hết kho
- **Có filesystem:** Hàng được xếp theo kệ, có nhãn, có sổ kiểm kê → tìm ngay lập tức

**Cụ thể hơn:**
- **File** = hàng hóa (data thực tế)
- **Directory** = kệ hàng/phòng (nhóm files lại)
- **Inode** = thẻ kiểm kê (metadata: size, owner, vị trí trên disk)
- **Filename** = nhãn dán trên hàng hóa (chỉ là tên gọi, trỏ đến inode)
- **Mount** = kết nối thêm 1 kho phụ vào kho chính

**Ví dụ khác:**

| Filesystem concept | Ví dụ thư viện |
|-------------------|----------------|
| Partition | Khu vực trong thư viện (Khu A: Khoa học, Khu B: Văn học) |
| Filesystem | Hệ thống tổ chức sách (Dewey Decimal, LC) |
| Directory | Kệ sách (Kệ 101: Vật lý, Kệ 102: Hóa học) |
| File | Cuốn sách cụ thể |
| Inode | Phiếu catalog (ghi tác giả, năm XB, vị trí kệ) |
| Hard link | 2 phiếu catalog cùng trỏ đến 1 cuốn sách |
| Soft link | Phiếu ghi "Xem sách ở Kệ 205" (trỏ đến phiếu khác) |

---

## 2. Kiến Thức Nền Tảng — Cách Ổ Cứng Lưu Data

### 2.1 Block Devices

```
Ổ cứng (HDD/SSD) lưu data theo BLOCKS (khối):
- Mỗi block = 512 bytes (sector) hoặc 4096 bytes (4K sector modern)
- Filesystem thường dùng block size = 4096 bytes (4KB)
- File 10 bytes vẫn chiếm 1 block (4KB) → lãng phí 4086 bytes!
- File 5000 bytes chiếm 2 blocks (8KB) → lãng phí 3192 bytes

Block addressing:
Block 0: [Superblock - metadata filesystem]
Block 1: [Inode table - danh sách inodes]
Block 2: [Data block - nội dung file A]
Block 3: [Data block - nội dung file A (tiếp)]
Block 4: [Data block - nội dung file B]
...
```

### 2.2 Partition Table

```
Physical Disk (ổ cứng vật lý):
┌─────────────────────────────────────────────────────────┐
│ MBR/GPT │ Partition 1 │ Partition 2 │ Partition 3 │ Free│
│ Header  │ (ext4, /)   │ (swap)      │ (xfs, /data)│     │
└─────────────────────────────────────────────────────────┘

MBR (Master Boot Record) — Legacy:
- Max 4 primary partitions
- Max 2TB per partition
- 32-bit addressing

GPT (GUID Partition Table) — Modern:
- Max 128 partitions
- Max 8 ZB (zettabytes) per partition
- 64-bit addressing
- Backup header at end of disk
- Checksum protection
```

### 2.3 Công Cụ Phân Vùng

```bash
# fdisk — MBR partitions (legacy)
sudo fdisk /dev/sdb
# m = help, n = new partition, d = delete, w = write, q = quit

# parted — GPT partitions (modern, recommended)
sudo parted /dev/sdb
# (parted) mklabel gpt
# (parted) mkpart primary ext4 0% 50%
# (parted) mkpart primary xfs 50% 100%
# (parted) print

# gdisk — GPT fdisk (alternative)
sudo gdisk /dev/sdb

# Xem partition hiện tại
lsblk                    # Tree view của block devices
cat /proc/partitions     # Kernel view
sudo fdisk -l            # Detailed partition info
```

---

## 3. Inodes — Thẻ Căn Cước Của File

### 3.1 Inode Là Gì?

**Ví dụ đời thường:** Mỗi công dân có CMND/CCCD với số duy nhất. CMND lưu thông tin (tên, ngày sinh, địa chỉ) nhưng KHÔNG phải là con người thật. Tương tự, **inode lưu metadata của file** nhưng KHÔNG chứa tên file hay nội dung file.

```
Inode chứa:
┌─────────────────────────────────────┐
│ Inode Number: 131073 (unique ID)    │
├─────────────────────────────────────┤
│ File type: regular file             │
│ Permissions: rwxr-xr-x (755)       │
│ Owner UID: 1000                     │
│ Group GID: 1000                     │
│ Size: 4096 bytes                    │
│ Number of hard links: 2            │
│ Timestamps:                         │
│   atime: 2026-06-01 10:00 (access)│
│   mtime: 2026-05-15 09:30 (modify)│
│   ctime: 2026-05-15 09:30 (change)│
│ Block pointers:                     │
│   Direct[0]: block 1024            │
│   Direct[1]: block 1025            │
│   Indirect: block 2048             │
└─────────────────────────────────────┘

KHÔNG chứa trong inode:
✗ Filename (tên file lưu trong directory entry!)
✗ File content (data lưu trong data blocks!)
```

### 3.2 Xem Thông Tin Inode

```bash
# Xem inode number
ls -i /etc/passwd
# 131073 /etc/passwd

# Xem chi tiết inode
stat /etc/passwd
#   File: /etc/passwd
#   Size: 2847       Blocks: 8          IO Block: 4096   regular file
# Device: 8,1        Inode: 131073      Links: 1
# Access: (0644/-rw-r--r--)  Uid: (0/root)   Gid: (0/root)
# Access: 2026-06-01 10:00:00
# Modify: 2026-05-15 09:30:00
# Change: 2026-05-15 09:30:00
#  Birth: 2026-01-01 00:00:00

# Đếm inodes used/free
df -i
# Filesystem     Inodes    IUsed   IFree  IUse%  Mounted on
# /dev/sda1     6553600   234567  6318033   4%   /
```

### 3.3 Inode Exhaustion — Hết Inode!

```bash
# Tình huống: Disk còn trống nhưng không tạo file được!
touch newfile.txt
# touch: cannot touch 'newfile.txt': No space left on device

# Check: df nói còn space nhưng df -i nói hết inodes!
df -h     # 50GB free (space OK)
df -i     # IUse% 100% (INODES HẾT!)

# Nguyên nhân: quá nhiều files nhỏ (millions of tiny files)
# Ví dụ: mail server lưu mỗi email = 1 file, cache files, session files

# Fix: Tìm directory có nhiều files nhất
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
# 1500000 /var/spool/mail    ← Quá nhiều files!
# 800000  /tmp/sessions

# Giải pháp:
# 1. Xóa files không cần
# 2. Recreate filesystem với nhiều inodes hơn (mkfs -N)
# 3. Dùng filesystem hỗ trợ dynamic inodes (XFS, btrfs)
```

---

## 4. Hard Links vs Soft Links (Symbolic Links)

### 4.1 Hard Link

**Ví dụ đời thường:** Một cuốn sách trong thư viện có 2 phiếu catalog (1 phiếu theo tên tác giả, 1 phiếu theo chủ đề). Xóa 1 phiếu → sách vẫn tìm được qua phiếu còn lại. Chỉ khi xóa TẤT CẢ phiếu → sách mới "mất" (thực tế sách vẫn nằm trên kệ cho đến khi bị thu dọn).

```bash
# Tạo hard link
ln /home/user/original.txt /home/user/hardlink.txt

# Cả 2 files có CÙNG inode number!
ls -li
# 131073 -rw-r--r-- 2 user user 100 Jun 1 original.txt
# 131073 -rw-r--r-- 2 user user 100 Jun 1 hardlink.txt
#  ↑ same inode      ↑ link count = 2

# Xóa original → hardlink vẫn hoạt động (data vẫn còn!)
rm original.txt
cat hardlink.txt    # Vẫn đọc được!
# Data chỉ bị xóa thật khi link count = 0
```

**Đặc điểm Hard Link:**
- Cùng inode = cùng data = cùng permissions = hoàn toàn giống nhau
- KHÔNG thể hard link đến directory (tránh loop)
- KHÔNG thể hard link cross filesystem/partition
- Xóa original → hard link vẫn sống (data còn đến khi link count = 0)
- Không tốn thêm disk space (chỉ thêm 1 directory entry)

### 4.2 Soft Link (Symbolic Link — Symlink)

**Ví dụ đời thường:** Biển chỉ đường "Bệnh viện ← 500m". Biển KHÔNG phải bệnh viện, nó chỉ CHỈ ĐƯỜNG đến bệnh viện. Nếu bệnh viện bị phá → biển vẫn còn nhưng chỉ vào chỗ trống.

```bash
# Tạo symlink
ln -s /home/user/original.txt /home/user/symlink.txt

# Symlink có INODE RIÊNG, nội dung = path đến target
ls -li
# 131073 -rw-r--r-- 1 user user 100 Jun 1 original.txt
# 131074 lrwxrwxrwx 1 user user  27 Jun 1 symlink.txt -> /home/user/original.txt
#  ↑ khác inode!  ↑ type 'l'         ↑ size = length of path string

# Xóa original → symlink BỊ HỎNG (dangling symlink!)
rm original.txt
cat symlink.txt
# cat: symlink.txt: No such file or directory (broken link!)
```

**Đặc điểm Soft Link:**
- Có inode RIÊNG (khác với target)
- CÓ THỂ link đến directory
- CÓ THỂ link cross filesystem/partition
- Xóa target → link bị hỏng (dangling)
- Tốn thêm 1 inode + data block (lưu path string)
- Permissions trên symlink không có ý nghĩa (luôn lrwxrwxrwx)

### 4.3 So Sánh

| Feature | Hard Link | Soft Link |
|---------|-----------|-----------|
| Inode | Same as target | Different (own inode) |
| Cross filesystem | ❌ | ✅ |
| Link to directory | ❌ | ✅ |
| Target deleted | Still works! | Broken! |
| Disk space | Minimal (directory entry) | 1 inode + path |
| Use case | Backup, multiple names | Shortcuts, version switching |

---

## 5. VFS (Virtual File System) — Tầng Trừu Tượng

### 5.1 VFS Là Gì?

**Ví dụ đời thường:** Bạn dùng remote TV. Dù TV là Samsung, LG, hay Sony, bạn đều bấm nút cùng cách (Vol+, Vol-, Channel). Remote là "interface thống nhất". **VFS là remote của Linux** — cho phép bạn dùng CÙNG lệnh (open, read, write, close) cho BẤT KỲ filesystem nào.

```
User Space:     open("/home/file.txt")   read()   write()   close()
                         │                  │        │         │
─────────────────────────┼──────────────────┼────────┼─────────┼──
Kernel Space:            │                  │        │         │
                    ┌────┴──────────────────┴────────┴─────────┴──┐
                    │            VFS (Virtual File System)          │
                    │   Provides UNIFORM interface to user space    │
                    └─────┬────────────┬─────────────┬────────────┘
                          │            │             │
                    ┌─────┴────┐ ┌─────┴────┐ ┌─────┴────┐
                    │   ext4   │ │   XFS    │ │  procfs  │
                    │ (disk)   │ │ (disk)   │ │ (virtual)│
                    └─────┬────┘ └─────┬────┘ └──────────┘
                          │            │
                    ┌─────┴────┐ ┌─────┴────┐
                    │  /dev/sda│ │ /dev/sdb │
                    │  (SSD)   │ │  (HDD)   │
                    └──────────┘ └──────────┘
```

### 5.2 VFS Objects

```
VFS quản lý 4 objects chính:

1. Superblock: Metadata của filesystem đã mount
   - Block size, total blocks, free blocks
   - Inode count, free inodes
   - Filesystem type, mount options
   
2. Inode: Metadata của file/directory
   - Permissions, owner, timestamps
   - Block pointers (vị trí data)
   - Operations: create, link, unlink, rename

3. Dentry (Directory Entry): Ánh xạ name → inode
   - Filename + inode number
   - Cached in dcache (directory cache) for speed
   - "foo.txt" → inode 131073

4. File: Đại diện cho file ĐANG MỞ
   - Current position (offset)
   - Open mode (read/write)
   - Operations: read, write, seek, close
```

### 5.3 /proc và /sys — Virtual Filesystems

```
/proc — Process Information (procfs):
- KHÔNG lưu trên disk! Generated by kernel on-the-fly
- /proc/PID/ — thông tin của mỗi process
- /proc/cpuinfo — CPU info
- /proc/meminfo — Memory info
- /proc/sys/ — Kernel tunable parameters

Ví dụ:
$ cat /proc/cpuinfo       # Thông tin CPU
$ cat /proc/meminfo       # RAM usage
$ cat /proc/1/status      # Process 1 (systemd) info
$ cat /proc/sys/net/ipv4/ip_forward   # IP forwarding enabled?
$ echo 1 > /proc/sys/net/ipv4/ip_forward  # Enable IP forwarding

/sys — Sysfs (Device/Driver info):
- Hardware devices tree
- /sys/class/net/ — network interfaces
- /sys/block/ — block devices
- /sys/fs/ — filesystem info

Ví dụ:
$ cat /sys/class/net/eth0/speed      # Link speed (Mbps)
$ cat /sys/block/sda/queue/scheduler # I/O scheduler
$ ls /sys/class/net/                 # List network interfaces
```

---

## 6. Filesystem Types — ext4, XFS, Btrfs, ZFS

### 6.1 ext4 (Fourth Extended Filesystem)

**Ví dụ đời thường:** ext4 như Toyota Camry — reliable, mature, phổ biến nhất, "mặc định" cho hầu hết mọi người.

```
Đặc điểm:
- Default filesystem cho hầu hết Linux distributions
- Max file size: 16 TB
- Max filesystem size: 1 EB (exabyte)
- Journaling: metadata + optional data
- Extents (thay vì block mapping) → hiệu quả cho large files
- Backward compatible: có thể mount ext3/ext2 as ext4

Ưu điểm:
✓ Extremely mature và stable (15+ năm)
✓ Tốt cho general purpose (OS, applications, databases)
✓ Low CPU overhead
✓ Excellent tooling (e2fsck, resize2fs, tune2fs)
✓ Works well on both HDD and SSD

Nhược điểm:
✗ Fixed inode count tại mkfs time (phải plan trước)
✗ Không có built-in snapshots
✗ Không có checksumming cho data (chỉ metadata)
✗ Không có native compression
✗ Max 16TB file size (giới hạn cho very large DBs)

Tạo ext4:
$ sudo mkfs.ext4 /dev/sdb1
$ sudo tune2fs -l /dev/sdb1    # Xem filesystem parameters
$ sudo resize2fs /dev/sdb1     # Resize (grow or shrink)
```

### 6.2 XFS

**Ví dụ đời thường:** XFS như xe tải lớn — chuyên chở hàng nặng (large files), chạy nhanh trên đường cao tốc (high throughput).

```
Đặc điểm:
- Default cho RHEL/CentOS/Rocky 7+
- Max file size: 8 EB
- Max filesystem size: 8 EB
- Excellent parallel I/O (allocation groups)
- Dynamic inode allocation (KHÔNG bao giờ hết inodes!)
- Online resize (chỉ grow, KHÔNG shrink!)

Ưu điểm:
✓ Excellent performance cho large files + high throughput
✓ Dynamic inodes → never run out
✓ Parallel allocation groups → great for multi-threaded I/O
✓ Excellent for large filesystems (100+ TB)
✓ Delayed allocation → reduced fragmentation

Nhược điểm:
✗ Cannot shrink! (chỉ grow được)
✗ Không có checksumming (đang develop)
✗ Delete nhiều small files chậm hơn ext4
✗ Recovery after crash có thể mất data (delayed allocation)

Tạo XFS:
$ sudo mkfs.xfs /dev/sdb1
$ sudo xfs_info /dev/sdb1      # Xem info
$ sudo xfs_growfs /mountpoint   # Grow filesystem
$ sudo xfs_repair /dev/sdb1    # Repair
```

### 6.3 Btrfs (B-tree Filesystem)

**Ví dụ đời thường:** Btrfs như smartphone đời mới — nhiều tính năng hiện đại (snapshot, compression, checksumming) nhưng đôi khi còn bugs.

```
Đặc điểm:
- Copy-on-Write (CoW) filesystem
- Built-in snapshots (instant, space-efficient)
- Built-in compression (zlib, lzo, zstd)
- Data + metadata checksumming (detect corruption!)
- Built-in RAID (0, 1, 5, 6, 10)
- Subvolumes (lightweight virtual filesystems)
- Online defragmentation
- Send/receive (incremental backup)

Ưu điểm:
✓ Snapshots = instant backup (rollback dễ dàng)
✓ Checksumming = phát hiện data corruption
✓ Compression = tiết kiệm disk space 30-50%
✓ Subvolumes = flexible partitioning
✓ Self-healing (with RAID: detect + correct corruption)

Nhược điểm:
✗ RAID 5/6 KHÔNG stable! (write hole, known bugs)
✗ Performance kém hơn ext4/XFS cho databases (CoW overhead)
✗ Fragmentation issues với random writes
✗ Maturity concerns (so với ext4/XFS)

Tạo btrfs:
$ sudo mkfs.btrfs /dev/sdb1
$ sudo btrfs subvolume create /mnt/@home         # Create subvolume
$ sudo btrfs subvolume snapshot /mnt/@home /mnt/@home-snap  # Snapshot
$ sudo btrfs filesystem df /mnt                  # Space usage
$ sudo btrfs scrub start /mnt                    # Verify checksums
```

### 6.4 ZFS

**Ví dụ đời thường:** ZFS như xe bọc thép — bảo vệ data tuyệt đối (checksumming mọi thứ), nhiều tính năng, nhưng nặng nề (RAM hungry) và license phức tạp.

```
Đặc điểm:
- Originally Sun Microsystems (Solaris), ported to Linux (OpenZFS)
- Combined volume manager + filesystem (no need LVM)
- End-to-end checksumming (EVERY block verified)
- Self-healing with redundancy (detect + auto-correct corruption)
- Snapshots + Clones (instant, space-efficient)
- Built-in RAID (RAID-Z1, Z2, Z3)
- Compression (lz4, zstd)
- Deduplication (nhưng EXTREMELY RAM hungry)
- Copy-on-Write
- 128-bit addressing (effectively unlimited capacity)

Ưu điểm:
✓ DATA INTEGRITY #1 priority (checksums + self-heal)
✓ Combined volume manager (zpools = PV+VG+LV in one)
✓ Enterprise-grade features
✓ Excellent for NAS/storage servers
✓ Mature + battle-tested (20+ years)

Nhược điểm:
✗ CDDL license (incompatible with GPL → not in Linux kernel!)
✗ Must install as kernel module (DKMS) or use user-space (FUSE)
✗ RAM hungry: needs 1GB RAM per 1TB storage minimum
✗ Dedup needs 5GB RAM per 1TB (!)
✗ Cannot shrink vdev (only grow)
✗ No native encryption until OpenZFS 0.8+

Commands:
$ sudo zpool create tank mirror /dev/sdb /dev/sdc  # Create mirrored pool
$ sudo zfs create tank/data                         # Create dataset
$ sudo zfs snapshot tank/data@today                 # Snapshot
$ sudo zfs send tank/data@today | ssh remote zfs receive backup/data
$ sudo zpool status                                 # Health check
$ sudo zpool scrub tank                             # Verify all checksums
```

### 6.5 So Sánh Filesystems

| Feature | ext4 | XFS | Btrfs | ZFS |
|---------|------|-----|-------|-----|
| Maturity | ★★★★★ | ★★★★★ | ★★★☆☆ | ★★★★★ |
| Max file size | 16TB | 8EB | 16EB | 16EB |
| Snapshots | ❌ | ❌ | ✅ | ✅ |
| Checksumming | Meta only | Meta only | ✅ All | ✅ All |
| Compression | ❌ | ❌ | ✅ | ✅ |
| CoW | ❌ | ❌ | ✅ | ✅ |
| Shrink | ✅ | ❌ | ✅ | ❌ |
| RAM usage | Low | Low | Medium | HIGH |
| Best for | General/OS | Large files/DB | Desktop/NAS | Enterprise NAS |
| Default in | Ubuntu/Debian | RHEL/Rocky | openSUSE | FreeBSD |

---

## 7. Mount Points — Điểm Gắn Kết

### 7.1 Mount Là Gì?

**Ví dụ đời thường:** Mount giống như cắm USB vào máy tính — bạn "gắn" một thiết bị lưu trữ vào một thư mục trong hệ thống file tree. Thư mục đó trở thành "cửa vào" thiết bị.

```bash
# Mount filesystem
sudo mount /dev/sdb1 /mnt/data
# Bây giờ /mnt/data → nội dung của partition /dev/sdb1

# Xem tất cả mounts hiện tại
mount
findmnt              # Tree view (dễ đọc hơn)
cat /proc/mounts     # Kernel view

# Mount với options
sudo mount -t ext4 -o noatime,noexec /dev/sdb1 /mnt/data

# Unmount
sudo umount /mnt/data
# Nếu busy: sudo umount -l /mnt/data (lazy unmount)
```

### 7.2 /etc/fstab — Auto Mount On Boot

```bash
# /etc/fstab format:
# <device>          <mount point>  <type>  <options>        <dump> <pass>
/dev/sda1           /              ext4    defaults         0      1
/dev/sda2           /home          ext4    defaults,noatime 0      2
UUID=abc-123        /data          xfs     defaults         0      2
/dev/sda3           none           swap    sw               0      0
tmpfs               /tmp           tmpfs   size=2G,noexec   0      0

# UUID (recommended over /dev/sdX vì device names có thể thay đổi)
blkid /dev/sdb1    # Xem UUID

# Test fstab trước khi reboot (tránh lỗi boot)
sudo mount -a      # Mount tất cả entries trong fstab
```

### 7.3 Mount Options Quan Trọng

```
defaults    = rw, suid, dev, exec, auto, nouser, async
noatime     = Không update access time (TĂNG performance đáng kể!)
nodiratime  = Không update directory access time
noexec      = Không cho phép execute binaries (security)
nosuid      = Không honor SUID/SGID bits (security)
nodev       = Không cho phép device files (security)
ro          = Read-only
sync        = Synchronous I/O (chậm nhưng safe)
discard     = Enable TRIM cho SSD
quota       = Enable disk quotas
```

---

## 8. LVM (Logical Volume Manager) — Quản Lý Volume Linh Hoạt

### 8.1 LVM Là Gì?

**Ví dụ đời thường:** Bình thường, mỗi partition cố định kích thước (như phòng có tường cứng — muốn mở rộng phải đập tường). **LVM giống như vách ngăn di động** — muốn phòng to hơn? Kéo vách qua là xong. Muốn thêm diện tích? Mua thêm đất (thêm ổ cứng) và nối vào.

```
Không LVM:
┌────────────────────────────────────────┐
│ /dev/sda1 (50GB, ext4, /) │ /dev/sda2 (50GB, ext4, /home) │
└────────────────────────────────────────┘
Vấn đề: / hết dung lượng, /home còn thừa → KHÔNG chuyển được!

Có LVM:
Physical Volumes     Volume Group          Logical Volumes
┌──────────┐        ┌──────────────┐      ┌───────────────┐
│/dev/sda1 │───┐    │              │  ┌───│ LV-root (70GB)│→ /
│  (50GB)  │   ├───→│   VG-main    │──┤   └───────────────┘
└──────────┘   │    │   (150GB)    │  │   ┌───────────────┐
┌──────────┐   │    │              │  └───│ LV-home (80GB)│→ /home
│/dev/sdb1 │───┘    └──────────────┘      └───────────────┘
│ (100GB)  │
└──────────┘
Ưu điểm: resize LV-root/LV-home thoải mái, thêm disk dễ dàng!
```

### 8.2 LVM Concepts: PV → VG → LV

```
PV (Physical Volume): Ổ cứng/partition vật lý
↓ gộp lại thành
VG (Volume Group): "Pool" chung dung lượng
↓ cắt ra thành
LV (Logical Volume): "Partition ảo" linh hoạt

Analogy:
PV = Các miếng đất riêng lẻ bạn mua
VG = Gộp tất cả đất thành 1 khu vực lớn
LV = Cắt khu vực lớn thành từng lô (nhà ở, vườn, garage)
    → Muốn nhà lớn hơn? Thu nhỏ vườn, mở rộng nhà (resize LV)!
```

### 8.3 LVM Commands

```bash
# === Physical Volumes (PV) ===
# Tạo PV từ partition/disk
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc

# Xem PVs
sudo pvs           # Summary
sudo pvdisplay     # Detailed

# === Volume Groups (VG) ===
# Tạo VG từ PVs
sudo vgcreate myvg /dev/sdb1 /dev/sdc

# Thêm PV vào VG (mở rộng pool!)
sudo vgextend myvg /dev/sdd1

# Xem VGs
sudo vgs           # Summary
sudo vgdisplay     # Detailed

# === Logical Volumes (LV) ===
# Tạo LV
sudo lvcreate -n lv_data -L 50G myvg       # Fixed size 50GB
sudo lvcreate -n lv_home -l 100%FREE myvg  # Tất cả space còn lại

# Xem LVs
sudo lvs           # Summary
sudo lvdisplay     # Detailed

# Format + Mount
sudo mkfs.ext4 /dev/myvg/lv_data
sudo mount /dev/myvg/lv_data /data

# === RESIZE (Power feature!) ===
# Grow LV + filesystem (online, không cần unmount!)
sudo lvextend -L +20G /dev/myvg/lv_data    # Thêm 20GB
sudo resize2fs /dev/myvg/lv_data            # Grow ext4 filesystem
# Hoặc 1 lệnh:
sudo lvextend -r -L +20G /dev/myvg/lv_data  # -r = auto resize FS

# Shrink LV (CẦN unmount trước! NGUY HIỂM - backup first!)
sudo umount /data
sudo e2fsck -f /dev/myvg/lv_data            # Check filesystem
sudo resize2fs /dev/myvg/lv_data 30G        # Shrink FS to 30G
sudo lvreduce -L 30G /dev/myvg/lv_data      # Shrink LV to 30G
sudo mount /dev/myvg/lv_data /data
```

### 8.4 LVM Snapshots

```bash
# Snapshot = "ảnh chụp" của LV tại 1 thời điểm
# CoW: Chỉ lưu THAY ĐỔI so với original → space efficient

# Tạo snapshot (cần free space trong VG)
sudo lvcreate -s -n lv_data_snap -L 5G /dev/myvg/lv_data
#                                   ↑ Space cho changes

# Mount snapshot (read-only backup)
sudo mount -o ro /dev/myvg/lv_data_snap /mnt/snap

# Rollback: Merge snapshot vào original (REVERT changes!)
sudo lvconvert --merge /dev/myvg/lv_data_snap
# LV-data quay lại trạng thái lúc tạo snapshot!

# Use cases:
# - Backup trước khi upgrade
# - Test changes, rollback nếu fail
# - Consistent backup (snapshot → backup snapshot → delete)
```

### 8.5 LVM Thin Provisioning

```bash
# Thin provisioning: Allocate VIRTUAL size lớn hơn PHYSICAL size
# Giống như overbooking hàng không: bán 110 vé cho máy bay 100 chỗ

# Tạo thin pool
sudo lvcreate --type thin-pool -n thin_pool -L 100G myvg

# Tạo thin volumes (có thể tổng > 100G!)
sudo lvcreate --type thin -n vm1 -V 50G --thinpool thin_pool myvg
sudo lvcreate --type thin -n vm2 -V 50G --thinpool thin_pool myvg
sudo lvcreate --type thin -n vm3 -V 50G --thinpool thin_pool myvg
# Tổng virtual: 150G, physical: chỉ 100G!
# Works because VMs thường KHÔNG dùng hết allocated space

# Monitor actual usage
sudo lvs -a
# vm1: Virtual 50G, Actual used: 15G
# vm2: Virtual 50G, Actual used: 20G
# vm3: Virtual 50G, Actual used: 10G
# Total actual: 45G (dư 55G cho growth)
```

---

## 9. Practical Scenarios

### 9.1 Scenario: Disk Full — Ổ Cứng Đầy

```bash
# Bước 1: Xác nhận disk nào đầy
df -h
# /dev/sda1      50G   47G  3G  95%  /

# Bước 2: Tìm thủ phạm (directory chiếm nhiều nhất)
du -sh /* | sort -rh | head -10
# 30G    /var
# 10G    /home
# 5G     /usr

du -sh /var/* | sort -rh | head -5
# 25G    /var/log    ← Thủ phạm!

# Bước 3: Tìm files lớn
find /var/log -type f -size +100M -exec ls -lh {} \;
# -rw-r--r-- 15G /var/log/syslog.1    ← File log 15GB!

# Bước 4: Giải pháp
# a) Xóa/truncate log cũ
sudo truncate -s 0 /var/log/syslog.1   # Không xóa file, chỉ clear content
sudo journalctl --vacuum-size=500M      # Giới hạn journal 500MB

# b) Nếu dùng LVM → extend
sudo lvextend -r -L +10G /dev/myvg/lv_root

# c) Setup logrotate proper (ngăn tái phát)
```

### 9.2 Scenario: Thêm Ổ Cứng Mới

```bash
# Ổ cứng mới: /dev/sdc (500GB)

# Bước 1: Partition (GPT)
sudo parted /dev/sdc mklabel gpt
sudo parted /dev/sdc mkpart primary 0% 100%

# Bước 2a: Format trực tiếp (không LVM)
sudo mkfs.xfs /dev/sdc1
sudo mkdir /data
sudo mount /dev/sdc1 /data
# Thêm vào fstab
echo "UUID=$(blkid -s UUID -o value /dev/sdc1) /data xfs defaults 0 2" | sudo tee -a /etc/fstab

# Bước 2b: Hoặc thêm vào LVM (recommended)
sudo pvcreate /dev/sdc1
sudo vgextend myvg /dev/sdc1          # Thêm vào existing VG
sudo lvextend -r -l +100%FREE /dev/myvg/lv_data  # Expand LV
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Filesystem Selection Guide

```
General purpose (OS, apps): ext4
Large files, high throughput: XFS  
Need snapshots + compression: Btrfs
Maximum data integrity (NAS): ZFS
Enterprise Linux default: XFS (RHEL) / ext4 (Ubuntu)
```

### 10.2 Key Takeaways

1. **Inode = metadata của file** (permissions, size, block locations) — filename KHÔNG nằm trong inode
2. **Hard link = thêm tên cho cùng inode**, Soft link = file mới trỏ đến path
3. **VFS** cho phép Linux dùng cùng commands cho mọi filesystem
4. **ext4** stable + proven cho general use; **XFS** cho large files; **Btrfs/ZFS** cho advanced features
5. **LVM = layer trừu tượng** giữa physical disks và filesystems → resize dễ dàng
6. **PV → VG → LV**: Physical disks gộp thành pool, cắt ra thành volumes linh hoạt
7. **/proc** và **/sys** là virtual filesystems — kernel generate on-the-fly
8. **noatime** mount option = free performance boost

### 10.3 Tài Liệu Tham Khảo

- Linux kernel documentation: Documentation/filesystems/
- man pages: ext4(5), xfs(5), btrfs(5), mount(8), fstab(5), lvm(8)
- "Understanding the Linux Kernel" by Bovet & Cesati — VFS chapter
- "Linux System Programming" by Robert Love — filesystem chapter
- Red Hat Storage Administration Guide
- Arch Wiki: File Systems, LVM
- OpenZFS documentation: https://openzfs.github.io/openzfs-docs/
- Btrfs Wiki: https://btrfs.wiki.kernel.org/

---

*Bài viết tiếp theo: File Permissions Advanced — SUID, SGID, Sticky Bit, ACLs*
