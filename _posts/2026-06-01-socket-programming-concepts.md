---
layout: post
title: "Socket Programming Concepts - Sockets, bind/listen/accept, epoll/kqueue và I/O Models"
date: 2026-06-01
categories: [networking]
tags: [socket, tcp, udp, epoll, kqueue, io-multiplexing, event-driven, non-blocking]
---

## 1. Giới thiệu — Ổ cắm điện cho Internet

Hãy tưởng tượng bạn muốn nói chuyện với ai đó qua **điện thoại cố định**:

1. **Mua điện thoại** → `socket()` — tạo thiết bị giao tiếp
2. **Gắn số điện thoại** → `bind()` — gán địa chỉ (IP:port)
3. **Bật chế độ chờ** → `listen()` — sẵn sàng nhận cuộc gọi
4. **Nhấc máy** → `accept()` — chấp nhận cuộc gọi đến
5. **Nói/Nghe** → `send()/recv()` — truyền dữ liệu
6. **Gác máy** → `close()` — kết thúc cuộc gọi

Hoặc nếu BẠN muốn gọi đi:
1. **Mua điện thoại** → `socket()` — tạo thiết bị
2. **Quay số** → `connect()` — gọi đến người khác
3. **Nói/Nghe** → `send()/recv()` — truyền dữ liệu
4. **Gác máy** → `close()` — kết thúc

**Socket** = "Ổ cắm" cho phép 2 chương trình trên mạng giao tiếp với nhau.

### Socket trong cuộc sống hàng ngày

Mỗi khi bạn:
- Mở trình duyệt web → Browser tạo socket, connect đến server port 443
- Gửi email → Email client tạo socket, connect đến SMTP server port 587
- SSH vào server → SSH client tạo socket, connect đến port 22
- Chơi game online → Game tạo UDP socket, sendto game server

**MỌI** network communication đều qua sockets!

### Socket = File Descriptor

Trên Unix/Linux, socket là một **file descriptor (fd)**:
```
"Everything is a file" in Unix:
  fd 0 = stdin (keyboard)
  fd 1 = stdout (screen)
  fd 2 = stderr (error output)
  fd 3 = opened file
  fd 4 = socket (network!) ← Same interface as files!
  
  read(fd, buffer, size)  → works for files AND sockets!
  write(fd, buffer, size) → works for files AND sockets!
```

---

## 2. Socket Types và Addressing

### Phép so sánh — Các loại "đường dây" giao tiếp

- **SOCK_STREAM (TCP)** = Điện thoại — kết nối trước, nói chuyện liên tục, đảm bảo
- **SOCK_DGRAM (UDP)** = Walkie-talkie — gửi thông điệp rời rạc, không kết nối
- **SOCK_RAW** = Radio HAM — truy cập trực tiếp tần số (raw IP packets)

### Socket Domain (Address Family)

| Domain | Constant | Mô tả |
|---|---|---|
| IPv4 Internet | AF_INET | TCP/UDP over IPv4 |
| IPv6 Internet | AF_INET6 | TCP/UDP over IPv6 |
| Unix Local | AF_UNIX (AF_LOCAL) | Inter-process trên cùng machine |
| Netlink | AF_NETLINK | Kernel ↔ Userspace (Linux) |
| Packet | AF_PACKET | Raw Ethernet frames |

### Socket Address Structure

```c
// IPv4 socket address
struct sockaddr_in {
    sa_family_t    sin_family;  // AF_INET
    in_port_t      sin_port;    // Port number (network byte order!)
    struct in_addr sin_addr;    // IPv4 address
};

// IPv6 socket address
struct sockaddr_in6 {
    sa_family_t     sin6_family;   // AF_INET6
    in_port_t       sin6_port;     // Port number
    uint32_t        sin6_flowinfo; // Flow information
    struct in6_addr sin6_addr;     // IPv6 address
    uint32_t        sin6_scope_id; // Scope ID
};

// Generic (used in function signatures)
struct sockaddr {
    sa_family_t sa_family;   // Address family
    char        sa_data[14]; // Address data
};
```

### Byte Order — Network vs Host

