---
layout: post
title: "Tutorial: Tạo VPC đầy đủ trên AWS Console — Giải thích từ A-Z cho người mới"
subtitle: "Bạn chưa biết VPC, Subnet, Internet Gateway là gì? Bài này sẽ giải thích từ zero rồi cầm tay hướng dẫn từng bước"
gh-repo: wayarmy/wayarmy.github.io
tags: [aws, vpc, networking, tutorial, learning-path, beginner]
comments: true
date: 2026-06-03
categories: [networking]
---

> **Bài này dành cho ai?** Người hoàn toàn mới — bạn chưa hiểu network là gì, chưa từng dùng AWS Console, chỉ biết máy tính cơ bản. Tôi sẽ giải thích **mọi thứ từ đầu**.
>
> **Sau bài này, bạn sẽ:**
> - Hiểu VPC, Subnet, Internet Gateway, NAT Gateway **là gì** và **tại sao cần**
> - Tự tay tạo một mạng hoàn chỉnh trên AWS
> - Hiểu tại sao cần chia "public" và "private"
>
> **Thời gian:** ~45 phút (bao gồm đọc hiểu)  
> **Chi phí:** ~$0.05 nếu xong trong 1 giờ. **Nhớ xóa sau lab!**

---

# PHẦN 1: HIỂU KHÁI NIỆM TRƯỚC KHI LÀM

> ⚠️ **Đừng bỏ qua phần này.** Nếu bạn nhảy thẳng vào bấm nút mà không hiểu, bạn sẽ tạo ra thứ gì đó nhưng không biết nó hoạt động thế nào. Hãy đọc chậm, hiểu rõ, rồi làm.

---

## 1. VPC là gì? — "Nhà riêng" của bạn trên Internet

### Vấn đề: Tại sao cần "nhà riêng"?

Hãy tưởng tượng Internet là **một thành phố khổng lồ** — hàng tỷ người sống ở đó, hàng tỷ "ngôi nhà" (máy tính, server) nằm trên các con đường.

Nếu bạn mở một cửa hàng online, bạn cần server để chạy website. Bạn có 2 lựa chọn:

**❌ Lựa chọn 1: Đặt server ngay giữa đường phố (public Internet)**
- Bất kỳ ai đi qua cũng nhìn thấy server của bạn
- Hacker có thể đến gõ cửa bất cứ lúc nào
- Không có rào, không có bảo vệ
- Tất cả mọi thứ đều lộ thiên

**✅ Lựa chọn 2: Xây một khu đô thị riêng (VPC)**
- Có **tường rào** bao quanh toàn bộ khu
- Bạn **tự quyết định** ai được vào, ai không
- Bên trong có nhiều "khu vực" khác nhau (public, private)
- Bạn kiểm soát hoàn toàn

### Vậy VPC là gì?

**VPC = Virtual Private Cloud = "Khu đô thị riêng ảo" của bạn trên AWS.**

Khi bạn tạo VPC, AWS tạo cho bạn **một mạng hoàn toàn riêng biệt**, tách biệt khỏi mạng của người khác. Giống như bạn mua một miếng đất lớn, xây tường rào xung quanh — bên trong bạn muốn làm gì tùy bạn.

