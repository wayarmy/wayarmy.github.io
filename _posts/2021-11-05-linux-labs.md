---
layout: post
title:  "Những bài labs cơ bản trên linux"
subtitle: Khởi đầu mới với Linux
gh-repo: wayarmy/wayarmy.github.io
gh-badge: [star, fork, follow]
tags: [sysadmin]
comments: true
date:   2021-11-05
categories: Sysadmin
---

**Những bài labs liên quan tới Linux**

> Danh sách các bài labs mà mình nghĩ cần thiết cho 1 bạn newbie về Linux
> Nội dung sẽ đi từ Fundamentals => Basic => Advanced
> Vì hướng dẫn có rất nhiều trên google, nên sẽ không có hướng dẫn làm ở đây
## 1. Thao tác với user/group trên linux

Khởi tạo một user trên Ubuntu:

- Khởi tạo user không có quyền login, không có thư mục `home directory` của user đó
- Khởi tạo user có thể login vào máy tính, có thư mục `home directory` của riêng user đó với shell mặc định là `/bin/bash`
- Xoá một user khác `root` khỏi hệ điều hành
- Xem danh sách các users đang có trên Ubuntu

Thao tác group trên Ubuntu:

- Khởi tạo 1 group
- Gán một user vào group
- Xem danh sách các groups đang có trên một hệ điều hành Ubuntu


## 2. Thao tác với files, thư mục

- Tạo mới file trống
- Tạo mới folder trống
- Đọc nội dung của 1 file 
- Thêm, sửa, xoá nội dung của 1 file
- Move 1 file từ folder này sang folder khác
- Copy 1 file từ folder này sang folder khác
- Xoá một folder chứa rất nhiều files
- `chmod` file chỉ cho phép user tạo ra file đó có thể `đọc+ghi+execute` được file
- `chmod` folder chỉ cho phép user tạo ra file đó có thể `đọc+ghi+execute` được file bên trong folder đó
- `chmod` cho một group bất kỳ có quyền `đọc+ghi` cho 1 file, nhưng không được quyền `execute` file đó
- `chmod` cho phép 1 user khác có quyền `đọc+ghi` cho 1 file.

~~~
$ chmod
$ chgroup
$ chown
~~~

## 3. Xem thông số phần cứng của một máy tính cài đặt linux

- Xem thông số của CPU

```
cat /proc/cpuinfo
```

- Xem máy tính của mình được cấp phát bao nhiêu ram, lượng thực tế đang được sử dụng thế nào

- Xem máy tính mình có bao nhiêu ổ cứng, dung lượng thực tế sử dụng đang là bao nhiêu.

## 4. Kiểm tra và quản lý các service/process trên linux

- Xem có bao nhiêu services đang hoạt động
- Xem có bao nhiêu process đang hoạt động
- Start/Stop 1 service trên OS
- Stop 1 process đang hoạt động trên OS

## 5. Kiểm tra các ports mạng trên linux

- Xem có bao nhiêu ports đang được mở trên một server linux

~~~
netstat -ano
~~~

- Xem một port bất kỳ đang được mở bằng process nào

~~~
netstat -nlp
~~~

## 6. Cài đặt các phần mềm cơ bản trên linux server

### 6.1 Cài đặt NFS server

Yêu cầu:

- Tạo 2 server A và B (Recommend sử dụng Ubuntu server)
- Cài đặt NFS lên cả 2 server A và B
- Tạo thư mục `/mount_a` trên server A và `/mount_b` trên server B
- Mount thư mục `/mount_a` trên server A vào `/mount_b` trên server B sử dụng NFS
- Tạo 1 file rỗng trên thư mục `/mount_b` trên server B và test thử xem có nhìn thấy file đó trên `/mount_a` trên server A hay không ?
- Thêm một nội dung bất kỳ vào file đó trên `/mount_a` của server A và đọc ra nội dung của file đó trên `/mount_b` của server B - Nếu nội dung trùng khớp nghĩa là bạn đã cài đặt thành công.


### 6.2 Cài đặt webserver lên trên server linux

Thường thì các bài labs về webserver sẽ sử dụng `apache httpd` để cài đặt lên server linux

Yêu cầu: 

- Cài đặt `apache httpd` sử dụng `apt-get` của Ubuntu
- Kiểm tra cài đặt thành công bằng cách check xem đã có services httpd start hay chưa

~~~
systemctl status httpd
~~~

- Tạo một folder chứa code của một trang html đơn giản khác với  `/var/www/html` (đây là thư mục mặc định chứa static html của `apache httpd`) - Ví dụ `/home/www/html`

- Tạo 1 file tên `index.html` bên trong thư mục vừa tạo với nội dung

~~~
<html>
    <title>Aapache HTTPD test page</title>

    <h1>This is H1 tag</h1>
</html>
~~~

- Thực hiện cấu hình `apache httpd` sao cho khi truy cập vào địa chỉ `<server_IP>:8080` thì sẽ hiển thị ra với HTML tương ứng.

> (to be continued)