```
Máy tính lưu số theo Little-Endian (Intel) hoặc Big-Endian:
  
  Port 80 = 0x0050:
    Little-Endian (host): 50 00
    Big-Endian (network): 00 50

Network LUÔN dùng Big-Endian!
→ Phải convert khi đặt port/IP vào struct!

Conversion functions:
  htons() — Host TO Network Short (16-bit port)
  htonl() — Host TO Network Long (32-bit IP)
  ntohs() — Network TO Host Short
  ntohl() — Network TO Host Long
```

---

## 3. TCP Server — bind(), listen(), accept()

### Phép so sánh — Mở nhà hàng

1. **socket()** = Thuê mặt bằng (tạo không gian)
2. **bind()** = Đăng ký địa chỉ kinh doanh (gán IP:port)
3. **listen()** = Treo biển "OPEN", chuẩn bị bàn ghế (sẵn sàng phục vụ)
4. **accept()** = Đón khách vào, xếp bàn (tạo connection riêng cho mỗi client)
5. **read/write** = Phục vụ (truyền data)
6. **close()** = Thanh toán, khách ra về (đóng connection)

### TCP Server — Complete C Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main() {
    // ===== 1. SOCKET() — Tạo socket =====
    int server_fd = socket(AF_INET,      // IPv4
                           SOCK_STREAM,   // TCP
                           0);            // Default protocol
    if (server_fd < 0) {
        perror("socket failed");
        exit(1);
    }
    
    // Socket options — tránh "Address already in use"
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // ===== 2. BIND() — Gán address =====
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;      // Listen on all interfaces
    addr.sin_port = htons(8080);             // Port 8080 (network byte order!)
    
    if (bind(server_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind failed");
        exit(1);
    }
    
    // ===== 3. LISTEN() — Start accepting =====
    if (listen(server_fd, 128) < 0) {  // backlog = 128 pending connections
        perror("listen failed");
        exit(1);
    }
    printf("Server listening on port 8080...\n");
    
    // ===== 4. ACCEPT() — Wait for client =====
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        
        int client_fd = accept(server_fd, 
                              (struct sockaddr*)&client_addr,
                              &client_len);
        if (client_fd < 0) {
            perror("accept failed");
            continue;
        }
        
        printf("Client connected: %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));
        
        // ===== 5. READ/WRITE — Communication =====
        char buffer[1024];
        int bytes_read = read(client_fd, buffer, sizeof(buffer)-1);
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Received: %s\n", buffer);
            
            // Send response
            const char *response = "HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nHello";
            write(client_fd, response, strlen(response));
        }
        
        // ===== 6. CLOSE() — Disconnect =====
        close(client_fd);
    }
    
    close(server_fd);
    return 0;
}
```

### TCP Client — connect()

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main() {
    // 1. Socket
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    
    // 2. Connect (no bind needed — OS assigns ephemeral port)
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);
    
    if (connect(sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect failed");
        return 1;
    }
    printf("Connected to server!\n");
    
    // 3. Send request
    const char *request = "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n";
    write(sock, request, strlen(request));
    
    // 4. Receive response
    char buffer[4096];
    int bytes = read(sock, buffer, sizeof(buffer)-1);
    buffer[bytes] = '\0';
    printf("Response: %s\n", buffer);
    
    // 5. Close
    close(sock);
    return 0;
}
```

### Python Equivalent (dễ đọc hơn)

```python
# === TCP Server ===
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))
server.listen(128)
print("Server listening on port 8080...")

while True:
    client_sock, client_addr = server.accept()
    print(f"Client connected: {client_addr}")
    
    data = client_sock.recv(1024)
    print(f"Received: {data.decode()}")
    
    client_sock.send(b"Hello from server!")
    client_sock.close()
```

```python
# === TCP Client ===
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('127.0.0.1', 8080))

sock.send(b"Hello from client!")
response = sock.recv(1024)
print(f"Response: {response.decode()}")

sock.close()
```

---

## 4. I/O Models — Cách xử lý nhiều connections

### Phép so sánh — Nhà hàng phục vụ nhiều bàn

