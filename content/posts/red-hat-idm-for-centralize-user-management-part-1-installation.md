---
title: "Red Hat Identity Management trong môi trường Homelab - Phần 1: Cài đặt"
author: "Aperture"
date: 2026-11-20T15:00:00+07:00
categories:
  - LDAP
  - IdM
  - Red Hat
  - RHEL
  - FreeIPA
  - Installation
  - Experience
  - Infrastructure
  - Day 1
  - Homelab
tags:
  - ldap
  - idm
  - ipa
  - free-ipa
  - 389-ds
  - redhat
  - rhel
  - installation
  - infra
  - infrastructure
  - day-1
  - homelab
cover: "/posts/red-hat-idm-for-centralize-user-management-part-1-installation/cover.png"
images:
  - "/posts/red-hat-idm-for-centralize-user-management-part-1-installation/cover.png"
---

# Mở đầu câu chuyện

Câu chuyện bắt đầu từ việc một khách hàng yêu cầu quản lý user tập trung cho toàn bộ RHEL fleet, ngoài ra còn phải quản lý về quyền hạn, về password policy, sâu xa hơn là tích hợp với các hệ thống khác như quản lý login OpenShift, vân vân. Tất cả những tiêu chí trên vô cùng phù hợp với sản phẩm [Red Hat Identity Management](https://access.redhat.com/products/red-hat-identity-management/) (giờ gọi là IdM cho ngắn). Mô tả thì dài dòng, nhưng tôi có thể tóm gọn một vài tính năng của nó ở đây:

- Quản lý user tập trung: Admin tạo user 1 lần, người dùng có thể sử dụng user đó để đăng các máy RHEL đã đăng kí tới IdM server.
- Quản lý quyền hạn truy cập của user và quyền `sudo` tới từng RHEL host: Admin phân quyền người dùng hoặc một nhóm người dùng truy cập tới từng máy hoặc một nhóm máy, cho phép họ tương tác giới hạn với `sudo`, ví dụ như chỉ cho `sudo systemctl restart posgresql` chẳng hạn.
- Cho phép các ứng dụng khác sử dụng IdM làm user database qua [LDAP](https://docs.redhat.com/en/documentation/red_hat_directory_server/13/html/planning_and_designing_directory_server/introduction-to-directory-services#about-ldap): LDAP khá thông dụng rồi, nên các ứng dụng khác có thể chọc vào IdM để lấy ra list user và auth trên list user đó luôn.
- Cung cấp tính năng cấp certificate và quản lý vòng đời của certificate thông qua Dogtag PKI. Ngoài ra Dogtag còn hỗ trợ [ACME](https://www.rfc-editor.org/info/rfc8555/), tức nếu bạn có [`certbot`](https://certbot.eff.org/) hoặc [`cert-manager`](https://cert-manager.io/) thì có thể tự động issue ra certificate từ IdM mà không cần phải thủ công ngồi renew và cấp lại cert hàng năm nữa.
- (và nhiều tính năng khác mà chúng ta sẽ không đề cập đến... vui lòng đọc thêm tại [link này](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/plan-overview_of_idm_and_access_control_in_rhel#introduction-to-idm))

Ban đầu tôi chỉ nghĩ đơn giản rằng, chắc phải có hệ thống lớn lắm thì mới cần một server IdM/LDAP để quản lý user tập trung. Nhưng mọi thứ thay đổi khi tôi nghiên cứu vọc vạch nhiều hơn ở homelab của mình. Hiện tại thì ngoài tôi ra, còn một vài người bạn cũng đang sử dụng homelab. Phần lớn dùng để lưu trữ file chung cho cả nhóm, host server phim, nghe nhạc. Ngoài ra, tôi còn phải nhớ account admin của tất cả các dịch vụ khác nhau, khá là bất tiện khi mỗi lần đăng nhập lại phải mở password app ra để lấy mật khẩu. Từ nhu cầu dùng của cá nhân và bạn bè, tổng hợp lại họ cần:

- Một user chung cho tất cả các máy và ứng dụng
- Đổi pass 1 lần ở một nơi, tất cả sẽ thay pass theo
- Cần có một nền tảng phổ biến chung để cấp user cho tất cả người dùng

Tự dưng tôi lại nghĩ rằng, dựng 1 con IdM nội bộ cũng khá là tiện đấy chứ, dù quả thật như vậy khá overkill trong môi trường homelab có vài người dùng.

# Cài đặt IdM

Theo như [documentation của Red Hat](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-install_an_idm_server_using_the_cli), thì với mỗi môi trường khác nhau chúng ta sẽ có một cách cài đặt khác nhau. Sau khi khảo sát hạ tầng của mình, mong muốn của tôi sẽ là:

  - Tận dụng DNS của Mikrotik làm DNS tập trung, tắt tính năng DNS của IdM
  - Do homelab chưa có PKI, nên tôi sẽ lấy luôn IdM làm PKI tập trung

Tôi cũng mô tả sơ qua homelab của tôi:

  - Một server RHEL 10 (subscription của RHEL [sẽ bao gồm luôn sản phẩm IdM](https://access.redhat.com/kb/7057494). Nếu bạn đã mua RHEL thì có thể dùng IdM mà không phải mua thêm.)
  - DNS tập trung có sẵn trong router của Mikrotik
  - Một server RHEL khác để enroll vào IdM
  - Một vài server hoặc ứng dụng khác hỗ trợ LDAP để đăng kí vào IdM (như [Jellyfin](https://github.com/jellyfin/jellyfin-plugin-ldapauth), [NAS Synology](https://kb.synology.com/en-global/DSM/help/DirectoryServer/ldap_server?version=7))
  - Một cụm OpenShift/Kubernetes đã cài sẵn `cert-manager` ([operator nếu bạn dùng OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift) hoặc các [phương pháp khác liệt kê trong documentation của `cert-manager`](https://cert-manager.io/docs/installation/)) để kết nối tới ACME server của IdM.

Vậy nên tôi sẽ lựa chọn cách cài đặt [IdM không kèm DNS nhưng dùng integrated CA làm Root CA](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-installing_an_idm_server_without_integrated_dns_with_an_integrated_ca_as_the_root_ca) để tận dụng tối đa những gì đã có.

> Lưu ý: Nếu bạn cài đặt [`FreeIPA`](https://www.freeipa.org/page/Documentation.html) thì cũng có thể theo documentation IdM của RHEL để cài đặt. Hướng dẫn của tôi dưới đây phỏng theo hướng dẫn của IdM nên theo lý thuyết cũng áp dụng được với FreeIPA.

## Cấu hình trước khi cài

<!-- ### Về phía server -->

Dựa vào [tài liệu của hãng](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-preparing_the_system_for_idm_server_installation) và thông tin đang có của homelab, tôi có thể liệt kê các yêu cầu hạ tầng như sau:

  - Chạy RHEL 7, 8, 9 và 10 (nếu bạn sử dụng FreeIPA và cài trên Fedora hay CentOS thì thực tế yêu cầu về OS sẽ ít chặt hơn). Bài viết này được thực hiện trên máy RHEL 10.
  - Máy RHEL đã cập nhật repo RPM `BaseOS` và `AppStream` mới nhất, nguồn repo có thể từ chính Red Hat hoặc lấy về từ Satellite
  - Trống khoảng 4G RAM và 4G swap, tuy nhiên, trong môi trường homelab ta có thể bỏ qua yêu cầu này
  - DNS server tập trung hỗ trợ các record: `A`, `TXT`, `SRV`. Nếu hỗ trợ thêm `URI` nữa thì càng tốt.
  - Máy còn cần phải còn trống các cổng (và mở cổng trên `firewalld`):
    - 80, 443 - TCP: Để cung cấp UI và API cho IdM
    - 389, 636 - TCP: Cấp cổng cho dịch vụ LDAP hoạt động
    - 88, 464 - TCP/UDP: Cấp cổng cho dịch vụ [Kerberos](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/reference_guide/ch-kerberos)
  - Kết nối tới DNS server tập trung đã nêu

<!-- ### Về phía client

Yêu cầu về phía client sẽ không nhiều như phía server (dựa theo [tài liệu của hãng](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/installing_identity_management/index?utm_source=chatgpt.com#preparing-the-system-for-ipa-client-installation_installing-identity-management)), tóm lại sẽ chỉ có yêu cầu sau:

  - Bất kì bản Linux nào cài đặt được `ipa-client`, tức không chỉ RHEL hay Fedora mà có cả gồm [Ubuntu](https://manpages.ubuntu.com/manpages/jammy/man1/ipa-client-install.1.html) hay [Debian](https://manpages.debian.org/unstable/freeipa-client/ipa-client-install.1.en.html).
  - Kết nối tới DNS server tập trung, cùng DNS server mà IdM server đã kết nối tới
  - Mở các cổng để kết nối tới IdM server, tương ứng với các port đã mở của IdM Server
    - 80, 443 - TCP: Để cung cấp UI và API cho IdM
    - 389, 636 - TCP: Cấp cổng cho dịch vụ LDAP hoạt động
    - 88, 464 - TCP/UDP: Cấp cổng cho dịch vụ [Kerberos](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/reference_guide/ch-kerberos) -->

## Bắt đầu cài đặt

Theo tài liệu ta có thể [sử dụng Ansible](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-installing_an_idm_server_with_an_integrated_ca_using_an_ansible_playbook) để cài đặt. Nhưng tôi sẽ theo phương pháp CLI truyền thống để cài đặt.

### Bước 1: Chọn realm và domain phù hợp

> Lưu ý: Trong bài viết này, tôi sẽ lấy domain của tôi là `haivq.local`. Vui lòng thay đổi domain này thành domain thực tế của bạn.

Chọn 1 domain, realm và server hostname mà bạn muốn sử dụng cho IdM như tôi sẽ chọn như sau:

```
Domain: haivq.local
Realm: HAIVQ.LOCAL
Server hostname: idm.haivq.local
```

Ghi nhớ lấy thông tin này.

> Chú ý không chọn trùng với domain và realm của dịch vụ khác, ví dụ như bạn có [Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) (AD) trong hệ thống thì phải né realm và domain của AD ra, tránh các lỗi do chồng lấn domain gây ra, và trong tương lai có thể trust và sync user từ AD về.

Sau đó, sửa file `/etc/hosts` và đặt IP IdM server phân giải ra server domain của bạn. Giả sử IP server của tôi là `192.168.1.38` thì thêm 1 entry vào file hosts như sau:

```
192.168.1.38 idm.haivq.local
```

### Bước 2: Chuẩn bị môi trường

Chuyển sang user `root` và set `umask` để tránh việc file user `root` tạo ra, các user khác không đọc được:
```bash
sudo -i
umask 0022
```

Cài package `ipa-server` để chuẩn bị cài đặt:

```bash
dnf install ipa-server ipa-healthcheck
```

### Bước 3: Cấu hình mạng

Mở toàn bộ port cần thiết cho `ipa-server`:

```bash
firewall-cmd --permanent --add-service=freeipa-4
firewall-cmd --reload
```

Cấu hình DNS server tới DNS tập trung đã nêu ở trên. Có rất nhiều cách để cấu hình DNS, như dùng `cockpit`, `nmtui`, `nmcli`, sửa trên GNOME UI. Hãy chọn cách mà bạn thấy thuận tiện nhất.

> Lưu ý rằng sửa trực tiếp trong `/etc/resolv.conf` không được khuyến khích vì file này được quản lý bởi `NetworkManager` nên nếu có thay đổi về `NetworkManager` thì nó sẽ ghi đè file `/etc/resolv.conf`.

Cài đặt hostname của IdM server cho trùng với hostname của server hostname đã chọn ở trên. Tương tự, hãy chọn cách sửa hostname mà bạn thấy dễ nhất. Tôi dedicate luôn con máy hiện tại để làm IdM server nên sửa luôn hostname máy đó cho dễ phân biệt. Trong trường hợp bạn muốn kết hợp 1 server homelab làm nhiều việc thì có thể giữ nguyên tên cũ ví dụ như `home-server-1.haivq.local` cũng được, không nhất thiết phải sửa lịa hostname.

### Bước 4: Cài đặt IdM

Sử dụng command sau để cài đặt IdM (Thay đổi directory server password và admin password theo ý bạn):
```bash
ipa-server-install \
  --domain haivq.local \
  --realm HAIVQ.LOCAL \
  --hostname=idm.haivq.local \
  --ds-password SomeDirectoryServerAdminPassword \
  --admin-password SomeIdMAdminUserPassword \
  --unattended
```

Đợi một lúc cho để installer làm việc của mình. Khi cài xong thì installer cho bạn 1 file các record DNS ở địa chỉ `/tmp/ipa.system.records.<random_characters>.db` mà bận cần phải add vào DNS server. Add tất cả các record đó vào DNS server.

Một ví dụ nội dung của file DNS record:
```
_kerberos-master._tcp.haivq.local. 3600 IN SRV 0 100 88 idm.haivq.local.
_kerberos-master._udp.haivq.local. 3600 IN SRV 0 100 88 idm.haivq.local.
_kerberos._tcp.haivq.local. 3600 IN SRV 0 100 88 idm.haivq.local.
_kerberos._udp.haivq.local. 3600 IN SRV 0 100 88 idm.haivq.local.
_kerberos.haivq.local. 3600 IN TXT "HAIVQ.LOCAL"
_kerberos.haivq.local. 3600 IN URI 0 100 "krb5srv:m:tcp:idm.haivq.local."
_kerberos.haivq.local. 3600 IN URI 0 100 "krb5srv:m:udp:idm.haivq.local."
_kpasswd._tcp.haivq.local. 3600 IN SRV 0 100 464 idm.haivq.local.
_kpasswd._udp.haivq.local. 3600 IN SRV 0 100 464 idm.haivq.local.
_kpasswd.haivq.local. 3600 IN URI 0 100 "krb5srv:m:tcp:idm.haivq.local."
_kpasswd.haivq.local. 3600 IN URI 0 100 "krb5srv:m:udp:idm.haivq.local."
_ldap._tcp.haivq.local. 3600 IN SRV 0 100 389 idm.haivq.local.
ipa-ca.haivq.local. 3600 IN A 192.168.1.38
```

> Trong file này sẽ yêu cầu bạn phải phải add các record URI, mà một số DNS sẽ không hỗ trợ (hoặc không hỗ trợ native). Để add record URI, tìm kiếm documentation từ DNS provider của bạn, hoặc tham khảo một số KCS sau từ Red Hat:
>
> - Windows DNS: [KCS 7004658](https://access.redhat.com/kb/7004658)
> - Infoblox DNS: [KCS 7056567](https://access.redhat.com/kb/7056567)
>
> Trong trường hợp của tôi thì Mikrotik hoàn toàn không hỗ trợ tạo record URI. Việc không có record URI sẽ khiến `ipa-client` fallback về record `SRV`, dẫn đến việc truy vấn DNS nhiều hơn.

Sau đó, bật ACME server để sử dụng về sau:
```bash
ipa-acme-manage enable
```

Sau khi add xong DNS, chạy `ipa-healthcheck` để kiểm tra tình trạng của IdM:
```bash
ipa-healthcheck
```

### Bước 5: Truy cập IdM

Sau khi hoàn thành cài đặt, truy cập IdM qua địa chỉ đã chọn (ví dụ https://idm.haivq.local/ipa/ui/). Nhập username `admin` và password bạn đã điền ở trên để đăng nhập vài IdM.

Vậy là bạn đã cài đặt xong IdM.

# Một số best practice nên làm ngay sau khi cài đặt

Các best practice ở đây đều không bắt buộc, nhưng khi đã tích hợp một hệ thống quản lý tập trung như IdM, thì đây luôn là target tấn công đầu tiên của cả hạ tầng. Vậy nên có thói quen cài đặt các best practice là một việc tốt.

## Tắt `allow_all` HBAC rule

Mặc định IdM có một rule HBAC default là `allow_all`, cho phép TẤT CẢ user truy cập vào TẤT CẢ các host đã đăng kí vào. Dù rule `allow_all` này không cho phép người dùng có quyền `sudo`, nhưng vẫn khá nguy hiểm nếu cho phép user có thể login được tất cả các máy.

> Lưu ý rằng sau khi bạn tắt HBAC rule `allow_all` này mà không tạo HBAC rule khác thì không user nào có quyền truy cập bất kì máy chủ nào trong hệ thống. Nếu bạn muốn thử, hãy giữ nguyên HBAC rule này và tắt nó đi ở sau khi bạn đã tạo HBAC rule ở phần 2.

## Chặn anonymous LDAP bind

Mặc định Directory Server của IdM không chặn anonymous bind, tức là không cần credential truy cập IdM, bất kì ai biết địa chỉ của IdM server có thể lấy được tất cả thông tin về config và dữ liệu (ví dụ như danh sách tất cả user) từ Directory Server. Để chặn việc này xảy ra, ta sẽ sửa 1 attribute trên Directory Server.

Chạy lệnh sau trên máy IdM:
```bash
ldapmodify -x -D "cn=Directory Manager" -W -h idm.haivq.local -p 389
```

Lệnh này sẽ đòi password của Directory Server, lưu ý rằng password của IPA admin và Directory Server là khác nhau. Chú ý nhập đúng password nếu bạn đặt password của 2 cái này khác nhau. Nếu nhập đúng password, thì terminal sẽ dừng lại ở 1 dòng trống. Lúc ấy, paste nội dung sau vào để sửa `nsslapd-allow-anonymous-access` attribute:

```
dn: cn=config
changetype: modify
replace: nsslapd-allow-anonymous-access
nsslapd-allow-anonymous-access: rootdse
```

Nhập xong enter 2 lần, ta sẽ có output như sau:
```
modifying entry "cn=config"
```

Sau đó restart Directory Server để nhận config:

```bash
sudo systemctl restart dirsrv.target
```

Sau khi restart xong, chạy thử lệnh liệt kê danh sách user để check chắc chắn rằng Directory Server đã block việc lấy thông tin (chú ý sửa thông tin DN thành DN của bạn, ở đây lấy ví dụ là `dc=haivq,dc=local`):
```bash
ldapsearch -LLL -x \
  -H ldaps://idm.haivq.local \
  -b "cn=users,cn=accounts,dc=haivq,dc=local" \
  "(uid=*)" dn uid cn
```

Nếu lệnh trên trả ra kết quả như sau là đã chặn anonymous LDAP bind:

```
Inappropriate authentication (48)
Additional information: Anonymous access is not allowed.
```

## Tắt Schema Compatibility (`compat`) plugin nếu không dùng

Nếu hệ thống của bạn không còn dùng NIS hay các phiên bản UNIX/Linux cổ lỗ sĩ như Solaris thì không cần thiết phải bật plugin `compat` trên Directory Server nữa. Chạy command sau để tắt tính năng này đi (lưu ý rằng bước này cũng sẽ đòi password của Directory Server):

```bash
ipa-compat-manage disable
```

# Kết thúc phần 1

Đến đây là đã kết thúc phần 1 về việc cài đặt IdM. Tôi sẽ làm phần 2 về việc sử dụng IdM, bao gồm:
- Enroll máy, tạo user/group và các HBAC rule
- Issue certificate từ IdM
- Tích hợp ACME với `cert-manager` để OCP/k8s tự động kết nối tới IdM để issue ra certificate
- Cấu hình các ứng dụng khác kết nối với IdM thông qua LDAP để lấy danh sách user

# Tài liệu tham khảo
- [About LDAP | Red Hat Documentatuib](https://docs.redhat.com/en/documentation/red_hat_directory_server/13/html/planning_and_designing_directory_server/introduction-to-directory-services#about-ldap) / [archive](https://web.archive.org/web/20261124165851/https://docs.redhat.com/en/documentation/red_hat_directory_server/13/html/planning_and_designing_directory_server/introduction-to-directory-services#about-ldap)
- [7 features of Red Hat Identity Management you need to know for the modern enterprise | Red Hat Blog](https://www.redhat.com/en/blog/7-features-red-hat-identity-management-you-need-know-modern-enterprise) / [archive](https://web.archive.org/web/20261123063022/https://www.redhat.com/en/blog/7-features-red-hat-identity-management-you-need-know-modern-enterprise)
- [FreeIPA - Identity, Policy, Audit](https://www.freeipa.org)
- [Identity Management in Red Hat Enterprise Linux | Red Hat Documentation](https://access.redhat.com/products/red-hat-identity-management/) / [archive](https://web.archive.org/web/20261123063159/https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/plan-overview_of_idm_and_access_control_in_rhel)
  - [Overview of IdM and access control in RHEL](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/plan-overview_of_idm_and_access_control_in_rhel) 
  - [Install an IdM server: Without integrated DNS, with an integrated CA as the root CA](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-installing_an_idm_server_without_integrated_dns_with_an_integrated_ca_as_the_root_ca) / [archive](https://web.archive.org/web/20261123063320/https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-installing_an_idm_server_without_integrated_dns_with_an_integrated_ca_as_the_root_ca)
  - [Deploy and manage the ACME service in IdM](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-deploying_and_managing_the_acme_service_in_idm_lightweight_sub_ca#the-acme-service-in-idm) / [archive](https://web.archive.org/web/20261123063428/https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/install-deploying_and_managing_the_acme_service_in_idm_lightweight_sub_ca#the-acme-service-in-idm)
  - [IdM security hardening implementation phases](https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/secure-idm_security_hardening_implementation_phases) / [archive](https://web.archive.org/web/20261123063535/https://docs.redhat.com/en/documentation/identity_management_in_red_hat_enterprise_linux/10/secure-idm_security_hardening_implementation_phases)


