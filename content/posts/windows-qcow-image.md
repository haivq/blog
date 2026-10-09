---
title: "Ví dụ về việc tạo Golden Image Windows định dạng QCOW sử dụng UI trên Cockpit"
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
    - Experience
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
    - experience
cover: "/posts/windows-qcow-image/cover.png"
images:
    - "/posts/windows-qcow-image/cover.png"
---

# Mở đầu câu chuyện

Gần đây tôi vừa gặp một trường hợp khách gặp trục trặc về việc cài đặt [Windows Server](https://www.microsoft.com/en-us/windows-server) trên [RHOSO](https://www.redhat.com/en/technologies/cloud-computing/openstack-services-on-openshift).

## 1. Installer của Windows Server báo thiếu driver VirtIO khi cài đặt

Khi boot ISO Windows Server lên, thay vì giao diện chọn ngôn ngữ hiện lên, thì installer sẽ đòi driver. Nhiều người sẽ thấy bỡ ngỡ khi họ không gặp phải vấn đề này trên VMWare. Vấn đề này không chỉ xuất hiện trên OpenStack, mà sẽ gặp trên mọi platform sử dụng KVM và chọn VirtIO device thay vì giả lập SATA controller.

{{< figure 
    src="/posts/windows-qcow-image/driver-asking.png"
    position="center"
    alt="Installer kẹt lại ở đoạn đòi driver"
    caption="Installer kẹt lại ở đoạn đòi driver" >}}

Lý do là vì Microsoft thường đính kèm driver của VMWare trong bộ cài Windows nhưng không cung cấp driver [VirtIO](https://github.com/virtio-win/kvm-guest-drivers-windows), khiến cho việc cài đặt bị kẹt lại ở đoạn tìm driver. Cách xử lý cho vấn đề này là mount thêm ISO chứa driver của VirtIO vào máy ảo và để Windows tự detect các driver còn thiếu.

Tuy nhiên, kể cả khi đã mount ISO VirtIO vào máy ảo thì installer vẫn hiện màn hình đòi driver thay vì tự nhận. Việc này xảy ra khi người dùng mount ISO dưới dạng bus SCSI. Để khắc phục, người dùng bắt buộc phải mount lại ISO dưới định dạng SATA CD-ROM. Việc này đã được đề cập trong documentation của [OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/managing-vms#virt-installing-virtio-drivers-existing-windows_virt-install-virtio-drivers-on-windows-vms). Chỉ đến khi mount lại xong thì Windows mới tự quét và nhận driver như bình thường.

## 2. Cài đặt Windows Server quá lâu

Khi mount VirtIO ISO đúng cách và Windows đã nhận driver được rồi thì mỗi lần cài đặt một máy Windows mới sẽ rất tốn thời gian và phải có người can thiệp vào quá trình cài đặt để ấn Next và cài thêm các phần mềm khác (ví dụ như [QEMU Agent](https://docs.openstack.org/nova/2026.2/admin/libvirt-misc.html#guest-agent-support) hoặc phần mềm bảo mật), trong khi đó mục tiêu của chúng ta là tạo ra được một [Golden Image](https://www.redhat.com/en/topics/linux/what-is-a-golden-image), chỉ cần chọn và ấn start là một cái máy ảo Windows xuất hiện.

Sau khi tham khảo tài liệu [tạo image Windows trong tài liệu của RHOSO 18](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/administer-assembly_glance_creating_rhosp_compatible_images#proc_create-windows-image), tôi thấy tài liệu này có phần hướng dẫn sử dụng CLI để tạo image Windows rồi. Tuy nhiên tài liệu lại trực tiếp đề cập đến việc VirtIO driver sẽ khiến việc cài đặt Windows Server bị kẹt lại ở đoạn đòi driver (vốn nằm ngoài scope của Red Hat). Vậy nên tôi viết bài viết này như kinh nghiệm bản thân chuẩn bị một Golden Image cho Windows Server ở định dạng [qcow2](https://www.qemu.org/docs/master/interop/qcow2.html), tương thích với nhiều hypervisor hiện nay.

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

# Hướng dẫn cài đặt

Vui lòng xem video sau:

{{< youtube id=uIt3QT-P1_0 caption="Ví dụ tạo Golden Image Windows Server 2025 định dạng QCOW trên Fedora" >}}

# Tham khảo
  - [Red Hat OpenStack Services on OpenShift](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0) | [archive](https://web.archive.org/web/20261008190406/https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0)
    - [Create custom images from ISO files | Create a Microsoft Windows image](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/administer-assembly_glance_creating_rhosp_compatible_images#proc_create-windows-image) | [archive](https://web.archive.org/web/20261008190734/https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/administer-assembly_glance_creating_rhosp_compatible_images#proc_create-windows-image)
  - [Red Hat Enterprise Linux 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/)
    - [21.2.1. Installing KVM paravirtualized drivers for Windows virtual machines](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/configuring_and_managing_virtualization/index#installing-kvm-paravirtualized-drivers-for-rhel-virtual-machines_optimizing-windows-virtual-machines-on-rhel) | [archive](https://web.archive.org/web/20261008191737/https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_and_managing_virtualization/index)
  - [OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/index) | [archive](https://web.archive.org/web/20261008191117/https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/index)
    - [9.3.5. Installing VirtIO drivers from a SATA CD drive on an existing Windows VM](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/managing-vms#virt-installing-virtio-drivers-existing-windows_virt-install-virtio-drivers-on-windows-vms) | [archive](https://web.archive.org/web/20261008191324/https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/managing-vms#virt-installing-virtio-drivers-existing-windows_virt-install-virtio-drivers-on-windows-vms)