**Model 1: Blocking I/O (1 phục vụ/bàn)**
- Mỗi bàn có 1 nhân viên đứng chờ
- Khách không gọi → nhân viên đứng **chờ** (block)
- 1000 bàn = 1000 nhân viên! (1000 threads!)
- Tốn tài nguyên!

**Model 2: Non-blocking I/O (nhân viên đi vòng)**
- 1 nhân viên đi qua từng bàn: "Anh cần gì không?"
- "Chưa" → đi bàn tiếp
- "Có!" → phục vụ
- Nhân viên **liên tục đi vòng** (busy polling — tốn CPU!)

**Model 3: I/O Multiplexing (bell system)**
- Mỗi bàn có **nút bấm chuông**
- Nhân viên ngồi 1 chỗ, chờ chuông rung
- Bàn 5 bấm chuông → đi phục vụ bàn 5
- Hiệu quả! 1 nhân viên phục vụ **hàng nghìn bàn**!

**Model 4: Async I/O (đội ship tự phục vụ)**
- Khách gọi → hệ thống tự xử lý → báo lại khi xong
- Nhân viên không cần chờ hay kiểm tra
- Hoàn toàn async!

### Model 1: Blocking I/O + Threading

```python
# Simple but doesn't scale:
import socket
import threading

def handle_client(client_sock, addr):
    """Mỗi client 1 thread — BLOCKS on recv()"""
    while True:
        data = client_sock.recv(1024)  # ← BLOCKS here until data arrives!
        if not data:
            break
        client_sock.send(data.upper())
    client_sock.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8080))
server.listen(128)

while True:
    client, addr = server.accept()  # ← BLOCKS here until new connection!
    thread = threading.Thread(target=handle_client, args=(client, addr))
    thread.start()  # New thread for EACH client!

# Problems:
# - 10,000 clients = 10,000 threads!
# - Each thread: ~8MB stack memory
# - 10K threads × 8MB = 80GB RAM needed??
# - Context switching overhead
# - C10K problem!
```

### Model 2: Non-blocking I/O (Busy Polling)

```python
import socket
import os

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8080))
server.listen(128)
server.setblocking(False)  # NON-BLOCKING!

clients = []

while True:
    # Try accept — won't block (returns immediately)
    try:
        client, addr = server.accept()
        client.setblocking(False)
        clients.append(client)
    except BlockingIOError:
        pass  # No new connection — continue
    
    # Check each client — won't block
    for client in clients[:]:
        try:
            data = client.recv(1024)  # Won't block!
            if data:
                client.send(data.upper())
            else:
                clients.remove(client)
                client.close()
        except BlockingIOError:
            pass  # No data ready — continue

# Problem: BUSY LOOP! CPU 100% even when no data!
# Wastes CPU cycling through all clients constantly
```

---

## 5. I/O Multiplexing — select(), poll(), epoll

### select() — Oldest (1983)

```c
// select() — watch multiple fds, block until one is ready
#include <sys/select.h>

fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(server_fd, &read_fds);  // Watch server socket
FD_SET(client_fd, &read_fds);  // Watch client socket

struct timeval timeout = {5, 0};  // 5 second timeout

// Block until at least 1 fd is ready:
int ready = select(max_fd + 1, &read_fds, NULL, NULL, &timeout);

if (FD_ISSET(server_fd, &read_fds)) {
    // New connection available! accept() won't block
    int new_client = accept(server_fd, ...);
}
if (FD_ISSET(client_fd, &read_fds)) {
    // Data available! recv() won't block
    int bytes = recv(client_fd, buffer, sizeof(buffer), 0);
}
```

**select() limitations:**
- **FD_SETSIZE** = 1024 (max 1024 fds on most systems!)
- **Linear scan** — kernel checks ALL fds every time
- **Copy** — must copy fd_set to kernel every call
- **Rebuild** — must rebuild fd_set after each call

### poll() — Better than select (1986)

