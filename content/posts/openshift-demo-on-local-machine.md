---
title: "Cài đặt OpenShift trên Windows"
author: "Aperture"
date: "2025-09-20T11:55:00+07:00"
categories:
    - kubernetes
    - openshift
    - development
    - hyperv
    - virtualization
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
draft: true
---

# Mở đầu câu chuyện

Một đề bài phỏng vấn vào cửa gần đây của tôi với Red Hat là làm một bài thuyết trình bao gồm:

- Cách thức triển khai OpenShift lên homelab của mình.
- Ví dụ một phương thức triển khai một ứng dụng trên OpenShift (từ lúc nó còn là code đến khi nó biến thành một cái Web App chạy được).
- Cách thức scale và monitor ứng dụng.

Tôi có 1 tuần để chuẩn bị, tìm hiểu. Với background Kubernetes rồi thì có lẽ tôi cần phải tìm hiểu thêm về OpenShift. Và bài blog này sẽ kể về quá trình tôi tìm hiểu, so sánh và nghiên cứu về nó, dù có được tuyển hay không thì tôi cũng có thêm nhiều kiến thức cho bản thân mình.

# OpenShift là gì?

Nói vui vui rằng, OpenShift thực chất chỉ là k8s thêm nhiều gia vị, vì như tôi tìm hiểu ban đầu thì lõi bên trong của OpenShift là k8s. Nhưng sau khi tìm hiểu sâu hơn, thì tôi nhận ra một vài điều sau:

1. OpenShift bắt buộc phải cài trên RHEL, nhân Linux mà yêu cầu đầu tiên và hàng đầu phải là security. Chính vì phải cài trên RHEL nên OpenShift kế thừa toàn bộ những tính năng security mà RHEL đã có. Chính vì thế mà RHEL sẽ luôn chậm hơn Linux vài phiên bản.

2. OpenShift không chỉ có mỗi k8s, mà nó là cả một giải pháp để triển khai, giám sát, theo dõi, cảnh báo, thậm chí có sẵn cả frontend luôn. K8s chay không bao hàm việc CI/CD, monitoring và alerting cũng phải cài thêm. Cũng chính vì thế mà yêu cầu phần cứng của OpenShift cũng tương đối cao.

Nếu như ngày trước tôi muốn sử dụng k8s mà muốn kèm theo các yếu tố trên, thì tôi phải cài thêm Rancher, cắm Rancher vào cái cluster k8s đã có, sau đó từ cái Rancher đó tôi add các helm chart vào rồi cài một loạt monitoring lên. Nhưng OpenShift thì bundle hết tất cả vào một gói luôn.

# Cài OpenShift ra sao?

Không có document nào cụ thể hướng dẫn tôi cài OpenShift local một cách đơn giản cả, phần lớn đều bắt tôi cài 1 cái OpenShift to nạc. May quá có một [blog post của Red Hat](https://www.redhat.com/en/blog/install-openshift-local) tại đây hướng dẫn cài OpenShift local, nhưng hướng dẫn lại ở trên Linux. Thực tế sau khi xem hướng dẫn và tải xuống file cài OpenShift local, thì tôi thấy thực ra mình có thể chạy trên Windows, chỉ là thay vì dùng KVM thì tôi dùng Hyper-V. Dưới đây là quá trình cài khá đơn giản của OpenShift local.

## Cài đặt ban đầu

1. Để dùng được Hyper-V thì bạn phải có Windows 10/11 Pro. Nếu bạn dùng Windows 10/11 Home thì sẽ không thể bật Hyper-V lên được. Vậy nên trước tiên bạn phải upgrade lên Windows 10/11 Pro.

2. Để bật được Hyper-V thì bạn phải Virtualization trong BIOS. Thường thường việc bật Virtualization sẽ khiến OS chậm đi khoảng 2 3%, nên thường các máy bình thường người ta sẽ tắt nó đi. Để bật Virtualization thì bạn phải vào BIOS, phần CPU Feature hoặc CPU Setting và bật AMD-V (nếu dùng AMD) hoặc VT-x nếu dùng Intel (bật thêm VT-d nữa thì tốt)
