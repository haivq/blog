---
title: "Tạo một cluster Kubernetes đơn giản trên PC, sử dụng k3s và Rancher 2.5"
author: "Aperture"
date: 2021-10-04T00:48:26+07:00
categories:
   - kubernetes
   - rancher
   - docker
   - development
tags:
   - k8s
   - k3s
   - rancher
   - kubernetes
   - docker
   - self hosted
   - pc
   - development
---

Kubernetes gần đây đã trở thành một công nghệ nổi tiếng. Không chỉ gắn liên với việc quản lý các cluster lớn, rất nhiều framework hay các stack công nghệ bây giờ đều hỗ trợ k8s (điển hình nhất là kubeflow, nginx). Việc học k8s gần như không còn là môt lựa chọn mà trở thành một yêu cầu quan trọng cho nhiều công việc. Những ai đã/đang sử dụng k8s đều phải công nhận đây là một công nghệ rất mạnh và mềm dẻo, nhưng kèm theo đó là một vài nhược điểm cố hữu: Rất khó sử dụng! Với những dev sử dụng máy tính thông thường, việc điều khiển một cluster k8s thực sự là một cực hình do phải điều khiển nó thông quan kubectl - một chương trình cli có quá nhiều thứ phải nhớ. Ngoài ra để deploy bất kì thứ gì lên đó, chúng ta phải viết một cái file yml dài ngoằng khó hiểu, liệt kê đầy đủ tất cả những gì cần phải có để một chương trình có thể chạy. Và tất nhiên k8s mặc định hoàn toàn không có giao diện. Kubernetes Dashboard (UI chính thức của k8s) cuối cùng vẫn bắt bạn phải viết yml. Rất nhiều config lưu trong k8s không thể lưu chính xác, và kết quả bạn lại phải viết thêm yml để phục vụ các config đó. Vậy nên yêu cầu của chúng ta khi deploy một cluster k8s ở máy bàn cá nhân cần phải thoả mãn các tiêu chí sau:

   * Có giao diện, dễ sử dụng
   * Hiệu năng phải cao
   * Không viết YML khi cần thiết

# Tìm đường để giải quyết vấn đề:

Để đáp ứng nhu cầu thứ nhất và thứ ba, chúng ta sẽ sử dụng Rancher - một phần mềm tạo ra một giao diện điều khiển cho k8s rất dễ sử dụng. Nhưng ở nhu cầu thứ 2, chúng ta cần liệt kê ra một vài ứng cử viên:

---

{{< figure 
    src="https://64.media.tumblr.com/cda8d40d9e3d13ed6907eb68021e295b/1d4062b521d6dee3-e3/s540x810/c6a421fe01e699d2babd7466a2e945edb7b00422.png"
    style="width:300px"
    position="center"
    alt="MicroK8s"
    title="MicroK8s"
    caption="MicroK8s"
    attr="MicroK8s"
    attrlink="https://microk8s.io"
    link="https://microk8s.io" >}}

## microk8s

Một phiên bản k8s được xuất bản và hỗ trợ bởi Canonical (Hãng phát triển Ubuntu). Bạn có thể đã gặp dòng chữ microk8s khá nhiều lần trong dòng “message of the day” lúc ssh vào bất kì server Ubuntu nào. Ta sẽ phân tích một vài ưu điểm và nhược điểm của nó:

Ưu điểm:
   * Cài đặt đơn giản
   * Tối ưu rất tốt cho Ubuntu
   * Đầy đủ tính năng của k8s
   * Các tính năng của k8s có thể bật/tắt theo nhu cầu, dưới dạng plugin
   * Không chạy trên máy ảo

Nhược điểm:

   * Tắt đi bật lại rất lâu và rất dễ lỗi (nhất khi tắt máy đi bật lại), kết quả xấu nhất là bạn không thể kết nối vào k8s và kết quả là phải xoá đi cài lại
   * Tối ưu tốt cho Ubuntu nhưng chưa chắc đã tối ưu tốt cho các OS còn lại

Nếu bạn đã làm quen được với microk8s, tôi nghĩ bạn có thể tự cài đặt Rancher trên này được rồi.

---

{{< figure 
    src="https://64.media.tumblr.com/7ec7f76ccd06eeb660d7259e352a7139/1d4062b521d6dee3-3d/s540x810/6e819c7eaf034881af92e9fdc5429beb356fd8a0.png"
    style="width:300px"
    position="center"
    alt="minikube"
    title="minikube"
    caption="minikube"
    attr="minikube"
    attrlink="https://minikube.sigs.k8s.io/docs/"
    link="https://minikube.sigs.k8s.io/docs/" >}}