```c
#include <poll.h>

struct pollfd fds[1024];
fds[0].fd = server_fd;
fds[0].events = POLLIN;  // Watch for readable

fds[1].fd = client_fd;
fds[1].events = POLLIN;

int nfds = 2;

// Block until at least 1 fd ready:
int ready = poll(fds, nfds, 5000);  // 5000ms timeout

for (int i = 0; i < nfds; i++) {
    if (fds[i].revents & POLLIN) {
        if (fds[i].fd == server_fd) {
            // New connection
            int new_client = accept(server_fd, ...);
        } else {
            // Data available
            int bytes = recv(fds[i].fd, buffer, sizeof(buffer), 0);
        }
    }
}
```

**poll() improvements over select:**
- No FD_SETSIZE limit (dynamic array)
- Events/revents separated (no rebuild needed)

**Still has:**
- Linear scan (O(n) per call)
- Copy to kernel every call

### epoll() — Linux Solution (2002)

```c
#include <sys/epoll.h>

// 1. Create epoll instance
int epoll_fd = epoll_create1(0);

// 2. Add fds to watch (one-time registration!)
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = server_fd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev);

ev.data.fd = client_fd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);

// 3. Wait for events
struct epoll_event events[1024];
while (1) {
    // ONLY returns READY fds! (not all fds)
    int num_ready = epoll_wait(epoll_fd, events, 1024, -1);
    
    for (int i = 0; i < num_ready; i++) {
        if (events[i].data.fd == server_fd) {
            // New connection
            int new_client = accept(server_fd, ...);
            // Add new client to epoll
            ev.events = EPOLLIN | EPOLLET;  // Edge-triggered
            ev.data.fd = new_client;
            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, new_client, &ev);
        } else {
            // Data available on client socket
            char buffer[4096];
            int bytes = recv(events[i].data.fd, buffer, sizeof(buffer), 0);
            if (bytes <= 0) {
                // Client disconnected
                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                close(events[i].data.fd);
            } else {
                // Process data...
                send(events[i].data.fd, buffer, bytes, 0);
            }
        }
    }
}
```

### epoll Trigger Modes

```
Level-Triggered (LT — default):
  - epoll_wait() trả về fd NẾU fd CÓ data pending
  - Trả về LẶP LẠI mỗi lần gọi epoll_wait() cho đến khi hết data
  - Simple, forgiving (OK nếu không đọc hết)
  - Similar to poll() behavior

Edge-Triggered (ET):
  - epoll_wait() trả về fd CHỈ KHI có data MỚI đến
  - Trả về 1 LẦN — nếu bạn không đọc hết, sẽ KHÔNG báo lại!
  - PHẢI đọc hết data (loop recv until EAGAIN)
  - Higher performance (fewer epoll_wait() returns)
  - Dùng bởi: nginx, Redis

Edge-Triggered pattern:
  events[i].events = EPOLLIN | EPOLLET;  // Enable ET
  
  // Must read ALL available data:
  while (1) {
      int bytes = recv(fd, buffer, sizeof(buffer), 0);
      if (bytes < 0) {
          if (errno == EAGAIN || errno == EWOULDBLOCK)
              break;  // No more data — done
      } else if (bytes == 0) {
          // Connection closed
          break;
      }
      // Process bytes...
  }
```

### kqueue — BSD/macOS Solution

```c
#include <sys/event.h>

// 1. Create kqueue
int kq = kqueue();

// 2. Register events
struct kevent changes[2];
EV_SET(&changes[0], server_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
EV_SET(&changes[1], client_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);

kevent(kq, changes, 2, NULL, 0, NULL);  // Register

// 3. Wait for events
struct kevent events[64];
while (1) {
    int num_events = kevent(kq, NULL, 0, events, 64, NULL);
    
    for (int i = 0; i < num_events; i++) {
        int fd = events[i].ident;
        if (fd == server_fd) {
            // New connection
            int new_client = accept(server_fd, ...);
            struct kevent new_ev;
            EV_SET(&new_ev, new_client, EVFILT_READ, EV_ADD, 0, 0, NULL);
            kevent(kq, &new_ev, 1, NULL, 0, NULL);
        } else {
            // Data available
            recv(fd, buffer, sizeof(buffer), 0);
        }
    }
}
```

### Performance Comparison

