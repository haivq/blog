---
title: "Cài đặt Self-Signed SSL để đảm bảo bảo mật connect tới Cloudflare"
author: "Aperture"
date: "2021-11-23T13:57:17+07:00"
categories:
    - cloudflare
    - cdn
    - ssl/tls
tags:
    - cloudflare
    - cloudflare cdn
    - ssl
    - self-signed
    - encryption
    - openssl
    - tls
---

> Lưu ý: Phương pháp này không hoàn toàn ngăn chặn Man-in-the-Middle attack nếu hacker can thiệp được vào tầng network, do Cloudflare không verify độ uy tín của certificate nên hacker có thể sniffing request bằng cách cung cấp certificate của riêng hắn rồi proxy ngược vào server. Để đảm bảo At Rest Encryption hoạt động một cách chính xác, tham khảo document này: [Full (strict) - SSL/TLS encryption modes](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/). Như vậy Cloudflare chỉ thực hiện request đến server nếu như server đó sử dụng certificate của mà Cloudflare cho là an toàn, bao gồm các certificate có CA yêu cầu việc issue khó khăn hoặc cerficate được sign bởi Cloudflare.

# Giải thích

Để đảm bảo bảo mật ổn định nhất từ người dùng đến server, cần thực hiện các bước sau:

  * Mã hóa dữ liệu từ người dùng đến Cloudflare (đã được Cloudflare thực hiện)
  * Mã hóa dữ liệu từ Cloudflare gọi đến server origin

Để đảm bảo mã hóa dữ liệu, ta cần phải setup certificate từ ở server Origin. Việc này xử lý 2 vấn đề như sau:
  - Không có lỗi SSL không hợp lệ (525: SSL Handshake failed), lỗi này xảy ra khi Cloudflare request https đến server không có SSL
  - Đảm bảo At Rest Encryption, tức giả sử nếu có người sniffing request giữa Cloudflare và origin server (thường là IP của load balancer) thì cũng không biết data gửi là gì.

Minh hoạ sẽ như sau:

{{< figure 
    src="/self-signed-cert-cloudflare/full-enc.png"
    position="center"
    alt="Full encryption"
    title="Full encryption"
    caption="Full encryption"
    attr="Full encryption"
    attrlink="https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full/"
    link="https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full/">}}

Để setup, ta cần phải setup như sau:
  * Setup một certificate hợp lệ ở cloudflare (bước này cloudflare đã tự xử lý)
  * Ở server Origin, cài đặt self-signed certificate chỉ để đảm bảo tính bảo mật

# Thực hiện

Yêu cầu:
  * Một máy tính chạy Linux, UNIX hoặc macOS đã cài đặt OpenSSL
  * SSH 

## 1. Tạo self-signed certificate

Dùng command sau để tạo Certificate, Private key:

```bash
openssl req -x509 -nodes -newkey rsa:4096 -keyout key.pem -out cert.pem -days 3650 -subj "/C=VN/ST=Hanoi/L=Thanh Xuan/O=House3D LLC/OU=IT/CN=house3d.com"
```

Ở trên ta đã tạo key RSA 4096 với tên key.pem, xuất ra certificate cert.pem với hạn sử dụng 10 năm.

Giải thích phần Subject: Bình thường khi tạo certificate, xuất hiện các bước để điền tên nước, tên tỉnh/thành, tên công ty, vân vân. Ở đây ta truyền các thông tin đó vào luôn trong mục `-subj`. trong này, các mục subject được mô tả như sau:
  * `/C` - Country: Mã nước sở (2 kí tự) tại của doanh nghiệp (ví dụ: Vietnam - VN, United States - US)
  * `/ST` - State: Tỉnh/thành/bang của doanh nghiệp (ví dụ: Hanoi, Texas)
  * `/L` - Locality Name: Tên thành phố/địa phương (ví dụ: Hoan Kiem, Brooklyn)
  * `/O` - Organization: Tên doanh nghiệp (ví dụ: Google, House3D)
  * `/OU` - Organizational Unit: Bộ phận tác nghiệp (ví dụ: IT, Development Department)
  * `/CN` - Common Nane: Trang web doanh nghiệp (ví dụ: house3d.com)

Dựa vào thông tin trên để thay đổi subject khi cần. Chúng ta có thể bỏ qua phần subject này cũng được, lúc tạo certificate nếu bị hỏi chỉ cần enter để bỏ qua là được

## 2. Setup certificate lên server

### Rancher/Kubernetes

Ta cần add certificate vào Ingress, thông qua mục certificate trong Secret

[Encrypting HTTP Communication](https://rancher.com/docs/rancher/v2.5/en/k8s-in-rancher/certificates/)

### Apache

Setup thông qua file Virtual Host

[SSL/TLS Strong Encryption: How-To](https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html)

[How to install an SSL Certificate on Apache](https://www.ssls.com/knowledgebase/how-to-install-an-ssl-certificate-on-apache/)

### NGINX

Setup thông qua `server` config

[NGINX SSL Termination](https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/)

## 3. Setup End-to-End Encryption trên Cloudflare

Setup Full encryption:

[End-to-end HTTPS with Cloudflare - Part 3: SSL options](https://support.cloudflare.com/hc/en-us/articles/200170416-End-to-end-HTTPS-with-Cloudflare-Part-3-SSL-options#h_845b3d60-9a03-4db0-8de6-20edc5b11057)