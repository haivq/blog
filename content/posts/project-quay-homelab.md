---
title: "Cài đặt Project Quay làm Registry cho homelab"
author: "Aperture"
date: "2026-09-19T20:30:00+07:00"
categories:
    - Quay
    - Registry
    - Kubernetes
    - OpenShift
    - Disconnected
    - Air-gapped
    - Red Hat
tags:
    - quay
    - mirror
    - registry
    - rhquay
    - kubernetes
    - k8s
    - openshift
    - ocp
    - disconnected
    - airgapped
    - redhat
    - rh
    - s3
    - devops
    - infrastructure
    - experience
draft: true
---

# Hoàn cảnh

Sau khi vào làm Red Hat một thời gian, đột nhiên tôi lại có hứng thú trở lại với việc setup homelab. Từ ý định ban đầu chỉ setup 1 2 con máy server chạy OCP để làm lab demo cho khách hàng, càng ngày tôi lại càng lún sâu vào việc xây dựng homelab. Ban đầu chỉ là lưu vài tấm ảnh, lưu một 2 bộ phim để xem rồi xoá, và làm hub smart home, cuối cùng tôi setup luôn cả server media, mua NAS để làm lưu trữ và sắm một con router ngon hơn để gánh tải mạng (dù sự thực tôi thấy con Mikrotik này hiệu năng Wifi 6 quá cùi).

{{< figure 
    src="/posts/project-quay-homelab/homelab.jpeg"
    position="center"
    alt="Khoe góc homelab tại nhà"
    caption="Khoe góc homelab tại nhà" >}}

Gần đây tôi lại có nhu cầu cài đặt OCP disconnected để thi thêm RHCOA và thử nghiệm một dự án nho nhỏ cũng yêu cầu air-gap, nên tôi tính cài đặt luôn một con registry để lưu trữ image để về sau tải lại cho nhanh, dù sao thì dung lượng NAS cũng đang thừa. Vì đã làm ở Red Hat nên chắc chắn tôi sẽ bị bias bởi các sản phẩm của Red Hat, tôi chọn giải pháp Project Quay làm giải pháp registry.

# Project Quay là cái gì?



# Nguồn tham khảo
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/):
    * [15.1.20.5 FOREIGN KEY Constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking) / [archive](https://web.archive.org/web/20240706020716/https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking)
    * [17.12.1 Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations) / [archive](https://web.archive.org/web/20240614132951/https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations)
    * [10.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release) / [archive](https://web.archive.org/web/20240710093303/https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release)
- [AWS Documentation - Amazon RDS](https://docs.aws.amazon.com/rds/):
    * [Common DBA tasks for MySQL DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End) / [archive](https://web.archive.org/web/20240713052429/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End)
    * [RDS for MySQL stored procedure reference - Ending a session or query](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html) / [archive](https://web.archive.org/web/20240225060423/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html)
    * [Managing an RDS Proxy - Avoiding pinning](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-managing.html#rds-proxy-pinning) / [archive](https://web.archive.org/web/20240704212205/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-managing.html#rds-proxy-pinning)
- [StackOverflow](https://stackoverflow.com)
    * [MySQL 5.6 - table locks even when ALGORITHM=inplace is used](https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used) / [archive](https://web.archive.org/web/20240717161527/https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used)
- [Percona Blog](https://www.percona.com/blog/)
    * [Chasing a Hung MySQL Transaction: InnoDB History Length Strikes Back](https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/) / [archive](https://web.archive.org/web/20240522170813/https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/)
    * [How small changes impact complex systems – MySQL example](https://www.percona.com/blog/small-changes-impact-complex-systems-mysql-example/) / [archive](https://web.archive.org/web/20240718070236/https://www.percona.com/blog/small-changes-impact-complex-systems-mysql-example/)
- [Planet MySQL](https://planet.mysql.com/)
    * [Tracking MySQL query history in long running transactions](https://planet.mysql.com/entry/?id=5988591) / [archive](https://web.archive.org/web/20240814092726/https://planet.mysql.com/entry/?id=5988591)