```
┌─── Internet (thành phố khổng lồ) ────────────────────────────┐
│                                                                │
│   Hàng tỷ máy tính, server, hacker, bot...                   │
│                                                                │
│   ┌─── VPC của bạn (khu đô thị riêng) ─────────────────┐     │
│   │                                                      │     │
│   │  • Tường rào bao quanh (isolated)                   │     │
│   │  • Bạn tự quyết IP addresses                        │     │
│   │  • Bạn tự quyết ai vào, ai ra                       │     │
│   │  • Bạn tự chia khu vực bên trong                     │     │
│   │                                                      │     │
│   └──────────────────────────────────────────────────────┘     │
│                                                                │
│   ┌─── VPC của người khác ──┐  ┌─── VPC của công ty X ──┐    │
│   │  (bạn không thấy)       │  │  (bạn không thấy)      │    │
│   └──────────────────────────┘  └────────────────────────┘    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Tại sao cần VPC?

| Không có VPC | Có VPC |
|-------------|--------|
| Server lộ thiên trên Internet | Server nằm trong mạng riêng |
| Không kiểm soát được ai truy cập | Bạn quyết định hoàn toàn |
| Tất cả servers "thấy" nhau | Chỉ servers trong VPC "thấy" nhau |
| Giống nhà ở vỉa hè | Giống nhà trong khu đô thị có bảo vệ |

**Trên AWS, bạn BẮT BUỘC phải có VPC để chạy bất kỳ server (EC2) nào.** Không có VPC = không có chỗ đặt server.

---

## 2. Subnet là gì? — "Phân khu" bên trong khu đô thị

### Vấn đề: Tại sao không để tất cả trong 1 chỗ?

Quay lại ví dụ khu đô thị. Bạn có miếng đất lớn (VPC). Bạn có nên xây **tất cả mọi thứ** vào cùng một khu không?

**❌ Không nên!** Ví dụ:
- Cửa hàng (nơi khách vào mua hàng) → Cần mặt tiền, cần khách vào
- Kho hàng (chứa hàng tồn) → Không cần khách vào, cần bảo mật
- Phòng két sắt (tiền, dữ liệu quan trọng) → KHÔNG AI được vào ngoài chủ

Nếu bạn để cửa hàng + kho + két sắt **cùng 1 phòng**, khách hàng vào mua hàng sẽ **thấy luôn** kho và két sắt → Nguy hiểm!

**✅ Giải pháp: Chia thành nhiều khu vực (subnets)**

### Vậy Subnet là gì?

**Subnet = "Phân khu" bên trong VPC.** Mỗi subnet là một khu vực nhỏ hơn với quy tắc riêng.

```
┌─── VPC (khu đô thị) ────────────────────────────────────────┐
│                                                               │
│  ┌─── Subnet A ──────┐    ┌─── Subnet B ──────────────────┐ │
│  │ "Khu mặt tiền"    │    │ "Khu bên trong"                │ │
│  │                    │    │                                 │ │
│  │ • Web server       │    │ • App server                   │ │
│  │ • Load balancer    │    │ • Database                     │ │
│  │ • Khách vào được   │    │ • Khách KHÔNG vào được         │ │
│  │                    │    │                                 │ │
│  └────────────────────┘    └─────────────────────────────────┘ │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Mỗi Subnet cần gì?

Mỗi subnet cần một **dải địa chỉ IP** riêng (CIDR block). Giống như mỗi khu phố cần **dải số nhà** riêng:
- Khu A: nhà số 1-100
- Khu B: nhà số 101-200

Trên AWS:
- Subnet A: `10.0.1.0/24` (256 địa chỉ IP)
- Subnet B: `10.0.2.0/24` (256 địa chỉ IP khác)

---

## 3. Public Subnet vs Private Subnet — Tại sao cần 2 loại?

### Đây là khái niệm QUAN TRỌNG NHẤT của bài này.

### Public Subnet — "Khu mặt tiền"

**Public Subnet = khu vực mà Internet CÓ THỂ truy cập vào.**

Ví dụ đời thường:
> Đây giống như **mặt tiền cửa hàng** trên phố lớn. Khách hàng (Internet users) đi ngang → thấy cửa hàng → bước vào mua đồ.

Đặc điểm:
- Machines trong public subnet **có thể nhận request từ Internet**
- Máy ở đây có **public IP** (địa chỉ mà cả thế giới thấy được)
- **Đặt gì ở đây?** Web server, Load Balancer — những thứ CẦN người ngoài truy cập

### Private Subnet — "Khu bên trong"

**Private Subnet = khu vực mà Internet KHÔNG THỂ truy cập trực tiếp.**

Ví dụ đời thường:
> Đây giống như **kho hàng phía sau cửa hàng**. Khách hàng không biết kho ở đâu, không vào được kho. Chỉ nhân viên (internal traffic) mới ra vào kho.

Đặc điểm:
- Machines trong private subnet **KHÔNG có public IP** → Internet không tìm thấy
- Không ai từ Internet truy cập trực tiếp được
- **Đặt gì ở đây?** Database, App server, Backend — những thứ KHÔNG CẦN (và KHÔNG NÊN) để Internet chạm vào

### Tại sao cần cả hai?

| Scenario | Chỉ dùng Public | Public + Private |
|----------|-----------------|------------------|
| Database | ❌ Database lộ ra Internet → hacker tấn công | ✅ Database ẩn trong private → an toàn |
| Web server | ✅ Khách truy cập được | ✅ Khách truy cập được |
| Bảo mật | ❌ Mọi thứ đều "lộ thiên" | ✅ Chỉ lộ những gì cần lộ |

### Ví dụ thực tế — Website bán hàng:

```
Khách hàng (Internet)
       │
       ▼
┌─ PUBLIC SUBNET ──────────────────┐
│                                   │
│  Web Server (hiển thị trang web)  │  ← Khách thấy và truy cập được
│                                   │
└───────────────┬───────────────────┘
                │ (internal traffic)
                ▼
┌─ PRIVATE SUBNET ─────────────────┐
│                                   │
│  Database (chứa thông tin khách,  │  ← Khách KHÔNG thấy, KHÔNG truy cập
│  đơn hàng, mật khẩu...)          │     trực tiếp được
│                                   │
└───────────────────────────────────┘
```

**Nếu hacker muốn tấn công database, họ phải:**
1. Trước tiên hack được Web Server (đã khó)
2. Rồi từ Web Server mới đến được Database (khó hơn nữa)
3. → **Defense in Depth** (bảo vệ nhiều lớp)

---

## 4. Internet Gateway — "Cổng chính" ra Internet

### Vấn đề:

Bạn đã có khu đô thị (VPC) với khu mặt tiền (public subnet). Nhưng khu đô thị vẫn **đóng cửa hoàn toàn** — chưa có đường nối ra phố lớn (Internet).

### Internet Gateway là gì?

**Internet Gateway (IGW) = "Cổng chính" kết nối VPC với Internet.**

Ví dụ đời thường:
> Giống như **cổng ra vào** của khu đô thị nối với đường quốc lộ. Không có cổng → không ai vào/ra được, khu đô thị bị cô lập.

```
Internet  ←────→  [Internet Gateway]  ←────→  Public Subnet (trong VPC)
(đường quốc lộ)        (cổng)                    (khu mặt tiền)
```

**Đặc điểm quan trọng:**
- IGW cho phép traffic đi **HAI CHIỀU**: từ Internet vào + từ VPC ra Internet
- Mỗi VPC chỉ cần **1 Internet Gateway**
- Chỉ **public subnet** có route đến IGW (private subnet KHÔNG CÓ)
- Miễn phí! (không mất tiền cho bản thân IGW)

---

## 5. NAT Gateway — "Người đại diện" cho Private Subnet

### Vấn đề:

Private subnet **không có đường ra Internet** (vì nó "private"). Nhưng máy trong private subnet vẫn cần:
- Tải updates hệ điều hành (`apt update`, `yum update`)
- Cài phần mềm (`pip install flask`)
- Gọi API bên ngoài (ví dụ: gọi API thanh toán)

Làm sao để **ra Internet** mà vẫn **không ai từ Internet vào được**?

### NAT Gateway là gì?

**NAT Gateway = "Người đại diện" giúp private subnet ra Internet ẩn danh.**

Ví dụ đời thường:
> Bạn là người nổi tiếng, không muốn lộ địa chỉ nhà. Khi cần mua đồ online, bạn thuê **người trợ lý** đặt hàng. Người giao hàng chỉ biết địa chỉ trợ lý (NAT Gateway), KHÔNG biết nhà bạn thật ở đâu.
>
> - Bạn gửi request ra ngoài (qua trợ lý) ✅
> - Bên ngoài gửi response về cho trợ lý → trợ lý chuyển lại cho bạn ✅
> - Bên ngoài muốn tự đến nhà bạn? ❌ Không biết địa chỉ!

```
Private Subnet                NAT Gateway              Internet
(nhà bạn - ẩn)       (trợ lý - có public IP)        (thế giới bên ngoài)
      │                        │                          │
      │── "Tải update" ──────→ │── dùng IP công khai ──→ │
      │                        │                          │
      │←── nhận response ─────│←── response ────────────│
      │                        │                          │
      │     Từ Internet vào?   │                          │
      │          ❌ BLOCKED     │←── ai đó cố truy cập ──│
```

**Đặc điểm:**
- NAT Gateway nằm trong **public subnet** (vì nó cần có public IP để ra Internet)
- Private subnet có route `0.0.0.0/0 → NAT Gateway`
- Traffic **1 chiều**: Private → Internet (ra được), Internet → Private (KHÔNG vào được)
- **Tốn tiền**: ~$0.045/giờ (~$32/tháng) + phí data → XÓA SAU LAB!

---

## 6. Route Table — "Biển chỉ đường" cho traffic

### Vấn đề:

Khi một packet (gói dữ liệu) rời khỏi máy, nó cần biết **đi đâu**. Route Table giống như "biển chỉ đường" trong khu đô thị.

### Route Table là gì?

**Route Table = bảng quy tắc cho biết "traffic đi đến đâu thì đi qua đâu".**