| Mechanism | Time Complexity | FD Limit | Platforms | Best For |
|---|---|---|---|---|
| select() | O(n) per call | 1024 | All Unix | < 100 connections |
| poll() | O(n) per call | No limit | All Unix | < 10,000 connections |
| **epoll()** | **O(1) per ready fd** | No limit | **Linux** | **10K-1M connections** |
| **kqueue()** | **O(1) per ready fd** | No limit | **BSD/macOS** | **10K-1M connections** |
| io_uring | O(1), kernel-side | No limit | Linux 5.1+ | Max performance |

---

## 6. Event Loop Pattern — Nền tảng Server hiện đại

### Phép so sánh — 1 đầu bếp, nhiều món

Thay vì 1 đầu bếp/món (threading), 1 đầu bếp giỏi nấu **nhiều món cùng lúc**:
1. Bỏ thịt vào lò → timer 30 min
2. Luộc rau → timer 5 min
3. Nấu nước sốt → timer 15 min
4. Khi timer nào kêu → xử lý món đó → set timer tiếp
5. Không ai đứng **CHỜ** — luôn làm việc khác!

### Event Loop Architecture

```
┌─────────────────────────────────────────────────┐
│                 Event Loop                       │
│                                                 │
│  while (running):                               │
│    ┌─────────────────────┐                     │
│    │ epoll_wait() / kevent│ ← Block until event│
│    └─────────┬───────────┘                     │
│              │                                  │
│    ┌─────────▼───────────┐                     │
│    │ For each ready fd:  │                     │
│    │   - New connection? │→ accept + register  │
│    │   - Data readable?  │→ read + process     │
│    │   - Writable?       │→ write buffered data│
│    │   - Error/Close?    │→ cleanup            │
│    └─────────────────────┘                     │
│                                                 │
│    ┌─────────────────────┐                     │
│    │ Run timers/callbacks │ ← Scheduled tasks  │
│    └─────────────────────┘                     │
│                                                 │
└─────────────────────────────────────────────────┘

Single thread — handles thousands of connections!
Used by: nginx, Redis, Node.js, HAProxy
```

### Python Event Loop (asyncio)

```python
import asyncio

async def handle_client(reader, writer):
    """Handle one client — cooperative multitasking"""
    addr = writer.get_extra_info('peername')
    print(f"Connected: {addr}")
    
    while True:
        data = await reader.read(1024)  # Yield to event loop until data arrives
        if not data:
            break
        
        message = data.decode()
        print(f"Received '{message}' from {addr}")
        
        writer.write(data.upper())
        await writer.drain()  # Yield until write buffer flushed
    
    writer.close()
    print(f"Disconnected: {addr}")

async def main():
    server = await asyncio.start_server(handle_client, '0.0.0.0', 8080)
    async with server:
        await server.serve_forever()

asyncio.run(main())

# Single thread, handles 10,000+ connections!
# await = "yield to event loop" (not blocking!)
```

### Real-world Servers using Event Loop

```
nginx:
  - Master process + Worker processes
  - Each worker: single-threaded event loop (epoll/kqueue)
  - 1 worker handles 10,000+ concurrent connections
  - Configuration: worker_connections 10240;

Redis:
  - Single-threaded (main loop)
  - All commands processed sequentially
  - epoll for connection handling
  - 100K+ operations/second (single core!)
  - Thread safety: no locks needed!

Node.js:
  - libuv event loop (epoll on Linux, kqueue on macOS)
  - Single JS thread + thread pool for file I/O
  - Excellent for I/O-bound workloads
  - Bad for CPU-bound (blocks event loop!)

HAProxy:
  - Multi-process, each with event loop
  - Millions of concurrent connections
  - < 1ms latency for proxying
```

---

## 7. Socket Options — Tùy chỉnh hành vi

### Common Socket Options

