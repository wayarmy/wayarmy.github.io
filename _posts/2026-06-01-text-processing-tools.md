---
layout: post
title: "Text Processing Tools - Công cụ xử lý văn bản dòng lệnh"
date: 2026-06-01
categories: [linux]
tags: [sed, awk, jq, xargs, parallel, text-processing]
---

## Mục lục
1. [Góc nhìn tổng quan - Dây chuyền sản xuất văn bản](#goc-nhin-tong-quan)
2. [sed - Stream Editor: Thợ sửa văn bản tự động](#sed-stream-editor)
3. [sed nâng cao: Hold Space, Pattern Space và Branching](#sed-nang-cao)
4. [awk - Ngôn ngữ xử lý dữ liệu có cấu trúc](#awk)
5. [awk nâng cao: Mảng, hàm và pattern matching](#awk-nang-cao)
6. [jq - Xử lý JSON từ dòng lệnh](#jq)
7. [xargs - Chuyển đổi input thành arguments](#xargs)
8. [GNU Parallel - Xử lý song song](#gnu-parallel)
9. [Kết hợp các công cụ trong pipeline](#ket-hop)
10. [Tổng kết và tài liệu tham khảo](#tong-ket)

---

## 1. Góc nhìn tổng quan - Dây chuyền sản xuất văn bản {#goc-nhin-tong-quan}

### Ví dụ đời thường

Hãy tưởng tượng bạn làm việc trong một **nhà máy đóng gói thư** (mail processing center):

- **sed** giống như một **máy dán nhãn tự động** - thư đi qua băng chuyền, máy đọc địa chỉ cũ, dán đè địa chỉ mới lên. Mỗi phong bì đi qua một lượt, xong thì ra khỏi máy
- **awk** giống như một **nhân viên phân loại thông minh** - đọc từng dòng trên phong bì, tách thành các trường (tên, địa chỉ, mã vùng), rồi quyết định: gửi đi, bỏ, hay ghi vào sổ thống kê
- **jq** giống như một **chuyên gia mở hộp lồng** (Russian nesting dolls) - dữ liệu JSON giống hộp trong hộp, jq biết cách mở đúng lớp bạn cần
- **xargs** giống như **người chia việc** - nhận danh sách tên file, rồi đưa từng cái (hoặc nhóm) cho lệnh tiếp theo xử lý
- **GNU parallel** giống như **quản đốc phân xưởng** - thay vì 1 người làm 100 việc lần lượt, chia cho 8 người làm song song

### Tại sao cần học những công cụ này?

```
Tình huống thực tế:
- Bạn có 50GB log files, cần tìm tất cả IP truy cập thất bại
- Bạn nhận 10,000 file JSON từ API, cần trích xuất 3 trường
- Bạn cần đổi tên 5,000 file theo pattern mới
- Bạn cần xử lý CSV 1 triệu dòng mà Excel không mở nổi

Các công cụ text processing giải quyết trong vài giây
thay vì viết script phức tạp hoặc mở từng file thủ công.
```

### Triết lý Unix Pipeline

```
Mỗi chương trình làm MỘT việc tốt.
Các chương trình nối với nhau qua pipe (|).
Text là giao diện chung (universal interface).
```

Ví dụ pipeline hoàn chỉnh:
```bash
# Tìm 10 IP đăng nhập thất bại nhiều nhất từ auth log
grep "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | \
  sort | uniq -c | sort -rn | head -10
```

---

## 2. sed - Stream Editor: Thợ sửa văn bản tự động {#sed-stream-editor}

### sed là gì?

**sed** (Stream Editor - Trình soạn thảo luồng) là công cụ xử lý text theo từng dòng. Nó đọc input line by line, áp dụng các lệnh biến đổi, rồi xuất kết quả.

Tài liệu gốc: GNU sed manual, POSIX sed specification.

### Mô hình hoạt động - Băng chuyền nhà máy

```
Input (stdin/file)
    │
    ▼
┌─────────────┐
│ Đọc 1 dòng  │ ──► Pattern Space (vùng làm việc)
└─────────────┘
    │
    ▼
┌─────────────┐
│ Áp dụng     │  Lệnh 1 → Lệnh 2 → Lệnh 3 ...
│ các lệnh    │
└─────────────┘
    │
    ▼
┌─────────────┐
│ In kết quả  │ ──► stdout
└─────────────┘
    │
    ▼
  Lặp lại cho dòng tiếp theo
```

### Cú pháp cơ bản

```bash
sed [OPTIONS] 'COMMAND' file

# OPTIONS phổ biến:
# -n    : Không tự động in (suppress default print)
# -i    : Sửa file tại chỗ (in-place editing)
# -E/-r : Extended regex (không cần escape +, ?, |)
# -e    : Nhiều lệnh (multiple commands)
```

### Substitution - Lệnh thay thế (s///)

Đây là lệnh sed được dùng nhiều nhất:

```bash
# Cú pháp: s/pattern/replacement/flags
# s = substitute, giống Find & Replace trong Word

# Thay thế lần xuất hiện đầu tiên mỗi dòng
sed 's/old/new/' file.txt

# Thay thế TẤT CẢ (g = global)
sed 's/old/new/g' file.txt

# Thay thế lần thứ 2 trở đi
sed 's/old/new/2' file.txt

# Case-insensitive (I flag)
sed 's/hello/Hi/gI' file.txt
```

### Addressing - Chỉ định dòng cần xử lý

```bash
# Dòng cụ thể
sed '5s/old/new/' file.txt        # Chỉ dòng 5
sed '1,10s/old/new/' file.txt     # Dòng 1 đến 10
sed '5,$s/old/new/' file.txt      # Dòng 5 đến cuối file

# Theo pattern (regex)
sed '/ERROR/s/old/new/' file.txt  # Chỉ dòng chứa "ERROR"
sed '/^#/d' file.txt              # Xóa dòng bắt đầu bằng #

# Kết hợp pattern range
sed '/START/,/END/s/old/new/' file.txt  # Từ dòng START đến END

# Bước nhảy (step)
sed '1~2s/old/new/' file.txt      # Dòng lẻ: 1, 3, 5, 7...
sed '2~2s/old/new/' file.txt      # Dòng chẵn: 2, 4, 6, 8...
```

### Các lệnh sed quan trọng

```bash
# d - Delete (xóa dòng)
sed '/^$/d' file.txt              # Xóa dòng trống
sed '1,5d' file.txt               # Xóa 5 dòng đầu

# p - Print (in, thường dùng với -n)
sed -n '/ERROR/p' file.txt        # Chỉ in dòng có ERROR (giống grep)

# a - Append (thêm dòng phía sau)
sed '/pattern/a\New line after' file.txt

# i - Insert (thêm dòng phía trước)
sed '1i\# Header line' file.txt

# c - Change (thay thế cả dòng)
sed '/deprecated/c\# This line was removed' file.txt

# y - Transliterate (thay từng ký tự, giống tr)
sed 'y/abc/ABC/' file.txt         # a→A, b→B, c→C
```

### Ví dụ thực tế

```bash
# 1. Xóa comment và dòng trống từ config file
sed '/^#/d; /^$/d' nginx.conf

# 2. Thêm prefix vào mỗi dòng
sed 's/^/[LOG] /' output.txt

# 3. Trích xuất text giữa 2 markers
sed -n '/BEGIN/,/END/p' file.txt

# 4. Sửa file in-place (backup .bak)
sed -i.bak 's/localhost/192.168.1.100/g' config.ini

# 5. Đổi line ending Windows → Unix
sed -i 's/\r$//' file.txt

# 6. Thay thế với capture group (nhóm bắt)
# Đổi "2024-01-15" thành "15/01/2024"
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/g' dates.txt
```

---

## 3. sed nâng cao: Hold Space, Pattern Space và Branching {#sed-nang-cao}

### Pattern Space và Hold Space

Đây là khái niệm quan trọng nhất để hiểu sed nâng cao:

```
┌─────────────────────────────────────────┐
│              sed Engine                   │
│                                          │
│  ┌──────────────────┐                   │
│  │  Pattern Space   │ ← Bàn làm việc   │
│  │  (working area)  │   chính           │
│  └──────────────────┘                   │
│           ↕  (x, h, H, g, G)           │
│  ┌──────────────────┐                   │
│  │  Hold Space      │ ← Ngăn kéo lưu   │
│  │  (storage)       │   trữ tạm        │
│  └──────────────────┘                   │
│                                          │
└─────────────────────────────────────────┘

Ví dụ đời thường:
- Pattern Space = bàn làm việc (đang xử lý)
- Hold Space = ngăn kéo bàn (cất tạm để dùng sau)
```

### Các lệnh điều khiển Hold Space

```bash
# h - Copy pattern space → hold space (ghi đè)
# H - Append pattern space → hold space (nối thêm)
# g - Copy hold space → pattern space (ghi đè)
# G - Append hold space → pattern space (nối thêm)
# x - Exchange (hoán đổi pattern ↔ hold)

# Ví dụ: Đảo ngược thứ tự dòng (giống tac)
sed -n '1!G; h; $p' file.txt

# Giải thích từng bước:
# 1!G  : Từ dòng 2 trở đi, append hold space vào pattern space
# h    : Copy pattern space vào hold space
# $p   : Ở dòng cuối, in pattern space
```

### Ví dụ Hold Space thực tế

```bash
# Nối 2 dòng liên tiếp thành 1 (join pairs)
sed 'N; s/\n/ /' file.txt

# In dòng trước dòng chứa pattern (context before match)
sed -n '/ERROR/{x;p;x;p}; h' logfile.txt
# Giải thích:
# h        : Mỗi dòng, cất vào hold space
# /ERROR/  : Khi gặp ERROR:
#   x      : Lấy dòng trước từ hold (hoán đổi)
#   p      : In dòng trước
#   x      : Hoán đổi lại (ERROR quay về pattern)
#   p      : In dòng ERROR

# Xóa dòng trùng lặp liên tiếp (giống uniq)
sed '$!N; /^\(.*\)\n\1$/!P; D' file.txt
```

### Branching - Điều khiển luồng

```bash
# :label  - Đặt nhãn
# b label - Nhảy đến nhãn (unconditional branch)
# t label - Nhảy nếu substitution thành công (conditional)
# T label - Nhảy nếu substitution KHÔNG thành công (GNU ext)

# Ví dụ: Xóa trailing whitespace và dòng trống liên tiếp
sed '/^$/{ 
  N
  /^\n$/d
}' file.txt

# Ví dụ: Nối continuation lines (dòng kết thúc bằng \)
sed ':a; /\\$/{N; s/\\\n//; ba}' file.txt
# Giải thích:
# :a        - Đặt nhãn 'a'
# /\\$/   - Nếu dòng kết thúc bằng \
# N         - Đọc thêm dòng tiếp theo
# s/\\\n// - Xóa \ và newline
# ba        - Nhảy lại nhãn 'a' (lặp nếu vẫn có \)
```

### Multi-line processing

```bash
# N - Append next line vào pattern space
# P - Print đến newline đầu tiên
# D - Delete đến newline đầu tiên, restart cycle

# Thay thế chuỗi nằm trên nhiều dòng
sed ':a; N; $!ba; s/foo\nbar/baz/' file.txt
# Đọc toàn bộ file vào pattern space, rồi thay thế

# Xóa block giữa 2 markers
sed '/START/,/END/d' file.txt

# Thay thế nội dung block
sed '/START/,/END/{
  /START/!{/END/!d}
  /START/a\  new content here
}' file.txt
```

---

## 4. awk - Ngôn ngữ xử lý dữ liệu có cấu trúc {#awk}

### awk là gì?

**awk** (Aho, Weinberger, Kernighan - tên 3 tác giả) là một ngôn ngữ lập trình chuyên xử lý text có cấu trúc (structured text). Nó mạnh hơn sed rất nhiều vì có biến, mảng, hàm, và control flow.

Tài liệu gốc: POSIX awk specification, GNU awk (gawk) manual.

### Mô hình hoạt động - Nhân viên kế toán

```
awk giống một nhân viên kế toán xử lý hóa đơn:

1. Lấy 1 hóa đơn (record/dòng) ra khỏi chồng
2. Tách thành các ô (fields): tên, số lượng, đơn giá
3. Kiểm tra điều kiện: "Nếu đơn giá > 100..."
4. Thực hiện hành động: "...thì ghi vào sổ cảnh báo"
5. Lặp lại cho hóa đơn tiếp theo

Công thức awk:
   pattern { action }
   (điều kiện) { hành động }
```

### Cú pháp cơ bản

```bash
awk 'pattern { action }' file

# Ví dụ đơn giản
echo "John 85 92 78" | awk '{print $1, ($2+$3+$4)/3}'
# Output: John 85

# Biến đặc biệt:
# $0     - Toàn bộ dòng hiện tại
# $1..$n - Field thứ n
# NR     - Số thứ tự dòng (Number of Record)
# NF     - Số field trong dòng (Number of Fields)
# FS     - Field Separator (mặc định: whitespace)
# RS     - Record Separator (mặc định: newline)
# OFS    - Output Field Separator
# ORS    - Output Record Separator
```

### Field splitting - Tách trường

```bash
# Mặc định: tách bởi whitespace (space, tab)
awk '{print $1}' file.txt

# Tách bằng ký tự khác
awk -F':' '{print $1, $3}' /etc/passwd  # user và UID
awk -F',' '{print $2}' data.csv          # CSV column 2

# Tách bằng regex
awk -F'[,;:]' '{print $1}' mixed.txt

# Đặt FS trong BEGIN block
awk 'BEGIN{FS=","} {print $1}' data.csv
```

### Patterns - Điều kiện lọc

```bash
# Regex pattern
awk '/ERROR/' logfile.txt        # Dòng chứa ERROR
awk '!/^#/' config.txt           # Dòng KHÔNG bắt đầu bằng #

# Comparison
awk '$3 > 100' sales.txt         # Field 3 > 100
awk '$1 == "John"' scores.txt    # Field 1 là "John"
awk 'NR > 1' file.txt            # Bỏ header (từ dòng 2)

# Range pattern
awk '/START/,/END/' file.txt     # Từ START đến END

# Special patterns
awk 'BEGIN { print "Header" }'   # Chạy trước khi đọc input
awk 'END { print NR " lines" }' file.txt  # Chạy sau khi đọc hết
```

### Actions - Hành động

```bash
# Print formatting
awk '{printf "%-10s %5d\n", $1, $2}' file.txt

# Tính toán
awk '{sum += $3} END {print "Total:", sum}' sales.txt
awk '{sum += $3; count++} END {print "Avg:", sum/count}' data.txt

# Điều kiện if/else
awk '{
  if ($3 >= 90) grade = "A"
  else if ($3 >= 80) grade = "B"
  else grade = "C"
  print $1, grade
}' scores.txt

# Loops
awk '{
  for (i = 1; i <= NF; i++)
    if ($i ~ /[0-9]+/) sum += $i
  print sum; sum = 0
}' numbers.txt
```

---

## 5. awk nâng cao: Mảng, hàm và pattern matching {#awk-nang-cao}

### Associative Arrays (Mảng kết hợp)

```bash
# awk arrays giống dictionary/hashmap trong Python/Java
# Key có thể là string bất kỳ

# Đếm số lần xuất hiện mỗi giá trị (frequency count)
awk '{count[$1]++} END {for (k in count) print k, count[k]}' access.log

# Nhóm và tính tổng theo category
awk -F',' '{
  category[$2] += $3
} END {
  for (cat in category)
    printf "%s: $%.2f\n", cat, category[cat]
}' sales.csv

# Multi-dimensional arrays (mô phỏng)
awk '{
  data[$1][$2] = $3    # gawk extension
  # Hoặc: data[$1 SUBSEP $2] = $3  (POSIX)
}' matrix.txt
```

### User-defined Functions

```bash
# Định nghĩa hàm
awk '
function max(a, b) {
  return (a > b) ? a : b
}
function trim(s) {
  gsub(/^[ \t]+|[ \t]+$/, "", s)
  return s
}
{
  print max($2, $3), trim($4)
}' data.txt
```

### String Functions

```bash
# length(s)   - Độ dài chuỗi
# substr(s, start, len) - Cắt chuỗi
# index(s, target)  - Vị trí substring
# split(s, array, separator) - Tách vào mảng
# sub(regex, replacement, target) - Thay thế lần đầu
# gsub(regex, replacement, target) - Thay thế tất cả
# match(s, regex) - Tìm regex, set RSTART, RLENGTH
# sprintf(format, ...) - Format string
# tolower(s), toupper(s) - Chuyển case

# Ví dụ: Parse log entry
awk '{
  # "2024-01-15 10:30:45 ERROR [auth] Login failed for user=admin"
  split($4, module, /[\[\]]/)
  match($0, /user=([^ ]+)/, arr)  # gawk ONLY
  printf "%s %s %s\n", $1, module[2], arr[1]
}' app.log
```

### Ví dụ awk thực tế phức tạp

```bash
# 1. Tạo báo cáo từ CSV (pivot table đơn giản)
awk -F',' 'NR > 1 {
  region[$2] += $4
  count[$2]++
} END {
  printf "%-15s %10s %10s\n", "Region", "Total", "Average"
  printf "%-15s %10s %10s\n", "------", "-----", "-------"
  for (r in region)
    printf "%-15s %10.2f %10.2f\n", r, region[r], region[r]/count[r]
}' sales_report.csv

# 2. Phân tích Apache access log - Top URLs theo status code
awk '{
  status = $9
  url = $7
  if (status == "404") missing[url]++
  if (status == "500") errors[url]++
} END {
  print "=== 404 Not Found ==="
  for (u in missing) print missing[u], u | "sort -rn | head -5"
  print "\n=== 500 Errors ==="
  for (u in errors) print errors[u], u | "sort -rn | head -5"
}' access.log

# 3. Transpose columns và rows
awk '{
  for (i = 1; i <= NF; i++) {
    a[NR][i] = $i
  }
}
NF > max_nf { max_nf = NF }
END {
  for (j = 1; j <= max_nf; j++) {
    for (i = 1; i <= NR; i++)
      printf "%s%s", a[i][j], (i < NR ? "\t" : "\n")
  }
}' matrix.txt
```

---

## 6. jq - Xử lý JSON từ dòng lệnh {#jq}

### jq là gì?

**jq** là bộ lọc và biến đổi JSON từ command line. Nó giống "sed cho JSON" - nhận JSON input, áp dụng filter, xuất JSON (hoặc text) output.

Tài liệu gốc: [jq manual](https://jqlang.github.io/jq/manual/)

### Tại sao cần jq?

```
API trả về JSON → cần trích xuất data
Config files (package.json, terraform) → cần đọc/sửa
Log structured (JSON lines) → cần filter và aggregate

Không có jq, bạn phải viết Python script cho mỗi tác vụ.
Với jq, 1 dòng lệnh là xong.
```

### Cú pháp cơ bản

```bash
# Đọc từ file
jq '.' data.json              # Pretty print
jq '.name' data.json          # Lấy field "name"
jq '.users[0]' data.json      # Array element đầu tiên
jq '.users[].name' data.json  # Tất cả name trong array

# Đọc từ pipe
curl -s api.example.com/users | jq '.[].email'
```

### Filters cơ bản

```bash
# Object access
echo '{"name":"John","age":30}' | jq '.name'
# Output: "John"

# Array access
echo '[1,2,3,4,5]' | jq '.[2]'      # 3
echo '[1,2,3,4,5]' | jq '.[2:4]'    # [3,4]
echo '[1,2,3,4,5]' | jq '.[-1]'     # 5 (cuối)

# Nested access
echo '{"a":{"b":{"c":42}}}' | jq '.a.b.c'   # 42

# Pipe trong jq (giống Unix pipe)
echo '{"users":[{"name":"A","age":25},{"name":"B","age":30}]}' | \
  jq '.users[] | .name'
# Output: "A" \n "B"

# Raw output (bỏ dấu ngoặc kép)
jq -r '.name' data.json       # John (không có quotes)
```

### Filters nâng cao

```bash
# select - Lọc theo điều kiện
jq '.users[] | select(.age > 25)' data.json
jq '.[] | select(.status == "active")' items.json

# map - Transform mỗi element
jq '.users | map(.name)' data.json           # Lấy danh sách tên
jq '.items | map(. * 2)' numbers.json        # Nhân đôi mỗi số
jq '.users | map({name, email})' data.json   # Chỉ giữ 2 fields

# Construct new objects
jq '.[] | {id: .user_id, full_name: (.first + " " + .last)}' users.json

# Reduce / Aggregate
jq '[.items[].price] | add' cart.json           # Tổng giá
jq '.users | length' data.json                  # Đếm users
jq '.items | group_by(.category) | map({key: .[0].category, count: length})' items.json

# Conditional
jq '.[] | if .score >= 90 then "A" elif .score >= 80 then "B" else "C" end' scores.json

# String interpolation
jq -r '.users[] | "\(.name) (\(.email))"' data.json
```

### jq với JSON Lines (NDJSON)

```bash
# Mỗi dòng là 1 JSON object riêng
# Rất phổ biến trong logs structured

# Filter log entries
cat app.log.json | jq -r 'select(.level == "error") | "\(.timestamp) \(.message)"'

# Aggregate errors per service
cat app.log.json | jq -s 'group_by(.service) | map({service: .[0].service, errors: length})'

# -s (slurp): đọc tất cả lines vào 1 array
# -c (compact): output 1 line per object
```

### Ví dụ thực tế với jq

```bash
# 1. Parse AWS CLI output
aws ec2 describe-instances | jq -r '
  .Reservations[].Instances[] |
  [.InstanceId, .State.Name, (.Tags[]? | select(.Key=="Name") | .Value)] |
  @tsv'

# 2. Merge 2 JSON files
jq -s '.[0] * .[1]' defaults.json overrides.json

# 3. Convert JSON to CSV
jq -r '.users[] | [.name, .email, .age] | @csv' data.json > output.csv

# 4. Update nested value
jq '.database.port = 5433' config.json > config_new.json

# 5. Sửa package.json version
jq '.version = "2.0.0"' package.json | sponge package.json
```

---

## 7. xargs - Chuyển đổi input thành arguments {#xargs}

### xargs là gì?

**xargs** đọc items từ stdin và chuyển chúng thành arguments cho command khác. Nó là cầu nối giữa output của lệnh này và input arguments của lệnh kia.

### Ví dụ đời thường

```
Bạn có danh sách 1000 số điện thoại cần gọi.
Thay vì bấm từng số một (for loop),
xargs giống như đưa cả danh sách cho hệ thống auto-dial.
Nó tự chia thành nhóm và gọi song song.
```

### Cú pháp cơ bản

```bash
# Cơ bản: chuyển stdin thành arguments
echo "file1 file2 file3" | xargs rm
# Tương đương: rm file1 file2 file3

# -I {} : Placeholder cho mỗi item
find . -name "*.log" | xargs -I {} mv {} /archive/

# -n N : Mỗi lần truyền N arguments
echo "1 2 3 4 5 6" | xargs -n 2 echo
# echo 1 2
# echo 3 4
# echo 5 6

# -P N : Chạy N processes song song
find . -name "*.png" | xargs -P 4 -I {} convert {} -resize 50% resized_{}

# -0 : Dùng null byte làm delimiter (an toàn với filename có space)
find . -name "*.tmp" -print0 | xargs -0 rm

# -p : Hỏi confirm trước khi chạy
find . -name "*.bak" | xargs -p rm
```

### Patterns phổ biến

```bash
# 1. Xóa files tìm được
find /tmp -mtime +7 -type f | xargs rm -f

# 2. Grep trong nhiều files
find . -name "*.py" | xargs grep "import os"

# 3. Batch rename
ls *.jpeg | xargs -I {} bash -c 'mv "$1" "${1%.jpeg}.jpg"' _ {}

# 4. Download nhiều URLs
cat urls.txt | xargs -P 5 -I {} curl -sO {}

# 5. Xử lý output từ command khác
docker ps -q | xargs docker stop       # Stop tất cả containers
git branch --merged | grep -v main | xargs git branch -d  # Xóa merged branches
```

---

## 8. GNU Parallel - Xử lý song song {#gnu-parallel}

### GNU Parallel là gì?

**GNU Parallel** chạy commands song song, tận dụng nhiều CPU cores. Nó giống xargs nhưng mạnh hơn nhiều với job control, progress bar, retry logic, và remote execution.

Tài liệu gốc: GNU Parallel manual (Ole Tange, 2011).

### So sánh với xargs -P

```
xargs -P:    Đơn giản, có sẵn, đủ dùng cho tác vụ nhẹ
GNU Parallel: Progress, resume, retry, remote, load balancing
```

### Cú pháp cơ bản

```bash
# Giống xargs nhưng mặc định song song
parallel echo ::: A B C D
# Output (thứ tự có thể khác): A B C D

# Từ file
parallel process_file {} < file_list.txt

# Từ pipe
find . -name "*.mp4" | parallel ffmpeg -i {} -vf scale=1280:720 720p_{}

# Nhiều input sources
parallel convert {1} -resize {2} resized_{1} ::: *.png ::: 50% 25%
```

### Job control và features

```bash
# -j N : Số jobs song song (mặc định: số CPU cores)
parallel -j 8 process {} ::: file*

# --progress : Hiển thị tiến độ
parallel --progress compress {} ::: *.log

# --eta : Estimated time of arrival
find . -name "*.raw" | parallel --eta process_image {}

# --joblog : Ghi log mỗi job
parallel --joblog jobs.log do_work {} ::: items*

# --resume --joblog : Chạy tiếp nếu bị interrupt
parallel --resume --joblog jobs.log do_work {} ::: items*

# --retry-failed --joblog : Retry jobs thất bại
parallel --retry-failed --joblog jobs.log

# --timeout : Giới hạn thời gian mỗi job
parallel --timeout 60 slow_command {} ::: inputs*
```

### Remote execution

```bash
# Chạy trên nhiều servers
parallel --sshlogin server1,server2,server3 process {} ::: data*

# Transfer file → execute → return result
parallel --sshlogin server1 --transferfile {} --return {}.out \
  'process {} > {}.out' ::: input*
```

### Ví dụ thực tế

```bash
# 1. Compress logs song song
find /var/log -name "*.log" -size +100M | \
  parallel -j 4 --progress gzip {}

# 2. Download và process song song
cat urls.txt | parallel -j 10 --colsep '\t' \
  'curl -s {1} | jq .data > results/{2}.json'

# 3. Database migrations trên nhiều databases
parallel -j 3 --joblog migration.log \
  'psql -h {} -f migrate.sql' ::: db1.host db2.host db3.host

# 4. Image processing pipeline
find photos/ -name "*.jpg" | parallel -j $(nproc) '
  convert {} -resize 800x600 -quality 85 optimized/{/}
'

# 5. Security scan nhiều hosts
parallel -j 20 --timeout 30 \
  'nmap -sV -p 80,443 {} 2>/dev/null' :::: hosts.txt > scan_results.txt
```

---

## 9. Kết hợp các công cụ trong pipeline {#ket-hop}

### Pipeline Pattern: Tìm → Lọc → Biến đổi → Tổng hợp

```bash
# Phân tích web server logs: Top 10 user agents gây 404
awk '$9 == 404 {print $0}' access.log | \
  sed 's/.*"\(.*\)"/\1/' | \
  sort | uniq -c | sort -rn | head -10

# API response processing
curl -s "https://api.github.com/repos/torvalds/linux/commits" | \
  jq -r '.[] | "\(.commit.author.date) \(.commit.author.name): \(.commit.message | split("\n")[0])"' | \
  head -20

# Multi-step log analysis
find /var/log -name "*.log" -mtime -1 | \
  xargs grep -l "CRITICAL" | \
  parallel 'echo "=== {} ==="; grep -c CRITICAL {}' | \
  awk -F: '{sum += $NF} END {print "Total CRITICAL:", sum}'
```

### Real-world scripts

```bash
#!/bin/bash
# Script: Phân tích slow queries từ PostgreSQL log

LOG="/var/log/postgresql/postgresql-*.log"

echo "=== Slow Query Report ==="
echo "Generated: $(date)"
echo ""

grep "duration:" $LOG | \
  awk '{
    # Extract duration in ms
    match($0, /duration: ([0-9.]+) ms/, arr)
    duration = arr[1]
    # Extract query (simplified)
    match($0, /statement: (.*)/, query)
    if (duration > 1000) {
      slow[query[1]] += duration
      count[query[1]]++
    }
  } END {
    for (q in slow)
      printf "%8.0f ms (x%d) %s\n", slow[q]/count[q], count[q], substr(q, 1, 80)
  }' | sort -rn | head -20
```

```bash
#!/bin/bash
# Script: Batch process JSON API responses

API_BASE="https://api.example.com"
OUTPUT_DIR="./results"
mkdir -p "$OUTPUT_DIR"

# Lấy danh sách IDs, download song song, extract data
curl -s "$API_BASE/items" | \
  jq -r '.[].id' | \
  parallel -j 10 --progress \
    "curl -s '$API_BASE/items/{}' | jq '{id: .id, name: .name, status: .status}'" | \
  jq -s '.' > "$OUTPUT_DIR/all_items.json"

# Summary
jq -r 'group_by(.status) | .[] | "\(.[0].status): \(length)"' "$OUTPUT_DIR/all_items.json"
```

---

## 10. Tổng kết và tài liệu tham khảo {#tong-ket}

### Bảng so sánh: Khi nào dùng công cụ nào?

| Tác vụ | Công cụ tốt nhất | Lý do |
|--------|------------------|-------|
| Find & Replace đơn giản | sed | Nhanh, 1 dòng lệnh |
| Xử lý multi-line text | sed (hold space) | Có buffer lưu trữ |
| Xử lý dữ liệu có cột | awk | Field splitting native |
| Tính toán, thống kê | awk | Có biến, mảng, math |
| Xử lý JSON | jq | Parser JSON chuyên dụng |
| Chuyển output → args | xargs | Cầu nối pipe → command |
| Batch processing | GNU parallel | Song song, retry, resume |
| Tác vụ phức tạp | Python/Perl | Khi pipeline quá rối |

### Bảng cheat sheet

```bash
# sed essentials
sed 's/old/new/g'           # Replace all
sed -n '/pattern/p'          # Print matches (grep)
sed '/pattern/d'             # Delete matches
sed -i.bak 's/a/b/g' f      # In-place with backup

# awk essentials
awk '{print $1}'             # First column
awk -F',' '{print $2}'      # CSV second column
awk '/pat/{sum+=$3}END{print sum}'  # Conditional sum
awk '!seen[$0]++'            # Remove duplicates

# jq essentials
jq '.'                       # Pretty print
jq '.key'                    # Access field
jq '.[] | .name'            # Array iteration
jq -r 'select(.x>5)'       # Filter

# xargs essentials
find . -name "*.x" | xargs cmd
cmd | xargs -I {} other_cmd {}
cmd | xargs -P4 -n1 process
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| GNU sed manual | Tài liệu chính thức cho sed |
| The AWK Programming Language (Aho et al.) | Sách gốc từ tác giả awk |
| GAWK: Effective AWK Programming | GNU awk manual đầy đủ |
| jq Manual (jqlang.github.io) | Tham khảo đầy đủ tất cả filters |
| GNU Parallel Tutorial (Ole Tange) | Hướng dẫn chính thức |
| POSIX sed specification | Chuẩn POSIX cho portable scripts |
| sed & awk (O'Reilly, Dale Dougherty) | Sách kinh điển |

### Bài tập thực hành

1. **sed**: Viết script chuyển đổi Markdown headers (## → <h2>) 
2. **awk**: Parse Apache log, tính bandwidth per IP
3. **jq**: Từ output `docker inspect`, trích xuất network info
4. **Kết hợp**: Pipeline phân tích git log → top contributors per month

---

*Bài viết tiếp theo: [Linux Performance Tuning](/2026/08/08/linux-performance-tuning/) - Tối ưu hiệu năng hệ thống Linux*
