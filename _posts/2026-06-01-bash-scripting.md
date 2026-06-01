---
layout: post
title: "Linux Fundamentals - Phần 9: Bash Scripting"
subtitle: "Tự động hóa công việc bằng shell script — từ cơ bản đến EC2 User Data"
gh-repo: wayarmy/wayarmy.github.io
tags: [linux, aws, learning-path]
comments: true
date: 2026-06-01
categories: [linux]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 9).
>
> **Đối tượng:** Người mới hoàn toàn — biết CLI cơ bản từ Phần 7-8.
>
> **Nguồn tham khảo:**
> - GNU Bash Reference Manual — [https://www.gnu.org/software/bash/manual/](https://www.gnu.org/software/bash/manual/)
> - TLDP Advanced Bash-Scripting Guide — [https://tldp.org/LDP/abs/html/](https://tldp.org/LDP/abs/html/)
> - Google Shell Style Guide — [https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)
> - ShellCheck — [https://www.shellcheck.net/](https://www.shellcheck.net/)
> - AWS EC2 User Data — [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

---

## 1. Bash Script là gì? — "Kịch bản" cho máy tính

### Ví dụ đời thường:

Mỗi sáng bạn thực hiện routine:
1. Tắt báo thức
2. Bật đèn
3. Pha cà phê
4. Mở email

Nếu có robot, bạn sẽ viết "kịch bản" (script) để robot làm tự động. **Bash script** chính là kịch bản cho Linux — thay vì gõ từng lệnh một, bạn viết tất cả vào file và chạy 1 lần.

### Script đầu tiên:

```bash
#!/bin/bash
# File: hello.sh
# Mô tả: Script đầu tiên

echo "Xin chào! Hôm nay là:"
date
echo "Bạn đang đăng nhập với user:"
whoami
echo "Thư mục hiện tại:"
pwd
```

```bash
# Tạo file
$ vim hello.sh

# Cấp quyền execute
$ chmod +x hello.sh

# Chạy
$ ./hello.sh
Xin chào! Hôm nay là:
Mon Jun  9 10:00:00 UTC 2026
Bạn đang đăng nhập với user:
alice
Thư mục hiện tại:
/home/alice
```

### Shebang (`#!`) — Dòng đầu tiên:

```bash
#!/bin/bash        ← "Hãy dùng Bash để chạy file này"
```

- `#!` = Shebang (sharp + bang)
- `/bin/bash` = Đường dẫn đến Bash interpreter
- Nếu không có shebang → shell hiện tại sẽ chạy (có thể không phải Bash!)
- Alternatives: `#!/bin/sh` (POSIX shell), `#!/usr/bin/env bash` (portable hơn)

---

## 2. Variables — Biến

### Ví dụ đời thường:

Biến giống **hộp có nhãn** — bạn bỏ đồ vào hộp, dán nhãn, và sau đó dùng nhãn để tìm lại đồ.

### Khai báo và sử dụng (GNU Bash Manual, Section 3.4):

```bash
#!/bin/bash

# Gán biến (KHÔNG có khoảng trắng quanh dấu =)
name="Alice"
age=25
greeting="Xin chào, ${name}!"

# Sử dụng biến (dùng $)
echo "$greeting"           # Xin chào, Alice!
echo "Tuổi: $age"         # Tuổi: 25
echo "Tên có ${#name} ký tự"  # Tên có 5 ký tự

# SAI: (có khoảng trắng)
# name = "Alice"    ← LỖI! Bash hiểu "name" là lệnh, "=" là argument
```

### Quy tắc đặt tên biến:

- Bắt đầu bằng chữ cái hoặc `_`
- Chỉ chứa chữ, số, `_`
- Case-sensitive: `Name` ≠ `name`
- Convention: UPPERCASE cho constants/environment, lowercase cho local

### Biến đặc biệt:

| Biến | Ý nghĩa |
|------|---------|
| `$0` | Tên script |
| `$1, $2, ...` | Arguments (tham số dòng lệnh) |
| `$#` | Số lượng arguments |
| `$@` | Tất cả arguments (mỗi arg là 1 word riêng) |
| `$*` | Tất cả arguments (gộp thành 1 string) |
| `$?` | Exit code của lệnh cuối (0 = thành công) |
| `$$` | PID của script hiện tại |
| `$!` | PID của background process cuối |

```bash
#!/bin/bash
# File: greet.sh

echo "Script: $0"
echo "Tên: $1"
echo "Tuổi: $2"
echo "Tổng arguments: $#"
echo "Tất cả: $@"

# Chạy: ./greet.sh Alice 25
# Output:
# Script: ./greet.sh
# Tên: Alice
# Tuổi: 25
# Tổng arguments: 2
# Tất cả: Alice 25
```

### Quoting — Ngoặc kép vs ngoặc đơn:

```bash
name="World"

echo "Hello $name"    # Hello World    ← Double quotes: expand variables
echo 'Hello $name'    # Hello $name    ← Single quotes: literal (không expand)
echo "Path: $(pwd)"   # Path: /home/alice  ← Command substitution
echo "2 + 2 = $((2+2))"  # 2 + 2 = 4  ← Arithmetic
```

### Command Substitution — Lấy output của lệnh:

```bash
# Cú pháp: $(command) hoặc `command` (legacy)
current_date=$(date +%Y-%m-%d)
hostname=$(hostname)
file_count=$(ls | wc -l)

echo "Hôm nay: $current_date"
echo "Máy: $hostname"
echo "Có $file_count files"
```

### Environment Variables:

```bash
# Xem environment variables
env
printenv PATH

# Set environment variable (cho child processes)
export MY_APP_ENV="production"
export DB_HOST="localhost"

# Chỉ set local (không export → child processes không thấy)
local_var="only here"
```

---

## 3. Conditionals — Điều kiện (if/else)

### Ví dụ đời thường:

"NẾU trời mưa THÌ mang ô, KHÔNG THÌ mang kính râm"

### Cú pháp if (GNU Bash Manual, Section 3.2.5.2):

```bash
if [ condition ]; then
    # code chạy nếu condition đúng (true)
elif [ condition2 ]; then
    # code chạy nếu condition2 đúng
else
    # code chạy nếu tất cả sai
fi
```

### Test conditions — Toán tử so sánh:

#### So sánh số:

| Operator | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| `-eq` | equal (=) | `[ $a -eq $b ]` |
| `-ne` | not equal (≠) | `[ $a -ne $b ]` |
| `-gt` | greater than (>) | `[ $a -gt $b ]` |
| `-ge` | greater or equal (≥) | `[ $a -ge $b ]` |
| `-lt` | less than (<) | `[ $a -lt $b ]` |
| `-le` | less or equal (≤) | `[ $a -le $b ]` |

#### So sánh string:

| Operator | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| `=` hoặc `==` | Bằng | `[ "$a" = "$b" ]` |
| `!=` | Không bằng | `[ "$a" != "$b" ]` |
| `-z` | String rỗng (zero length) | `[ -z "$a" ]` |
| `-n` | String không rỗng | `[ -n "$a" ]` |

#### Kiểm tra file:

| Operator | Ý nghĩa |
|----------|---------|
| `-f` | Là file thông thường |
| `-d` | Là directory |
| `-e` | Tồn tại (exists) |
| `-r` | Readable |
| `-w` | Writable |
| `-x` | Executable |
| `-s` | File không rỗng (size > 0) |

### Ví dụ thực tế:

```bash
#!/bin/bash
# File: check_server.sh — Kiểm tra web server

URL="http://localhost:80"
LOG_FILE="/var/log/health-check.log"

# Kiểm tra HTTP response
status_code=$(curl -s -o /dev/null -w "%{http_code}" "$URL")

if [ "$status_code" -eq 200 ]; then
    echo "$(date): OK — Server responding normally" >> "$LOG_FILE"
elif [ "$status_code" -eq 503 ]; then
    echo "$(date): WARNING — Server overloaded (503)" >> "$LOG_FILE"
    systemctl restart nginx
else
    echo "$(date): ERROR — Server returned $status_code" >> "$LOG_FILE"
    # Gửi alert
    echo "Server down! Status: $status_code" | mail admin@company.com
fi
```

### `[[ ]]` vs `[ ]`:

`[[ ]]` là Bash extension — mạnh hơn `[ ]`:

```bash
# [[ ]] hỗ trợ regex matching
if [[ "$filename" =~ \.log$ ]]; then
    echo "Đây là file log"
fi

# [[ ]] hỗ trợ pattern matching
if [[ "$answer" == y* ]]; then
    echo "User said yes"
fi

# [[ ]] không cần quote variables (an toàn hơn)
if [[ $var == "hello" ]]; then  # OK ngay cả khi $var rỗng
    echo "match"
fi
```

### Logical operators:

```bash
# AND (&&) / OR (||)
if [ -f "$file" ] && [ -r "$file" ]; then
    echo "File tồn tại VÀ đọc được"
fi

if [ -z "$DB_HOST" ] || [ -z "$DB_PORT" ]; then
    echo "ERROR: Missing database configuration"
    exit 1
fi

# Short-circuit evaluation
[ -d "/tmp" ] && echo "tmp exists"      # Chạy echo NẾU dir tồn tại
[ -f "nofile" ] || echo "file missing"  # Chạy echo NẾU file KHÔNG tồn tại
```

### Case statement (switch):

```bash
#!/bin/bash
# File: deploy.sh

environment="$1"

case "$environment" in
    production|prod)
        echo "Deploying to PRODUCTION"
        INSTANCES=10
        ;;
    staging|stg)
        echo "Deploying to STAGING"
        INSTANCES=2
        ;;
    development|dev)
        echo "Deploying to DEV"
        INSTANCES=1
        ;;
    *)
        echo "ERROR: Unknown environment: $environment"
        echo "Usage: $0 {production|staging|development}"
        exit 1
        ;;
esac
```

---

## 4. Loops — Vòng lặp

### For loop:

```bash
# Lặp qua danh sách
for fruit in apple banana cherry; do
    echo "Trái cây: $fruit"
done

# Lặp qua files
for file in /var/log/*.log; do
    echo "Log file: $file ($(wc -l < "$file") lines)"
done

# Lặp qua range
for i in {1..10}; do
    echo "Server $i"
done

# C-style for loop
for ((i=0; i<5; i++)); do
    echo "Count: $i"
done

# Lặp qua command output
for user in $(cat /etc/passwd | cut -d: -f1); do
    echo "User: $user"
done
```

### While loop:

```bash
# Đếm
counter=1
while [ $counter -le 10 ]; do
    echo "Lần: $counter"
    ((counter++))
done

# Đọc file từng dòng
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Infinite loop với break
while true; do
    status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost)
    if [ "$status" -eq 200 ]; then
        echo "Server ready!"
        break
    fi
    echo "Waiting... (status: $status)"
    sleep 5
done
```

### Until loop (ngược với while):

```bash
# Chạy cho đến khi condition TRUE
until [ -f /tmp/ready.flag ]; do
    echo "Waiting for ready flag..."
    sleep 2
done
echo "Flag found! Proceeding..."
```

### Ví dụ thực tế — Retry logic:

```bash
#!/bin/bash
# Retry command với exponential backoff

MAX_RETRIES=5
RETRY_DELAY=2

retry_command() {
    local cmd="$1"
    local attempt=1

    while [ $attempt -le $MAX_RETRIES ]; do
        echo "Attempt $attempt/$MAX_RETRIES: $cmd"

        if eval "$cmd"; then
            echo "Success on attempt $attempt!"
            return 0
        fi

        echo "Failed. Retrying in ${RETRY_DELAY}s..."
        sleep $RETRY_DELAY
        RETRY_DELAY=$((RETRY_DELAY * 2))    # Exponential backoff
        ((attempt++))
    done

    echo "ERROR: Failed after $MAX_RETRIES attempts"
    return 1
}

retry_command "curl -sf http://api.example.com/health"
```

---

## 5. Functions — Hàm

### Ví dụ đời thường:

Function giống **công thức nấu ăn con** — bạn viết 1 lần "Cách làm nước sốt", rồi mỗi khi cần nước sốt chỉ gọi tên, không cần viết lại từ đầu.

### Khai báo và gọi function:

```bash
#!/bin/bash

# Khai báo function
log_info() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] INFO: $*"
}

log_error() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $*" >&2
}

# Gọi function
log_info "Application starting"
log_info "Loading configuration"
log_error "Database connection failed"
```

### Function với arguments và return:

```bash
#!/bin/bash

# Function nhận arguments (qua $1, $2, ...)
create_user() {
    local username="$1"    # local = biến chỉ sống trong function
    local email="$2"

    if [ -z "$username" ]; then
        echo "ERROR: Username required"
        return 1    # Return non-zero = lỗi
    fi

    echo "Creating user: $username ($email)"
    # useradd "$username" ...
    return 0    # Return 0 = thành công
}

# Gọi function
create_user "alice" "alice@example.com"
if [ $? -eq 0 ]; then
    echo "User created successfully"
fi

# Hoặc dùng trực tiếp trong if
if create_user "bob" "bob@example.com"; then
    echo "OK"
else
    echo "FAILED"
fi
```

### Function trả về giá trị:

```bash
# Bash functions không có "return value" giống Python/JS
# Cách 1: echo + command substitution
get_instance_id() {
    local id
    id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    echo "$id"    # "return" value bằng echo
}

instance_id=$(get_instance_id)    # Capture output
echo "Instance: $instance_id"

# Cách 2: Dùng global variable
RESULT=""
calculate() {
    RESULT=$(( $1 + $2 ))
}
calculate 5 3
echo "Sum: $RESULT"    # 8
```

---

## 6. Error Handling — Xử lý lỗi

### Exit codes:

Mỗi command kết thúc với **exit code** (0-255):
- `0` = Thành công
- `1-255` = Lỗi (số khác nhau = loại lỗi khác nhau)

```bash
# Kiểm tra exit code
ls /existing_folder
echo $?    # 0 (success)

ls /nonexistent
echo $?    # 2 (no such file)
```

### `set` options — Bắt lỗi tự động:

```bash
#!/bin/bash
set -e          # Exit ngay khi bất kỳ lệnh nào fail (exit code ≠ 0)
set -u          # Exit khi dùng biến chưa khai báo
set -o pipefail # Pipe fail nếu BẤT KỲ lệnh nào trong pipe fail
set -x          # Debug: in mỗi lệnh trước khi chạy

# Viết gọn:
set -euo pipefail
```

**Ví dụ hiệu quả của `set -e`:**

```bash
#!/bin/bash
# KHÔNG có set -e:
cd /nonexistent         # Fail, nhưng script tiếp tục!
rm -rf *                # XÓA THƯ MỤC HIỆN TẠI (vì cd fail → vẫn ở chỗ cũ!)

# CÓ set -e:
set -e
cd /nonexistent         # Fail → script DỪNG ngay → an toàn!
rm -rf *                # Không chạy đến đây
```

### Trap — Bắt signals và cleanup:

```bash
#!/bin/bash
set -euo pipefail

TEMP_DIR=$(mktemp -d)
LOCK_FILE="/tmp/myscript.lock"

# Cleanup function
cleanup() {
    echo "Cleaning up..."
    rm -rf "$TEMP_DIR"
    rm -f "$LOCK_FILE"
}

# Trap: chạy cleanup khi script exit (bình thường hay lỗi)
trap cleanup EXIT
# Trap: bắt Ctrl+C
trap 'echo "Interrupted!"; exit 1' INT

# Main logic
touch "$LOCK_FILE"
echo "Working in $TEMP_DIR..."
# ... long running work ...
echo "Done!"
# cleanup tự động chạy khi exit (nhờ trap)
```

### Error handling pattern:

```bash
#!/bin/bash
set -euo pipefail

# Function log có màu
log() { echo -e "\033[32m[INFO]\033[0m $*"; }
err() { echo -e "\033[31m[ERROR]\033[0m $*" >&2; }
warn() { echo -e "\033[33m[WARN]\033[0m $*"; }

# Die function
die() {
    err "$*"
    exit 1
}

# Usage
[ -f "$CONFIG_FILE" ] || die "Config file not found: $CONFIG_FILE"
command -v docker >/dev/null || die "Docker not installed"
```

---

## 7. Practical Scripts — Ví dụ thực tế

### Script 1: Automated Server Setup

```bash
#!/bin/bash
# setup_web_server.sh — Cài đặt web server tự động
set -euo pipefail

log() { echo "[$(date '+%H:%M:%S')] $*"; }

# Kiểm tra root
if [ "$(id -u)" -ne 0 ]; then
    echo "Script này cần chạy với sudo!"
    exit 1
fi

log "Updating system packages..."
apt update -y && apt upgrade -y

log "Installing Nginx..."
apt install -y nginx

log "Configuring firewall..."
ufw allow 'Nginx Full'
ufw --force enable

log "Starting Nginx..."
systemctl enable --now nginx

log "Creating index page..."
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>Server Ready</title></head>
<body>
<h1>Server configured successfully!</h1>
<p>Hostname: $(hostname)</p>
<p>Setup date: $(date)</p>
</body>
</html>
EOF

log "Verifying..."
curl -s http://localhost | grep -q "Server Ready" && log "✓ Web server working!" || log "✗ Something wrong"

log "Setup complete!"
```

### Script 2: Log Rotation & Cleanup

```bash
#!/bin/bash
# cleanup_logs.sh — Dọn dẹp log files
set -euo pipefail

LOG_DIR="/var/log/myapp"
MAX_AGE_DAYS=30
MAX_SIZE_MB=100
ARCHIVE_DIR="/var/log/myapp/archive"

mkdir -p "$ARCHIVE_DIR"

echo "=== Log Cleanup: $(date) ==="

# 1. Compress log files lớn hơn MAX_SIZE_MB
find "$LOG_DIR" -name "*.log" -size "+${MAX_SIZE_MB}M" | while read -r logfile; do
    echo "Compressing: $logfile ($(du -h "$logfile" | cut -f1))"
    gzip "$logfile"
    mv "${logfile}.gz" "$ARCHIVE_DIR/"
done

# 2. Xóa archives cũ hơn MAX_AGE_DAYS
deleted=$(find "$ARCHIVE_DIR" -name "*.gz" -mtime "+$MAX_AGE_DAYS" -delete -print | wc -l)
echo "Deleted $deleted old archives (>$MAX_AGE_DAYS days)"

# 3. Report disk usage
echo "Current log usage: $(du -sh "$LOG_DIR" | cut -f1)"
echo "=== Done ==="
```

### Script 3: AWS EC2 User Data — Full Stack Setup

```bash
#!/bin/bash
# EC2 User Data script — Deploy web application
set -euo pipefail
exec > >(tee /var/log/user-data.log) 2>&1    # Log everything

echo "=== Starting setup at $(date) ==="

# Variables
APP_USER="appuser"
APP_DIR="/opt/myapp"
APP_PORT=8080
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

echo "Region: $REGION, Instance: $INSTANCE_ID"

# 1. System update
echo "--- Updating system ---"
yum update -y

# 2. Install dependencies
echo "--- Installing packages ---"
yum install -y docker git

# 3. Create app user
echo "--- Creating app user ---"
useradd -r -s /sbin/nologin "$APP_USER" || true

# 4. Start Docker
echo "--- Starting Docker ---"
systemctl enable --now docker
usermod -aG docker "$APP_USER"

# 5. Deploy application
echo "--- Deploying application ---"
mkdir -p "$APP_DIR"
cd "$APP_DIR"

# Pull and run container
docker pull myregistry/myapp:latest
docker run -d \
    --name myapp \
    --restart unless-stopped \
    -p "${APP_PORT}:${APP_PORT}" \
    -e "REGION=${REGION}" \
    -e "INSTANCE_ID=${INSTANCE_ID}" \
    myregistry/myapp:latest

# 6. Install and configure Nginx as reverse proxy
echo "--- Configuring reverse proxy ---"
amazon-linux-extras install nginx1 -y

cat > /etc/nginx/conf.d/myapp.conf <<EOF
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:${APP_PORT};
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }

    location /health {
        access_log off;
        proxy_pass http://localhost:${APP_PORT}/health;
    }
}
EOF

systemctl enable --now nginx

# 7. Verify
echo "--- Verifying deployment ---"
sleep 5
if curl -sf http://localhost/health; then
    echo "✓ Application healthy!"
else
    echo "✗ Application not responding"
    docker logs myapp
fi

echo "=== Setup complete at $(date) ==="
```

---

## 8. Arrays và String Manipulation

### Arrays (Bash 4+):

```bash
# Indexed array
servers=("web-01" "web-02" "web-03" "db-01")

# Truy cập
echo "${servers[0]}"       # web-01
echo "${servers[2]}"       # web-03
echo "${servers[@]}"       # Tất cả elements
echo "${#servers[@]}"      # Số lượng: 4

# Lặp qua array
for server in "${servers[@]}"; do
    echo "Checking $server..."
    ping -c 1 "$server" &>/dev/null && echo "  ✓ UP" || echo "  ✗ DOWN"
done

# Append
servers+=("cache-01")

# Associative array (key-value, Bash 4+)
declare -A config
config[host]="localhost"
config[port]="5432"
config[db]="myapp"

echo "Connecting to ${config[host]}:${config[port]}/${config[db]}"
```

### String operations:

```bash
str="Hello, World! Welcome to Bash"

# Length
echo "${#str}"              # 30

# Substring
echo "${str:0:5}"           # Hello
echo "${str:7:5}"           # World

# Replace
echo "${str/World/Linux}"   # Hello, Linux! Welcome to Bash
echo "${str//l/L}"          # HeLLo, WorLd! WeLcome to Bash (all occurrences)

# Remove prefix/suffix
filepath="/var/log/nginx/access.log"
echo "${filepath##*/}"      # access.log (remove longest prefix matching */)
echo "${filepath%/*}"       # /var/log/nginx (remove shortest suffix matching /*)
echo "${filepath%.log}"     # /var/log/nginx/access (remove .log)

# Default values
echo "${UNDEFINED_VAR:-default_value}"    # Dùng default nếu var rỗng/unset
echo "${UNDEFINED_VAR:=default_value}"    # Set VÀ dùng default
DB_HOST="${DB_HOST:-localhost}"            # Common pattern
```

---

## 9. Input và Interactive Scripts

### Read user input:

```bash
#!/bin/bash

# Đọc input
echo -n "Nhập tên: "
read name
echo "Xin chào, $name!"

# Read với prompt
read -p "Username: " username
read -sp "Password: " password    # -s = silent (không hiện ký tự)
echo

# Read với timeout
read -t 10 -p "Confirm (y/n)? " answer || answer="n"

# Read vào array
echo "Nhập danh sách (space-separated):"
read -a items
echo "Bạn nhập ${#items[@]} items: ${items[*]}"
```

### Here Document (Heredoc):

```bash
# Heredoc — ghi multi-line string vào file/command
cat > /etc/nginx/sites-available/mysite <<EOF
server {
    listen 80;
    server_name $DOMAIN;
    root /var/www/$DOMAIN;
}
EOF

# Heredoc không expand variables (dùng 'EOF' với quotes)
cat > script.sh <<'EOF'
#!/bin/bash
echo "This $variable won't be expanded"
echo "Current user: $(whoami)"
EOF
```

---

## 10. Best Practices — Thực hành tốt nhất

### Theo Google Shell Style Guide:

```bash
#!/bin/bash
# =============================================================================
# Script: deploy.sh
# Description: Deploy application to production
# Author: DevOps Team
# Date: 2026-06-09
# Usage: ./deploy.sh <environment> <version>
# =============================================================================

set -euo pipefail

# Constants (UPPERCASE)
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/deploy.log"
readonly MAX_RETRIES=3

# Functions (lowercase, underscore separated)
show_usage() {
    cat <<EOF
Usage: $(basename "$0") <environment> <version>

Arguments:
    environment    Target environment (production|staging|development)
    version        Application version to deploy (e.g., v1.2.3)

Examples:
    $(basename "$0") production v1.2.3
    $(basename "$0") staging v1.3.0-rc1
EOF
}

validate_inputs() {
    if [ $# -lt 2 ]; then
        show_usage
        exit 1
    fi
}

main() {
    validate_inputs "$@"

    local environment="$1"
    local version="$2"

    log "Deploying $version to $environment"
    # ... deploy logic ...
    log "Deployment complete"
}

# Entry point
main "$@"
```

### Checklist cho production scripts:

1. ✅ Bắt đầu với `set -euo pipefail`
2. ✅ Có shebang `#!/bin/bash`
3. ✅ Quote tất cả variables: `"$var"` (không phải `$var`)
4. ✅ Dùng `local` cho biến trong function
5. ✅ Có error handling (trap, exit codes)
6. ✅ Có logging (timestamps)
7. ✅ Có usage/help message
8. ✅ Validate inputs trước khi chạy
9. ✅ Chạy qua [ShellCheck](https://www.shellcheck.net/) để tìm bugs
10. ✅ Test trên staging trước production

---

## 11. Thực hành: Lab tự làm

### Lab 1: Script cơ bản

```bash
# Viết script system-info.sh hiển thị:
# - Hostname, OS, Kernel version
# - CPU info (cores, model)
# - RAM (total, used, free)
# - Disk usage
# - Top 5 processes by CPU
# - Active network connections count
```

### Lab 2: Script xử lý file

```bash
# Viết script organize.sh:
# - Nhận argument: folder path
# - Tạo subfolders: images/, documents/, videos/, others/
# - Di chuyển files vào folder tương ứng theo extension
# - In report: bao nhiêu file mỗi loại
```

### Lab 3: EC2 monitoring script

```bash
# Viết script monitor.sh cho EC2:
# - Check disk usage → alert nếu > 80%
# - Check memory → alert nếu > 90%
# - Check if nginx running → restart nếu dead
# - Log kết quả vào file
# - Cài vào crontab chạy mỗi 5 phút
```

### Lab 4: Deployment script

```bash
# Viết deploy.sh:
# - Accept: environment, git tag
# - Pull code từ git
# - Run tests
# - Build Docker image
# - Push to ECR
# - Update ECS service
# - Health check
# - Rollback nếu health check fail
```

---

## 12. Tổng kết

| Khái niệm | Ví dụ đời thường | Cú pháp |
|-----------|-----------------|---------|
| Script | Kịch bản cho robot | `#!/bin/bash` + lệnh |
| Variable | Hộp có nhãn | `name="value"`, `$name` |
| Condition | "Nếu... thì..." | `if [ cond ]; then ... fi` |
| Loop | Lặp lại công việc | `for`, `while`, `until` |
| Function | Công thức con | `func_name() { ... }` |
| Error handling | Phanh khẩn cấp | `set -euo pipefail`, `trap` |
| Arrays | Danh sách | `arr=("a" "b" "c")` |
| User Data | Hướng dẫn setup EC2 | Script chạy lần đầu boot |

---

## Tài liệu tham khảo

1. **GNU Bash Reference Manual** — [https://www.gnu.org/software/bash/manual/bash.html](https://www.gnu.org/software/bash/manual/bash.html)
2. **TLDP Advanced Bash-Scripting Guide** — [https://tldp.org/LDP/abs/html/](https://tldp.org/LDP/abs/html/)
3. **Google Shell Style Guide** — [https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)
4. **ShellCheck** — Static analysis tool. [https://www.shellcheck.net/](https://www.shellcheck.net/)
5. **"The Linux Command Line"** — William Shotts. [https://linuxcommand.org/tlcl.php](https://linuxcommand.org/tlcl.php)
6. **AWS EC2 User Data** — [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
7. **Bash Pitfalls** — Greg's Wiki. [https://mywiki.wooledge.org/BashPitfalls](https://mywiki.wooledge.org/BashPitfalls)

---

**Bài tiếp theo:** [Phần 10: Docker & Containers — Ảo hóa ứng dụng](/2026-06-01-docker-containers/)