```c
// === SO_REUSEADDR ===
// Cho phép bind port đang ở TIME_WAIT
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
// LUÔN SET cho servers! Tránh "Address already in use" khi restart

// === SO_REUSEPORT ===
// Nhiều processes/threads bind CÙNG port
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
// Load balance connections across processes (kernel handles)
// Used by: nginx, envoy

// === SO_KEEPALIVE ===
// Enable TCP keepalive probes
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
// Detect dead connections

// === TCP_NODELAY ===
// Disable Nagle's algorithm (send immediately)
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
// For: real-time apps, games, SSH

// === SO_RCVBUF / SO_SNDBUF ===
// Set receive/send buffer sizes
int bufsize = 1048576;  // 1MB
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));

// === SO_LINGER ===
// Control close() behavior
struct linger ling = {1, 0};  // linger_on=1, timeout=0
setsockopt(fd, SOL_SOCKET, SO_LINGER, &ling, sizeof(ling));
// timeout=0: close() sends RST immediately (abort, no TIME_WAIT)
// timeout>0: close() blocks until data sent or timeout

// === TCP_QUICKACK ===
// Disable delayed ACK (Linux only)
setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &opt, sizeof(opt));
// Reduce latency for request-response patterns
```

### Non-blocking Mode

```c
// Method 1: fcntl
#include <fcntl.h>
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// Method 2: socket creation (Linux 2.6.27+)
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

// Method 3: accept4 (Linux)
int client = accept4(server_fd, &addr, &addrlen, SOCK_NONBLOCK);

// Non-blocking behavior:
//   recv() → returns -1 with errno=EAGAIN if no data
//   send() → returns -1 with errno=EAGAIN if buffer full
//   connect() → returns -1 with errno=EINPROGRESS (check later)
//   accept() → returns -1 with errno=EAGAIN if no connection
```

---

## 8. UDP Socket Programming

### UDP Server/Client

```python
# === UDP Server (simple!) ===
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # DGRAM = UDP
sock.bind(('0.0.0.0', 9999))
print("UDP server listening on port 9999")

while True:
    data, client_addr = sock.recvfrom(65535)  # Max UDP datagram
    print(f"Received from {client_addr}: {data.decode()}")
    sock.sendto(b"ACK: " + data, client_addr)  # Reply to sender

# Note: NO listen(), NO accept()!
# UDP = connectionless → just recv from anyone!
```

```python
# === UDP Client ===
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto(b"Hello UDP!", ('127.0.0.1', 9999))  # No connect needed!

data, server_addr = sock.recvfrom(65535)
print(f"Response: {data.decode()}")
sock.close()
```

### Multicast Socket

```python
# === Multicast Receiver ===
import socket
import struct

MCAST_GROUP = '224.1.1.1'
MCAST_PORT = 5007

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('', MCAST_PORT))

# Join multicast group
mreq = struct.pack('4sL', socket.inet_aton(MCAST_GROUP), socket.INADDR_ANY)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

while True:
    data, addr = sock.recvfrom(1024)
    print(f"Multicast from {addr}: {data.decode()}")
```

---

## 9. Advanced Patterns — Connection Pooling, Graceful Shutdown

### Connection Pool

```python
import socket
import queue
import threading

class ConnectionPool:
    """Reuse TCP connections instead of creating new ones each time"""
    
    def __init__(self, host, port, max_size=10):
        self.host = host
        self.port = port
        self.pool = queue.Queue(max_size)
        self.lock = threading.Lock()
        
    def get_connection(self):
        """Get a connection from pool or create new"""
        try:
            conn = self.pool.get_nowait()
            # Verify connection still alive
            try:
                conn.getpeername()  # Raises if disconnected
                return conn
            except:
                pass  # Dead connection, create new
        except queue.Empty:
            pass
        
        # Create new connection
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((self.host, self.port))
        return sock
    
    def return_connection(self, conn):
        """Return connection to pool for reuse"""
        try:
            self.pool.put_nowait(conn)
        except queue.Full:
            conn.close()  # Pool full, discard

# Usage:
pool = ConnectionPool('db-server', 5432, max_size=20)
conn = pool.get_connection()
# ... use connection ...
pool.return_connection(conn)  # Don't close! Return to pool!
```

### Graceful Shutdown

