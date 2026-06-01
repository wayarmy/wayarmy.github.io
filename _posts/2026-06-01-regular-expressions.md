---
layout: post
title: "Regular Expressions Deep Dive - BRE, ERE, PCRE"
date: 2026-06-01
categories: [linux]
tags: [regex, grep, sed, awk, text-processing]
---

# Regular Expressions Deep Dive - BRE, ERE, PCRE

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

**Regular Expression (Regex)** = ngôn ngữ mô tả PATTERN (mẫu) trong text.

**Ví dụ đời thường:** Bạn tìm kiếm trong danh bạ điện thoại:
- "Tìm tất cả số bắt đầu bằng 09" → pattern: `09*`
- "Tìm email có đuôi @gmail.com" → pattern: `*@gmail.com`
- "Tìm tên bắt đầu bằng Nguyễn, kết thúc bằng An" → pattern: `Nguyễn.*An`

Regex mạnh hơn wildcard thông thường — nó mô tả được patterns CỰC KỲ phức tạp:
- Validate email format
- Extract phone numbers from text
- Find and replace patterns in code
- Parse log files

**Ba "phương ngữ" Regex:**

| Loại | Tên đầy đủ | Dùng trong |
|------|-----------|-----------|
| BRE | Basic Regular Expression | grep (default), sed (default) |
| ERE | Extended Regular Expression | grep -E (egrep), sed -E, awk |
| PCRE | Perl Compatible Regular Expression | grep -P, Python, JavaScript, PHP |

---

## 2. Kiến Thức Nền Tảng — Regex Building Blocks

### 2.1 Literal Characters

```
Pattern "hello" → matches chuỗi "hello" CHÍNH XÁC
Pattern "abc" → matches "abc" trong "xyzabcdef"

Chỉ cần gõ ký tự bình thường = match chính xác ký tự đó
```

### 2.2 Metacharacters — Ký Tự Đặc Biệt

```
.       Match BẤT KỲ 1 ký tự nào (trừ newline)
^       Match ĐẦU dòng (anchor)
$       Match CUỐI dòng (anchor)
*       Lặp lại 0 hoặc nhiều lần (ký tự trước)
+       Lặp lại 1 hoặc nhiều lần (ERE/PCRE)
?       Lặp lại 0 hoặc 1 lần (ERE/PCRE)
\       Escape metacharacter (biến thành literal)
[]      Character class (match 1 trong nhóm ký tự)
|       OR (alternation, ERE/PCRE)
()      Group (capture group)
{}      Quantifier (lặp lại N lần)
```

### 2.3 Character Classes [...]

```bash
[abc]      # Match 'a' OR 'b' OR 'c' (1 ký tự)
[a-z]      # Match bất kỳ chữ thường nào (range)
[A-Z]      # Match bất kỳ chữ HOA nào
[0-9]      # Match bất kỳ chữ số nào
[a-zA-Z]   # Match bất kỳ chữ cái nào
[a-zA-Z0-9] # Match chữ cái hoặc số (alphanumeric)

[^abc]     # Match bất kỳ NGOẠI TRỪ a, b, c (negate)
[^0-9]     # Match bất kỳ KHÔNG phải số

# POSIX character classes (portable):
[[:alpha:]]  # Alphabetic [a-zA-Z]
[[:digit:]]  # Digits [0-9]
[[:alnum:]]  # Alphanumeric [a-zA-Z0-9]
[[:space:]]  # Whitespace (space, tab, newline)
[[:upper:]]  # Uppercase [A-Z]
[[:lower:]]  # Lowercase [a-z]
[[:punct:]]  # Punctuation
[[:print:]]  # Printable characters

# Ví dụ thực tế:
grep '[0-9]\{3\}-[0-9]\{4\}' file    # Phone: 123-4567
grep '[[:upper:]][[:lower:]]*' file   # Capitalized words
```

### 2.4 Quantifiers — Lặp Lại