## minikube

Khi bạn tìm hiểu về k8s, đọc hướng dẫn trên trang chủ của k8s, tutorial trên trang này ngay lập tức mời bạn cài đặt minikube. Vậy nên tôi nghĩ đây nhiều người đã từng thử cài và tìm hiểu k8s qua nó rồi.

Ưu điểm:

   * Cài đặt rất dễ, chạy được cả trên Windows, macOS.
   * Tắt bật rất nhanh, dễ xoá đi cài lại
   * Do chạy trên máy ảo nên rất dễ giới hạn tài nguyên

Nhược điểm:

   * Chạy trên máy ảo loại 2 (type 2 Hypervisor) VirtualBox nên hiệu năng sẽ không được cao như chạy trên máy thật hay type 1 Hypervisor
   * Để lấy được toàn bộ tính năng của k8s (như nvidia-gpu), yêu cầu người dùng phải cài đặt các loại addons trên cả k8s và máy ảo
   * Bật tắt các tính năng không tiện được như microk8s, thành ra vẫn phải setup khá lằng nhằng
   * Chiếm dụng tài nguyên phần cứng kể cả khi không sử dụng

Vậy nên theo tôi thì minikube không tối ưu để chạy thực tế vì thời gian setup tưởng nhanh mà đổi lại hiệu suất có thể không như ý muốn.

---

{{< figure 
    src="https://64.media.tumblr.com/344d71c978037162fd6bff6195ab8cb1/1d4062b521d6dee3-9c/s540x810/1dbbc8b8600d0404a895ca8e25e64f9bea5a171e.png"
    style="width:300px"
    position="center"
    alt="kubeadm"
    title="kubeadm"
    caption="kubeadm"
    attr="kubeadm"
    attrlink="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/"
    link="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/" >}}

## kubeadm

Công cụ được phát hành bởi chính k8s, dùng để tạo cluster k8s. Đây là cách khó nhất để tạo một cluster k8s, và cũng là chương trình duy nhất giúp bạn tạo ra một cluster k8s đúng theo sơ đồ thiết kế mà k8s đã đề ra.

Ưu điểm:

   * Thực sự tạo ra một cluster k8s chuẩn theo thiết kế đề xuất, có thể dùng trực tiếp trong môi trường production
   * Không sử dụng máy ảo

Nhược điểm:

   * Siêu khó sử dụng
   * Rất cồng kềnh và nặng nề nếu cài đặt hết lên 1 máy (theo thiết kế của k8s, chúng ta sẽ có các node controlplane chỉ để điều khiển, các node worker chỉ để chạy job và hàng loạt node etcd để lưu trữ các loại thông tin config)

Tóm lại, kubeadm rất tốt, nhưng hoàn toàn không phù hợp với ai không phải là DevOps hay SysAdmin (mà kể cả 2 đối tượng trên không chắc đã sử dụng được kubeadm)

---

{{< figure 
    src="https://64.media.tumblr.com/73706a3d1285b5d142bed47c7e673059/1d4062b521d6dee3-02/s540x810/831f6af1ff940001c2fee90c8714fbe93acb55e8.png"
    position="center"
    style="width:300px"
    alt="k3s"
    title="k3s"
    caption="k3s"
    attr="k3s"
    attrlink="https://k3s.io"
    link="https://k3s.io" >}}

## k3s

Công cụ được phát hành bởi chính Rancher. Đây là một bản k8s được rút gọn đi rất nhiều thứ, rất nhẹ và thậm chí có thể chạy ổn trên các con Pi. Dù vậy nhưng nó vẫn đạt các yêu cầu tính năng mà k8s yêu cầu.

Ưu điểm:

   * Cài đặt rất nhanh
   * Rất nhẹ, không tốn tài nguyên khi vận hành
   * Không chạy trên máy ảo

Nhược điểm
   * Một vài thành phần trong k8s bị cắt bỏ (như `etcd`, cloud storage)

Trong hướng dẫn này, tôi sẽ lựa chọn k3s vì nó phù hợp với yêu cầu của tôi: nhẹ, cài đặt nhanh, hiệu năng cao. Mọi người có thể chọn các distro khác nếu muốn.

# Cài đặt k8s và Rancher

> Mặc định k3s sẽ sử dụng traefik để làm ingress, nhưng tôi sẽ sử dụng `nginx` do thói quen.
> Bài viết này giả định tình huống máy của bạn chưa cài bất kì phần mềm nào liên quan đến k8s. Vậy nên nếu bạn đang sử dụng phần mềm nào (ví dụ như `kubectl`), hãy để ý tới các mục 2, 3 của hướng dẫn vì nó sẽ GHI ĐÈ lên file config hiện có, có thể khiến bạn có thể mất quyền truy cập vào server đang điều khiển hiện tại.