Ví dụ đời thường:
> Trong khu đô thị có biển chỉ đường:
> - "Đi đến khu A, B, C (nội bộ) → đi thẳng trong khu" 
> - "Đi ra ngoài thành phố → đi qua CỔNG CHÍNH (Internet Gateway)"

### Sự khác biệt giữa Public và Private Subnet:

**CÂU HỎI LỚN:** Public subnet và Private subnet khác nhau ở đâu?

**ĐÁP ÁN:** Chỉ khác nhau ở **ROUTE TABLE**! Không có gì khác.

| | Public Subnet Route Table | Private Subnet Route Table |
|---|---|---|
| Traffic nội bộ VPC | `10.0.0.0/16 → local` | `10.0.0.0/16 → local` |
| Traffic ra Internet | `0.0.0.0/0 → Internet Gateway` ✅ | `0.0.0.0/0 → NAT Gateway` 🔒 |
| Kết quả | Internet vào/ra tự do | Chỉ ra được, không ai vào được |

> 🔑 **KEY INSIGHT:** Một subnet trở thành "public" hay "private" **KHÔNG phải vì tên gọi**, mà vì **route table** gắn với nó có chỉ đến Internet Gateway hay không!

---

## 7. Tổng kết khái niệm — Bức tranh toàn cảnh

```
                         Internet
                            │
                    ┌───────┴───────┐
                    │  Internet     │ ← Cổng chính (miễn phí)
                    │  Gateway      │
                    └───────┬───────┘
                            │
          Route: 0.0.0.0/0 → IGW
                            │
              ┌─────────────┴─────────────┐
              │                           │
       ┌──────┴──────┐            ┌──────┴──────┐
       │   PUBLIC    │            │   PUBLIC    │
       │  SUBNET    │            │  SUBNET    │
       │  (AZ-a)    │            │  (AZ-b)    │
       │             │            │             │
       │ Web Server  │            │ Web Server  │
       │ NAT Gateway │            │             │
       └──────┬──────┘            └─────────────┘
              │
    Route: 0.0.0.0/0 → NAT GW
              │
       ┌──────┴──────┐            ┌─────────────┐
       │  PRIVATE    │            │  PRIVATE    │
       │  SUBNET    │            │  SUBNET    │
       │  (AZ-a)    │            │  (AZ-b)    │
       │             │            │             │
       │ App Server  │            │ Database    │
       │             │            │             │
       └─────────────┘            └─────────────┘

Tóm tắt:
• VPC = mạng riêng (có tường rào)
• Public Subnet = mặt tiền (khách vào được) — route đến IGW
• Private Subnet = khu bên trong (khách không vào) — route đến NAT GW
• Internet Gateway = cổng ra Internet (2 chiều)
• NAT Gateway = người đại diện (1 chiều: ra được, vào không được)
• Route Table = biển chỉ đường
```

---

# PHẦN 2: THỰC HÀNH — TẠO VPC TRÊN AWS CONSOLE

> Bây giờ bạn đã hiểu các khái niệm. Hãy tạo thật trên AWS!

---

## Step 1: Đăng nhập AWS Console

