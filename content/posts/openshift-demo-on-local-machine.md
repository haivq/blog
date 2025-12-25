---
title: "Cài đặt OpenShift trên Homelab"
author: "Aperture"
date: "2025-09-24T12:58:00+07:00"
categories:
    - kubernetes
    - openshift
    - development
    - hyperv
    - virtualization
    - windows
tags:
    - k8s
    - homelab
    - openshift
    - kubernetes
    - docker
    - self hosted
    - pc
    - development
    - virtualization
    - windows
---

# Mở đầu câu chuyện

Một đề bài phỏng vấn vào cửa gần đây của tôi với Red Hat là làm một bài thuyết trình bao gồm:

- Cách thức triển khai OpenShift lên homelab của mình.
- Ví dụ một phương thức triển khai một ứng dụng trên OpenShift (từ lúc nó còn là code đến khi nó biến thành một cái Web App chạy được).
- Cách thức scale và monitor ứng dụng.

Tôi có 1 tuần để chuẩn bị, tìm hiểu. Với background Kubernetes rồi thì có lẽ tôi cần phải tìm hiểu thêm về OpenShift. Và bài blog này sẽ kể về quá trình tôi cài đặt nó lên trên môi trường homelab của mình.

# OpenShift là gì?

Nói vui vui rằng, OpenShift thực chất chỉ là k8s thêm nhiều gia vị, vì như tôi tìm hiểu ban đầu thì lõi bên trong của OpenShift là k8s. Nhưng sau khi tìm hiểu sâu hơn, thì tôi nhận ra một vài điều sau:

1. OpenShift bắt buộc phải cài trên RHEL, nhân Linux mà yêu cầu đầu tiên và hàng đầu phải là security. Chính vì phải cài trên RHEL nên OpenShift kế thừa toàn bộ những tính năng security mà RHEL đã có. Chính vì thế mà RHEL sẽ luôn chậm hơn Linux vài phiên bản.

2. OpenShift không chỉ có mỗi k8s, mà nó là cả một giải pháp để triển khai, giám sát, theo dõi, cảnh báo, thậm chí có sẵn cả frontend luôn. K8s chay không bao hàm việc CI/CD; monitoring và alerting cũng phải cài thêm mới có. Cũng chính vì thế mà yêu cầu phần cứng của OpenShift cũng tương đối cao.

Nếu như ngày trước tôi muốn sử dụng k8s mà muốn kèm theo các yếu tố trên, thì tôi thường dùng giải pháp của Rancher, tạo một cluster k8s chỉ để chứa Rancher, sau đó kết nối Rancher vào cái cluster k8s đã có bằng cách cài cattle agent lên đó, sau đó từ giao diện của Rancher đó tôi add các helm chart vào rồi cài một loạt các package lên. Nhưng OpenShift thì bundle hết tất cả vào một gói luôn.

# Cài OpenShift ra sao?

