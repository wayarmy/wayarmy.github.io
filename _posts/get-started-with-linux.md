---
layout: post
title:  "Bắt đầu với hệ điều hành Linux"
subtitle: Khởi đầu mới với Linux
gh-repo: wayarmy/wayarmy.github.io
gh-badge: [star, fork, follow]
tags: [sysadmin]
comments: true
date:   2021-10-10
categories: Sysadmin
---

**Bắt đầu với Linux như thế nào là ổn**

> Bài viết tập hợp những kinh nghiệm của bản thân mình sau 1 thời gian rất dài làm việc với các hệ điều hành Linux

> Bài viết này dành cho những anh em chưa từng biết gì về Linux, và không có nhiều kiến thức về Computer Scienes.

## 1. Linux là gì ?

Có thể hiểu một cách đơn giản Linux là 1 tên gọi chỉ chung các hệ điều hành có cùng một cấu trúc hệ điều hành được phát triển từ một nhân Linux (Kernel - Sau này nếu có thời gian sẽ viết lại 1 bài riêng về Kernel của Linux. Ở đây cứ tạm hiểu đó là core của một hệ điều hành).

Ví dụ một số hệ điều hành: Ubuntu, Centos, Redhat, Debian, Gentoo, Solaris,...

(Vì bài viết dành )

## 2. Bắt đầu với Linux từ đâu ?

> Theo kinh nghiệm của bản thân mình từ ngày trước, bạn nên bắt đầu với việc cài đặt một hệ điều hành Linux, có thể chọn Ubuntu, hoặc Centos tuỳ ý. Ngày trước mình bắt đầu với CentOS, nhưng theo thời gian mình khuyên các bạn nên bắt đầu bằng Ubuntu, vì Ubuntu có cảm giác friendly hơn so với CentOS, đồng thời cộng đồng người sử dụng Ubuntu nhiều hơn rất nhiều so với CentOS.

### 2.1 Cài đặt Ubuntu

Mình sẽ không hướng dẫn quá sâu về việc cài đặt hệ điều hành Ubuntu, vì các bạn có thể search google ra rất nhiều hướng dẫn. Ở đây mình chỉ recommend với những bạn đang sử dụng windows rồi bắt đầu chuyển sang thử bắt đầu với Linux có thể tham khảo cách cài đặt Ubuntu như một hệ điều hành thử nghiệm trên VMWare Workstation để nếu có lỗi gì đó trong cài đặt, các bạn có thể cài đặt lại một cách dễ dàng trên VMWare Workstation. 

Tiện tay search google ra một [link hướng dẫn](https://openplanning.net/11327/cai-dat-ubuntu-desktop-trong-vmware) tiếng Việt mà mình nghĩ tác giả chụp màn hình và hướng dẫn khá chi tiết.

### 2.2 Bắt đầu với Ubuntu

#### 2.2.1 Làm quen với giao diện Terminal

Trong quá trình làm việc thực tế với Ubuntu hoặc các hệ điều hành Linux khác, kể cả khi mình làm việc với MacOSX, thời gian sử dụng terminal để làm việc cũng chiếm khoảng 50% thời gian mình ngồi làm việc trên máy tính. Nên việc đầu tiên khi bắt đầu với Linux theo mình là tập làm quen với giao diện terminal của các hệ điều hành Linux.

Trên Ubuntu thì Terminal sẽ có giao diện kiểu thế này:
![Ubuntu Terminal](/assets/images/1-25.png)

Sau khi cài đặt xong Ubuntu, bạn có thể bật thử terminal có sẵn trên thanh task bar của Ubuntu hoặc tìm kiếm `terminal` trong thanh công cụ `search` của Ubuntu.

#### 2.2.2 Thử làm việc với một số lệnh Linux trên terminal

Khi sử dụng một hệ điều hành, việc mà chúng ta làm thường xuyên nhất thường là:

- Truy vấn files, folders trong 1 folders.
- Đọc, thêm/sửa, xoá files
- Sửa xoá nội dung trong files

Để làm việc đó trên `Terminal`, có khá nhiều cách, nhưng mình sẽ bắt đầu với một số câu lệnh đơn giản sau:

~~~
ls # để list ra danh sách các files/folders trong 1 folders
ls -alh # để list danh sách các files/folders trong 1 folders với đầy đủ thông tin của các files/folders đó
cat file.txt # để đọc toàn bộ nội dung của 1 file với tên là file.txt
mkdir /folder # để tạo thêm 1 folder trống mới 
touch file.txt # để tạo ra một file trống với tên là file.txt ở thư mục hiện tại.
~~~
