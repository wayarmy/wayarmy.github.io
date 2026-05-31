---
layout: post
title: "Networking Fundamentals - Phần 1: OSI Model & TCP/IP"
subtitle: "Hiểu cách máy tính giao tiếp với nhau — giải thích bằng ví dụ đời thường"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-01
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation**. Đây là bài đầu tiên trong phần Networking Fundamentals (Tuần 1).
>
> **Đối tượng:** Người mới hoàn toàn, chưa có kiến thức IT. Bạn chỉ cần biết máy tính là gì.
>
> **Nguồn tham khảo:**
> - ISO/IEC 7498-1:1994 — OSI Basic Reference Model
> - RFC 1122 (1989) — Requirements for Internet Hosts
> - RFC 793 (1981) — Transmission Control Protocol
> - AWS Documentation: [What is OSI Model?](https://aws.amazon.com/what-is/osi-model/)
> - Wikipedia: [Internet Protocol Suite](https://en.wikipedia.org/wiki/Internet_protocol_suite)
> - Textbook: "Computer Networks" — Andrew S. Tanenbaum, 5th Edition

---

## 1. Máy tính "nói chuyện" với nhau như thế nào?

Hãy tưởng tượng bạn muốn gửi một bức thư cho bạn mình ở thành phố khác. Bạn không thể tự đi giao — bạn cần **hệ thống bưu điện**.

Hệ thống bưu điện hoạt động theo nhiều bước:
1. Bạn **viết thư** (nội dung)
2. Bạn **bỏ vào phong bì**, ghi địa chỉ người nhận (đóng gói)
3. Bưu điện **phân loại** theo mã bưu chính, xác định đường đi (định tuyến)
4. Thư được **chuyển** bằng xe tải, máy bay (vận chuyển vật lý)
5. Bưu điện đầu kia **phân phối** đến đúng nhà (giao hàng cuối)
6. Bạn mình **mở phong bì**, đọc thư (nhận nội dung)

**Mạng máy tính hoạt động y hệt như vậy**, chỉ là thay "thư" bằng "dữ liệu số", thay "bưu điện" bằng "các thiết bị mạng". Và để quản lý sự phức tạp này, người ta chia thành **các lớp (layers)** — mỗi lớp lo một việc cụ thể.

---

## 2. Tại sao cần chia thành "lớp"?

### Ví dụ đời thường: Giao hàng Shopee

Khi bạn mua hàng trên Shopee, quá trình giao hàng có nhiều "lớp" khác nhau:

| Lớp | Ai làm | Việc gì |
|-----|--------|---------|
| **Người bán** | Shop | Đóng gói sản phẩm, dán nhãn |
| **Kho phân loại** | Trung tâm logistics | Phân loại theo khu vực, tỉnh thành |
| **Vận chuyển** | Xe tải, máy bay | Chuyển hàng giữa các kho |
| **Shipper cuối** | Người giao hàng | Giao đến tận nhà bạn |

Mỗi "lớp" chỉ cần biết việc của mình:
- Shipper **không cần biết** trong gói hàng có gì
- Xe tải **không cần biết** ai là người nhận cuối cùng
- Shop **không cần biết** xe tải đi đường nào

**Đây chính là nguyên tắc phân lớp (layering)** — mỗi lớp làm tốt việc của mình, không cần quan tâm chi tiết bên trong của lớp khác.

### Lợi ích:
1. **Dễ sửa chữa:** Nếu xe tải hỏng, chỉ cần đổi xe — không ảnh hưởng đến cách đóng gói hay nội dung hàng
2. **Dễ nâng cấp:** Có thể đổi từ giao bằng xe máy sang drone mà shop không cần thay đổi gì
3. **Nhiều người cùng làm:** Mỗi đội ngũ chuyên một việc, không ai cần hiểu toàn bộ hệ thống

---

## 3. OSI Model — 7 Lớp

### Lịch sử ngắn gọn

Vào thập niên 1970-80, mỗi hãng máy tính (IBM, DEC, Honeywell...) đều có hệ thống mạng **riêng**, không tương thích với nhau — giống như mỗi nước có hệ thống bưu điện khác nhau, thư không gửi qua lại được.

**Tổ chức Tiêu chuẩn Quốc tế (ISO)** đã tạo ra mô hình OSI (Open Systems Interconnection) năm 1984, tiêu chuẩn **ISO 7498**, để mọi hệ thống có thể "nói chung một ngôn ngữ".

OSI chia quá trình giao tiếp mạng thành **7 lớp**, từ lớp thấp nhất (vật lý) đến lớp cao nhất (ứng dụng).

---

### 🧱 Layer 1: Physical Layer (Lớp Vật Lý)

#### Ví dụ đời thường:
> Đây giống như **con đường vật lý** — đường nhựa, cầu, hầm chui — mà xe chạy trên đó. Không có đường, không có gì di chuyển được.

#### Chức năng:
Truyền **các bit thô (0 và 1)** từ máy này sang máy khác qua môi trường vật lý.

#### Chi tiết:
- **Bit** là đơn vị nhỏ nhất của dữ liệu số: chỉ có giá trị 0 hoặc 1
- Bit được biểu diễn bằng: xung điện (trên dây đồng), ánh sáng (trên cáp quang), hoặc sóng radio (Wi-Fi)
- Layer 1 quy định: tốc độ truyền bao nhiêu bit/giây (ví dụ: 1 Gbps), dùng loại cáp nào, đầu cắm hình gì

#### Các "con đường" phổ biến:
| Loại | Ví dụ | Tốc độ | Khoảng cách |
|------|-------|--------|-------------|
| **Cáp đồng (Copper)** | Dây mạng RJ-45 (dây xanh cắm vào máy tính) | 1-10 Gbps | ~100m |
| **Cáp quang (Fiber Optic)** | Sợi thủy tinh mảnh truyền ánh sáng | 10-400 Gbps | Hàng km |
| **Sóng radio (Wireless)** | Wi-Fi, Bluetooth, 4G/5G | 0.1-10 Gbps | 10m - vài km |

#### Thiết bị hoạt động ở Layer 1:
- **Hub:** Nhận tín hiệu từ 1 cổng, phát lại cho TẤT CẢ các cổng khác (giống loa phát thanh — ai cũng nghe)
- **Repeater:** Khuếch đại tín hiệu yếu (giống trạm tiếp sóng)

#### Trong AWS:
Bạn KHÔNG bao giờ chạm tới Layer 1 — AWS quản lý toàn bộ phần cứng vật lý (data center, cáp, switch...) thông qua hệ thống **AWS Nitro**. Đây là phần "**of the cloud**" mà AWS chịu trách nhiệm trong Shared Responsibility Model.

---

### 🔗 Layer 2: Data Link Layer (Lớp Liên Kết Dữ Liệu)

#### Ví dụ đời thường:
> Đây giống như **biển số xe**. Trong cùng một con đường (mạng LAN), mỗi xe có biển số khác nhau để phân biệt. Khi bạn muốn gửi đồ cho xe cụ thể, bạn ghi biển số xe đó.

#### Chức năng:
Truyền dữ liệu giữa **hai thiết bị nằm trên cùng một mạng cục bộ** (LAN — Local Area Network). Sử dụng **MAC address** để xác định "ai là ai" trên cùng mạng.

#### MAC Address là gì?

MAC (Media Access Control) address giống như **"số CMND" của card mạng** — mỗi thiết bị mạng được gán một MAC address duy nhất khi xuất xưởng.

- Dài 48-bit (6 bytes), viết dạng: `AA:BB:CC:DD:EE:FF`
- Ví dụ: `00:1A:2B:3C:4D:5E`
- Không thể thay đổi (burned into hardware) — khác với IP address (có thể thay đổi)

#### Frame là gì?

Dữ liệu ở Layer 2 được đóng gói thành **frame** — giống như một bao thư có ghi địa chỉ:

```
┌──────────────────────────────────────────────────┐
│ Dest MAC │ Source MAC │ Type │ DATA │ FCS (CRC) │
│  6 bytes │   6 bytes  │  2B  │ ...  │  4 bytes  │
└──────────────────────────────────────────────────┘
```

- **Dest MAC:** MAC address người nhận (trên cùng mạng)
- **Source MAC:** MAC address người gửi
- **FCS (Frame Check Sequence):** Mã kiểm tra lỗi — giống như "dấu niêm phong" để biết frame có bị hỏng trên đường đi không

#### Thiết bị hoạt động ở Layer 2:
- **Switch:** Giống "bưu tá thông minh" — nhận frame, đọc MAC address đích, chuyển đến ĐÚNG cổng của thiết bị đó (khác với Hub phát cho tất cả)
- Switch lưu một **bảng MAC address** — giống danh bạ: "MAC này nằm ở cổng nào"

#### Khác biệt Hub vs Switch:
| | Hub (Layer 1) | Switch (Layer 2) |
|---|---|---|
| Nhận frame | Phát cho TẤT CẢ cổng | Chỉ gửi đến cổng đích |
| Thông minh? | Không — không đọc địa chỉ | Có — đọc MAC address |
| Hiệu quả | Lãng phí bandwidth | Hiệu quả cao |
| Ví dụ | Loa phát thanh (ai cũng nghe) | Bưu tá (giao đúng nhà) |

#### Giao thức tiêu biểu:
- **Ethernet (IEEE 802.3):** Tiêu chuẩn mạng có dây phổ biến nhất (dây mạng bạn cắm vào máy tính)
- **Wi-Fi (IEEE 802.11):** Mạng không dây

#### Trong AWS:
- Mỗi **ENI (Elastic Network Interface)** gắn vào EC2 instance đều có một MAC address riêng
- Security Groups bắt đầu hoạt động từ Layer 2 trở lên

---

### 🌐 Layer 3: Network Layer (Lớp Mạng)

#### Ví dụ đời thường:
> Đây giống như **hệ thống địa chỉ + bản đồ** của bưu điện. MAC address chỉ giúp tìm nhau trên cùng một con phố (LAN). Nhưng khi bạn gửi thư đi thành phố khác, bạn cần **mã bưu chính** và bưu điện cần **bản đồ** để biết đường đi.
>
> IP address = mã bưu chính + số nhà  
> Routing = bưu điện tra bản đồ để tìm đường

#### Chức năng:
**Định tuyến (routing)** — tìm đường đi tốt nhất để chuyển dữ liệu từ mạng này sang mạng khác. Sử dụng **IP address** (địa chỉ logic) để xác định vị trí.

#### IP Address là gì?

IP address giống như **địa chỉ nhà của bạn trên Internet**. Mỗi thiết bị kết nối mạng đều cần ít nhất một IP address.

**IPv4** (phiên bản phổ biến nhất):
- Dài 32-bit, viết thành 4 số thập phân cách nhau bởi dấu chấm
- Ví dụ: `192.168.1.100`
- Mỗi số có giá trị 0-255
- Tổng cộng: ~4.3 tỷ địa chỉ (đã cạn kiệt!)

**IPv6** (thế hệ mới):
- Dài 128-bit, ví dụ: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Gần như vô hạn địa chỉ

#### So sánh MAC address vs IP address:

| | MAC Address | IP Address |
|---|---|---|
| **Ví dụ** | Số CMND (cố định) | Địa chỉ nhà (có thể đổi khi chuyển nhà) |
| **Phạm vi** | Chỉ dùng trong mạng LAN | Dùng toàn cầu (Internet) |
| **Ai gán?** | Nhà sản xuất (cố định) | Quản trị mạng hoặc DHCP (thay đổi được) |
| **Dùng khi nào** | Giao tiếp trên cùng mạng | Giao tiếp giữa các mạng khác nhau |

#### Routing (Định tuyến) là gì?

Khi bạn gửi thư đi Đà Nẵng, bưu điện Hà Nội không gửi thẳng — nó chuyển qua nhiều trạm trung gian. Mỗi trạm quyết định: "Bước tiếp theo nên gửi đi đâu?"

Router cũng làm y hệt:
1. Nhận packet (gói dữ liệu)
2. Đọc IP address đích
3. Tra **bảng định tuyến (routing table)**: "Mạng đích nằm ở hướng nào?"
4. Chuyển packet đến router tiếp theo (next hop)

```
Máy bạn → Router nhà → Router ISP → ... → Router ISP đích → Server
```

#### Thiết bị:
- **Router:** "Bưu điện trung chuyển" — kết nối nhiều mạng khác nhau, quyết định đường đi

#### Trong AWS:
| Concept | Ý nghĩa |
|---------|---------|
| **VPC (Virtual Private Cloud)** | Mạng riêng ảo — giống khu đô thị có rào riêng |
| **Subnet** | Phân chia VPC thành các "phường/xã" nhỏ hơn |
| **Route Table** | Bảng chỉ đường — packet đi đâu qua đâu |
| **Internet Gateway** | "Cổng ra quốc tế" — kết nối VPC với Internet |
| **NAT Gateway** | "Người đại diện" — cho máy trong mạng riêng truy cập Internet mà không bị lộ IP |
| **NACLs** | "Bảo vệ khu phố" — kiểm tra mọi xe vào/ra (stateless) |

---

### 🚚 Layer 4: Transport Layer (Lớp Vận Chuyển)

#### Ví dụ đời thường:
> Layer 3 giúp thư đến đúng **tòa nhà** (IP address). Nhưng tòa nhà có nhiều phòng — Layer 4 giúp thư đến đúng **phòng** (port number).
>
> Hãy nghĩ **IP address = địa chỉ tòa nhà**, **Port number = số phòng** trong tòa nhà đó.
>
> Ví dụ: `192.168.1.100:443` = Tòa nhà 192.168.1.100, phòng 443 (đây là phòng chuyên phục vụ HTTPS)

#### Chức năng:
Đảm bảo dữ liệu được truyền **end-to-end** giữa hai ứng dụng cụ thể trên hai máy khác nhau, sử dụng **port numbers** (0-65535).

#### Port Numbers phổ biến:
| Port | Dịch vụ | Ví dụ thực tế |
|------|---------|---------------|
| 22 | SSH | Remote login vào server |
| 53 | DNS | Tra cứu tên miền |
| 80 | HTTP | Web không mã hóa |
| 443 | HTTPS | Web có mã hóa (bank, shopping) |
| 3306 | MySQL | Database |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |

#### Hai "kiểu giao hàng" — TCP vs UDP:

**TCP (Transmission Control Protocol) — Giao hàng đảm bảo:**

Giống như gửi hàng qua **bưu điện có ký nhận**:
- Phải "bắt tay" trước khi gửi (thiết lập kết nối)
- Mỗi gói hàng có **số thứ tự** — nếu nhận sai thứ tự, sắp xếp lại
- Nếu gói hàng **bị mất**, gửi lại
- Người nhận phải **ký xác nhận** (ACK) cho mỗi gói
- Chậm hơn vì có nhiều bước kiểm tra

*Dùng khi: Web (HTTP), Email, tải file — khi cần DỮ LIỆU HOÀN CHỈNH*

**UDP (User Datagram Protocol) — Giao hàng tốc độ:**

Giống như **ném giấy qua cửa sổ**:
- Không cần "bắt tay" — gửi luôn
- Không biết người nhận có nhận được không
- Không gửi lại nếu mất
- Nhanh hơn vì không có bước kiểm tra

*Dùng khi: Video call, game online, livestream — khi CẦN NHANH hơn cần đầy đủ (mất vài frame video không sao, nhưng lag thì rất khó chịu)*

#### TCP 3-Way Handshake — "Bắt tay 3 bước"

Trước khi TCP gửi dữ liệu, hai bên phải "bắt tay" để đồng ý kết nối. Giống như gọi điện:

```
Bạn:     "Alo, nghe rõ không?" (SYN)
Bạn bè:  "Nghe rõ! Còn bạn nghe tôi rõ không?" (SYN + ACK)
Bạn:     "Nghe rõ! Bắt đầu nói nhé!" (ACK)
→ Kết nối được thiết lập, bắt đầu nói chuyện
```

Kỹ thuật hơn (theo RFC 793):
```
Client → Server:  SYN, Seq=100
Server → Client:  SYN+ACK, Seq=300, Ack=101
Client → Server:  ACK, Seq=101, Ack=301
→ Connection established — data transfer begins
```

#### Trong AWS:
| Service | Layer 4 liên quan |
|---------|-------------------|
| **NLB (Network Load Balancer)** | Cân bằng tải ở Layer 4 — chỉ xem IP + Port, không đọc nội dung |
| **Security Groups** | Filter theo: Protocol (TCP/UDP) + Port + Source IP |

---

### 💬 Layer 5: Session Layer (Lớp Phiên)

#### Ví dụ đời thường:
> Giống như **cuộc gọi điện thoại**: bạn nhấc máy (mở session), nói chuyện (trao đổi data), rồi gác máy (đóng session). Nếu cuộc gọi bị ngắt giữa chừng, bạn gọi lại và tiếp tục từ chỗ đang dở.

#### Chức năng:
- **Mở** phiên làm việc giữa hai ứng dụng
- **Duy trì** phiên (giữ kết nối sống)
- **Đóng** phiên khi xong
- **Đồng bộ:** Đặt "checkpoint" — nếu mất kết nối, phục hồi từ checkpoint gần nhất (không cần bắt đầu lại từ đầu)

#### Giao thức: NetBIOS, RPC, NFS

#### Ghi chú thực tế:
Trong mô hình TCP/IP (thực tế), Layer 5 thường được **gộp vào Layer 7 (Application)**. Ví dụ: HTTP tự quản lý session qua cookies, WebSocket duy trì kết nối liên tục.

---

### 🎭 Layer 6: Presentation Layer (Lớp Trình Diễn)

#### Ví dụ đời thường:
> Bạn viết thư bằng tiếng Việt, nhưng bạn mình chỉ đọc được tiếng Anh. Bạn cần **người phiên dịch** trước khi gửi. Layer 6 chính là "người phiên dịch" — đảm bảo hai bên hiểu cùng một ngôn ngữ.
>
> Ngoài ra, nếu bạn muốn gửi thư bí mật, bạn cần **mã hóa** trước khi gửi (và người nhận giải mã khi nhận). Layer 6 cũng làm việc này.

#### Chức năng:
- **Encoding/Decoding:** Chuyển đổi định dạng dữ liệu (ASCII, Unicode, UTF-8)
- **Encryption/Decryption:** Mã hóa dữ liệu (SSL/TLS — bạn thấy icon 🔒 trên trình duyệt chính là nhờ layer này)
- **Compression:** Nén dữ liệu cho nhỏ lại để truyền nhanh hơn (JPEG, ZIP)

#### Ví dụ cụ thể:
- Khi bạn vào trang web bank (HTTPS), dữ liệu được **mã hóa TLS** ở Layer 6 — dù ai đó "nghe lén" trên đường truyền, họ cũng chỉ thấy dữ liệu vô nghĩa
- File JSON, XML, HTML đều là cách **trình diễn dữ liệu** ở layer này

#### Trong AWS:
- **ACM (AWS Certificate Manager):** Quản lý SSL/TLS certificates
- CloudFront, ALB đều sử dụng TLS để mã hóa traffic

---

### 📱 Layer 7: Application Layer (Lớp Ứng Dụng)

#### Ví dụ đời thường:
> Đây là **nội dung bức thư** — phần mà bạn và bạn mình thực sự quan tâm. Tất cả các layer bên dưới chỉ là phương tiện để đưa nội dung này từ điểm A đến điểm B.
>
> Khi bạn mở Google.com, bạn nhìn thấy trang web — đó là Layer 7. Bạn không cần biết (và không thấy) hàng triệu packet, frame, bit đã chạy bên dưới.

#### Chức năng:
Cung cấp **giao diện mạng** cho các ứng dụng người dùng. Đây là lớp duy nhất mà end-user tương tác trực tiếp.

#### Giao thức phổ biến:
| Giao thức | Port | Dùng cho | Ví dụ |
|-----------|------|----------|-------|
| **HTTP/HTTPS** | 80/443 | Web | Mở google.com, đọc tin tức |
| **DNS** | 53 | Phân giải tên miền | "google.com" → IP address |
| **SMTP** | 25/587 | Gửi email | Gmail gửi mail |
| **POP3/IMAP** | 110/143 | Nhận email | Gmail nhận mail |
| **FTP** | 21 | Truyền file | Upload file lên server |
| **SSH** | 22 | Remote access | Đăng nhập server từ xa |

#### Trong AWS:
| Service | Chức năng ở Layer 7 |
|---------|---------------------|
| **ALB (Application Load Balancer)** | Đọc nội dung HTTP — route theo URL path, hostname |
| **API Gateway** | Proxy cho API (REST, WebSocket) |
| **CloudFront** | CDN — cache và phân phối nội dung |
| **WAF (Web Application Firewall)** | Bảo vệ khỏi tấn công Layer 7 (SQL injection, XSS) |
| **Route 53** | DNS service — phân giải tên miền |

---

## 4. Data Encapsulation — "Đóng gói" dữ liệu

### Ví dụ đời thường:

Bạn muốn gửi một chiếc **vòng tay** cho bạn ở Đà Nẵng:

1. **Vòng tay** = Dữ liệu gốc (Layer 7)
2. Bạn bọc vòng tay trong **giấy chống sốc** = Thêm header Layer 6 (encoding)
3. Bạn bỏ vào **hộp nhỏ**, ghi "Hàng dễ vỡ" = Thêm header Layer 4 (TCP segment — thêm port, seq number)
4. Bạn dán lên hộp **phiếu gửi hàng** với địa chỉ người nhận = Thêm header Layer 3 (IP packet — thêm IP đích)
5. Bưu điện bỏ vào **thùng carton lớn** cùng nhiều hộp khác, dán mã vận đơn = Thêm header Layer 2 (Ethernet frame — thêm MAC address)
6. Thùng carton được **chất lên xe tải** = Layer 1 (truyền physical)

### Kỹ thuật:

```
Gửi (đi xuống):

Layer 7-5: [DATA]                                    ← Application data
Layer 4:   [TCP Header | DATA]                       ← Segment
Layer 3:   [IP Header | TCP Header | DATA]           ← Packet
Layer 2:   [Eth Header | IP | TCP | DATA | FCS]      ← Frame
Layer 1:   10110100101101001...                       ← Bits
```

```
Nhận (đi lên):

Layer 1:   Nhận bits → chuyển lên Layer 2
Layer 2:   Bóc Eth Header → kiểm tra MAC → chuyển lên Layer 3
Layer 3:   Bóc IP Header → kiểm tra IP đích → chuyển lên Layer 4
Layer 4:   Bóc TCP Header → chuyển data lên ứng dụng đúng port
Layer 7-5: Nhận [DATA] → hiển thị cho người dùng
```

**Key insight:** Mỗi layer chỉ đọc header CỦA MÌNH, không quan tâm bên trong payload chứa gì. Switch Layer 2 không biết (và không cần biết) data bên trong là email hay video — nó chỉ xem MAC address.

---

## 5. TCP/IP Model — Mô hình thực tế của Internet

### Tại sao cần biết TCP/IP khi đã có OSI?

OSI là mô hình **lý thuyết** — tuyệt vời để học và giải thích. Nhưng Internet thực tế **không dùng OSI**, mà dùng **TCP/IP**.

TCP/IP Model được phát triển bởi **DARPA** (Cơ quan nghiên cứu quốc phòng Mỹ) trong thập niên 1970, tiêu chuẩn hóa trong **RFC 1122** (1989). Internet bạn đang dùng ngay bây giờ hoạt động theo mô hình này.

### 4 Lớp TCP/IP:

```
         OSI Model                    TCP/IP Model
    ┌─────────────────┐          ┌─────────────────┐
    │  7. Application │          │                 │
    ├─────────────────┤          │   Application   │ ← HTTP, DNS, SSH, SMTP
    │  6. Presentation│          │                 │
    ├─────────────────┤          ├─────────────────┤
    │  5. Session     │          │                 │
    ├─────────────────┤          ├─────────────────┤
    │  4. Transport   │  ═══════ │   Transport     │ ← TCP, UDP
    ├─────────────────┤          ├─────────────────┤
    │  3. Network     │  ═══════ │   Internet      │ ← IP, ICMP
    ├─────────────────┤          ├─────────────────┤
    │  2. Data Link   │          │  Network Access │
    ├─────────────────┤          │  (Link Layer)   │ ← Ethernet, Wi-Fi
    │  1. Physical    │          │                 │
    └─────────────────┘          └─────────────────┘
```

### Khác biệt chính:

| | OSI | TCP/IP |
|---|---|---|
| **Số lớp** | 7 | 4 |
| **Layer 5,6,7** | Tách riêng | Gộp thành 1 "Application" |
| **Layer 1,2** | Tách riêng | Gộp thành 1 "Network Access" |
| **Thực tế** | Mô hình lý thuyết | Internet thực hoạt động theo |
| **Tiêu chuẩn** | ISO 7498 (1984) | RFC 1122 (1989) |
| **Dùng cho** | Giáo dục, troubleshooting | Triển khai, vận hành |

### Tại sao vẫn cần học OSI?

Vì trong công việc thực tế và AWS exams, người ta dùng **ngôn ngữ OSI** để nói chuyện:
- "Layer 7 load balancer" = ALB
- "Layer 4 load balancer" = NLB
- "Layer 3 DDoS attack" = Tấn công ở tầng IP
- "Layer 7 firewall" = WAF

---

## 6. Tóm tắt — Bảng mapping sang AWS

| OSI Layer | Ví dụ đời thường | AWS Service |
|-----------|-----------------|-------------|
| **L7** Application | Nội dung bức thư | ALB, API Gateway, CloudFront, WAF, Route 53 |
| **L6** Presentation | Mã hóa / giải mã | ACM (TLS certificates) |
| **L5** Session | Cuộc gọi điện thoại | (tích hợp trong L7) |
| **L4** Transport | Số phòng trong tòa nhà | NLB, Security Groups |
| **L3** Network | Mã bưu chính + bản đồ | VPC, Subnets, Route Tables, IGW, NAT GW, NACLs |
| **L2** Data Link | Biển số xe | ENI (MAC address) |
| **L1** Physical | Con đường vật lý | AWS Nitro (managed by AWS) |

---

## 7. Bài tập thực hành

### Bài 1: Trace một HTTP request qua 7 layers (trên giấy)

Bạn mở browser, gõ `https://google.com`. Hãy viết ra giấy từng bước:

1. **L7:** Browser tạo HTTP GET request cho `/`
2. **L6:** Mã hóa request bằng TLS (HTTPS)
3. **L5:** Thiết lập session (TCP connection)
4. **L4:** Đóng thành TCP segment — source port: 52341 (random), dest port: 443
5. **L3:** Đóng thành IP packet — source IP: 192.168.1.5 (IP nhà bạn), dest IP: 142.250.185.14 (IP Google)
6. **L2:** Đóng thành Ethernet frame — source MAC: (card mạng bạn), dest MAC: (router nhà bạn)
7. **L1:** Gửi qua Wi-Fi dưới dạng sóng radio

### Bài 2: Dùng Wireshark quan sát thực tế

1. Tải Wireshark (wireshark.org) — miễn phí
2. Mở Wireshark, chọn network interface (Wi-Fi hoặc Ethernet)
3. Mở browser, truy cập `http://example.com` (dùng HTTP, không phải HTTPS, để đọc được nội dung)
4. Quay lại Wireshark, tìm packet HTTP GET
5. Click vào packet đó, xem từng layer:
   - **Ethernet II** (Layer 2): MAC addresses
   - **Internet Protocol v4** (Layer 3): IP addresses
   - **Transmission Control Protocol** (Layer 4): Port numbers
   - **Hypertext Transfer Protocol** (Layer 7): HTTP GET request

### Bài 3: Vẽ sơ đồ so sánh

Vẽ trên giấy A4:
- Bên trái: OSI 7 layers (ghi tên + PDU + 2 giao thức mỗi layer)
- Bên phải: TCP/IP 4 layers
- Vẽ mũi tên mapping giữa hai mô hình
- Ghi thiết bị hoạt động tại mỗi layer (Hub, Switch, Router)

---

## Tài liệu đọc thêm

| Nguồn | Link | Ghi chú |
|-------|------|---------|
| ISO 7498 (OSI Model) | [iso.org](https://www.iso.org/standard/20269.html) | Tiêu chuẩn gốc |
| RFC 1122 (TCP/IP) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc1122.html) | Internet host requirements |
| RFC 793 (TCP) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc793.html) | TCP specification |
| AWS: What is OSI? | [aws.amazon.com](https://aws.amazon.com/what-is/osi-model/) | AWS explanation |
| Wikipedia: Internet Protocol Suite | [wikipedia.org](https://en.wikipedia.org/wiki/Internet_protocol_suite) | TCP/IP overview |

---

*Bài tiếp theo: [Networking Fundamentals - Phần 2: IP Addressing & Subnetting](/2026-06-02-ip-addressing-subnetting/) — Hiểu cách đánh "địa chỉ" cho máy tính trên mạng*
