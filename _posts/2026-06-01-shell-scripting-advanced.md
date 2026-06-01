---
layout: post
title: "Shell Scripting Advanced Deep Dive - trap, getopts, Process Substitution"
date: 2026-06-01
categories: [linux]
tags: [bash, shell-scripting, automation, debugging]
---

# Shell Scripting Advanced Deep Dive - trap, getopts, Process Substitution

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Bạn đã biết shell scripting cơ bản (if, for, while, functions). Giờ hãy học các kỹ thuật nâng cao:

- **trap** = "Kế hoạch B" — nếu script bị gián đoạn (Ctrl+C, error), vẫn dọn dẹp sạch sẽ trước khi thoát. Giống như hệ thống tự động tắt bếp gas khi phát hiện rò rỉ.
- **getopts** = "Menu nhà hàng" — xử lý options/flags mà user truyền vào script (`./script -f file -v`). Giống như quán phở có menu: bạn chọn -t (tái), -c (chín), -n (nạm).
- **Here-docs** = "Template email" — tạo khối text lớn trong script mà không cần echo từng dòng.
- **Process substitution** = "Đường ống ngầm" — chuyển output của command thành file ảo để command khác đọc.

---

## 2. trap — Xử Lý Signals và Cleanup

### 2.1 trap Là Gì?

**Ví dụ đời thường:** Bạn đang nấu ăn. Nếu đột ngột phải ra ngoài:
- **Không có trap:** Bếp gas vẫn cháy → cháy nhà!
- **Có trap:** Hệ thống tự động tắt gas khi bạn ra khỏi bếp → an toàn!

```bash
# trap command — "Bẫy" signal và thực hiện hành động
# Syntax: trap 'commands' SIGNAL_LIST

#!/bin/bash
# Tạo temp file
TMPFILE=$(mktemp)

# trap: Nếu script bị interrupt → xóa temp file
trap 'echo "Cleaning up..."; rm -f "$TMPFILE"; exit 1' INT TERM EXIT

# Script logic
echo "Working with $TMPFILE..."
echo "some data" > "$TMPFILE"
sleep 30    # Giả sử đang xử lý lâu

# Nếu user nhấn Ctrl+C ở đây:
# → trap catches SIGINT
# → Runs cleanup commands
# → Temp file gets deleted
# → Script exits cleanly!
```

### 2.2 Common trap Patterns

```bash
# Pattern 1: Cleanup on exit (PHỔBIẾN NHẤT)
cleanup() {
    echo "Cleaning up..."
    rm -rf "$WORK_DIR"
    [ -n "$PID" ] && kill "$PID" 2>/dev/null  # Kill background process
}
trap cleanup EXIT  # EXIT = luôn chạy khi script kết thúc (bất kể lý do)

# Pattern 2: Ignore signals (prevent interruption)
trap '' INT TERM  # Empty string = IGNORE signal
echo "Critical section - cannot be interrupted!"
# ... critical code ...
trap - INT TERM   # Reset to default handlers

# Pattern 3: Graceful shutdown with state
RUNNING=true
shutdown() {
    echo "Shutting down gracefully..."
    RUNNING=false  # Signal main loop to exit
}
trap shutdown TERM INT

while $RUNNING; do
    # Process work
    echo "Working..."
    sleep 1
done
echo "Shutdown complete"

# Pattern 4: Lock file management
LOCKFILE="/var/run/myscript.lock"
cleanup() { rm -f "$LOCKFILE"; }
trap cleanup EXIT

if [ -f "$LOCKFILE" ]; then
    echo "Script already running!" >&2
    exit 1
fi
echo $$ > "$LOCKFILE"  # Write PID to lock file
# ... main script ...

# Pattern 5: Error handling with line number
trap 'echo "Error on line $LINENO. Exit code: $?"' ERR
```

### 2.3 Signals Hay Dùng Với trap

```bash
trap 'command' EXIT      # Luôn chạy khi script exit (success hoặc error)
trap 'command' ERR       # Chạy khi command trả non-zero exit code
trap 'command' INT       # SIGINT (Ctrl+C)
trap 'command' TERM      # SIGTERM (kill command)
trap 'command' HUP       # SIGHUP (terminal closed)
trap 'command' DEBUG     # Chạy TRƯỚC mỗi command (cho debugging)
trap 'command' RETURN    # Chạy khi function/source returns
```

