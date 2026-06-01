---
layout: post
title: "Email Protocols Deep Dive - SMTP Relay Chain, MX Records, IMAP vs POP3, SPF/DKIM/DMARC"
date: 2026-06-01
categories: [networking]
tags: [smtp, imap, pop3, email, spf, dkim, dmarc]
---

# Email Protocols Deep Dive — SMTP Relay Chain, MX Records, IMAP vs POP3, SPF/DKIM/DMARC

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [Email System Architecture — Kiến trúc tổng quan](#2-email-system-architecture--kiến-trúc-tổng-quan)
3. [SMTP — Simple Mail Transfer Protocol](#3-smtp--simple-mail-transfer-protocol)
4. [SMTP Relay Chain — Hành trình của email](#4-smtp-relay-chain--hành-trình-của-email)
5. [MX Records — Tìm đường cho email](#5-mx-records--tìm-đường-cho-email)
6. [IMAP vs POP3 — Đọc email](#6-imap-vs-pop3--đọc-email)
7. [SPF — Sender Policy Framework](#7-spf--sender-policy-framework)
8. [DKIM — DomainKeys Identified Mail](#8-dkim--domainkeys-identified-mail)
9. [DMARC — Domain-based Message Authentication](#9-dmarc--domain-based-message-authentication)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### Email như hệ thống bưu chính

Hệ thống email hoạt động giống hệ thống bưu chính truyền thống:

| Bưu chính | Email |
|---|---|
| Bạn viết thư, bỏ vào hòm thư | Soạn email, nhấn Send |
| Hòm thư → Bưu cục gần nhà | Email client → SMTP server (submission) |
| Bưu cục phân loại theo mã zip | SMTP server tra cứu MX record |
| Xe chuyển phát giữa các bưu cục | SMTP relay giữa các mail servers |
| Bưu cục đến → hòm thư người nhận | MX server → mailbox |
| Người nhận mở hòm thư lấy thư | IMAP/POP3 → email client |
| Con dấu bưu điện (xác thực) | SPF/DKIM/DMARC |
| Phong bì (địa chỉ ngoài) | Envelope (MAIL FROM, RCPT TO) |
| Nội dung thư bên trong | Message body (From:, To:, Subject:) |

### Các giao thức email

```
SENDING (gửi):
  SMTP (port 25, 587, 465) — "Xe chuyển phát" — vận chuyển email

RECEIVING (đọc):
  IMAP (port 143, 993) — "Đọc thư TẠI bưu cục" — sync, online
  POP3 (port 110, 995) — "Mang thư VỀ NHÀ" — download, offline

AUTHENTICATION (xác thực):
  SPF — "Danh sách bưu tá hợp lệ"
  DKIM — "Con dấu chống giả mạo"
  DMARC — "Chính sách xử lý thư giả"
```

---

## 2. Email System Architecture — Kiến trúc tổng quan

### 2.1 Các thành phần

```
┌─────────────────────────────────────────────────────────────────┐
│                      Email Infrastructure                         │
│                                                                   │
│  Sender Side                              Recipient Side          │
│  ┌──────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌──────┐       │
│  │ MUA  │───→│ MSA │───→│ MTA │···→│ MTA │───→│ MDA  │       │
│  │      │587 │     │ 25 │     │ 25 │     │    │      │       │
│  └──────┘    └─────┘    └─────┘    └─────┘    └──┬───┘       │
│  (Gmail)   (submit)    (relay)    (destination)   │            │
│                                                    ↓            │
│                                              ┌──────────┐      │
│                                              │ Mailbox  │      │
│                                              │ (Storage)│      │
│                                              └────┬─────┘      │
│                                                   │             │
│                                              IMAP/POP3          │
│                                                   ↓             │
│                                              ┌──────┐          │
│                                              │ MUA  │          │
│                                              │(Reader)│          │
│                                              └──────┘          │
└─────────────────────────────────────────────────────────────────┘
```

| Component | Tên đầy đủ | Vai trò | Ví dụ |
|---|---|---|---|
| **MUA** | Mail User Agent | Client soạn/đọc email | Gmail app, Outlook, Thunderbird |
| **MSA** | Mail Submission Agent | Nhận email từ MUA (port 587) | Postfix (submission) |
| **MTA** | Mail Transfer Agent | Chuyển tiếp email (port 25) | Postfix, Sendmail, Exchange |
| **MDA** | Mail Delivery Agent | Giao email vào mailbox | Dovecot, procmail |
| **MRA** | Mail Retrieval Agent | Cung cấp IMAP/POP3 | Dovecot |

### 2.2 Ports và Encryption

| Port | Protocol | Encryption | Mục đích |
|---|---|---|---|
| 25 | SMTP | None / STARTTLS | MTA-to-MTA relay |
| 587 | SMTP Submission | STARTTLS (required) | Client → server (send email) |
| 465 | SMTPS | Implicit TLS | Client → server (legacy, revived) |
| 143 | IMAP | None / STARTTLS | Read email (sync) |
| 993 | IMAPS | Implicit TLS | Read email (encrypted) |
| 110 | POP3 | None / STARTTLS | Download email |
| 995 | POP3S | Implicit TLS | Download email (encrypted) |

---

## 3. SMTP — Simple Mail Transfer Protocol

### 3.1 SMTP Session Flow

```
Telnet simulation (manual SMTP):

Client                                  Server (port 25)
  │                                       │
  │──── TCP Connect ─────────────────────→│
  │←─── 220 mail.example.com SMTP Ready ──│  Banner
  │                                       │
  │──── EHLO client.example.com ─────────→│  Introduce yourself
  │←─── 250-mail.example.com             ──│  Capabilities list
  │     250-SIZE 52428800                  │
  │     250-8BITMIME                       │
  │     250-STARTTLS                       │
  │     250 AUTH PLAIN LOGIN               │
  │                                       │
  │──── STARTTLS ────────────────────────→│  Upgrade to TLS
  │←─── 220 Ready for TLS ────────────────│
  │     [TLS Handshake]                    │
  │                                       │
  │──── AUTH PLAIN [base64 credentials] ──→│  Authenticate
  │←─── 235 Authentication successful ────│
  │                                       │
  │──── MAIL FROM:<alice@sender.com> ────→│  Envelope sender
  │←─── 250 OK ───────────────────────────│
  │                                       │
  │──── RCPT TO:<bob@recipient.com> ─────→│  Envelope recipient
  │←─── 250 OK ───────────────────────────│
  │                                       │
  │──── DATA ────────────────────────────→│  Start message body
  │←─── 354 Start mail input ─────────────│
  │                                       │
  │──── From: Alice <alice@sender.com>    │  Headers
  │     To: Bob <bob@recipient.com>       │
  │     Subject: Hello!                    │
  │     Date: Thu, 22 Jul 2026 10:00:00   │
  │                                       │
  │     Hi Bob, how are you?              │  Body
  │                                       │
  │──── .  (single dot on line) ─────────→│  End of message
  │←─── 250 OK: queued as ABC123 ─────────│  Accepted!
  │                                       │
  │──── QUIT ────────────────────────────→│
  │←─── 221 Bye ──────────────────────────│
```

### 3.2 SMTP Commands

| Command | Mục đích | Ví dụ |
|---|---|---|
| `EHLO` | Introduce client (Extended HELLO) | `EHLO mail.sender.com` |
| `MAIL FROM:` | Envelope sender (Return-Path) | `MAIL FROM:<alice@sender.com>` |
| `RCPT TO:` | Envelope recipient(s) | `RCPT TO:<bob@recipient.com>` |
| `DATA` | Bắt đầu message content | `DATA` → 354 response |
| `QUIT` | Kết thúc session | `QUIT` → 221 response |
| `RSET` | Reset session state | Cancel current message |
| `VRFY` | Verify address (usually disabled) | `VRFY bob@example.com` |
| `AUTH` | Authenticate (RFC 4954) | `AUTH PLAIN [base64]` |
| `STARTTLS` | Upgrade to TLS | RFC 3207 |

### 3.3 SMTP Response Codes

| Code | Ý nghĩa | Hành động |
|---|---|---|
| 220 | Service ready | Proceed with EHLO |
| 250 | OK, completed | Continue |
| 354 | Start mail input | Send message, end with `.` |
| 421 | Service not available | Retry later |
| 450 | Mailbox unavailable (temp) | Retry later |
| 452 | Insufficient storage | Retry later |
| 500 | Syntax error | Fix command |
| 550 | Mailbox not found (permanent) | Don't retry |
| 552 | Message size exceeded | Reduce size |
| 554 | Transaction failed | Check policy |

### 3.4 Envelope vs Headers (QUAN TRỌNG!)

```
Ví dụ đời thường: Phong bì có địa chỉ khác với nội dung thư bên trong.
Bạn có thể viết "Gửi: Ông A" trên phong bì, nhưng trong thư viết "Dear Ông B"

SMTP Envelope (dùng cho routing):
  MAIL FROM: <bounce@newsletter.company.com>    ← Return-Path
  RCPT TO: <bob@recipient.com>                   ← Actual delivery address

Message Headers (user sees):
  From: "Company Newsletter" <news@company.com>  ← Display address
  To: "Bob" <bob@recipient.com>
  Reply-To: support@company.com                   ← Reply goes here
  
Thực tế: Envelope FROM ≠ Header From (hợp lệ nhưng dễ bị lạm dụng → cần SPF/DKIM/DMARC)
```

---

## 4. SMTP Relay Chain — Hành trình của email

### 4.1 Store-and-Forward Model

SMTP dùng mô hình **store-and-forward** — giống bưu chính:

```
alice@gmail.com gửi email tới bob@company.com:

Step 1: Gmail MUA → Gmail MSA (smtp.gmail.com:587)
        [Authenticated submission with STARTTLS]

Step 2: Gmail MTA looks up MX record for company.com
        → mx1.company.com (priority 10)
        → mx2.company.com (priority 20, backup)

Step 3: Gmail MTA → company.com MTA (mx1.company.com:25)
        [STARTTLS if supported, verify SPF/DKIM]

Step 4: company.com MTA → company.com MDA
        [Deliver to bob's mailbox]

Step 5: Bob's MUA ← IMAP/POP3 ← company.com mailbox
```

### 4.2 Received Headers — Tracking route

Mỗi MTA thêm một `Received:` header. Đọc từ dưới lên = hành trình email.

```email
Received: from mx1.company.com (mx1.company.com [203.0.113.10])
    by internal-mda.company.com with ESMTPS id abc123
    for <bob@company.com>; Thu, 22 Jul 2026 10:00:05 +0000
Received: from mail-yw1-f169.google.com (mail-yw1-f169.google.com [209.85.128.169])
    by mx1.company.com with ESMTPS id def456
    for <bob@company.com>; Thu, 22 Jul 2026 10:00:02 +0000
Received: by mail-yw1-f169.google.com with SMTP id xyz789
    for <bob@company.com>; Thu, 22 Jul 2026 09:59:58 +0000

Đọc từ dưới lên:
1. 09:59:58 — Gmail internal processing
2. 10:00:02 — Gmail → company MX server
3. 10:00:05 — company MX → internal MDA
Total transit: 7 seconds
```

### 4.3 Email Bounce và NDR

```
Nếu delivery fail:

Scenario: bob@company.com không tồn tại

1. Gmail MTA → company.com MTA
2. company.com MTA respond: 550 User unknown
3. Gmail MTA tạo NDR (Non-Delivery Report) → gửi về MAIL FROM address
4. NDR gửi tới alice@gmail.com: "Your email could not be delivered"

NDR = "Return to Sender" stamp trên thư
```

---

## 5. MX Records — Tìm đường cho email

### 5.1 MX Record là gì?

**MX (Mail Exchanger) record** là DNS record chỉ định **server nào nhận email** cho domain.

```dns
company.com.    IN    MX    10    mx1.company.com.
company.com.    IN    MX    20    mx2.company.com.
company.com.    IN    MX    30    mx-backup.company.com.
```

| Priority | Server | Ý nghĩa |
|---|---|---|
| 10 (thấp nhất = ưu tiên nhất) | mx1.company.com | Primary — thử trước |
| 20 | mx2.company.com | Secondary — nếu primary down |
| 30 | mx-backup.company.com | Tertiary — backup cuối |

### 5.2 MX Lookup Process

```bash
# Sending MTA muốn gửi email tới bob@company.com:

# 1. Lookup MX records
$ dig company.com MX +short
10 mx1.company.com.
20 mx2.company.com.

# 2. Resolve MX hostnames to IPs
$ dig mx1.company.com A +short
203.0.113.10

# 3. Connect to lowest priority (10) first
# → TCP connect to 203.0.113.10:25

# 4. Nếu mx1 down (timeout/reject) → try mx2 (priority 20)
```

### 5.3 Fallback khi không có MX record

```
Nếu domain KHÔNG có MX record:
→ Sender MTA fall back tới A record (RFC 5321 Section 5.1)
→ Kết nối trực tiếp tới IP của domain

Ví dụ:
$ dig tiny-site.com MX  → no results
$ dig tiny-site.com A   → 93.184.216.34
→ Gửi email trực tiếp tới 93.184.216.34:25

Null MX (RFC 7505) — chỉ rõ domain KHÔNG nhận email:
tiny-site.com. IN MX 0 .
→ "Domain này không nhận email" → sender trả ngay 550 error
```

---

## 6. IMAP vs POP3 — Đọc email

### 6.1 POP3 (Post Office Protocol v3) — RFC 1939

**Ví dụ đời thường**: POP3 giống **lấy thư từ hòm thư về nhà** — thư được chuyển về máy bạn, không còn ở server (mặc định).

```
POP3 Session:
Client → Server (port 110/995):

+OK POP3 server ready
USER bob
+OK
PASS [REDACTED_PASSWORD]
+OK bob has 3 messages (12345 octets)
STAT
+OK 3 12345
LIST
+OK 3 messages
1 4567
2 3456
3 4322
.
RETR 1
+OK 4567 octets
[message content...]
.
DELE 1
+OK message 1 deleted
QUIT
+OK POP3 server signing off
```

### 6.2 IMAP (Internet Message Access Protocol) — RFC 9051

**Ví dụ đời thường**: IMAP giống **đọc thư TẠI bưu cục** — thư vẫn lưu ở server, bạn chỉ xem copy. Mọi thiết bị đều thấy cùng thư.

```
IMAP Session:
Client → Server (port 143/993):

* OK IMAP server ready
a1 LOGIN bob [REDACTED_PASSWORD]
a1 OK LOGIN completed
a2 SELECT INBOX
* 150 EXISTS
* 3 RECENT
* FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
a2 OK [READ-WRITE] SELECT completed
a3 FETCH 150 (BODY[HEADER] FLAGS)
* 150 FETCH (FLAGS (\Recent) BODY[HEADER] {345}
From: alice@sender.com
Subject: Hello!
Date: Thu, 22 Jul 2026 10:00:00 +0000
)
a3 OK FETCH completed
a4 STORE 150 +FLAGS (\Seen)
a4 OK STORE completed
a5 LOGOUT
* BYE IMAP server closing connection
a5 OK LOGOUT completed
```

### 6.3 So sánh IMAP vs POP3

| Feature | POP3 | IMAP |
|---|---|---|
| **Email storage** | Downloaded to client | Stays on server |
| **Multi-device sync** | ❌ (mỗi device có copy riêng) | ✅ (cùng view mọi device) |
| **Folders** | ❌ Chỉ INBOX | ✅ Server-side folders |
| **Search** | Client-side only | ✅ Server-side search |
| **Partial fetch** | ❌ Download toàn bộ | ✅ Fetch headers/parts |
| **Offline** | ✅ Đọc offline (downloaded) | ⚠️ Cần cache |
| **Server storage** | Thấp (xóa sau download) | Cao (lưu mọi thư) |
| **Bandwidth** | Tốn (download all) | Tiết kiệm (fetch on demand) |
| **RFC** | 1939 | 9051 (IMAP4rev2) |
| **Best for** | Single device, backup | Multiple devices, modern usage |

### 6.4 IMAP Extensions quan trọng

| Extension | RFC | Mục đích |
|---|---|---|
| IDLE | 2177 | Push notifications (không cần poll) |
| CONDSTORE | 4551 | Conditional operations, change tracking |
| MOVE | 6851 | Atomic move between folders |
| SORT/THREAD | 5256 | Server-side sort and threading |
| COMPRESS | 4978 | DEFLATE compression |
| QUOTA | 9208 | Mailbox quota management |

---

## 7. SPF — Sender Policy Framework

### 7.1 Vấn đề: Email Spoofing

**Vấn đề**: SMTP cho phép BẤT KỲ AI gửi email với BẤT KỲ FROM address nào. Giống như viết thư với tên người gửi giả trên phong bì.

```
Kẻ xấu gửi:
MAIL FROM:<ceo@company.com>    ← Giả mạo CEO!
RCPT TO:<finance@company.com>
Subject: Urgent wire transfer needed
"Please transfer $50,000 to account XYZ..."

→ Finance nhận email "từ CEO" → chuyển tiền → MẤT TIỀN!
```

### 7.2 SPF là gì?

**SPF (Sender Policy Framework)** — RFC 7208 — là DNS TXT record liệt kê **servers nào được phép gửi email** cho domain.

**Ví dụ đời thường**: SPF giống **danh sách bưu tá hợp lệ** — "Chỉ bưu tá A, B, C được phép gửi thư nhân danh công ty chúng tôi. Ai khác gửi = giả mạo."

### 7.3 SPF Record Syntax

```dns
company.com.  IN  TXT  "v=spf1 ip4:203.0.113.0/24 include:_spf.google.com include:amazonses.com -all"
```

| Phần | Ý nghĩa |
|---|---|
| `v=spf1` | SPF version 1 |
| `ip4:203.0.113.0/24` | Cho phép IP range 203.0.113.x |
| `include:_spf.google.com` | Cho phép Google Workspace servers |
| `include:amazonses.com` | Cho phép Amazon SES |
| `-all` | REJECT tất cả IP khác |

**Qualifiers:**

| Qualifier | Ý nghĩa | Action |
|---|---|---|
| `+` (default) | Pass | Cho phép |
| `-` | Fail (Hard) | Reject |
| `~` | SoftFail | Accept nhưng mark suspicious |
| `?` | Neutral | Không opinion |

### 7.4 SPF Validation Flow

```
Receiving server nhận email:

1. Extract MAIL FROM domain: "company.com"
2. DNS lookup: company.com TXT record
3. Parse SPF: "v=spf1 ip4:203.0.113.0/24 -all"
4. Check: sending IP = 203.0.113.15? → YES → PASS ✅
   Check: sending IP = 192.168.1.100? → NO match → FAIL ❌ (reject)
```

### 7.5 SPF Limitations

| Limitation | Giải thích |
|---|---|
| Checks envelope MAIL FROM only | Không check header From: (user sees) |
| Breaks on forwarding | Email forwarded = new IP = SPF fail |
| 10 DNS lookup limit | Quá nhiều `include` = permerror |
| Doesn't verify content | Chỉ check IP, không check nội dung |

---

## 8. DKIM — DomainKeys Identified Mail

### 8.1 DKIM là gì?

**DKIM (DomainKeys Identified Mail)** — RFC 6376 — thêm **chữ ký số** vào email headers. Receiver verify chữ ký bằng public key lưu trong DNS.

**Ví dụ đời thường**: DKIM giống **con dấu sáp** (wax seal) trên thư thời xưa. Chỉ người có con dấu (private key) mới đóng được, ai cũng có thể kiểm tra (public key trong DNS). Nếu nội dung bị sửa, dấu sáp sẽ bị vỡ.

### 8.2 DKIM Hoạt động thế nào

```
SENDING (ký):
1. Sending MTA chọn headers + body để ký
2. Hash (canonicalized headers + body)
3. Sign hash với private key (RSA/Ed25519)
4. Thêm DKIM-Signature header vào email

RECEIVING (verify):
1. Receiver lấy DKIM-Signature header
2. Extract "d=" (domain) và "s=" (selector)
3. DNS lookup: selector._domainkey.domain → public key
4. Verify signature bằng public key
5. PASS ✅ hoặc FAIL ❌
```

### 8.3 DKIM-Signature Header

```email
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
    d=company.com; s=selector1;
    h=from:to:subject:date:message-id;
    bh=base64-hash-of-body;
    b=base64-signature-of-headers;
```

| Tag | Ý nghĩa | Ví dụ |
|---|---|---|
| `v` | Version | 1 |
| `a` | Algorithm | rsa-sha256, ed25519-sha256 |
| `c` | Canonicalization | relaxed/relaxed (header/body) |
| `d` | Signing domain | company.com |
| `s` | Selector (DNS key name) | selector1 |
| `h` | Signed headers | from:to:subject:date |
| `bh` | Body hash (Base64) | hash of canonicalized body |
| `b` | Signature (Base64) | RSA signature |

### 8.4 DKIM DNS Record

```dns
; DNS TXT record for DKIM public key
selector1._domainkey.company.com.  IN  TXT  (
    "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQ..."
)
```

```bash
# Lookup DKIM key
dig selector1._domainkey.company.com TXT +short
# "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4..."
```

### 8.5 DKIM Alignment

DKIM signature domain (`d=`) phải "align" với header From: domain:
```
Header From: alice@company.com
DKIM d=company.com             → Aligned ✅ (exact match)
DKIM d=mail.company.com        → Aligned ✅ (relaxed, subdomain)
DKIM d=otherdomain.com         → NOT aligned ❌
```

---

## 9. DMARC — Domain-based Message Authentication

### 9.1 DMARC là gì?

**DMARC (Domain-based Message Authentication, Reporting and Conformance)** — RFC 7489 — kết hợp SPF + DKIM và thêm:
1. **Policy**: Nói receiver phải làm gì khi authentication fail (none/quarantine/reject)
2. **Alignment**: Kiểm tra SPF domain hoặc DKIM domain phải match header From:
3. **Reporting**: Receiver gửi report về authentication results cho domain owner

**Ví dụ đời thường**: DMARC giống **chính sách bảo mật** — "Nếu thư không có con dấu hợp lệ (DKIM) HOẶC không được gửi từ bưu tá đăng ký (SPF), thì HÃY VỨT ĐI (reject) và báo cáo cho chúng tôi."

### 9.2 DMARC Record

```dns
_dmarc.company.com.  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@company.com; ruf=mailto:dmarc-forensic@company.com; adkim=r; aspf=r; pct=100"
```

| Tag | Ý nghĩa | Giá trị |
|---|---|---|
| `v` | Version | DMARC1 |
| `p` | Policy | none / quarantine / reject |
| `sp` | Subdomain policy | none / quarantine / reject |
| `rua` | Aggregate report email | mailto:dmarc@company.com |
| `ruf` | Forensic report email | mailto:forensic@company.com |
| `adkim` | DKIM alignment mode | r (relaxed) / s (strict) |
| `aspf` | SPF alignment mode | r (relaxed) / s (strict) |
| `pct` | Percentage to apply policy | 100 (apply to all) |

### 9.3 DMARC Policy Levels

| Policy | Ý nghĩa | Khi nào dùng |
|---|---|---|
| `p=none` | Chỉ monitor, không hành động | Bắt đầu triển khai, thu thập data |
| `p=quarantine` | Đánh dấu suspicious (spam folder) | Sau khi confirm data, trước reject |
| `p=reject` | Từ chối email fail | Full deployment, confident |

### 9.4 DMARC Validation Flow

```
Email arrives at recipient MTA:

┌───────────────────────────────────────────────────────────┐
│ Header From: alice@company.com                             │
│                                                             │
│ Step 1: Check SPF                                          │
│   MAIL FROM: alice@company.com                             │
│   Sending IP: 203.0.113.15                                 │
│   SPF record → IP matches → SPF PASS ✅                   │
│   Alignment: MAIL FROM domain = From domain → ALIGNED ✅   │
│                                                             │
│ Step 2: Check DKIM                                         │
│   DKIM-Signature d=company.com                             │
│   Lookup public key → verify signature → DKIM PASS ✅      │
│   Alignment: DKIM d= matches From domain → ALIGNED ✅      │
│                                                             │
│ Step 3: DMARC evaluation                                   │
│   SPF PASS + Aligned = ✅ OR                               │
│   DKIM PASS + Aligned = ✅                                 │
│   → DMARC PASS ✅                                         │
│                                                             │
│ Step 4: Policy (p=reject)                                  │
│   PASS → Deliver to inbox                                  │
│   FAIL → Reject (bounce back)                              │
└───────────────────────────────────────────────────────────┘
```

### 9.5 SPF + DKIM + DMARC Together

```
┌──────────────────────────────────────────────────────────────┐
│              Email Authentication Stack                        │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ DMARC (RFC 7489) — Policy & Reporting                    │ │
│  │ "What to do when SPF/DKIM fail + alignment check"        │ │
│  └────────┬───────────────────────────────┬────────────────┘ │
│           │                               │                    │
│  ┌────────┴────────┐            ┌────────┴────────┐          │
│  │ SPF (RFC 7208)  │            │ DKIM (RFC 6376) │          │
│  │                  │            │                  │          │
│  │ "Who is allowed │            │ "Is the message │          │
│  │  to send for    │            │  intact and from │          │
│  │  this domain?"  │            │  authorized      │          │
│  │                  │            │  signer?"        │          │
│  │ Checks: IP      │            │ Checks: Content │          │
│  │ Against: DNS TXT│            │ Against: DNS key │          │
│  └─────────────────┘            └─────────────────┘          │
│                                                                │
│  Benefits:                                                     │
│  SPF alone:  Stops spoofing from wrong IP                     │
│  DKIM alone: Detects content tampering                        │
│  DMARC:      Enforces policy + provides visibility             │
│  All three:  Comprehensive protection against spoofing         │
└──────────────────────────────────────────────────────────────┘
```

### 9.6 Deployment Roadmap

```
Phase 1 (Month 1): Monitor
  _dmarc.company.com TXT "v=DMARC1; p=none; rua=mailto:dmarc@company.com"
  → Collect reports, identify legitimate senders

Phase 2 (Month 2-3): Soft enforcement
  _dmarc.company.com TXT "v=DMARC1; p=quarantine; pct=25; rua=..."
  → 25% of failing emails quarantined
  → Gradually increase pct to 100

Phase 3 (Month 4+): Full enforcement
  _dmarc.company.com TXT "v=DMARC1; p=reject; rua=...; ruf=..."
  → All failing emails rejected
```

---

## 10. Tổng kết và Best Practices

### 10.1 Email Security Checklist

```
□ SPF record configured (list all legitimate sending IPs/services)
□ DKIM configured (sign outgoing email, publish public key)
□ DMARC configured (start p=none, move to p=reject)
□ DMARC reports monitored (rua= emails)
□ TLS enforced (STARTTLS on port 587)
□ MTA-STS configured (force TLS between servers)
□ DANE/TLSA records (certificate pinning for MTA-to-MTA)
□ Reverse DNS (PTR record) matches sending hostname
□ Not on blacklists (mxtoolbox.com/blacklists)
□ BIMI configured (brand logo in email clients)
```

### 10.2 Troubleshooting Email

```bash
# 1. Check MX records
dig company.com MX +short

# 2. Check SPF record
dig company.com TXT +short | grep spf

# 3. Check DKIM key
dig selector1._domainkey.company.com TXT +short

# 4. Check DMARC policy
dig _dmarc.company.com TXT +short

# 5. Test SMTP connection
openssl s_client -connect mx1.company.com:25 -starttls smtp

# 6. Send test email and check headers
# Look for: Authentication-Results header in received email
# Authentication-Results: mx.google.com;
#   spf=pass (google.com: domain of alice@company.com)
#   dkim=pass header.d=company.com
#   dmarc=pass (p=REJECT)

# 7. Use online tools:
# - mxtoolbox.com (MX, SPF, DKIM, blacklist check)
# - mail-tester.com (send test email, get score)
# - dmarcian.com (DMARC reports analysis)
```

### 10.3 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| RFC 5321 | SMTP — Simple Mail Transfer Protocol |
| RFC 6409 | Message Submission (port 587) |
| RFC 9051 | IMAP4rev2 |
| RFC 1939 | POP3 |
| RFC 7208 | SPF — Sender Policy Framework |
| RFC 6376 | DKIM — DomainKeys Identified Mail |
| RFC 7489 | DMARC — Domain-based Message Authentication |
| RFC 8461 | MTA-STS — SMTP MTA Strict Transport Security |
| RFC 8617 | ARC — Authenticated Received Chain |

### 10.4 Câu hỏi ôn tập

1. Mô tả hành trình email từ sender → recipient. Qua những components nào?
2. SMTP envelope (MAIL FROM/RCPT TO) khác header (From:/To:) thế nào? Tại sao quan trọng?
3. MX record priority hoạt động ra sao? Nếu không có MX thì sao?
4. IMAP vs POP3: khác nhau thế nào? Khi nào dùng POP3 hợp lý?
5. SPF kiểm tra gì? Limitation chính của SPF?
6. DKIM hoạt động thế nào? Selector là gì?
7. DMARC kết hợp SPF và DKIM bằng cách nào? "Alignment" nghĩa là gì?
8. Triển khai DMARC nên bắt đầu với policy gì? Tại sao?
9. Email forwarding ảnh hưởng SPF/DKIM thế nào? ARC giải quyết ra sao?
10. Liệt kê các DNS records cần cho email security đầy đủ.

---

*Bài viết được tham khảo từ RFC 5321 (SMTP), RFC 9051 (IMAP4rev2), RFC 7208 (SPF), RFC 6376 (DKIM), RFC 7489 (DMARC), RFC 8461 (MTA-STS), và tài liệu của Google Postmaster Tools, Microsoft 365 admin.*