1. Cài đặt k3s và helm:

```bash
curl -sfL https://get.k3s.io | sh -s - server --no-deploy traefik
sudo snap install helm --classic
```
Bỏ tùy chọn `--no-deploy traefik` nếu bạn thích traefik làm ingress hơn. Do tôi đã khá quen với NGINX nên tôi sẽ cài đặt NGINX làm ingress sau.

Trong ví dụ này tôi sử dụng snapcraft để tải helm. Bạn có thể tải nó trực tiếp trên trang chủ của Helm nếu muốn.

2. Copy file config của kubernetes:

> LƯU Ý: Hành động này sẽ ghi đè lên file config cùng tên hiện tại, hãy để tâm!

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/local_config
```

Để đảm bảo user của bạn có thể sử dụng được file config này, chạy lệnh sau:

```bash
sudo chown $USER: ~/.kube/local_config
```

Để kiểm tra độ sẵn sàng của cluster, sử dụng các lệnh sau:

```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get pods --all-namespaces
```

3. Thêm config vào file .bashrc để có thể dùng kubectl:

Thêm dòng sau vào file config .bashrc:
```bash
export KUBECONFIG=~/.kube/local_config
# Lệnh dưới này để update môi trường
source ~/.bashrc
```

Nhưng nếu bạn chỉ muốn sử dụng nó tạm thời, chỉ cần chạy lệnh sau mỗi khi mở một session terminal/SSH mới:
```bash
export KUBECONFIG=~/.kube/local_config
```

4. Thêm các repository cần thiết vào Helm:

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
# sau đó cập nhật helm
helm repo update
```

5. Tạo các namespace cần thiết cho Rancher:

```bash
kubectl create namespace cattle-system
kubectl create namespace cert-manager
kubectl create namespace ingress-nginx
```

6. Cài đặt cert-manager:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.crds.yaml
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.0.4
```

Kiểm tra tình trạng của cert-manager thông qua lệnh sau:
```bash
kubectl get pods --namespace cert-manager
```

7. Cài đặt ingress-nginx cho k8s:
```bash
helm install \
  ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.watchIngressWithoutClass=true
```

Kiểm tra tình trạng của ingress bằng lệnh sau:
```bash
kubectl get pods --namespace ingress-nginx
```

8. Cuối cùng, cài Rancher:

LƯU Ý: cần phải có một domain để cho Rancher hoạt động.  Vì vậy dẫn tới 2 cách cài đặt sau đây:

* Cách 1: Có domain thật:
Lúc này còn gì tuyệt vời hơn khi bạn có thể tạo certificate TLS/HTTPS chuẩn chỉ thông qua ACME của Let's Encrypt. Bạn có thể dùng câu lện dưới đây, thay mục domain của bạn thành domain mà bạn muốn. Nhưng nếu bạn không thích dùng tên miền của mình để host rancher, bạn có thể chuyển sang cách 2.

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname={domain của bạn} \
  --set ingress.tls.source=letsEncrypt \
  --set replicas=1
```  
* Cách 2: Không có domain thật:

Bây giờ bạn không thể xác thực một certificate sử dụng ACME của Let's Encrypt được nữa. Chỉ còn cách sử dụng một certificate fake trên một domain local. Chúng ta không thể sử dụng trực tiếp tên miền localhost để host rancher được, cách tốt nhất là dùng một domain giả define trong file hosts.

Thêm dòng sau vào trong file hosts:
```bash
127.0.0.1 rancher.local
```

Cuối cùng sử dụng lệnh sau để cài rancher sử dụng certificate fake của Rancher:
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local \
  --set replicas=1
```

Sau khi chạy 1 trong 2 cách, chạy lệnh sau để check rancher đã deploy xong hay chưa. 

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

Quá trình deploy sẽ dài từ 5 - 10 phút, đặc biệt nếu bạn chọn cách 1. Hãy mở trình duyệt và vào rancher rồi config nó theo ý bạn!

## Tóm lại:
Mong qua bài viết này, bạn có thể tự cài được rancher trên máy tình của mình và sử dụng nó trong stack công nghệ hàng ngày của bạn.

## Các nguồn tham khảo:
   * https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/
   * https://rancher.com/docs/k3s/latest/en/installation/install-options/
   * https://kubernetes.github.io/ingress-nginx/deploy/#using-helm