---

## 3. getopts — Option Parsing

### 3.1 getopts Là Gì?

**Ví dụ đời thường:** Khi bạn gọi `ls -la /tmp`, options `-l` và `-a` thay đổi hành vi. **getopts** giúp script CỦA BẠN xử lý options tương tự.

```bash
#!/bin/bash
# Script: deploy.sh
# Usage: ./deploy.sh -e production -v -f config.yml

ENVIRONMENT=""
VERBOSE=false
CONFIG_FILE=""

usage() {
    echo "Usage: $0 -e <environment> [-v] [-f <config_file>]"
    echo "  -e  Environment (required): staging, production"
    echo "  -v  Verbose mode"
    echo "  -f  Config file path"
    echo "  -h  Show this help"
    exit 1
}

while getopts "e:vf:h" opt; do
    case $opt in
        e) ENVIRONMENT="$OPTARG" ;;     # -e requires argument (có dấu : sau e)
        v) VERBOSE=true ;;              # -v is flag (không có :)
        f) CONFIG_FILE="$OPTARG" ;;     # -f requires argument
        h) usage ;;
        \?) echo "Invalid option: -$OPTARG" >&2; usage ;;
        :) echo "Option -$OPTARG requires an argument" >&2; usage ;;
    esac
done

# Shift processed options
shift $((OPTIND - 1))
# $@ now contains remaining non-option arguments

# Validate required options
if [ -z "$ENVIRONMENT" ]; then
    echo "Error: -e environment is required!" >&2
    usage
fi

# Use the options
echo "Deploying to: $ENVIRONMENT"
$VERBOSE && echo "Verbose mode enabled"
[ -n "$CONFIG_FILE" ] && echo "Using config: $CONFIG_FILE"
echo "Extra args: $@"
```

### 3.2 getopts Rules

```bash
# Option string "e:vf:h"
# e: = option -e REQUIRES an argument (OPTARG)
# v  = option -v is a FLAG (no argument)
# f: = option -f REQUIRES an argument
# h  = option -h is a FLAG
# Leading : → silent error mode (bạn tự handle errors)

# getopts chỉ hỗ trợ short options (-e, -v)
# Cho long options (--environment), dùng manual parsing hoặc getopt (external)
```

### 3.3 Long Options (Manual Parsing)

```bash
#!/bin/bash
# Hỗ trợ cả short và long options

while [[ $# -gt 0 ]]; do
    case "$1" in
        -e|--environment)
            ENVIRONMENT="$2"
            shift 2
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -f|--file)
            CONFIG_FILE="$2"
            shift 2
            ;;
        -h|--help)
            usage
            ;;
        --)  # End of options
            shift
            break
            ;;
        -*)
            echo "Unknown option: $1" >&2
            usage
            ;;
        *)
            break  # Non-option argument
            ;;
    esac
done
```

---

## 4. Here-Documents và Here-Strings

### 4.1 Here-Document (<<)

**Ví dụ đời thường:** Thay vì gõ email từng dòng, bạn có template sẵn và chỉ điền thông tin. Here-doc = template text block trong script.

```bash
# Basic here-doc
cat << EOF
Hello, $USER!
Today is $(date).
Your home directory is $HOME.
EOF

# Output:
# Hello, john!
# Today is Mon Jun 1 10:00:00 UTC 2026.
# Your home directory is /home/john.

# Here-doc KHÔNG expand variables (dùng quotes quanh delimiter)
cat << 'EOF'
This is literal: $USER
This is literal: $(date)
No expansion happens here!
EOF

# Here-doc với indentation (<<- strips leading TABS)
if true; then
    cat <<-EOF
    This is indented in source
    But output starts at column 0
    (Only TABs are stripped, not spaces!)
    EOF
fi

# Here-doc gửi vào command
mysql -u root << EOF
CREATE DATABASE myapp;
GRANT ALL ON myapp.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
EOF

# Here-doc ghi vào file
cat > /etc/nginx/conf.d/myapp.conf << 'EOF'
server {
    listen 80;
    server_name myapp.example.com;
    location / {
        proxy_pass http://localhost:3000;
    }
}
EOF

# Multi-line SSH command
ssh user@server << 'REMOTE'
cd /opt/app
git pull origin main
sudo systemctl restart app
echo "Deploy complete!"
REMOTE
```