```bash
*      # 0 hoặc nhiều lần: "ab*c" → "ac", "abc", "abbc", "abbbc"
+      # 1 hoặc nhiều lần: "ab+c" → "abc", "abbc" (NOT "ac")
?      # 0 hoặc 1 lần: "colou?r" → "color", "colour"
{n}    # Đúng n lần: "a{3}" → "aaa"
{n,}   # Ít nhất n lần: "a{2,}" → "aa", "aaa", "aaaa"...
{n,m}  # Từ n đến m lần: "a{2,4}" → "aa", "aaa", "aaaa"

# GREEDY vs LAZY (PCRE):
# Greedy (default): match NHIỀU NHẤT có thể
# Lazy (thêm ?): match ÍT NHẤT có thể

"<.*>"     # Greedy: "<b>hello</b>" → matches "<b>hello</b>" (tất cả!)
"<.*?>"    # Lazy:   "<b>hello</b>" → matches "<b>" (ít nhất)
```

### 2.5 Anchors — Neo Vị Trí

```bash
^      # Đầu dòng: "^Hello" → dòng BẮT ĐẦU bằng "Hello"
$      # Cuối dòng: "world$" → dòng KẾT THÚC bằng "world"
^...$  # Match TOÀN BỘ dòng: "^Hello$" → dòng chỉ chứa "Hello"

# PCRE anchors:
\b     # Word boundary: "\bcat\b" → "cat" nhưng KHÔNG "category"
\B     # Non-word boundary: "\Bcat" → "category" nhưng KHÔNG "cat"
\A     # Start of string (khác ^ khi multiline)
\Z     # End of string (khác $ khi multiline)
```

---

## 3. BRE vs ERE vs PCRE — Sự Khác Biệt

### 3.1 Bảng So Sánh Metacharacters

| Feature | BRE | ERE | PCRE |
|---------|-----|-----|------|
| Literal match | abc | abc | abc |
| Any char | . | . | . |
| Zero or more | * | * | * |
| One or more | \\+ | + | + |
| Zero or one | \\? | ? | ? |
| Quantifier | \\{n,m\\} | {n,m} | {n,m} |
| Grouping | \\(..\\) | (..) | (..) |
| Alternation | \\| | \| | \| |
| Backreference | \\1 | \\1 | \\1 or $1 |
| Word boundary | \\b (GNU) | \\b (GNU) | \\b |
| Lookahead | ❌ | ❌ | (?=..) |
| Lookbehind | ❌ | ❌ | (?<=..) |
| Non-greedy | ❌ | ❌ | *? +? |
| Named groups | ❌ | ❌ | (?P<name>..) |

**Key insight:** BRE cần escape `\+`, `\?`, `\{`, `\(`, `\|` → ERE KHÔNG cần escape!

### 3.2 Sử Dụng Trong Các Tools

```bash
# grep (default: BRE)
grep 'pattern' file                    # BRE
grep -E 'pattern' file                 # ERE (same as egrep)
grep -P 'pattern' file                 # PCRE (most powerful)

# sed (default: BRE)
sed 's/pattern/replacement/' file      # BRE
sed -E 's/pattern/replacement/' file   # ERE

# awk (always ERE)
awk '/pattern/ {print}' file           # ERE

# Ví dụ: Match "hello" or "world"
grep 'hello\|world' file               # BRE (escape |)
grep -E 'hello|world' file             # ERE (no escape)
grep -P 'hello|world' file             # PCRE (no escape)
```

---

## 4. Groups và Backreferences

### 4.1 Capture Groups

```bash
# Groups: (...) capture matched text cho reuse

# Tìm duplicate words: "the the", "is is"
grep -E '\b(\w+)\s+\1\b' file
#       ↑ group 1  ↑ \1 = nội dung group 1 (backreference)

# sed: Swap first and last name
echo "John Smith" | sed -E 's/^(\w+) (\w+)$/\2 \1/'
# Output: "Smith John"
# \1 = "John" (group 1), \2 = "Smith" (group 2)

# Extract parts of a pattern
echo "2026-06-01" | grep -oP '(\d{4})-(\d{2})-(\d{2})'
# Captures: \1=2026, \2=06, \3=01
```