1. Mở trình duyệt → vào [https://console.aws.amazon.com](https://console.aws.amazon.com)
2. Đăng nhập bằng tài khoản AWS (email + password)
3. **Chọn Region** (góc trên bên phải): chọn **Asia Pacific (Singapore)** hoặc region gần bạn
   - Tại sao? Mọi thứ bạn tạo sẽ nằm trong region này. Singapore gần Việt Nam → nhanh hơn.

---

## Step 2: Mở VPC Console

1. Thanh search phía trên → gõ **"VPC"**
2. Click kết quả **VPC** (dưới mục Services)
3. Bạn thấy **VPC Dashboard** — đây là "bảng điều khiển" quản lý mạng

---

## Step 3: Tạo VPC (dùng wizard tự động)

### 3.1. Click "Create VPC"
- Nút màu cam **"Create VPC"** → góc trên bên phải → Click

### 3.2. Chọn "VPC and more"

Bạn thấy 2 lựa chọn:
- ❌ "VPC only" — chỉ tạo VPC trống (phải tự tạo subnet, gateway... sau)
- ✅ **"VPC and more"** — tạo VPC + tất cả components cùng lúc ← **CHỌN CÁI NÀY**

> 💡 "VPC and more" là cách AWS giúp người mới tạo đầy đủ components chỉ trong 1 click. Rất tiện!

### 3.3. Điền thông tin

Tôi sẽ giải thích **từng field**:

---

**Name tag auto-generation:** `my-first-vpc`

> Đây là tên. AWS sẽ tự đặt: `my-first-vpc-vpc`, `my-first-vpc-subnet-public1`, etc. Giúp bạn dễ nhận biết resources sau này.

---

**IPv4 CIDR block:** `10.0.0.0/16`

> **Đây là "phạm vi địa chỉ nhà"** trong VPC. Giống bạn mua đất — phải nói rõ "từ số nhà mấy đến số nhà mấy".
> 
> `10.0.0.0/16` nghĩa là: bạn có **65,536 địa chỉ IP** để dùng (từ 10.0.0.0 đến 10.0.255.255).
> 
> Chưa hiểu CIDR? Không sao — chỉ cần biết `/16` = rất nhiều địa chỉ, đủ cho hầu hết mọi use case. Giữ nguyên giá trị này.

---

**IPv6 CIDR block:** Chọn **"No IPv6 CIDR block"**

> IPv6 là phiên bản địa chỉ mới. Bỏ qua cho lab đơn giản.

---

**Tenancy:** Chọn **"Default"**

> "Default" = server của bạn chạy trên hardware chung với người khác (rẻ). "Dedicated" = hardware riêng (đắt). Chọn Default.

---

**Number of Availability Zones:** Chọn **2**

> **AZ (Availability Zone) là gì?** Mỗi AZ là một **data center vật lý riêng biệt** — nằm ở vị trí địa lý khác nhau trong cùng region.
>
> **Tại sao cần 2 AZ?** Giống mở 2 chi nhánh cửa hàng ở 2 quận. Nếu quận A mất điện → chi nhánh quận B vẫn phục vụ khách. Đây gọi là **High Availability** (HA).
>
> Với lab: 2 AZ là đủ. Production nên dùng 3.

---

**Number of public subnets:** Chọn **2**

> Mỗi AZ sẽ có 1 public subnet. Đặt Load Balancer, NAT Gateway ở đây.

---

**Number of private subnets:** Chọn **2**

> Mỗi AZ sẽ có 1 private subnet. Đặt app servers, databases ở đây.

---

**NAT gateways:** Chọn **"In 1 AZ"**

> Cho lab: 1 NAT Gateway đủ (tiết kiệm tiền ~$32/tháng).
> Production: nên chọn "1 per AZ" (HA).
>
> ⚠️ **NAT Gateway tốn tiền!** Nhớ xóa sau khi lab xong.

---

**VPC endpoints:** Chọn **"None"**

> Bỏ qua cho lab đơn giản.

---

**DNS options:** Giữ nguyên (cả 2 ticked ✅)

> Cho phép VPC dùng DNS — machines sẽ có hostname tự động.

---

### 3.4. Review & Create

- Nhìn panel **Preview** bên phải — bạn thấy diagram hiển thị VPC + subnets + IGW + NAT GW
- Kiểm tra xong → Click **"Create VPC"** 🎉

---

## Step 4: Chờ AWS tạo (2-3 phút)

AWS sẽ tạo lần lượt:
```
✅ VPC
✅ Subnets (4 cái: 2 public + 2 private)
✅ Route Tables (public + private)
✅ Internet Gateway (attached to VPC)
✅ NAT Gateway (mất 1-2 phút — cần allocate Elastic IP)
```

Khi tất cả **✅ xanh** → Click **"View VPC"**

---

## Step 5: Kiểm tra — Hiểu những gì AWS vừa tạo

### 5.1. Xem VPC
- Menu trái → **Your VPCs** → thấy `my-first-vpc-vpc`
- CIDR: `10.0.0.0/16` ✅

### 5.2. Xem Subnets
- Menu trái → **Subnets** → filter by VPC → thấy 4 subnets:

| Tên | Loại | Giải thích |
|-----|------|-----------|
| ...subnet-public1... | Public | Mặt tiền AZ-a — có route đến Internet Gateway |
| ...subnet-public2... | Public | Mặt tiền AZ-b |
| ...subnet-private1... | Private | Khu bên trong AZ-a — route đến NAT Gateway |
| ...subnet-private2... | Private | Khu bên trong AZ-b |

### 5.3. Xem Route Tables (PHẦN QUAN TRỌNG NHẤT)

Menu trái → **Route Tables** → click vào từng route table:

**Route Table của PUBLIC subnet:**

| Destination | Target | Ý nghĩa |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | "Traffic đi nội bộ VPC → gửi trực tiếp" |
| `0.0.0.0/0` | `igw-xxxx` | "Traffic đi bất cứ đâu khác → ra Internet Gateway" |

→ **Kết luận:** Public subnet có đường ra Internet trực tiếp ✅

**Route Table của PRIVATE subnet:**

| Destination | Target | Ý nghĩa |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | "Traffic nội bộ → gửi trực tiếp" |
| `0.0.0.0/0` | `nat-xxxx` | "Traffic ra ngoài → đi qua NAT Gateway" |

→ **Kết luận:** Private subnet ra Internet qua NAT (ẩn danh, 1 chiều) ✅

### 5.4. Xem Internet Gateway
- Menu trái → **Internet Gateways** → thấy `my-first-vpc-igw`
- State: **Attached** (đã gắn vào VPC)

### 5.5. Xem NAT Gateway
- Menu trái → **NAT Gateways** → thấy NAT Gateway
- State: **Available**
- Nằm trong: public subnet
- Có Elastic IP (đây là public IP của NAT — Internet thấy IP này khi private subnet ra ngoài)

---

## Step 6: 🧹 XÓA RESOURCES — QUAN TRỌNG!

> ⚠️⚠️⚠️ **Nếu bạn KHÔNG xóa, NAT Gateway sẽ tốn ~$32/tháng!**

### Cách xóa (theo thứ tự):

**Bước 1:** Xóa NAT Gateway
- VPC Console → NAT Gateways → chọn NAT → Actions → **Delete NAT gateway**
- Gõ "delete" để xác nhận
- Chờ state = `Deleted` (1-2 phút)

**Bước 2:** Release Elastic IP
- VPC Console → Elastic IPs → chọn IP (status: disassociated) → Actions → **Release Elastic IP address**

**Bước 3:** Xóa VPC
- VPC Console → Your VPCs → chọn `my-first-vpc-vpc` → Actions → **Delete VPC**
- AWS sẽ tự xóa: subnets, route tables, internet gateway, security groups
- Gõ "delete" → Confirm

**Verify:** Quay lại VPC Dashboard → không còn resources nào → ✅ Sạch!

---

# PHẦN 3: TỔNG KẾT

## Bạn vừa học được gì?

| Khái niệm | Giải thích 1 câu | Ví dụ đời thường |
|-----------|------------------|------------------|
| **VPC** | Mạng riêng của bạn trên AWS | Khu đô thị có tường rào |
| **Subnet** | Phân khu bên trong VPC | Khu vực mặt tiền / khu bên trong |
| **Public Subnet** | Subnet có route đến Internet Gateway | Cửa hàng mặt đường — khách vào được |
| **Private Subnet** | Subnet route qua NAT Gateway | Kho hàng — khách không vào được |
| **Internet Gateway** | Cổng kết nối VPC ↔ Internet (2 chiều) | Cổng chính khu đô thị |
| **NAT Gateway** | Cho private subnet ra Internet 1 chiều | Người đại diện / P.O. Box |
| **Route Table** | Quy tắc traffic đi đâu | Biển chỉ đường |
| **AZ** | Data center vật lý riêng biệt | Chi nhánh ở quận khác |

## Bí quyết nhớ:

```
Public vs Private = KHÁC NHAU CHỈ Ở ROUTE TABLE!
  • Route → IGW = PUBLIC (Internet vào/ra)
  • Route → NAT = PRIVATE (ra được, không ai vào)
```

---

## 🔗 Bài học liên quan

| Bài | Link | Mô tả |
|-----|------|-------|
| Lab Cloud Journey | [000003.awsstudygroup.com](https://000003.awsstudygroup.com/) | Lab VPC tiếng Việt (step-by-step có hình) |
| Lý thuyết IP/Subnet | [Phần 2: IP Addressing](/2026-06-01-ip-addressing-subnetting/) | Hiểu CIDR, subnet mask |
| Lý thuyết Routing | [Phần 3: Routing](/2026-06-01-routing-how-packets-travel/) | Hiểu route table chi tiết |

---

## 📖 Nguồn tham khảo

| Nguồn | Link |
|-------|------|
| AWS: Create a VPC | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html) |
| AWS: VPC example (private + NAT) | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html) |
| AWS: Internet Gateway | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) |
| AWS: NAT Gateway | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) |

---

*Bài viết thuộc series [AWS Learning Path](/2026-05-31-aws-learning-path/). Xem lộ trình đầy đủ [tại đây](/2026-05-31-aws-learning-path/).*