### 4.2 Here-String (<<<)

```bash
# Here-string: pass STRING as stdin (thay vì echo | command)

# Thay vì:
echo "hello world" | wc -w

# Dùng here-string:
wc -w <<< "hello world"

# Dùng với variable:
greeting="Hello, World!"
wc -c <<< "$greeting"

# Dùng với read:
IFS=',' read -r name age city <<< "Alice,30,Hanoi"
echo "$name is $age from $city"
# Alice is 30 from Hanoi

# JSON parsing with jq:
data='{"name":"Alice","age":30}'
jq -r '.name' <<< "$data"
# Alice
```

---

## 5. Process Substitution — <() và >()

### 5.1 Process Substitution Là Gì?

**Ví dụ đời thường:** Bạn muốn so sánh 2 danh sách. Bình thường phải lưu ra 2 file rồi diff. Process substitution = tạo "file ảo" từ output của command → không cần file tạm!

```bash
# Diff output of 2 commands (KHÔNG cần temp files!)
diff <(ls /dir1) <(ls /dir2)

# <(command) = tạo file descriptor chứa output của command
# Giống như: command > /tmp/file; cat /tmp/file
# Nhưng KHÔNG tạo file thật, dùng named pipe (FIFO)

# Ví dụ thực tế: So sánh config 2 servers
diff <(ssh server1 cat /etc/nginx/nginx.conf) \
     <(ssh server2 cat /etc/nginx/nginx.conf)

# So sánh sorted output
diff <(sort file1.txt) <(sort file2.txt)

# Paste columns from different sources
paste <(cut -d',' -f1 data.csv) <(cut -d',' -f3 data.csv)

# While loop không bị subshell issue
while IFS= read -r line; do
    echo "Processing: $line"
done < <(find /var/log -name "*.log" -mtime -1)
# Dùng < <(...) thay vì pipe để tránh subshell!
```

### 5.2 Output Process Substitution >()

```bash
# Ghi output đồng thời vào nhiều nơi
echo "Important log" | tee >(gzip > log.gz) >(mail admin@example.com)

# Tee vào file VÀ process
command | tee >(grep ERROR > errors.txt) >(grep WARN > warnings.txt) > full.log
```

---

## 6. Coprocesses (coproc)

```bash
# coproc: chạy command trong background VỚI bidirectional communication

coproc WORKER { 
    while IFS= read -r line; do
        echo "PROCESSED: ${line^^}"  # Uppercase
    done
}

# Gửi data đến coprocess
echo "hello" >&"${WORKER[1]}"

# Đọc response từ coprocess
read -r response <&"${WORKER[0]}"
echo "$response"   # "PROCESSED: HELLO"

# Đóng khi xong
exec {WORKER[1]}>&-
wait $WORKER_PID
```

---

## 7. Debugging Shell Scripts

### 7.1 set Options

```bash
#!/bin/bash
set -euo pipefail    # "Strict mode" — HIGHLY RECOMMENDED!

# set -e: Exit immediately if command returns non-zero
# set -u: Treat unset variables as ERROR (prevent typos!)
# set -o pipefail: Pipeline fails if ANY command fails (not just last)

# set -x: Print each command before execution (DEBUGGING)
set -x
echo "This line printed before execution"
# + echo 'This line printed before execution'
# This line printed before execution
set +x  # Turn off tracing

# Trace specific section only:
# PS4 customizes the trace prefix
PS4='+${BASH_SOURCE}:${LINENO}: '
set -x
# +script.sh:15: echo 'debug info'
```

### 7.2 Debugging Techniques

```bash
# 1. Run with debug flag
bash -x script.sh           # Trace entire script
bash -xv script.sh          # Trace + print input lines

# 2. Debug specific section
debug_start() { set -x; }
debug_stop() { set +x; }

debug_start
# ... problematic code ...
debug_stop

# 3. Trap DEBUG for line-by-line execution
trap 'read -p "Line $LINENO: $BASH_COMMAND [Enter to continue]"' DEBUG

# 4. Error trap with context
trap 'echo "ERROR: Line $LINENO, Command: $BASH_COMMAND, Exit: $?"' ERR

# 5. Shellcheck (static analysis - run BEFORE executing!)
# shellcheck script.sh
# Finds: quoting issues, deprecated syntax, common bugs
```

---

## 8. Advanced Patterns