### 4.2 Non-Capturing Groups (PCRE)

```bash
# (?:...) — group nhưng KHÔNG capture (không tạo \1)
# Dùng khi chỉ cần grouping cho alternation/quantifier, không cần backreference

grep -P '(?:https?|ftp)://\S+' file
# Match URLs starting with http://, https://, or ftp://
# (?:...) groups https?|ftp nhưng không capture
```

---

## 5. Lookahead và Lookbehind (PCRE Only)

### 5.1 Lookahead (?=...) và (?!...)

```bash
# Positive lookahead (?=...): Match nếu THEO SAU là pattern
# "Tìm 'foo' mà sau nó có 'bar'"
grep -P 'foo(?=bar)' file
# Matches "foo" trong "foobar" nhưng KHÔNG "foobaz"
# "foo" là match, "bar" chỉ là condition (không nằm trong match)

# Negative lookahead (?!...): Match nếu KHÔNG theo sau là pattern
# "Tìm 'foo' mà sau nó KHÔNG có 'bar'"
grep -P 'foo(?!bar)' file
# Matches "foo" trong "foobaz" nhưng KHÔNG "foobar"

# Ví dụ thực tế: Password validation
# Ít nhất 1 uppercase, 1 lowercase, 1 digit, min 8 chars
grep -P '^(?=.*[A-Z])(?=.*[a-z])(?=.*\d).{8,}$' passwords.txt
```

### 5.2 Lookbehind (?<=...) và (?<!...)

```bash
# Positive lookbehind (?<=...): Match nếu TRƯỚC ĐÓ là pattern
# "Tìm số mà trước nó có '$'"
grep -oP '(?<=\$)\d+' file
# Input: "Price: $500" → Output: "500" (match số, không match $)

# Negative lookbehind (?<!...): Match nếu TRƯỚC ĐÓ KHÔNG phải pattern
grep -P '(?<!un)happy' file
# Matches "happy" nhưng KHÔNG "unhappy"
```

---

## 6. PCRE Special Features

### 6.1 Shorthand Character Classes

```bash
\d     # Digit [0-9]
\D     # Non-digit [^0-9]
\w     # Word character [a-zA-Z0-9_]
\W     # Non-word character [^a-zA-Z0-9_]
\s     # Whitespace [ \t\n\r\f\v]
\S     # Non-whitespace [^ \t\n\r\f\v]

# Ví dụ:
grep -P '\d{3}-\d{3}-\d{4}' file     # Phone: 123-456-7890
grep -P '\b\w+@\w+\.\w+\b' file      # Simple email
grep -P '\s{2,}' file                 # 2+ whitespace (formatting issues)
```

### 6.2 Named Groups

```bash
# (?P<name>...) — named capture group
echo "2026-06-01" | grep -oP '(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})'

# Backreference by name:
grep -P '(?P<word>\w+)\s+(?P=word)' file    # Duplicate words using named backref
```

---

## 7. Real-World Regex Patterns

### 7.1 Common Validation Patterns

```bash
# Email (simplified, RFC 5322 compliant regex is 6000+ chars!)
grep -P '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

# IPv4 address
grep -P '\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b'

# URL
grep -P 'https?://[^\s<>"{}|\\^`\[\]]+'

# Date (YYYY-MM-DD)
grep -P '\b\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])\b'

# Vietnamese phone number
grep -P '\b(0[3-9]\d{8}|\+84[3-9]\d{8})\b'

# MAC address
grep -Pi '([0-9a-f]{2}[:-]){5}[0-9a-f]{2}'

# Strong password (8+ chars, upper, lower, digit, special)
grep -P '^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%^&*]).{8,}$'
```

### 7.2 Log Parsing

```bash
# Apache access log: extract IP, status code, URL
grep -oP '^\S+' access.log                       # IP addresses
grep -oP '"\w+ \K[^"]+' access.log               # URLs
grep -oP '" \K\d{3}' access.log                   # Status codes