Không có document nào cụ thể hướng dẫn tôi cài OpenShift local một cách đơn giản cả, phần lớn đều bắt tôi cài 1 cái OpenShift to nạc. May quá có một [blog post của Red Hat](https://www.redhat.com/en/blog/install-openshift-local) tại đây hướng dẫn cài OpenShift local, nhưng hướng dẫn lại ở trên Linux. Thực tế sau khi xem hướng dẫn và tải xuống file cài OpenShift local, thì tôi thấy thực ra mình có thể chạy trên Windows, chỉ là thay vì dùng KVM thì tôi dùng Hyper-V. Dưới đây là quá trình cài khá đơn giản của OpenShift local.

## Cài đặt ban đầu

> Lưu ý: Máy ảo sẽ luôn cài ở ổ C, vậy nên hãy đảm bảo ổ C của bạn trống khoảng 60-70G để có thể vận hành máy ảo 1 cách trơn tru.

Để dùng được Hyper-V thì bạn phải có Windows 10/11 Pro. Nếu bạn dùng Windows 10/11 Home thì sẽ không thể bật Hyper-V lên được. Vậy nên trước tiên bạn phải upgrade lên Windows 10/11 Pro.

Để bật được Hyper-V thì bạn phải Virtualization trong BIOS. Thường thường việc bật Virtualization sẽ khiến OS chậm đi khoảng 2 3%, nên thường các máy bình thường người ta sẽ tắt nó đi. Để bật Virtualization thì bạn phải vào BIOS, phần CPU Feature hoặc CPU Setting và bật AMD-V (thêm AMD-Vi hay IOMMU) nếu dùng AMD hoặc VT-x (bật thêm VT-d nữa thì tốt) nếu dùng Intel.

Tải OpenShift xuống từ Red Hat và cài nó vào máy. Đầu tiên ta [đăng kí một tài khoản Red Hat Developers](https://developers.redhat.com/register) trước, rồi sau đó vào [Red Hat Console](https://console.redhat.com/openshift/create/local) để tải Red hat OpenShift local về máy.

{{< figure 
    src="/posts/openshift-demo-on-local-machine/download-openshift.png"
    position="center"
    alt="Tải OpenShift xuống từ console"
    caption="Tải OpenShift xuống từ console">}}

Tải OpenShift local về ta cài đặt vào máy, cài xong máy có thể bắt restart. Restart xong, mở Powershell (hay Terminal) và chạy `crc setup` để tải image của OpenShift xuống máy. Script cũng sẽ bật Hyper-V nếu chưa có.

{{< figure 
    src="/posts/openshift-demo-on-local-machine/setup-openshift.png"
    position="center"
    alt="Setup OpenShift"
    caption="Setup OpenShift">}}

Sau khi setup xong ta cũng tắt máy đi bật lại cho chắc. Sau bước này thì ta chuyển sang bước 2: thực sự bật OpenShift lên và thao tác.

## Config OpenShift sau bước chuẩn bị ban đầu:

Trước tiên ta phải config lại OpenShift trước khi bật. Có những mục sau cần config lại:
- Chỉnh số nhân CPU, tốt nhất là 4 vCore trở lên cho đỡ lag vì OpenShift có rất nhiều component bên trong: `crc config set cpus {số vCore}`
- Chỉnh dung lượng RAM, ít nhất là 16G ram trở lên (lý do tương tự như nhân CPU ở trên): `crc config set memory {dung lượng RAM theo MiB, nhớ là bội số của 2}`
- Bật cluster monitoring, nếu không sẽ không có monitoring cho cluster local này: `crc config set enable-cluster-monitoring true`
- Chỉnh dung lượng ổ cứng máy ảo, vì mặc định image của OpenShift chỉ có khoảng 30-32Gi thôi, phải tăng lên thì mới không bị dính lỗi `disk pressure`, tốt nhất nên để khoảng 40-50Gi: `crc config set disk-size {dung lượng ổ cứng theo Gi}`

Sau khi setup xong ta kiểm tra lại bằng lệnh `crc config view`:

{{< figure 
    src="/posts/openshift-demo-on-local-machine/config-openshift.png"
    position="center"
    alt="Config của OpenShift"
    caption="Config của OpenShift">}}

Tạm thời config như thế là được rồi, ta sẽ bật OpenShift lên và dùng.

## Bật OpenShift local:

Để bật OpenShift, ta chạy lệnh sau:

```bash
crc start
```

Lúc này máy ảo Hyper-V sẽ bật lên và bắt đầu chạy OpenShift. Quá trình bật lần đầu tiên sẽ khá mất thời gian, vì lúc ấy OpenShift local bắt đầu bung image máy ảo, start hệ thống và bắt đầu setup cluster lần đầu tiên. Sau khoảng 10p chờ đợi thì ta sẽ có kết quả màn hình như sau:

{{< figure 
    src="/posts/openshift-demo-on-local-machine/start-openshift.png"
    position="center"
    alt="Kết quả start OpenShift"
    caption="Kết quả start OpenShift">}}

Lúc này ta có thể vào URL trên màn hình và bắt đầu dùng thử OpenShift. Login screen của OpenShift sẽ trông như sau:

{{< figure 
    src="/posts/openshift-demo-on-local-machine/openshift-login-screen.png"
    position="center"
    alt="Màn hình login của OpenShift"
    caption="Màn hình login của OpenShift">}}

Khi vào được OpenShift màn hình Home Dashboard sẽ như sau:

{{< figure 
    src="/posts/openshift-demo-on-local-machine/openshift-home-dashboard.png"
    position="center"
    alt="Home dashboard của OpenShift"
    caption="Home dashboard của OpenShift">}}

Đến đây là bạn đã cài xong OpenShift trên máy local

# Kết luận

Mong bài viết này có thể giúp ích cho bạn để setup 1 môi trường OpenShift local trên homelab của bạn. Mong rằng tôi có thể pass phỏng vấn và trở thành một Red Hatter chính hiệu!

> Cập nhật: Tôi đã nhận được offer làm việc tại Red Hat!

# Tài liệu tham khảo

- [How to install Red Hat OpenShift Local on your laptop](https://www.redhat.com/en/blog/install-openshift-local) / [Archive](https://web.archive.org/web/20250924055521/https://www.redhat.com/en/blog/install-openshift-local):
