---
title: "Cài Project Quay 3.18 làm Container Registry cho Homelab với Podman và S3"
author: "Aperture"
date: 2026-09-19T15:00:00+07:00
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
    - projectquay
    - redhatquay
    - rhquay
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
cover: "/posts/project-quay-homelab/cover.png"
images:
    - "/posts/project-quay-homelab/cover.png"
---

# Hoàn cảnh

Sau khi vào làm Red Hat một thời gian, đột nhiên tôi lại có hứng thú trở lại với việc setup homelab. Từ ý định ban đầu chỉ setup 2 con máy mini PC chạy OCP để làm lab demo cho khách hàng, càng ngày tôi lại càng lún sâu vào cái hố vôi này. Khởi điểm chỉ là lưu catalog ảnh chụp, lưu thêm một vài bộ phim để xem với vợ và làm hub smart home để điều khiển mấy món đồ không tương thích với Homekit, cuối cùng tôi setup luôn cả server media, cài 1 cụm OCP lớn, mua NAS để làm lưu trữ trung tâm và sắm một con router ngon hơn để gánh tải mạng (dù sự thực tôi thấy con Mikrotik hAP ax³ này hiệu năng Wifi 6 quá cùi so với giá thành của nó).

{{< figure
    src="/posts/project-quay-homelab/homelab.jpeg"
    position="center"
    alt="Khoe góc homelab tại nhà"
    caption="Khoe góc homelab tại nhà" >}}

Gần đây tôi lại có nhu cầu cài đặt OCP disconnected để có thêm kinh nghiệm và có môi trường ôn RHCOA, tiện thể thử nghiệm một dự án nho nhỏ cũng yêu cầu air-gap, nên tôi tính cài đặt luôn một con registry để lưu trữ image, làm mirror cho việc cài đặt OCP disconnected và làm image proxy để pull image cho nhanh (dù sao thì dung lượng NAS cũng đang thừa). Do khách hàng sử dụng Quay làm registry tương đối nhiều nên chọn luôn Quay làm giải pháp registry.

# Vậy Quay là cái gì?

## Giới thiệu về Quay

Mặc dù người tìm đến bài viết này nhiều khi vốn đã hiểu về Quay hay registry rồi, nhưng tôi xin phép được giới thiệu ngắn gọn như sau:

[Project Quay](https://github.com/quay/quay) là một registry open source, ban đầu chính là công nghệ đứng sau [quay.io](quay.io), di sản của team [CoreOS](https://www.redhat.com/en/technologies/cloud-computing/openshift/what-was-coreos). Sau khi được Red Hat mua lại thì project được open source thành Project Quay. Tất nhiên đúng cách làm việc của Red Hat, Project Quay chính là upstream của sản phẩm enterprise mà họ đem đi bán - [Red Hat Quay](https://www.redhat.com/en/technologies/cloud-computing/quay).

Không chỉ làm registry đơn thuần, Project Quay còn rất nhiều tính năng khác hay ho mà chắc chắn tôi sẽ không giới thiệu trong bài viết này, người đọc vui lòng tự đọc ở [trang giới thiệu này](https://docs.projectquay.io/red_hat_quay_overview.html). Tôi cũng sẽ cài Project Quay thay vì Red Hat Quay, nhưng phương pháp cài giữa 2 phiên bản này là như nhau, bạn có thể cài Red Hat Quay nếu bạn muốn (nhớ tạo tài khoản Red Hat và active trial license trước nhé).

## Mô tả kiến trúc của Quay

Nếu bạn đã xài [Docker registry](https://hub.docker.com/_/registry) thì sẽ thấy cài đặt nó rất đơn giản, chỉ cần start 1 cái container registry lên là xong. Nhưng với Quay thì mọi thứ lằng nhằng hơn thế, vì ngoài là chỗ chứa registry ra thì nó còn host rất nhiều các tính năng khác (mà trong bài viết này sẽ tắt đi kha khá). Vì vậy để deploy Quay thì cần phải có các component sau:

  - Quay: Chính là cái Quay instance làm đầu não xử lý logic
  - PostgreSQL: 1 cái database để chứa thông tin, backup định kì vì không thể thay thế
  - Redis: Làm cache để Quay lưu dữ liệu tạm thời, nên nếu có sự cố thì có thể tạo nhanh một Redis mà không sợ ảnh hưởng tới dữ liệu quan trọng.
  - S3/S3-compatible storage (tuỳ chọn): Nơi thực sự chứa image data. Thực tế không bắt buộc phải có S3, nhưng không ai muốn quản 1 cái ổ cứng nặng trịch trong máy cả, đẩy được ra S3 cho nó nhẹ đầu óc, scale ra cũng dễ hơn

Trong documentation của Quay có 2 phương pháp:
  - Cài [Proof-of-Concept](https://docs.projectquay.io/quay_jtbd-install.html#install-red-hat-quay-proof-of-concept_install_red_hat_quay_on_openshift_container_platform): Nhét tất cả mọi thứ vào 1 máy, lưu data của PostgreSQL và blob của image trên chính disk của máy đó, với mục đích start Quay lên nhanh nhất có thể để trải nghiệm.
  - Cài [High-availability](https://docs.projectquay.io/quay_jtbd-install.html#preparing-for-quay-ha): Thực sự cài 1 con Quay phù hợp cho production, yêu cầu phải có 2-3 node cài Quay + Redis, 1 node làm HAproxy + PostgreSQL, 1 node làm Clair và trong documentation có gợi ý cài 5 node làm CEPH cluster cho S3 nếu chưa có S3 provider nào khác.

Cả 2 phương pháp trên, cài kiểu Proof-of-Concept thì quá nhỏ, không phải best practice và không thực sự dạy ta được cái gì trong lúc cài, cài kiểu High-availability như trong tài liệu thì quá lớn và kềnh càng (tận 3 node quay, 1 node DB và LB, 1 node Clair, 5 node CEPH ko tính vì họ chỉ lấy ví dụ), không đủ tài nguyên để dựng mà quản lý cũng nhọc óc, vậy nên ta sẽ đi theo một con đường dung hoà cả 2:
  - 1 node cài toàn bộ Quay, Redis và PostgreSQL để tiết kiệm tài nguyên như Proof-of-Concept
  - Đẩy toàn bộ blob của image ra một S3-compatible thay vì lưu hết vào ổ đĩa máy cài Quay như High-availability.
  - Bỏ Clair vì registry này tôi chỉ cần lưu trữ image, không cần scan security

# Chuẩn bị trước khi cài đặt Quay

## Cấu hình của tôi

Vậy là sau khi chọn con đường hybrid, ta sẽ cần phải sizing tài nguyên trước khi cài. Dựa vào [tài liệu sizing của Quay](https://docs.projectquay.io/quay_jtbd-plan.html#sizing-intro), tôi lựa chọn cấu hình deploy như sau:

  - OS: RHEL 10 (vì tôi đang sẵn có server RHEL 10, thực tế bạn có thể cài trên Fedora, CentOS hay Alma Linux, Rocky Linux, chúng đều có chung nguồn gốc là từ RHEL mà ra. Tôi chưa test trên các Linux Distro khác, hoan nghênh bạn đọc đóng góp ý kiến)
  - Container runtime: Podman (vì nó là sản phẩm mặc định trong server RHEL 10, chạy được rootless container, cũng không phải đương đầu với userland proxy của Docker)
  - Disk: Trống 40G cho chắc ăn
  - RAM: Trống 8G
  - CPU: 2 core cho Quay, Redis và PostgreSQL mỗi cái 1 core, tổng là 4 core
  - S3-compatible: MinIO đặt trên máy NAS

## Chuẩn bị các image trước khi cài

Vì tất cả cài qua container, nên tôi cũng cài hết các component trên bằng container cho tiện. Dưới đây là các image mà tôi chọn để cài:

  - [Project Quay 3.18](https://www.projectquay.io/): quay.io/projectquay/quay:3.18.0
  - [PostgreSQL 18.6](https://images.redhat.com/?search=postgres&name=postgresql&version=18.6): registry.access.redhat.com/hi/postgresql:18.6
  - [Valkey 9.0.6 thay cho Redis](https://images.redhat.com/?search=valkey&name=valkey&version=9.0.6): registry.access.redhat.com/hi/valkey:9.0.6

Bạn có thể sẽ có 3 câu hỏi sau, và tôi xin trả lời luôn:
  1. Tại sao lại dùng Valkey thay vì Redis: Redis đã thay đổi license của mình từ BSD sang [SSPL](https://www.mongodb.com/legal/licensing/server-side-public-license)/[RSALv2](https://redis.io/legal/rsalv2-agreement/), tức là người dùng end-user có thể dùng miễn phí và contribute cho Redis như bình thường, nhưng sẽ là cú đấm cho các nền tảng khác dựng Redis lên và bán lại (như AWS ElastiCache). Bạn có thể đọc bài giải thích về sự kiện này trong [một bài viết của TechCrunch](https://techcrunch.com/2024/03/31/why-aws-google-and-oracle-are-backing-the-valkey-redis-fork/). Dù Redis 8 đã bổ sung giấy phép [AGPLv3](https://www.gnu.org/licenses/agpl-3.0.en.html), nhưng tôi đang muốn tìm hiểu tương thích giữa Valkey so với Redis, và trước mắt đang hoạt động ổn trong lab của tôi, nên tôi lựa chọn Valkey. Tuy nhiên, trong tài liệu của Project Quay không nói rõ ràng về việc Valkey đã được test và thay thế hoàn toàn cho Redis, nên hãy cân nhắc trước khi sử dụng Valkey trong môi trường prouction.
  2. Image của PostgreSQL và Valkey là gì trông lạ vậy, cái HI là gì thế: HI thực ra chính là Hardened Image của Red Hat, được thiết kế ra để giảm các vấn đề về Security đến mức tối thiểu, bắt nguồn từ dự án [Humming Bird](https://hummingbird-project.io/). Tôi đang tìm hiểu về Hardened Image nên sử dụng luôn. Thực tế tôi đã tìm image Redis thay vì Valkey, nhưng không tìm thấy trong [catalog Hardened Image của Red Hat](https://images.redhat.com/), nên đổi sang sử dụng thử Valkey.
  3. Dùng Valkey có ổn không: Valkey là một bản fork của Redis, giống như MariaDB và MySQL vậy. Nếu sử dụng một cách cơ bản bình thường thì theo tôi thấy không có gì khác so với Redis. Hơn nữa Redis/Valkey cũng chỉ là cache và không chứa thông tin gì quan trọng cả, nên khi cần ta có thể dựng một con Redis lên thay cho Valkey. 

# Các bước cài đặt Quay

OK sau khi bạn đã chuẩn bị xong xuôi các bước ở trên, ta bắt đầu quá trình cài đặt Quay.

## Cấu hình sơ bộ cho Quay host

### Chuẩn bị directory chứa toàn bộ dữ liệu của Quay

Để gom toàn bộ deployment vào một chỗ, tôi tạo 1 directory chứa toàn bộ dữ liệu của Quay. Trong ví dụ này tôi để luôn ở `/home/haivu` vì `/home` đang có dung lượng lớn, chứa dữ liệu ở đây sẽ tiện.

```bash
mkdir ~/quay
cd ~/quay
```

Từ bây giờ ta sẽ lấy `~/quay` làm gốc và mọi thư mục cho các component mới đều sẽ để tại đây.

### Cấu hình network của Podman cho tiện lợi

Sau khi tạo directory `~/quay`, tạo network `quay-net` để ta có thể dùng container name thẳng trong cùng 1 network mà không phải expose port ra ngoài

```bash
podman network create quay-net
```

### Cấu hình linger để ngăn việc container bị exit sau khi thoát SSH session.

Do rootless podman chạy trong context của một user nhất định, nên sau khi user logout ra khỏi SSH, tất cả các container tạo bởi rootless Podman sau một thời gian sẽ bị stop lại. Cách xử lý chính là bật `linger` lên để giữ cho container không bị tắt kể cả khi user đã logout ra khỏi máy.

Để bật tính năng này lên, chạy câu lệnh sau:

```bash
loginctl enable-linger
```

> Lưu ý: Đây có thể không phải behavior mà bạn mong muốn, vậy nên nếu bạn muốn chỉ mỗi mình các component của Quay được sống, còn các process đang chạy của bạn sẽ tự kết thúc sau khi thoát SSH, hãy tạo 1 user mới và bật `linger` cho user đó.

### Cấu hình DNS cho Quay

Chọn một hostname mà bạn muốn đặt cho Quay. Trong ví dụ này tôi lấy luôn `quay.haivq.local`. Tạo một record A trên DNS nội bộ của bạn trỏ vào hostname này.

```
A quay.haivq.local 192.168.1.123 
```

Sau khi chọn hostname và cập nhật DNS, sửa file `/etc/hosts`, đặt IP của Quay host trùng với hostname của Quay để phân giải DNS cho nhanh, tránh các lỗi khù khoằm do DNS gây ra (sửa IP của máy hiện tại của bạn vào đây, tôi sẽ giả sử IP sẽ là `192.168.1.123`):

```
192.168.1.123 quay.haivq.local
```

### Sinh Root CA và certificate cho Quay

Theo chuẩn sách giáo khoa, ta buộc phải tạo một SSL certificate cho Quay.

> Trong bài viết này tôi sẽ issue ra luôn RootCA và Certificate cho nhanh. Nếu bạn đã có một CA chung cho toàn bộ hệ thống (ví dụ Dogtag PKI CA) thì chỉ cần tạo CSR và gửi cho CA để lấy certificate về.

Về lại `~/quay`, tạo một thư mục `certs` và tạo certificate tại đó (phần này tôi làm theo hướng dẫn của ChatGPT, lưu ý thay giá trị SAN và CN cho sát nhu cầu thực tế):

```bash
cd ~/quay
mkdir certs
cd certs
# tạo private key cho rootCA
openssl genpkey \
    -algorithm RSA \
    -pkeyopt rsa_keygen_bits:4096 \
    -out rootCA.key\
chmod 600 rootCA.key

# tạo RootCA certificate 100 năm
openssl req \
    -x509 \
    -new \
    -key rootCA.key \
    -sha256 \
    -days 36500 \
    -out rootCA.crt \
    -subj "/C=VN/O=HAIVQ Homelab/CN=HAIVQ Root CA" \
    -addext "basicConstraints=critical,CA:TRUE" \
    -addext "keyUsage=critical,keyCertSign,cRLSign" \
    -addext "subjectKeyIdentifier=hash"

# tạo private key cho Quay
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:4096 \
  -out quay.key
chmod 600 quay.key

# tạo CSR cho Quay
openssl req \
  -new \
  -key quay.key \
  -out quay.csr \
  -subj "/C=VN/O=HAIVQ Homelab/CN=quay.haivq.local" \
  -addext "subjectAltName=DNS:quay.haivq.local"

# Tạo extension cho Quay
cat > quay.ext <<'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:quay.haivq.local
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
EOF

# Dùng Root CA ký cho Quay CSR cert 10 năm
openssl x509 \
  -req \
  -in quay.csr \
  -CA rootCA.crt \
  -CAkey rootCA.key \
  -CAcreateserial \
  -out quay.crt \
  -days 3650 \
  -sha256 \
  -extfile quay.ext
```

Bây giờ bạn đã có các file sau:
```
rootCA.key
rootCA.crt
rootCA.srl

quay.key
quay.csr
quay.crt
quay.ext
```

Về sau khi cần kết nối tới Quay sử dụng các công cụ như [`oc-mirror`](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/disconnected_environments/about-installing-oc-mirror-v2) hay sử dụng Quay làm mirror registry cho OCP, ta sẽ phải trust Root CA đã tạo. Vì vậy lưu ý lưu các file `rootCA.*` cẩn thận, vì mất CA thì ta sẽ phải sinh lại CA rồi tiến hành trust ở tất cả các máy đang sử dụng Qua, gây tốn rất nhiều thời gian không mong muốn, đặc biệt khi trust một CA mới cho OCP sẽ gây reboot toàn bộ các node.

Sau khi chuẩn bị xong các bước khởi tạo ban đầu, ta bắt đầu cài Quay và các component của nó.

## Cấu hình cho PostgreSQL và Valkey

Ta chuẩn bị trước PostgreSQL và Valkey trước khi khởi chạy Quay

### Khởi tạo PostgreSQL

1. Tại `~/quay`, tạo directory `postgresql/data` để chứa data của postgresql:

```bash
mkdir -p postgresql/data
```

2. Bật `postgresql` lên để khởi tạo database:

```bash
podman run -d --name postgresql \
    --network quay-net \
    --restart unless-stopped \
    -v /home/haivu/quay/postgresql/data:/var/lib/postgresql/data:Z,U \
    -e POSTGRES_USER=quayuser \
    -e POSTGRES_PASSWORD=quaypassword \
    -e POSTGRES_DB=quaydb \
    registry.access.redhat.com/hi/postgresql:18.6
```

3. Kết nối tới container `postgresql` và cài đặt các extension cần thiết
```bash
# tạo extension pg_trgm cho quaydb
podman exec -it postgresql psql -U quayuser -d quaydb -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"

# verify để chắc chắn pg_trgm đã tồn tại trong quaydb
podman exec -it postgresql psql -U quayuser -d quaydb -c "SELECT * FROM pg_extension"
```

### Khởi tạo Valkey

Valkey chỉ dùng làm in-memory cache nên start lên sẽ đơn giản hơn.

```bash
cd ~/quay
podman run -d --name redis \
    --network quay-net \
    --restart unless-stopped \
    registry.access.redhat.com/hi/valkey:9.0.6 --protected-mode "no" --save "" --appendonly "no"
```

Giải thích các params truyền vào cho Valkey như sau:
  - `--protected-mode "no"`: Cho phép các container khác có thể connect vào Valkey, vì Valkey mặc định phải cùng trong `localhost` (trong trường hợp này là trong cùng một container) mới kết nối được
  - `--save ""` và `--appendonly "no"`: Tắt tính năng [dump data ra disk và AOF của Valkey](https://valkey.io/topics/persistence/), biến Valkey trở thành một in-memory cache thuần tuý.

## Cài đặt Quay

Sau khi PostgreSQL và Valkey đã chạy, ta bắt đầu khởi tạo Quay

### Cấu hình config trước khi bật Quay

1. Về lại `~/quay`, tạo directory `quay` (cùng tên) để chứa `config`:

```bash
cd ~/quay
mkdir -p quay/config
```

2. Khởi tạo config của Quay tại `~/quay/quay/config`

Sử dụng `nano`, `vi` hay bất kì method nào để ghi nội dung sau vào file `quay/config/config.yaml` rồi sửa đổi cho hợp nhu cầu.

```yml
AUTHENTICATION_TYPE: Database
PREFERRED_URL_SCHEME: https
# Đổi hostname của bạn vào đây
SERVER_HOSTNAME: quay.haivq.local:8443
SECRET_KEY: somesecretkey
DATABASE_SECRET_KEY: somedbsecretkey
DB_URI: postgresql://quayuser:quaypassword@postgresql:5432/quaydb

BUILDLOGS_REDIS:
  host: redis
  port: 6379
  ssl: false

USER_EVENTS_REDIS:
  host: redis
  port: 6379
  ssl: false

DISTRIBUTED_STORAGE_CONFIG:
  minioDSM:
    - RadosGWStorage
    - access_key: somes3accesskey # thay access key
      bucket_name: quay # thay bucket name
      hostname: miniohostname # thay hostname của s3
      is_secure: false
      port: '9000'
      secret_key: somesecretkey # thay secret key
      storage_path: /datastorage/registry # đổi storage path thành cái khác nếu muốn, lưu ý đấy là storage path trong s3, không phải volume của Quay
      signature_version: v4

DISTRIBUTED_STORAGE_PREFERENCE:
  - minioDSM
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []

SUPER_USERS:
  - quayadmin

# Giảm số lượng worker để tiết kiệm RAM
# NOTE: trong doc có nói WORKER_COUNT_REGISTRY min là 8, nhưng do tôi dùng ít nên để là 4 cho tiết kiệm
WORKER_COUNT_REGISTRY: 4
WORKER_COUNT_WEB: 2
WORKER_CONNECTION_COUNT_REGISTRY: 10
WORKER_CONNECTION_COUNT_WEB: 5

# Tắt một loạt các feature không cần thiết để nhẹ gánh cho Quay
FEATURE_BUILD_SUPPORT: false
FEATURE_SECURITY_SCANNER: false
FEATURE_MAILING: false
FEATURE_ORG_MIRROR: false
FEATURE_REPO_MIRROR: false
FEATURE_PROXY_CACHE: true
FEATURE_QUOTA_MANAGEMENT: false
FEATURE_STORAGE_REPLICATION: false
FEATURE_RATE_LIMITS: false
FEATURE_ANONYMOUS_ACCESS: false
FEATURE_USER_CREATION: false
ROBOTS_DISALLOW: true

# Bật các tính năng cần thiết lên phục vụ cho việc mirror image
CREATE_NAMESPACE_ON_PUSH: true
CREATE_PRIVATE_REPO_ON_PUSH: true
FEATURE_EXTENDED_REPOSITORY_NAMES: true
FEATURE_GENERAL_OCI_SUPPORT: true

# Điều chỉnh một số cấu hình để giảm tải cho Quay và ổ cứng
CLEAN_BLOB_UPLOAD_FOLDER: true
GARBAGE_COLLECTION_FREQUENCY: 300

# 2 flag này cần có để dựng Quay production
TESTING: false
SETUP_COMPLETE: true

# Bật tính năng này lên để có thể khởi tạo username/password, sẽ tắt đi sau khi khởi tạo xong
FEATURE_USER_INITIALIZE: true
```

Để cho Quay chạy nhẹ nhàng, tôi đã tắt một loạt tính năng và giảm cấu hình mặc định của Quay xuống, vui lòng xem ý nghĩa của các config trên ở các mục này:
  - [Configure Project Quay](https://docs.projectquay.io/config_quay.html)
  - [Manage Project Quay](https://docs.projectquay.io/manage_quay.html)
  - [Optimize](https://docs.projectquay.io/quay_jtbd-optimize.html)

Để generate `SECRET_KEY` và`DATABASE_SECRET_KEY`, tôi dùng command sau cho nhanh:

```bash
openssl rand -hex 32
```

3. Copy các certificate đã tạo vào trong thư mục `config` của Quay

Để quay có thể sử dụng certificate đã sinh ra, bắt buộc phải để các file certificate này vào thư mục config của Quay với đúng tên `ssl.key` và `ssl.crt`. Ta phải copy đúng 2 file này vào đúng vị trí cạnh file `config.yaml` ở trên thì Quay mới nhận certificate và hoạt động:


```bash
cp ~/quay/certs/quay.key ssl.key
cp ~/quay/certs/quay.crt ssl.crt
```

Giờ trong thư mục `config` sẽ chứa:

```
config.yaml
ssl.key
ssl.crt
```

### Bật Quay và khởi tạo user

1. Đến thời điểm này, ta có thể khởi động Quay lên được rồi:
```bash
podman run -d --name quay \
    --network quay-net \
    --restart unless-stopped \
    -p 8443:8443 \
    -v /home/haivu/quay/quay/config:/conf/stack:Z,U \
    quay.io/projectquay/quay:3.18.0
```

Đợi một lúc cho quay migrate database, ta có thể vào được giao diện của Quay ngay tại: `quay.haivq.local:8443` (Nhớ đổi domain của bạn đi đấy).

2. Sau khi Quay đã online, ta phải khởi tạo password cho user `quayadmin` thì mới login được (chú ý thay đổi username, password, email cho phù hợp với mục đích sử dụng):
```bash
curl -k -X POST https://quay.haivq.local:8443/api/v1/user/initialize -H 'Content-Type: application/json' -d '{
    "username": "quayadmin",
    "password": "quayadmin",
    "email": "quayadmin@haivq.local",
    "access_token": true
}'
```
Sau khi khởi tạo xong, ta có thể dùng username/password đã đặt để vào Quay

3. **QUAN TRỌNG**: Ngay lập tức bỏ trường `FEATURE_USER_INITIALIZE: true` ra khỏi file `config.yaml` để ngăn việc cho phép init password xảy ra mà không qua login, sau đó restart Quay để áp dụng config mới:
```bash
podman container restart quay
```

Vậy là đến đây ta đã cài xong Project Quay.

## Backup cho PostgreSQL

Như đã đề cập ở trên, PostgreSQL cần phải được backup định kì vì nó chứa toàn bộ dữ liệu của Quay. Như bình thường ta có thể chạy 1 container backup, kết nối tới container của PostgreSQL và chạy backup cho `quaydb`, chính là database chứa data của Quay. Trong trường hợp gặp sự cố, ta có thể sử dụng file backup đã dump ra để restore lại PostgreSQL. Để chạy job backup, ta sẽ sử dụng chính [`pg_dump`](https://www.postgresql.org/docs/18/app-pgdump.html) trong chính image của PostgreSQL và nhét nó vào một cronjob chạy hàng ngày.

Để cho tiện, tôi tạo một shell script dưới đây, với mục đích backup vào folder `/mnt/backup/quay/postgresql` - là một NFS tôi mount vào máy RHEL để chứa các file backup. NFS này đã align user và group với user tôi đang chạy Quay nên sẽ không phải gặp lại các vấn đề về quyền nữa. Để tránh gặp phải vấn đề về SELinux và permission giữa container và NFS, tôi sẽ dump nó vào folder `/tmp` trước rồi copy vào NFS sau.

Để cho dễ minh hoạ, file này được đặt ở `/home/haivu/quay/postgresql-quay-backup.sh`. Dưới đây là nội dung của script, vui lòng thay đổi các nội dung như NFS mountpoint, local backup directory, vân vân.

> Script này được viết dựa theo gợi ý của [Bash Coding Standard (BCS)](https://github.com/Open-Technology-Foundation/bash-coding-standard), mục [Atomic file write](https://github.com/Open-Technology-Foundation/bash-coding-standard/blob/main/docs/BCS-Bash-Ref/12_Signals-and-Traps/15_Atomic-file-write.md)

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

log() {
  printf '[%(%Y-%m-%d %H:%M:%S)T] %s\n' -1 "$*"
}

log_error() {
  printf '[%(%Y-%m-%d %H:%M:%S)T] ERROR: %s\n' -1 "$*" >&2
}

# Đảm bảo không có 2 backup job chạy cùng một lúc
LOCK_FILE="/run/user/$(id -u)/quay-postgresql-backup.lock"
exec 9>"$LOCK_FILE"
if ! flock -n 9; then
  log_error "Another backup is already running."
  exit 1
fi

# Đảm bảo file mới tạo có permission restrictive, thay vì phụ thuộc vào umask mặc định của môi trường
umask 077

# Thay đổi cho matching với môi trường thực tế
PG_PASSWORD="quaypassword"
PG_IMAGE="registry.access.redhat.com/hi/postgresql:18.6"
PG_NETWORK="quay-net"

TIMESTAMP="$(date '+%Y-%m-%d_%H-%M-%S')"

BACKUP_NAME="quaydb-$TIMESTAMP.dump"

# Thay đổi local backup dir nếu cần
TMP_BACKUP_DIR="/tmp"
TMP_BACKUP_PATH="$TMP_BACKUP_DIR/$BACKUP_NAME"
TMP_PARTIAL_BACKUP_PATH="$(mktemp "$TMP_BACKUP_PATH.XXXXXX.partial")"

# Thay đổi NFS backup dir cho phù hợp với môi trường thật
NFS_MOUNTPOINT="/mnt/backup"
NFS_BACKUP_DIR="$NFS_MOUNTPOINT/quay/postgresql"
NFS_BACKUP_PATH="$NFS_BACKUP_DIR/$BACKUP_NAME"
NFS_PARTIAL_BACKUP_PATH=""

# Xoá các file partial
cleanup_partials() {
  # Xoá file partial backup dù có lỗi hay ko
  rm -f -- "$TMP_PARTIAL_BACKUP_PATH"

  # xoá file partial trên NFS
  if [[ -n "$NFS_PARTIAL_BACKUP_PATH" ]]; then
    rm -f -- "$NFS_PARTIAL_BACKUP_PATH"
  fi
}

# Chạy cleanup sau khi script exit
trap cleanup_partials EXIT

log "Backing up PostgreSQL to $TMP_BACKUP_PATH..."

if podman run --rm \
    --network "$PG_NETWORK" \
    -e PGPASSWORD="$PG_PASSWORD" \
    --entrypoint pg_dump \
    "$PG_IMAGE" \
    -h postgresql \
    -U quayuser \
    -d quaydb \
    -Fc \
    > "$TMP_PARTIAL_BACKUP_PATH"
then
  mv "$TMP_PARTIAL_BACKUP_PATH" "$TMP_BACKUP_PATH"
else
  rm -f -- "$TMP_PARTIAL_BACKUP_PATH"
  log_error "Backup failed, please check again!"
  exit 1
fi

chmod 600 "$TMP_BACKUP_PATH"

log "Backup file created at $TMP_BACKUP_PATH"

# Trigger NFS automount vì trong hạ tầng của tôi thì tôi đang mount NFS bằng autofs
log "Triggering NFS automount..."

if ! timeout 10 stat "$NFS_BACKUP_DIR" >/dev/null 2>&1; then
  log_error "Unable to access NFS backup directory: $NFS_BACKUP_DIR"
  exit 1
fi

echo "Copying backup to NFS at $NFS_BACKUP_PATH..."

NFS_PARTIAL_BACKUP_PATH="$(
  mktemp "$NFS_BACKUP_PATH.XXXXXX.partial"
)"

if ! cp "$TMP_BACKUP_PATH" "$NFS_PARTIAL_BACKUP_PATH"; then
  log_error "Failed to copy backup to NFS."
  log_error "Local backup preserved at $TMP_BACKUP_PATH"
  exit 1
fi

mv "$NFS_PARTIAL_BACKUP_PATH" "$NFS_BACKUP_PATH"
NFS_PARTIAL_BACKUP_PATH=""

log "Backup file copied to $NFS_BACKUP_PATH"

rm -f -- "$TMP_BACKUP_PATH"

log "Local backup at $TMP_BACKUP_PATH removed"

log "Backup completed!"
```

Giờ ta có thể chạy file này định kì bằng việc biến nó vào Cronjob. Để thêm Cronjob hiện tại, mở Crontab hiện tại bằng lệnh `crontab -e` và cho nội dung như sau vào:

```
CRON_TZ=Asia/Ho_Chi_Minh
TZ=Asia/Ho_Chi_Minh

0 2 * * * /home/haivu/quay/postgresql-quay-backup.sh >> /home/haivu/quay/postgresql/log/backup.log 2>&1
```

Như vậy là job này sẽ luôn chạy lúc 2h sáng theo đúng giờ Việt Nam (múi giờ UTC+7).


# Quản lý deployment bằng compose file

Để đơn giản hoá việc quản lý Quay, ta có thể sử dụng file `docker-compose.yml` để quản trị Quay đơn giản hơn. Config healthcheck, thời gian chờ start/stop container và đợi các container sử dụng compose file nhàn hơn rất nhiều là ngồi truy lại cái command `podman`. Tôi sẽ lấy một ví dụ file compose mà tôi đang sử dụng ở đây, file này được đặt trong directory `~/quay` cho dễ quản lý, vui lòng sửa lại theo nhu cầu của mỗi người:

```yaml
services:
  postgresql:
    image: registry.access.redhat.com/hi/postgresql:18.6
    networks:
      - quay-net
    restart: unless-stopped
    stop_grace_period: 300s
    cpus: "1.0"
    volumes:
      - /home/haivu/quay/postgresql/data:/var/lib/postgresql/data:Z,U
    environment:
      POSTGRES_USER: quayuser
      POSTGRES_PASSWORD: quaypassword
      POSTGRES_DB: quaydb

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"
        ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  redis:
    image: registry.access.redhat.com/hi/valkey:9.0.6
    networks:
      - quay-net
    restart: unless-stopped
    stop_grace_period: 60s
    cpus: "1.0"
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s
    command:
      - --protected-mode
      - "no"
      - --save
      - ""
      - --appendonly
      - "no"

  quay:
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    image: quay.io/projectquay/quay:3.18.0
    networks:
      - quay-net
    restart: unless-stopped
    stop_grace_period: 120s
    cpus: "2.0"
    volumes:
      - /home/haivu/quay/quay/config:/conf/stack:Z,U
    ports:
      - "8443:8443"
    healthcheck:
      test:
        [
          "CMD",
          "curl",
          "-k",
          "-fsS",
          "https://localhost:8443/health/endtoend"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s

networks:
  quay-net:
    name: quay-net
```

Như đã thấy, tôi có thể setup healthcheck cho tất cả component và đặt `depends_on` của Quay vào PostgreSQL và Valkey, đảm bảo 2 component này sống trước rồi mới bật Quay lên và Quay phải xuống trước khi 2 component đó xuống. Tôi cũng đặt luôn `stop_grace_period` cho các component khác nhau để đảm bảo chúng đủ thời gian để gracefully shutdown, thay vì mặc định 10s rồi sẽ bị kill. Thay vì dùng network mặc định mà `podman compose` tạo ra, tôi tạo hẳn một network riêng, việc này sẽ phục vụ cho tác vụ backup về sau.

Khi đặt xong healthcheck, bạn có thể theo dõi trạng thái của Quay trên Cockpit khá là tiện lợi:

{{< figure 
    src="/posts/project-quay-homelab/cockpit-quay.png"
    position="center"
    alt="Xem trạng thái của Quay trên Cockpit"
    caption="Xem trạng thái của Quay trên Cockpit" >}}

Lưu ý rằng khi sử dụng phương pháp compose này, bạn vẫn sẽ cần phải:
  
  - Cấu hình DNS và certificate
  - Chuẩn bị các directory cần thiết
  - Khởi tạo PostgreSQL và cài plugin `pg_trgm` vào `quaydb`
  - Cấu hình `config.yaml` của Quay
  - Cấu hình backup Cronjob cho Quay

# Tổng kết

Ở trên là kinh nghiệm của tôi trong việc cài Quay. Mong bạn đọc sẽ thấy bài viết hữu ích và giúp đỡ bạn làm quen nhanh chóng với Quay và xây dựng một registry trong homelab đơn giản và hiệu quả.

# Nguồn tham khảo
  - [Project Quay Documentation](https://docs.projectquay.io/welcome.html) / [archive](https://web.archive.org/web/20260919091034/https://docs.projectquay.io/welcome.html):
  - [Red Hat Quay Documentation](https://docs.redhat.com/en/documentation/red_hat_quay/3.18) / [archive](https://web.archive.org/web/20260919091907/https://docs.redhat.com/en/documentation/red_hat_quay/3.18)
  - [Red Hat Hardened Images](https://www.redhat.com/en/products/hardened-images) / [archive](https://web.archive.org/save/https://www.redhat.com/en/products/hardened-images)
  - [TechCrunch - Why AWS, Google and Oracle are backing the Valkey Redis fork](https://techcrunch.com/2024/03/31/why-aws-google-and-oracle-are-backing-the-valkey-redis-fork/) / [archive](https://web.archive.org/web/20260120044327/https://techcrunch.com/2024/03/31/why-aws-google-and-oracle-are-backing-the-valkey-redis-fork/)
  - [pg_dump - PostgreSQL 18 Documentation](https://www.postgresql.org/docs/18/app-pgdump.html) / [archive](https://web.archive.org/web/20260918185222/https://www.postgresql.org/docs/18/app-pgdump.html)
  - [Bash Coding Standard (BCS)](https://github.com/Open-Technology-Foundation/bash-coding-standard) / [archive](https://web.archive.org/web/20260925103119/https://github.com/Open-Technology-Foundation/bash-coding-standard)
    * [Atomic file write](https://github.com/Open-Technology-Foundation/bash-coding-standard/blob/main/docs/BCS-Bash-Ref/12_Signals-and-Traps/15_Atomic-file-write.md) / [archive](https://web.archive.org/save/https://github.com/Open-Technology-Foundation/bash-coding-standard/blob/main/docs/BCS-Bash-Ref/12_Signals-and-Traps/15_Atomic-file-write.md)