```python
import signal
import socket
import select

class GracefulServer:
    def __init__(self, host, port):
        self.running = True
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server.bind((host, port))
        self.server.listen(128)
        self.server.setblocking(False)
        self.clients = []
        
        # Handle SIGTERM/SIGINT gracefully
        signal.signal(signal.SIGTERM, self._shutdown)
        signal.signal(signal.SIGINT, self._shutdown)
    
    def _shutdown(self, signum, frame):
        """Graceful shutdown: finish existing, reject new"""
        print(f"\nShutting down gracefully...")
        self.running = False
        
        # Stop accepting new connections
        self.server.close()
        
        # Wait for existing clients to finish
        for client in self.clients:
            client.shutdown(socket.SHUT_WR)  # Send FIN
        
        # Give clients time to finish
        import time
        time.sleep(2)
        
        # Force close remaining
        for client in self.clients:
            client.close()
    
    def run(self):
        while self.running:
            readable, _, _ = select.select(
                [self.server] + self.clients, [], [], 1.0
            )
            for sock in readable:
                if sock == self.server:
                    client, addr = self.server.accept()
                    client.setblocking(False)
                    self.clients.append(client)
                else:
                    data = sock.recv(1024)
                    if not data:
                        self.clients.remove(sock)
                        sock.close()
                    else:
                        sock.send(data.upper())
```

---

## 10. Tổng kết và C10K/C10M Problem

### C10K Problem (1999)

```
Challenge: Handle 10,000 concurrent connections on 1 server

Solutions that emerged:
  1. epoll/kqueue (event-driven I/O)
  2. Non-blocking sockets
  3. Event loops (single-threaded)
  4. Connection pooling
  5. Zero-copy (sendfile, splice)
  
Result: nginx, Node.js, Go, etc. easily handle 10K+
```

### C10M Problem (2010+)

```
Challenge: Handle 10,000,000 concurrent connections!

Solutions:
  1. io_uring (Linux 5.1+) — async I/O, kernel-side processing
  2. DPDK/XDP — bypass kernel networking stack
  3. SO_REUSEPORT — multi-core load balancing
  4. Zero-copy networking (MSG_ZEROCOPY)
  5. Huge pages — reduce TLB misses
  6. NUMA-aware — keep data close to CPU
  
  Tools: Seastar (ScyllaDB), DPDK apps, AF_XDP
```

### Socket Programming Summary

```
┌─────────────────────────────────────────────────────────┐
│ Socket System Calls:                                    │
│                                                         │
│ Server: socket→bind→listen→accept→read/write→close     │
│ Client: socket→connect→read/write→close                │
│                                                         │
│ I/O Models (evolution):                                 │
│ 1. Blocking + Threading (simple, doesn't scale)        │
│ 2. Non-blocking polling (wastes CPU)                   │
│ 3. select/poll (O(n), limited)                         │
│ 4. epoll/kqueue (O(1), modern standard)               │
│ 5. io_uring (async, highest performance)               │
│                                                         │
│ Key Socket Options:                                     │
│ • SO_REUSEADDR (servers: avoid "address in use")       │
│ • SO_REUSEPORT (multi-process load balancing)          │
│ • TCP_NODELAY (disable Nagle for low-latency)          │
│ • SO_KEEPALIVE (detect dead connections)               │
│ • O_NONBLOCK (non-blocking mode)                       │
│                                                         │
│ Production Architecture:                                │
│ • Event loop (single-threaded, epoll/kqueue)           │
│ • Connection pooling (reuse, don't recreate)           │
│ • Graceful shutdown (finish in-flight, reject new)     │
│ • Buffer management (avoid copying)                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- Beej's Guide to Network Programming — beej.us/guide/bgnet
- Stevens, W.R. — UNIX Network Programming, Volume 1 (The Sockets Bible)
- Linux man pages: socket(2), bind(2), listen(2), accept(2), epoll(7)
- FreeBSD man pages: kqueue(2)
- RFC 9293 — TCP (socket behavior defined)
- Linux io_uring documentation — kernel.org
- The C10K Problem — Dan Kegel (kegel.com/c10k.html)
- nginx Architecture — nginx.org/en/docs
- Redis Event Library — redis.io/docs/reference/internals/internals-rediseventlib