### 8.1 Robust Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Script metadata
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"

# Logging
log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >&2; }
info() { log "INFO: $*"; }
warn() { log "WARN: $*"; }
error() { log "ERROR: $*"; }
die() { error "$*"; exit 1; }

# Cleanup
TMPDIR=$(mktemp -d)
cleanup() {
    rm -rf "$TMPDIR"
    info "Cleanup complete"
}
trap cleanup EXIT

# Usage
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <argument>

Options:
    -v, --verbose     Enable verbose output
    -d, --dry-run     Show what would be done
    -h, --help        Show this help
EOF
    exit "${1:-0}"
}

# Parse options
VERBOSE=false
DRY_RUN=false

while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose) VERBOSE=true; shift ;;
        -d|--dry-run) DRY_RUN=true; shift ;;
        -h|--help) usage 0 ;;
        --) shift; break ;;
        -*) die "Unknown option: $1" ;;
        *) break ;;
    esac
done

[[ $# -lt 1 ]] && die "Missing required argument. Use -h for help."

# Main logic
main() {
    local arg="$1"
    info "Processing: $arg"
    $VERBOSE && info "Verbose: extra details here"
    $DRY_RUN && { info "DRY RUN: would process $arg"; return; }
    
    # Actual work here
    echo "Done!"
}

main "$@"
```

### 8.2 Parallel Execution

```bash
# GNU Parallel: Run commands in parallel
find . -name "*.jpg" | parallel convert {} -resize 800x600 {.}_thumb.jpg

# Background jobs with wait
for server in server1 server2 server3; do
    ssh "$server" "sudo apt update && sudo apt upgrade -y" &
done
wait  # Wait for all background jobs
echo "All servers updated!"

# Parallel with max jobs
MAX_JOBS=4
for file in *.csv; do
    (($(jobs -r | wc -l) >= MAX_JOBS)) && wait -n  # Wait if max reached
    process_file "$file" &
done
wait
```

---

## 9. Common Pitfalls và Best Practices

```bash
# ❌ PITFALL: Unquoted variables (word splitting + glob expansion!)
file="my file.txt"
rm $file          # BAD: rm "my" "file.txt" (2 files!)
rm "$file"        # GOOD: rm "my file.txt" (1 file)

# ❌ PITFALL: [ vs [[ 
[ $var == "hello" ]     # BAD: fails if $var empty or has spaces
[[ $var == "hello" ]]   # GOOD: [[ handles empty/spaces

# ❌ PITFALL: Pipe subshell
count=0
cat file | while read line; do ((count++)); done
echo $count    # 0! (while loop ran in subshell due to pipe!)
# FIX: Use process substitution
while read line; do ((count++)); done < <(cat file)

# ❌ PITFALL: cd without error check
cd /some/dir        # If fails, rest of script runs in WRONG directory!
cd /some/dir || exit 1  # GOOD: exit if cd fails

# ✅ BEST PRACTICES:
# 1. Always quote variables: "$var"
# 2. Use set -euo pipefail
# 3. Use [[ ]] instead of [ ]
# 4. Use shellcheck
# 5. trap cleanup EXIT
# 6. Use functions for reusable code
# 7. Local variables in functions: local var="value"
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Key Takeaways

1. **trap EXIT** = cleanup guarantee (temp files, lock files, processes)
2. **getopts** cho short options; manual parsing cho long options
3. **Here-docs** cho multi-line text/configs; here-strings cho single-line stdin
4. **Process substitution** `<()` = pipe output as file (no temp files!)
5. **set -euo pipefail** = bash strict mode (LUÔN DÙNG!)
6. **PS4 + set -x** cho debugging; shellcheck cho static analysis
7. **Quote everything**: `"$var"` prevents word splitting + globbing
8. **Template scripts** with proper error handling, logging, cleanup

### 10.2 Tài Liệu Tham Khảo

- GNU Bash Manual: https://www.gnu.org/software/bash/manual/
- man bash (comprehensive)
- Advanced Bash-Scripting Guide (ABS)
- ShellCheck: https://www.shellcheck.net/
- Google Shell Style Guide
- "Classic Shell Scripting" by Robbins & Beebe (O'Reilly)
- Bash Hackers Wiki: https://wiki.bash-hackers.org/

---

*Bài viết tiếp theo: Regular Expressions — BRE, ERE, PCRE patterns*
