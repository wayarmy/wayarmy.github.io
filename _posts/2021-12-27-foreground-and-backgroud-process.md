---
layout: post
title:  "Foreground và background process trên các hệ điều hành linux/unix-like"
subtitle: Linux process
gh-repo: wayarmy/wayarmy.github.io
tags: [sysadmin]
comments: true
date:   2021-10-10
categories: Sysadmin
---

**Làm việc với foreground và background process**

> Vì một số concerns của mình và một số bạn khác nên trong bài viết này mình sẽ sử dụng `process` để thay thế cho `job`. Nếu bạn đọc ở đâu đó thấy họ nói về `background job`, hoặc `foreground job` thì có thể hiểu `job` là `process` như trong bài viết này mình đề cập


## 1. Foreground process là gì ?

`Foreground process` có thể hiểu đơn giản là các process đã/đang được tạo ra trên shell mình đang làm việc. Ví dụ khi bạn gõ lệnh `nginx -g daemon off`, bạn sẽ khởi động process `nginx` này ngay trên env shell mà bạn đang làm việc, do đó process này sẽ bị phụ thuộc vào chính môi trường shell mà bạn đang làm việc. Ví dụ nếu bạn terminate shell env này đi thì process đó cũng sẽ bị terminate đi.

Một số tính chất của `foreground process`

- Bị phụ thuộc trực tiếp vào môi trường shell mà bạn đang làm việc trên đó

- Có thể start/stop bằng các phím tắt cơ bản trên shell (vì nó hoạt động trên chính shell env)

## 2. Background process là gì ?

`Background process` được hiểu là các process hoạt động không phụ thuộc vào shell mà bạn làm việc cùng. Cách tạo 1 background process khá đơn giản: sử dụng dấu `&` vào sau command khởi tạo process đó hoặc sử dụng `nohup command`. Ví dụ: `nginx -g daemon off &`. Do đó `background process có 1 số tính chất sau:

- Không phụ thuộc vào trạng thái của bất kỳ shell env nào. Do đó nếu bạn đang làm việc một shell nào đó và start 1 `background process`, khi bạn thoát terminal đang làm việc, `background process` đó sẽ vẫn hoạt động bình thường.
- Không thể sử dụng các phím tắt của shell để stop `background process`

(và đồng thời `foreground` hay `background` process đều mang đầy đủ các tính chất của các process thông thường trên hệ điều hành)

## 3. Làm việc với foreground/background process

Mình sẽ ví dụ về 1 chương trình: đếm từ 1 đến vô cùng, và in ra một file trên ổ đĩa

> `ps.sh`

```
#!/bin/bash

set -eo pipefail

i=0

while true
do
	echo $i >> ps.txt
	((i=i+1))
done
```

### 3.1 Quản lý foreground process

- Start một `foreground` process: (sử dụng một câu lệnh để khởi tạo process)

`./ps.sh`

- Start 1 terminal mới và kiểm tra status của process:

```
$ ps aux | grep ps.sh                                                                                                             
quanphuong        8142  99.2  0.0 408628864   2816 s000  R+    5:24PM   0:12.84 /bin/bash ./ps.sh
quanphuong        8144   0.0  0.0 408637552   1728 s005  S+    5:25PM   0:00.00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox ps.sh
```

ở đây bạn có thể nhìn thấy PID của process trên là `8142`. Từ đó bạn hoàn toàn có thể `kill` process này một cách bình thường như các process khác

```
$ kill -STOP 8142
```

quay trở lại terminal trước đó bạn sử dụng để start process kia, bạn sẽ thấy message thông báo process đã bị kill

```
[1]    8142 killed     ./ps.sh
```

### 3.2 Quản lý background process

- Start một `background process`:

```
$ ./ps.sh &
[1] 8289
```

Bạn sẽ thấy process đó hoạt động ngay lập tức, nhưng bạn vẫn có thể thao tác được ở ngay chính terminal mà bạn vừa gõ lệnh đó, lúc đó chúng ta có thể xem process vừa được start kia hoạt động như thế nào bằng cách gõ lệnh: `jobs`

```
$ jobs                                                                                                                            
[1]  + running    ./ps.sh
```

và tương tự như với `foreground`, có thể sử dụng lệnh `ps` và `kill` để xem và terminate `background process`

- Suspend process đang chạy:

```
$ kill -STOP 8289
[1]  + 8289 suspended (signal)  ./ps.sh
```

khi process bị suspend, sử dụng lệnh `jobs` bạn sẽ thấy các jobs bị suspend:

```
$ jobs
[1]  + suspended (signal)  ./ps.sh
```

khi đó bạn muốn start lại job này, có thể sử dụng lệnh `bg` như sau:

```
$ bg %1
bg: job already in background
```

> với `%1` là id của process khi bạn show ra trong lệnh `jobs`

- Chuyển `background process` về `foreground process`:

Đầu tiên bạn bắt buộc phải suspend `background process` đang hoạt động bằng cách sử dụng lệnh `kill -STOP` như phía trên mình đã đề cập

```
$ kill -STOP 8289
[1]  + 8289 suspended (signal)  ./ps.sh
```

Sau đó sử dụng lệnh `fg` để start lại process vừa được suspend dưới dạng `foreground`

```
$ fg %1
[1]  + 8289 continued  ./ps.sh
```

> với `%1` là id của process khi bạn show ra trong lệnh `jobs`

Hi vọng mọi người đã có góc nhìn bao quát hơn về 2 cách tạo ra process và quản lý ngay trên terminal của mình.!