# Find 5xx errors with timestamp
grep -P '\d{4}-\d{2}-\d{2}.*\s5\d{2}\s' app.log

# Extract JSON values
grep -oP '"error":\s*"\K[^"]+' response.json      # Extract error messages
```

---

## 8. Performance Tips

```bash
# 1. Anchor khi có thể (^, $, \b) → giảm backtracking
grep -P '^\d{4}-\d{2}-\d{2}' log     # FAST (anchored)
grep -P '\d{4}-\d{2}-\d{2}' log      # SLOWER (scans entire line)

# 2. Cụ thể hơn tốt hơn
grep -P '[0-9]+' file                  # OK
grep -P '\d+' file                     # Tương đương nhưng ngắn hơn

# 3. Tránh catastrophic backtracking
# BAD: (a+)+b — exponential backtracking on "aaaaaaaaac"
# GOOD: a+b — linear

# 4. Dùng possessive quantifiers (PCRE)
# *+ ++ ?+ — không backtrack, fail fast
grep -P '\d++\.' file    # Faster than \d+\.

# 5. grep -F cho fixed strings (no regex, fastest)
grep -F 'exact string' file    # Literal match, no regex processing
```

---

## 9. sed và awk Regex Usage

### 9.1 sed Regex

```bash
# Substitute: s/pattern/replacement/flags
sed 's/old/new/g' file              # Replace all occurrences
sed -E 's/([0-9]+)/[\1]/g' file     # Wrap numbers in brackets
sed '/^#/d' file                    # Delete comment lines
sed -n '/ERROR/p' file              # Print only ERROR lines (like grep)
sed 's/^[[:space:]]*//' file        # Remove leading whitespace
sed '/^$/d' file                    # Remove empty lines
```

### 9.2 awk Regex

```bash
# awk uses ERE natively
awk '/pattern/ {action}' file

# Match lines containing "error" (case insensitive)
awk 'tolower($0) ~ /error/ {print NR": "$0}' file

# Extract fields matching pattern
awk -F: '$1 ~ /^admin/ {print $1, $3}' /etc/passwd

# Field-specific regex
awk '$3 ~ /^[0-9]+$/ && $3 > 100 {print}' data.txt
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Regex Cheat Sheet

```
.          Any character         \d  Digit [0-9]
^          Start of line         \w  Word char [a-zA-Z0-9_]
$          End of line           \s  Whitespace
*          0+ times              \b  Word boundary
+          1+ times              []  Character class
?          0 or 1                [^] Negated class
{n,m}      n to m times          ()  Capture group
|          OR                    \1  Backreference
(?=..)     Lookahead             (?<=..) Lookbehind
(?!..)     Neg lookahead         (?<!..) Neg lookbehind
```

### 10.2 Key Takeaways

1. **BRE** (grep, sed default): escape special chars `\+`, `\?`, `\(`, `\|`
2. **ERE** (grep -E, awk): no escaping needed for `+`, `?`, `()`, `|`
3. **PCRE** (grep -P): most powerful — lookahead/behind, `\d\w\s`, named groups
4. **Always anchor** (`^`, `$`, `\b`) for performance and precision
5. **Character classes** `[...]` cho flexibility, POSIX `[[:class:]]` cho portability
6. **Capture groups** `()` + backreferences `\1` cho find-and-replace
7. **Greedy default**, thêm `?` cho lazy matching
8. **Test regex** trước khi dùng: https://regex101.com/

### 10.3 Tài Liệu Tham Khảo

- Regular-Expressions.info: https://www.regular-expressions.info/
- regex101.com — interactive regex tester + debugger
- POSIX standard: IEEE Std 1003.1 (BRE/ERE specification)
- man pcrepattern (PCRE syntax)
- man grep, man sed, man awk
- "Mastering Regular Expressions" by Jeffrey Friedl (O'Reilly)
- GNU grep manual: https://www.gnu.org/software/grep/manual/

---

*Bài viết tiếp theo: Text Processing Tools — sed, awk, jq, xargs*
