---
title: "Một ví dụ về việc chuẩn Windows QCOW golden image kèm cloudbase-init sử dụng UI trên Cockpit"
author: "Aperture"
date: 2026-10-08T15:00:00+07:00
categories:
    - KVM
    - Windows
    - QCOW
    - Golden Image
    - Virtualization
    - Red Hat
    - OpenStack
tags:
    - kvm
    - qcow
    - windows
    - virtualization
    - golden-image
    - openstack
    - rhoso
    - rhosp
    - ocpv
cover: "/posts/windows-qcow-image/cover.png"
images:
    - "/posts/windows-qcow-image/cover.png"
---

# Mở đầu câu chuyện

Gần đây tôi vừa gặp một case gặp trục trặc về việc cài đặt [Windows Server](https://www.microsoft.com/en-us/windows-server) trên [RHOSO](https://www.redhat.com/en/technologies/cloud-computing/openstack-services-on-openshift) như sau:

  - Khi cài đặt, ngoài việc mount ISO của Windows Server, sẽ phải mount một ISO nữa chứa driver [VirtIO](https://github.com/virtio-win/kvm-guest-drivers-windows). Nhiều khách quen cài Windows Server trên VMWare sẽ thấy bỡ ngỡ, khi boot vào ISO sẽ hiện ra một cái cửa sổ đòi driver và không cho click Next đề đi tiếp. Kể cả mount VirtIO ISO vào rồi vẫn hiện màn hình đòi driver rất khó hiểu.
  - Khi mount VirtIO ISO đúng cách và Windows nhận driver được rồi thì mỗi lần cài đặt một máy Windows mới sẽ rất tốn thời gian. Hơn nữa để build một [golden image](https://www.redhat.com/en/topics/linux/what-is-a-golden-image) cũng cần phải cài đặt thêm một số phần mềm (trong đó có [Cloudbase-Init](https://cloudbase.it/cloudbase-init/) và [QEMU Agent](https://docs.openstack.org/nova/2026.2/admin/libvirt-misc.html#guest-agent-support))

Sau khi tham khảo tài liệu [tạo image Windows trong tài liệu của RHOSO 18](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/administer-assembly_glance_creating_rhosp_compatible_images#proc_create-windows-image), tôi thấy tài liệu này sử dụng command line và không trực tiếp nói về việc thiếu VirtIO driver sẽ khiến việc cài đặt Windows Server bị kẹt lại ở đoạn đòi driver. Vậy nên tôi viết bài viết này như kinh nghiệm bản thân chuẩn bị một golden image cho Windows Server ở định dạng [qcow2](https://www.qemu.org/docs/master/interop/qcow2.html), tương thích với nhiều hypervisor hiện nay.

# Chuẩn bị

Để tạo một image Qcow2, ta cần chuẩn bị như sau:

  - Một máy trạm đã bật tính năng ảo hoá trong BIOS (bật VT-x trên CPU Intel, AMD-V/SVM trên CPU AMD)
  - Máy trạm đó đã cài các công cụ sau:
    - RHEL hoặc Fedora (trong ví dụ này tôi sử dụng Fedora)
    - Cockpit (để có UI điều khiển máy ảo)
  - Các file ISO cần thiết cho việc cài Windows Server
    - ISO Windows Server
    - ISO VirtIO Driver (bạn tự chuẩn bị hoặc tải [từ đây](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/))
    - File cài Cloudbase-Init, có thể tải trên web của [Cloudbase Solution](https://cloudbase.it/cloudbase-init/)

# Bắt đầu cài đặt

Trước tiên ta vào màn hình screen

# Tham khảo
  - [Red Hat OpenStack Services on OpenShift](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0)
    - [Create custom images from ISO files | Create a Microsoft Windows image](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/administer-assembly_glance_creating_rhosp_compatible_images#proc_create-windows-image)
  - [Red Hat Enterprise Linux 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/)
    - [21.2.1. Installing KVM paravirtualized drivers for Windows virtual machines](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/configuring_and_managing_virtualization/index#installing-kvm-paravirtualized-drivers-for-rhel-virtual-machines_optimizing-windows-virtual-machines-on-rhel)