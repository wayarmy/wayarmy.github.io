---
layout: post
title: "gRPC & Protocol Buffers Deep Dive - Binary Protocol, HTTP/2, Streaming, .proto, Code Generation & vs REST"
date: 2026-06-01
categories: [networking]
tags: [grpc, protobuf, http2, microservices, rpc]
---

# gRPC & Protocol Buffers Deep Dive — Binary Protocol, HTTP/2, Streaming, .proto, Code Generation & vs REST

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [gRPC Fundamentals — Nền tảng](#2-grpc-fundamentals--nền-tảng)
3. [Protocol Buffers — Ngôn ngữ mô tả dữ liệu](#3-protocol-buffers--ngôn-ngữ-mô-tả-dữ-liệu)
4. [HTTP/2 — Lớp vận chuyển của gRPC](#4-http2--lớp-vận-chuyển-của-grpc)
5. [Streaming Types — Bốn kiểu giao tiếp](#5-streaming-types--bốn-kiểu-giao-tiếp)
6. [.proto File và Code Generation](#6-proto-file-và-code-generation)
7. [gRPC vs REST — So sánh chi tiết](#7-grpc-vs-rest--so-sánh-chi-tiết)
8. [Advanced Features — Tính năng nâng cao](#8-advanced-features--tính-năng-nâng-cao)
9. [Hands-on Implementation](#9-hands-on-implementation)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### RPC như gọi điện nhờ ai đó làm việc

Hãy tưởng tượng bạn cần tính thuế thu nhập. Thay vì tự tính (phức tạp, dễ sai), bạn **gọi điện cho kế toán** và nói: "Tính thuế cho tôi với thu nhập 50 triệu". Kế toán tính xong, nói lại kết quả. Bạn cảm thấy như mình vừa "gọi hàm tinhThue(50000000)" — nhưng thực tế hàm đó chạy ở máy tính của kế toán, không phải máy bạn.

**RPC (Remote Procedure Call)** — Gọi hàm từ xa — hoạt động đúng như vậy: code của bạn "gọi hàm" nhưng hàm đó thực thi trên server khác.

| Cuộc gọi kế toán | gRPC |
|---|---|
| Bạn gọi điện | Client gọi remote method |
| Nói bằng tiếng Việt (cả hai hiểu) | Dùng Protocol Buffers (format chung) |
| Đường dây điện thoại | HTTP/2 connection |
| Kế toán tính toán | Server xử lý logic |
| Kế toán trả lời kết quả | Server trả response |
| Bạn không cần biết kế toán dùng Excel hay máy tính | Client không cần biết server viết bằng ngôn ngữ gì |

### gRPC trong hệ sinh thái hiện đại

```
┌─────────────┐     gRPC      ┌──────────────┐
│  Go Service │ ←───────────→ │ Python Service│
└─────────────┘               └──────────────┘
       ↑                              ↑
       │ gRPC                         │ gRPC
       ↓                              ↓
┌─────────────┐               ┌──────────────┐
│ Java Service│               │ Node.js BFF   │
└─────────────┘               └──────────────┘
                                      ↑
                                      │ REST/GraphQL
                                      ↓
                              ┌──────────────┐
                              │   Browser     │
                              └──────────────┘
```

---

## 2. gRPC Fundamentals — Nền tảng

### 2.1 gRPC là gì?

**gRPC** (g = originally "Google", now the "g" means different things each release) là:
- **Open-source** RPC framework, phát triển bởi Google (2015)
- Dùng **Protocol Buffers (Protobuf)** cho serialization
- Dùng **HTTP/2** cho transport
- Hỗ trợ **nhiều ngôn ngữ**: Go, Java, Python, C++, Node.js, C#, Ruby, Kotlin, Swift, Dart...
- **IDL-first** (Interface Definition Language first) — define contract trước, generate code sau

### 2.2 Kiến trúc gRPC

```
┌──────────────────────────────────────────────────────────┐
│                    gRPC Architecture                       │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐            ┌──────────────┐           │
│  │  gRPC Client  │            │  gRPC Server  │           │
│  │               │            │               │           │
│  │  ┌─────────┐ │            │ ┌───────────┐ │           │
│  │  │Generated │ │            │ │ Service    │ │           │
│  │  │  Stub    │ │  HTTP/2    │ │ Implement  │ │           │
│  │  │ (proxy)  │←┼───────────→┼ │ (your code)│ │           │
│  │  └─────────┘ │  Protobuf  │ └───────────┘ │           │
│  │       ↑       │  (binary)  │       ↑       │           │
│  │       │       │            │       │       │           │
│  │  ┌─────────┐ │            │ ┌───────────┐ │           │
│  │  │  Your   │ │            │ │ Generated  │ │           │
│  │  │  Code   │ │            │ │  Server    │ │           │
│  │  │ (caller)│ │            │ │  Skeleton  │ │           │
│  │  └─────────┘ │            │ └───────────┘ │           │
│  └──────────────┘            └──────────────┘           │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐│
│  │              .proto file (contract)                    ││
│  │  → protoc generates client stub + server skeleton     ││
│  └──────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

### 2.3 Workflow phát triển với gRPC

```
Bước 1: Viết .proto file (define service + messages)
         ↓
Bước 2: Chạy protoc compiler → generate code
         ↓ (tự động)
Bước 3: Server: Implement service methods
         Client: Gọi methods qua generated stub
         ↓
Bước 4: Run — Client gọi server qua HTTP/2 + Protobuf
```

---

## 3. Protocol Buffers — Ngôn ngữ mô tả dữ liệu

### 3.1 Protobuf là gì?

**Protocol Buffers (Protobuf)** là format **serialization nhị phân (binary)** do Google phát triển. Nó:
- Nhỏ hơn JSON 3-10x
- Parse nhanh hơn JSON 20-100x
- Strongly typed (kiểu dữ liệu chặt chẽ)
- Schema-driven (có .proto file mô tả cấu trúc)

**Ví dụ đời thường**: 
- JSON giống viết thư bằng tiếng Việt: ai đọc cũng hiểu, nhưng dài
- Protobuf giống mã Morse: ngắn gọn, nhanh truyền, nhưng cần bảng mã (schema) để đọc

### 3.2 So sánh JSON vs Protobuf

**Cùng dữ liệu — Person với name="Alice", age=30, email="alice@example.com":**

```json
// JSON: 71 bytes (human-readable)
{"name":"Alice","age":30,"email":"alice@example.com"}
```

```protobuf
// Protobuf binary: 35 bytes (binary, không đọc trực tiếp)
0A 05 41 6C 69 63 65 10 1E 1A 11 61 6C 69 63 65
40 65 78 61 6D 70 6C 65 2E 63 6F 6D
```

**Tại sao Protobuf nhỏ hơn?**
- Không lặp field names (dùng field number thay vì "name", "age")
- Không có delimiters (`{`, `}`, `,`, `:`, `"`)
- Dùng varint encoding cho integers (30 = 1 byte thay vì 2 bytes "30")

### 3.3 Protobuf Encoding Deep Dive

```protobuf
message Person {
  string name = 1;    // field number 1
  int32 age = 2;      // field number 2
  string email = 3;   // field number 3
}
```

**Wire format:**
```
Mỗi field = [Tag] [Value]
Tag = (field_number << 3) | wire_type

Wire types:
0 = Varint (int32, int64, uint32, uint64, bool, enum)
1 = 64-bit (fixed64, sfixed64, double)
2 = Length-delimited (string, bytes, embedded messages, repeated fields)
5 = 32-bit (fixed32, sfixed32, float)
```

```
Encoding "Alice" (field 1, string):
0A           = Tag: field 1, wire type 2 (length-delimited)
              (1 << 3) | 2 = 0x0A
05           = Length: 5 bytes
41 6C 69 63 65 = "Alice" in UTF-8

Encoding age=30 (field 2, int32):
10           = Tag: field 2, wire type 0 (varint)
              (2 << 3) | 0 = 0x10
1E           = Varint 30 (fits in 1 byte)
```

### 3.4 Scalar Types

| .proto Type | Go | Java | Python | Notes |
|---|---|---|---|---|
| `double` | float64 | double | float | 64-bit IEEE 754 |
| `float` | float32 | float | float | 32-bit IEEE 754 |
| `int32` | int32 | int | int | Varint, inefficient for negative |
| `int64` | int64 | long | int | Varint |
| `uint32` | uint32 | int | int | Unsigned varint |
| `sint32` | int32 | int | int | ZigZag encoding (efficient for negatives) |
| `bool` | bool | boolean | bool | |
| `string` | string | String | str | UTF-8 |
| `bytes` | []byte | ByteString | bytes | Arbitrary binary |

### 3.5 Complex Types

```protobuf
syntax = "proto3";

package ecommerce;

import "google/protobuf/timestamp.proto";

// Enum
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // Luôn có zero value
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

// Nested message
message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

// Main message with various types
message Order {
  string order_id = 1;                          // Scalar
  repeated OrderItem items = 2;                 // List (repeated)
  Address shipping_address = 3;                 // Nested message
  OrderStatus status = 4;                       // Enum
  map<string, string> metadata = 5;            // Map
  google.protobuf.Timestamp created_at = 6;    // Well-known type
  oneof payment_method {                        // One-of (union type)
    CreditCard credit_card = 7;
    BankTransfer bank_transfer = 8;
  }
  optional string notes = 9;                    // Optional field
}

message OrderItem {
  string product_id = 1;
  string name = 2;
  int32 quantity = 3;
  double price = 4;
}

message CreditCard {
  string last_four = 1;
  string brand = 2;
}

message BankTransfer {
  string bank_name = 1;
  string account_number = 2;
}
```

### 3.6 Backward/Forward Compatibility

Protobuf được thiết kế để **schema evolution** an toàn:

```protobuf
// Version 1
message User {
  string name = 1;
  string email = 2;
}

// Version 2 (SAFE changes):
message User {
  string name = 1;
  string email = 2;
  string phone = 3;        // ✅ Thêm field mới (old clients ignore)
  // Field 4 removed        // ✅ Xóa field (old field number reserved)
  reserved 4;              // ✅ Reserve field number (tránh reuse)
  reserved "old_field";    // ✅ Reserve field name
}
```

**Rules cho compatibility:**

| Hành động | An toàn? | Lý do |
|---|---|---|
| Thêm field mới | ✅ | Old clients bỏ qua unknown fields |
| Xóa field | ✅ | New clients nhận default value |
| Đổi field number | ❌ | Binary encoding dựa trên number |
| Đổi field type | ❌ | Wire format không tương thích |
| Rename field | ✅ | Binary chỉ dùng number, không dùng name |
| Đổi repeated ↔ scalar | ❌ | Wire format khác |

---

## 4. HTTP/2 — Lớp vận chuyển của gRPC

### 4.1 Tại sao gRPC dùng HTTP/2?

**HTTP/1.1 limitations:**
- **Head-of-line blocking**: Mỗi connection chỉ xử lý 1 request tại 1 thời điểm
- **Không multiplexing**: Muốn song song → mở nhiều TCP connections
- **Header overhead**: Gửi lại toàn bộ headers mỗi request (no compression)

**HTTP/2 advantages cho gRPC:**

| Feature | Benefit cho gRPC |
|---|---|
| **Multiplexing** | Nhiều RPC calls song song trên 1 TCP connection |
| **Header compression** (HPACK) | Giảm metadata overhead |
| **Binary framing** | Hiệu quả hơn text parsing |
| **Server push** | Basis cho server streaming |
| **Flow control** | Backpressure cho streaming |
| **Stream prioritization** | Ưu tiên RPC quan trọng |

### 4.2 gRPC over HTTP/2 Flow

```
┌─────────────────────────────────────────────────────┐
│              Single TCP Connection                    │
│                                                       │
│  Stream 1: CreateOrder RPC                           │
│  ┌─────────────────────────────────────────────────┐│
│  │ HEADERS frame: :method POST, :path /svc/Method  ││
│  │ DATA frame: Protobuf request (length-prefixed)  ││
│  │ ... server processing ...                        ││
│  │ HEADERS frame: :status 200                      ││
│  │ DATA frame: Protobuf response                   ││
│  │ HEADERS frame (trailers): grpc-status 0         ││
│  └─────────────────────────────────────────────────┘│
│                                                       │
│  Stream 3: GetUser RPC (CÙNG LÚC với Stream 1)      │
│  ┌─────────────────────────────────────────────────┐│
│  │ HEADERS + DATA + response...                    ││
│  └─────────────────────────────────────────────────┘│
│                                                       │
│  Stream 5: HealthCheck (CÙNG LÚC)                   │
│  ┌─────────────────────────────────────────────────┐│
│  │ HEADERS + DATA + response...                    ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

### 4.3 gRPC Request Anatomy

```
HTTP/2 Request:
  :method = POST
  :scheme = http (or https)
  :path = /{package}.{ServiceName}/{MethodName}
  :authority = server:port
  content-type = application/grpc
  te = trailers
  grpc-timeout = 5S
  grpc-encoding = gzip
  
  DATA frame:
    [Compressed Flag (1 byte)] [Message Length (4 bytes)] [Message (protobuf)]
    [0x00]                     [0x00 0x00 0x00 0x23]     [35 bytes of protobuf]
```

```
HTTP/2 Response:
  :status = 200
  content-type = application/grpc
  grpc-encoding = gzip
  
  DATA frame:
    [0x00] [0x00 0x00 0x00 0x42] [66 bytes of protobuf response]
  
  TRAILERS:
    grpc-status = 0 (OK)
    grpc-message = (empty, success)
```

### 4.4 Length-Prefixed Message Framing

```
gRPC message format trên wire:
┌────────────────────┬───────────────────────┬──────────────────────┐
│ Compressed (1 byte)│ Message Length (4 bytes)│ Serialized Protobuf  │
│ 0 = no, 1 = yes   │ big-endian uint32      │ (variable length)    │
└────────────────────┴───────────────────────┴──────────────────────┘

Ví dụ:
00 00000023 [35 bytes protobuf data]
^^          ^^^^^^^^
uncompressed  length=35
```

---

## 5. Streaming Types — Bốn kiểu giao tiếp

### 5.1 Overview

gRPC hỗ trợ **4 kiểu** giao tiếp:

| Kiểu | Client gửi | Server trả | Ví dụ đời thường |
|---|---|---|---|
| **Unary** | 1 message | 1 message | Hỏi 1 câu → nhận 1 câu trả lời |
| **Server Streaming** | 1 message | N messages | Đặt báo → nhận báo mỗi ngày |
| **Client Streaming** | N messages | 1 message | Upload nhiều file → nhận 1 kết quả |
| **Bidirectional Streaming** | N messages | N messages | Chat — cả hai gửi/nhận liên tục |

### 5.2 Unary RPC (Request-Response đơn giản)

```protobuf
// .proto
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}
```

```
Client                          Server
  │                               │
  │── GetUserRequest ───────────→ │
  │                               │ (xử lý)
  │←── User response ──────────── │
  │                               │
```

**Sử dụng khi**: Mọi operation CRUD đơn giản — lấy 1 user, tạo 1 order, xóa 1 item.

### 5.3 Server Streaming RPC

```protobuf
service StockService {
  // Server gửi stream prices liên tục
  rpc WatchStock(StockRequest) returns (stream StockPrice);
}
```

```
Client                          Server
  │                               │
  │── StockRequest("AAPL") ─────→ │
  │                               │
  │←── StockPrice($150.00) ────── │
  │←── StockPrice($150.05) ────── │
  │←── StockPrice($149.98) ────── │
  │←── StockPrice($150.10) ────── │
  │←── ... (continues) ────────── │
  │                               │
```

**Sử dụng khi**: 
- Download large dataset (stream từng chunk)
- Real-time price feeds
- Log streaming
- Progress updates

### 5.4 Client Streaming RPC

```protobuf
service UploadService {
  // Client gửi nhiều chunks, server trả 1 kết quả
  rpc UploadFile(stream FileChunk) returns (UploadResult);
}
```

```
Client                          Server
  │                               │
  │── FileChunk (bytes 0-1MB) ──→ │
  │── FileChunk (bytes 1-2MB) ──→ │  (receiving...)
  │── FileChunk (bytes 2-3MB) ──→ │
  │── [END] ──────────────────→ │
  │                               │ (process complete file)
  │←── UploadResult ───────────── │
  │                               │
```

**Sử dụng khi**:
- File upload (chunked)
- Batch insert (nhiều records → 1 response)
- Sensor data collection

### 5.5 Bidirectional Streaming RPC

```protobuf
service ChatService {
  // Cả hai bên gửi/nhận liên tục
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

```
Client                          Server
  │                               │
  │── "Hello" ──────────────────→ │
  │←── "Hi there!" ────────────── │
  │── "How are you?" ───────────→ │
  │←── "Good!" ────────────────── │
  │── "Send me file X" ─────────→ │
  │←── [file data chunk 1] ────── │
  │←── [file data chunk 2] ────── │
  │── "Thanks!" ────────────────→ │
  │                               │
```

**Sử dụng khi**:
- Chat/messaging
- Multiplayer gaming
- Real-time collaboration
- Interactive voice/video

### 5.6 Streaming Flow Control

HTTP/2 cung cấp **flow control** — receiver nói cho sender biết "tôi sẵn sàng nhận thêm bao nhiêu":

```
Client                                 Server
  │                                      │
  │ WINDOW_UPDATE(65535)                 │  "Tôi có buffer 64KB"
  │──────────────────────────────────→   │
  │                                      │
  │         DATA (32KB)                  │  Server gửi 32KB
  │←─────────────────────────────────    │
  │                                      │
  │         DATA (32KB)                  │  Server gửi thêm 32KB
  │←─────────────────────────────────    │  (window đầy!)
  │                                      │
  │ WINDOW_UPDATE(32768)                 │  Client xử lý xong, mở thêm 32KB
  │──────────────────────────────────→   │
  │                                      │
  │         DATA (32KB)                  │  Server tiếp tục gửi
  │←─────────────────────────────────    │
```

---

## 6. .proto File và Code Generation

### 6.1 Service Definition

```protobuf
syntax = "proto3";

package order.v1;

option go_package = "github.com/mycompany/order/v1;orderv1";
option java_package = "com.mycompany.order.v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Service definition — defines all RPC methods
service OrderService {
  // Unary
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  
  // Server streaming
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  
  // Client streaming
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (BatchCreateResult);
  
  // Bidirectional streaming
  rpc OrderUpdates(stream OrderUpdateRequest) returns (stream OrderEvent);
}

// Request/Response messages
message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  Address shipping_address = 3;
}

message GetOrderRequest {
  string order_id = 1;
}

message ListOrdersRequest {
  string customer_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message Order {
  string order_id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  double total_amount = 5;
  google.protobuf.Timestamp created_at = 6;
}

message OrderItem {
  string product_id = 1;
  string name = 2;
  int32 quantity = 3;
  double unit_price = 4;
}

message Address {
  string line1 = 1;
  string line2 = 2;
  string city = 3;
  string state = 4;
  string postal_code = 5;
  string country = 6;
}

message BatchCreateResult {
  int32 total_created = 1;
  repeated string order_ids = 2;
  repeated string errors = 3;
}

message OrderUpdateRequest {
  string order_id = 1;
  OrderStatus new_status = 2;
}

message OrderEvent {
  string order_id = 1;
  OrderStatus old_status = 2;
  OrderStatus new_status = 3;
  google.protobuf.Timestamp timestamp = 4;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_PROCESSING = 3;
  ORDER_STATUS_SHIPPED = 4;
  ORDER_STATUS_DELIVERED = 5;
  ORDER_STATUS_CANCELLED = 6;
}
```

### 6.2 Code Generation

```bash
# Install protoc compiler
# macOS
brew install protobuf

# Generate Go code
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/order/v1/order.proto

# Generate Python code
python -m grpc_tools.protoc \
  --python_out=./generated \
  --pyi_out=./generated \
  --grpc_python_out=./generated \
  -I. proto/order/v1/order.proto

# Generate Java code
protoc --java_out=./src/main/java \
       --grpc-java_out=./src/main/java \
       proto/order/v1/order.proto

# Generate TypeScript (using ts-proto)
protoc --plugin=protoc-gen-ts_proto=./node_modules/.bin/protoc-gen-ts_proto \
       --ts_proto_out=./src/generated \
       proto/order/v1/order.proto
```

### 6.3 Generated Code Structure

```
Từ .proto file, protoc sinh ra:

For Go:
├── order.pb.go           # Message types (struct definitions)
│   ├── type Order struct { ... }
│   ├── func (*Order) Marshal() ([]byte, error)
│   └── func (*Order) Unmarshal([]byte) error
│
└── order_grpc.pb.go      # Service interface + client stub
    ├── type OrderServiceClient interface { ... }
    ├── type OrderServiceServer interface { ... }
    ├── func NewOrderServiceClient(cc grpc.ClientConnInterface) OrderServiceClient
    └── func RegisterOrderServiceServer(s *grpc.Server, srv OrderServiceServer)

For Python:
├── order_pb2.py          # Message classes
├── order_pb2.pyi         # Type stubs
└── order_pb2_grpc.py     # Service stubs + servicers
```

### 6.4 Buf — Modern Protobuf Tooling

```yaml
# buf.yaml — thay thế protoc, thân thiện hơn
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
lint:
  use:
    - DEFAULT
breaking:
  use:
    - FILE
```

```bash
# Lint proto files
buf lint

# Detect breaking changes
buf breaking --against '.git#branch=main'

# Generate code
buf generate

# Push to Buf Schema Registry (BSR)
buf push
```

---

## 7. gRPC vs REST — So sánh chi tiết

### 7.1 Bảng so sánh toàn diện

| Tiêu chí | REST | gRPC |
|---|---|---|
| **Philosophy** | Resource-oriented | Procedure-oriented |
| **Protocol** | HTTP/1.1 (thường) | HTTP/2 (bắt buộc) |
| **Data format** | JSON (text, ~100%) | Protobuf (binary) |
| **Contract** | OpenAPI/Swagger (optional) | .proto (bắt buộc) |
| **Code generation** | Optional (codegen tools) | Built-in (protoc) |
| **Streaming** | Limited (SSE, WebSocket riêng) | Native (4 kiểu) |
| **Browser support** | ✅ Native | ⚠️ Cần grpc-web proxy |
| **Human-readable** | ✅ JSON dễ đọc | ❌ Binary không đọc được |
| **Tooling** | curl, Postman, browser | grpcurl, BloomRPC, Postman |
| **Caching** | ✅ HTTP caching (ETag, Cache-Control) | ❌ Không có native caching |
| **Performance** | Baseline | 5-10x nhanh hơn |
| **Payload size** | Larger (JSON + headers) | Smaller (3-10x nhỏ hơn) |
| **Type safety** | Weak (JSON is dynamic) | Strong (.proto = contract) |
| **Versioning** | URL/header versioning | Package versioning |

### 7.2 Performance Comparison

```
Benchmark: 1000 requests, message 1KB payload

REST (JSON over HTTP/1.1):
├── Serialization: 50μs
├── Network overhead: 500 bytes headers/request
├── Parse: 80μs
├── Connections: 6 parallel (browser limit)
└── Total: ~200ms for 1000 requests

gRPC (Protobuf over HTTP/2):
├── Serialization: 5μs (10x faster)
├── Network overhead: 50 bytes headers (HPACK compressed)
├── Parse: 3μs (26x faster)
├── Connections: 1 (multiplexed)
└── Total: ~30ms for 1000 requests (6.7x faster overall)
```

### 7.3 Khi nào dùng gì?

| Use Case | Recommend | Lý do |
|---|---|---|
| Public API (external devs) | **REST** | Browser support, dễ dùng, documentation |
| Microservice-to-microservice | **gRPC** | Performance, type safety, streaming |
| Mobile app backend | **gRPC** | Smaller payloads, battery saving |
| Browser client | **REST** | Native support, no proxy needed |
| Real-time streaming | **gRPC** | Native bidirectional streaming |
| Simple CRUD | **REST** | Đơn giản, đủ tốt |
| Polyglot systems | **gRPC** | Code generation cho mọi ngôn ngữ |
| High-throughput internal | **gRPC** | 5-10x performance |

### 7.4 Hybrid Architecture (Phổ biến nhất)

```
┌─────────────────────────────────────────────────────┐
│                   Production Architecture            │
│                                                       │
│  [Browser/Mobile]                                    │
│       │                                               │
│       │ REST/GraphQL (public-facing)                 │
│       ↓                                               │
│  ┌─────────────┐                                    │
│  │  API Gateway │ ← REST/GraphQL → External clients  │
│  │  (BFF)       │                                    │
│  └──────┬──────┘                                    │
│         │                                             │
│         │ gRPC (internal, high-performance)          │
│         ↓                                             │
│  ┌──────┴──────┬──────────────┬────────────────┐   │
│  │ User Svc    │  Order Svc   │  Payment Svc   │   │
│  │ (Go)        │  (Java)      │  (Python)      │   │
│  └──────┬──────┴──────┬───────┴────────┬───────┘   │
│         │              │                │            │
│         └──── gRPC ────┴──── gRPC ──────┘            │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## 8. Advanced Features — Tính năng nâng cao

### 8.1 gRPC Status Codes

| Code | Name | HTTP equiv | Khi nào dùng |
|---|---|---|---|
| 0 | OK | 200 | Thành công |
| 1 | CANCELLED | 499 | Client hủy request |
| 2 | UNKNOWN | 500 | Lỗi không xác định |
| 3 | INVALID_ARGUMENT | 400 | Input không hợp lệ |
| 4 | DEADLINE_EXCEEDED | 504 | Timeout |
| 5 | NOT_FOUND | 404 | Resource không tồn tại |
| 6 | ALREADY_EXISTS | 409 | Resource đã tồn tại |
| 7 | PERMISSION_DENIED | 403 | Không có quyền |
| 8 | RESOURCE_EXHAUSTED | 429 | Rate limited/quota exceeded |
| 9 | FAILED_PRECONDITION | 400 | State không đúng cho operation |
| 10 | ABORTED | 409 | Transaction conflict |
| 11 | OUT_OF_RANGE | 400 | Outside valid range |
| 12 | UNIMPLEMENTED | 501 | Method chưa implement |
| 13 | INTERNAL | 500 | Internal server error |
| 14 | UNAVAILABLE | 503 | Service temporarily unavailable |
| 15 | DATA_LOSS | 500 | Unrecoverable data loss |
| 16 | UNAUTHENTICATED | 401 | Chưa authenticate |

### 8.2 Deadlines và Timeouts

```go
// Go client — set deadline
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

response, err := client.GetOrder(ctx, &GetOrderRequest{OrderId: "123"})
if err != nil {
    st, ok := status.FromError(err)
    if ok && st.Code() == codes.DeadlineExceeded {
        log.Println("Request timed out!")
    }
}
```

**Deadline propagation** — deadline được truyền qua service chain:
```
Client (deadline: 5s)
  → Service A (remaining: 4.8s)
    → Service B (remaining: 4.5s)
      → Service C (remaining: 4.2s)
      
Nếu Service C mất 5s → DEADLINE_EXCEEDED propagate ngược lên
```

### 8.3 Interceptors (Middleware)

```go
// Unary interceptor (Go)
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    
    // Call actual handler
    resp, err := handler(ctx, req)
    
    // Log after
    log.Printf("Method: %s, Duration: %v, Error: %v",
        info.FullMethod, time.Since(start), err)
    
    return resp, err
}

// Register interceptor
server := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
    grpc.StreamInterceptor(streamLoggingInterceptor),
)
```

### 8.4 Load Balancing

```
gRPC load balancing strategies:

1. Proxy-based (L7 LB):
   Client → [Envoy/Nginx] → Server 1/2/3
   ✅ Simple, client không cần biết
   ❌ Extra hop, single point of failure

2. Client-side (built-in):
   Client knows all server IPs → pick one
   ✅ No extra hop
   ❌ Client complexity

3. Look-aside (external LB):
   Client → asks [LB service] → gets server list → direct connect
   ✅ Flexible, scalable
   ❌ Additional infrastructure
```

### 8.5 Health Checking

```protobuf
// Standard health checking protocol (grpc.health.v1)
service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
```

### 8.6 gRPC-Web

```
Problem: Browser không hỗ trợ HTTP/2 trailers → không dùng gRPC trực tiếp

Solution: grpc-web proxy (Envoy)

Browser → [HTTP/1.1 + grpc-web format] → [Envoy Proxy] → [gRPC HTTP/2] → Server

// JavaScript client (grpc-web)
const client = new OrderServiceClient('https://api.example.com');
const request = new GetOrderRequest();
request.setOrderId('123');

client.getOrder(request, {}, (err, response) => {
  console.log(response.getOrderId());
});
```

### 8.7 Reflection

```bash
# Server reflection — cho phép client khám phá service mà không cần .proto
# Giống Swagger cho REST

# Dùng grpcurl để explore
grpcurl -plaintext localhost:50051 list
# → order.v1.OrderService

grpcurl -plaintext localhost:50051 describe order.v1.OrderService
# → Shows all methods

grpcurl -plaintext -d '{"order_id": "123"}' \
  localhost:50051 order.v1.OrderService/GetOrder
```

---

## 9. Hands-on Implementation

### 9.1 Server Implementation (Go)

```go
package main

import (
    "context"
    "log"
    "net"
    
    "google.golang.org/grpc"
    pb "github.com/mycompany/order/v1"
)

type orderServer struct {
    pb.UnimplementedOrderServiceServer
    orders map[string]*pb.Order
}

func (s *orderServer) GetOrder(ctx context.Context, req *pb.GetOrderRequest) (*pb.Order, error) {
    order, ok := s.orders[req.OrderId]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "order %s not found", req.OrderId)
    }
    return order, nil
}

func (s *orderServer) ListOrders(req *pb.ListOrdersRequest, stream pb.OrderService_ListOrdersServer) error {
    for _, order := range s.orders {
        if order.CustomerId == req.CustomerId {
            if err := stream.Send(order); err != nil {
                return err
            }
        }
    }
    return nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    
    server := grpc.NewServer()
    pb.RegisterOrderServiceServer(server, &orderServer{
        orders: make(map[string]*pb.Order),
    })
    
    log.Println("Server listening on :50051")
    server.Serve(lis)
}
```

### 9.2 Client Implementation (Python)

```python
import grpc
from order.v1 import order_pb2, order_pb2_grpc

# Create channel
channel = grpc.insecure_channel('localhost:50051')
stub = order_pb2_grpc.OrderServiceStub(channel)

# Unary call
request = order_pb2.GetOrderRequest(order_id="123")
try:
    response = stub.GetOrder(request, timeout=5.0)
    print(f"Order: {response.order_id}, Status: {response.status}")
except grpc.RpcError as e:
    print(f"Error: {e.code()} - {e.details()}")

# Server streaming
request = order_pb2.ListOrdersRequest(customer_id="user_456", page_size=10)
for order in stub.ListOrders(request):
    print(f"Order: {order.order_id}")

# Client streaming
def generate_orders():
    for i in range(100):
        yield order_pb2.CreateOrderRequest(
            customer_id="user_456",
            items=[order_pb2.OrderItem(product_id=f"prod_{i}", quantity=1)]
        )

result = stub.BatchCreateOrders(generate_orders())
print(f"Created {result.total_created} orders")
```

### 9.3 Testing gRPC

```bash
# grpcurl — curl for gRPC
# Install
brew install grpcurl

# List services
grpcurl -plaintext localhost:50051 list

# Describe a service
grpcurl -plaintext localhost:50051 describe order.v1.OrderService

# Call a method
grpcurl -plaintext -d '{"order_id": "123"}' \
  localhost:50051 order.v1.OrderService/GetOrder

# Server streaming
grpcurl -plaintext -d '{"customer_id": "user_456"}' \
  localhost:50051 order.v1.OrderService/ListOrders

# With TLS
grpcurl -cacert ca.pem -cert client.pem -key client-key.pem \
  api.example.com:443 order.v1.OrderService/GetOrder
```

---

## 10. Tổng kết và Best Practices

### 10.1 Khi nào dùng gRPC

```
✅ Dùng gRPC khi:
• Microservice-to-microservice communication
• Performance-critical paths
• Polyglot services (nhiều ngôn ngữ)
• Cần streaming (real-time, large data)
• Strong type safety quan trọng
• Internal APIs

❌ Không dùng gRPC khi:
• Public API cho browser/external developers
• Simple CRUD (REST đủ tốt)
• Cần HTTP caching
• Team nhỏ, MVP (REST đơn giản hơn)
• Debug/inspect dễ dàng cần thiết
```

### 10.2 Best Practices

| Category | Practice |
|---|---|
| **Proto design** | Dùng package versioning (v1, v2) |
| **Proto design** | Enum luôn có UNSPECIFIED = 0 |
| **Proto design** | Field numbers không bao giờ reuse (reserved) |
| **Proto design** | Request/Response message riêng cho mỗi RPC |
| **Performance** | Enable gzip compression cho large payloads |
| **Performance** | Reuse connections (connection pooling) |
| **Reliability** | Set deadlines cho mọi RPC call |
| **Reliability** | Implement retry with exponential backoff |
| **Observability** | Interceptors cho logging, metrics, tracing |
| **Security** | Luôn dùng TLS trong production |
| **Compatibility** | Dùng `buf breaking` check trước deploy |
| **Testing** | Mock generated stubs cho unit tests |

### 10.3 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| grpc.io | Official gRPC documentation |
| protobuf.dev | Protocol Buffers documentation |
| RFC 7540 | HTTP/2 specification |
| Google API Design Guide | gRPC API design best practices |
| Buf docs (buf.build) | Modern protobuf tooling |
| gRPC-Go examples | Official Go examples |
| grpc.github.io/grpc/core | gRPC Core documentation |

### 10.4 Câu hỏi ôn tập

1. gRPC khác REST ở điểm nào cơ bản? (philosophy, transport, format)
2. Protocol Buffers encode data thế nào? Tại sao nhỏ hơn JSON?
3. Mô tả 4 kiểu streaming trong gRPC. Cho ví dụ use case mỗi kiểu.
4. HTTP/2 giúp gì cho gRPC? (multiplexing, HPACK, flow control)
5. Code generation workflow: .proto → protoc → code. Lợi ích?
6. Backward compatibility rules cho proto files?
7. gRPC status codes nào tương đương HTTP 400, 404, 500?
8. Deadline propagation hoạt động thế nào trong service chain?
9. Thiết kế .proto cho hệ thống chat real-time.
10. So sánh gRPC vs REST cho: (a) public mobile API, (b) internal microservices, (c) IoT sensors.

---

*Bài viết được tham khảo từ grpc.io official docs, protobuf.dev specification, RFC 7540 (HTTP/2), Google API Design Guide, và Buf Schema Registry documentation.*
