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
- Cách thức setup OpenShift lên homelab của mình
- Ví dụ một phương thức triển khai một ứng dụng trên OpenShift
- Cách thức scale ứng dụng 


# Tư liệu tham khảo

- [Cloudflare Docs](https://developers.cloudflare.com)
    - [R2 / Pricing](https://developers.cloudflare.com/r2/pricing/#r2-pricing) / [archive](https://web.archive.org/web/20240827210539/https://developers.cloudflare.com/r2/pricing/#r2-pricing)
    - [Cloudflare Image Optimization / Pricing](https://developers.cloudflare.com/images/pricing/#images-transformed) / [archive](https://web.archive.org/web/20240810001916/https://developers.cloudflare.com/images/pricing/#images-transformed)
    - [Workers / Runtime APIs / Cache](https://developers.cloudflare.com/workers/runtime-apis/cache/) / [archive](https://web.archive.org/web/20240822091425/https://developers.cloudflare.com/workers/runtime-apis/cache/)
- [GitHub](https://github.com)
    - [fineshopdesign/cf-wasm](https://github.com/fineshopdesign/cf-wasm) / [archive](https://web.archive.org/web/20240831145937/https://github.com/fineshopdesign/cf-wasm)
