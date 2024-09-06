---
title: "Quản lý tài nguyên cho cluster Kubernetes"
author: "Aperture"
date: 2021-10-04T00:46:21+07:00
categories:
    - docker
    - kubernetes
tags:
    - k8s
    - docker
    - kubernetes
    - resource management
    - experience
    - kubectl
    - dev
    - development
---

# Đặt vấn đề

Một trong những thiếu sót khi sử dụng k8s trong môi trường production là không thiết đặt giới hạn tài nguyên cho hệ thống. Khi bạn không giới hạn tài nguyên cho các pod trong k8s, không sớm thì muộn sẽ có một thời điểm server của bạn sẽ hết sạch tài nguyên (điển hình là hết CPU và RAM). Việc này rất dễ xảy ra chạy các workload nặng trên nhiều node có cấu hình khác nhau. Cuối cùng server của bạn sẽ crash, hoặc là chạy rất chậm, khiến hệ thống trở nên kém ổn định, thậm chí là mất mát dữ liệu, gây tổn thất về uy tín và tiền bạc. Vậy một trong những việc đầu tiên khi một đưa bất kì thứ gì lên k8s là phải thiết đặt tài nguyên cho nó.

# Ý tưởng về cấp phát và giới hạn tài nguyên

Mặc định k8s chỉ có thể kiểm soát tài nguyên về CPU và RAM của hệ thống. Component chịu trách nhiệm kiểm soát tài nguyên và deploy pod lên các node được gọi là `kube-scheduler`. Có hai thuộc tính quan trọng để cấp phát và giới hạn tài nguyên được định nghĩa lúc bạn tạo workload:

 - Requests: Bảo đảm về lượng tài nguyên mà một pod trong workload sẽ được cấp. Mặc định các thuộc tính này sẽ để trống hoặc lấy theo yêu cầu mặc định của namespace (sẽ được đề cập sau). Tài nguyên request tối đa phải nhỏ hơn lượng tài nguyên mà một server mạnh nhất trong cluster có thể tải được.
	 - Ví dụ: `request: 100mCPU, 256MiB RAM` mang ý nghĩa bạn sẽ luôn đảm bảo rằng k8s sẽ cấp cho bạn ít nhất 100 mCPU và 256MiB RAM. Hệ thống sẽ chỉ deploy pod của bạn trên node có đủ số lượng tài nguyên như trên. Tất nhiên `kube-scheduler` có đánh dấu lượng tài nguyên này đã bị chiếm dụng.
	 - Giả sử bạn có 2 node, node 1 có 2 vCore và 8GiB RAM, node 2 có 4 vCore 16GiB RAM, nếu workload của bạn yêu cầu `request: 2500mCPU, 8GiB RAM` thì server sẽ chỉ deploy pod của workload này vào node 2.
	 - Trong trường hợp cũng với cấu hình trên mà bạn yêu cầu tài nguyên `request: 4000mCPU, 8GiB RAM`, sẽ không có pod nào được deploy cả.
 - Limits: Giới hạn lượng tài nguyên mà một pod trong workload sẽ được sử dụng
	 - Ví dụ: `limits: 1000mCPU, 2GiB RAM` mang ý nghĩa rằng bạn chỉ có thể dùng tối đa 1 CPU và 2GiB RAM.

Để thiết đặt chúng khi tạo một deployment, đây là YML ví dụ tạo deployment `ubuntu` (chú ý phần `resources` trong file YML):

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ubuntu-test
  labels:
    app: ubuntu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubuntu
  template:
    metadata:
      labels:
        app: ubuntu
    spec:
      containers:
      - name: ubuntu
        image: ubuntu:bionic
        ports:
        - containerPort: 80
        # chú ý mục resources này
        resources:
          limits:
            cpu: 1
            memory: 2Gi
    	  requests:
    	    cpu: 500
    	    memory: 2Gi
```

Bạn để ý ở mục `spec.template.spec.containers[0].resource`, sẽ thấy có hai setting là limits và requests. Đây chính là nơi thay đổi về lượng tài nguyên đảm bảo và giới hạn.

Qua 2 ví dụ trên, cần phải lưu ý hai điều sau:
1. k8s hoàn toàn có thể deploy pod của bạn vào một node mà lượng tài nguyên còn lại ít hơn lượng tài nguyên giới hạn. Ví dụ k8s sẽ có thể deploy workload `request: 100 mCPU, 256MiB RAM`, `limits: 1000mCPU, 2GiB RAM` vào một server chỉ còn trống 700 mCPU và 1 GiB RAM. Vậy nên nếu nếu workload của bạn yêu cầu nhiều tài nguyên hơn thì bạn cần chú ý cấp phát thêm tài nguyên cho chúng, vì trong trường hợp xấu, một pod dùng quá số lượng tài nguyên request mà node đã hết tài nguyên, node sẽ crash và buộc k8s phải kill một số pod khác hoặc tệ hơn là cả node đó sẽ không thể nào truy cập được nữa. Cơ chế kill pod đòi lại tài nguyên của k8s sẽ được đề cập ở mục dưới.
2. Kể cả khi bạn không dùng thì một pod cũng đã k8s đã tính pod của bạn đã chiếm dụng lượng tài nguyên bằng với lượng tài nguyên yêu cầu.
3. Tính chất tài nguyên CPU và RAM là khác nhau. Nếu pod của bạn vượt quá lượng tài nguyên CPU, k8s có thể "hãm" pod của bạn lại và không cho nó vượt quá giới hạn. Nhưng tài nguyên về bộ nhớ không thể bị giới hạn như vậy. Ngay khi pod của bạn vượt quá tài nguyên RAM cho phép, k8s sẽ kill luôn pod đó. Vậy nên cần chú ý về yêu cầu bộ nhớ của workload để tránh trường hợp pod bị kill ngoài ý muốn.
4. Việc deploy pod vào các node còn phải tùy việc server đó còn bao nhiêu tài nguyên (khá hiển nhiên nhưng vẫn phải đề cập). Giả sử một node có 4 vCore và 16GiB RAM nhưng đã bị nhiều pod chiếm dụng mất 3 vCore thì khi bạn request lượng tài nguyên `request: 2500mCPU, 8GiB RAM` thì pod của bạn cũng sẽ không bao giờ được deploy lên đó.

# Cơ chế deploy và kill của `kube-scheduler`

Nếu bạn tạo một cluster k8s từ các công cụ như k3s, kubeadm hay từ các bên cung cấp nền tảng như AKS (của Azure), EKS (của Amazon) thì mặc định cluster đó sẽ sử dụng `kube-scheduler` để kiểm soát việc quản lý tài nguyên và deploy pod. Việc deploy hay kill pod của `kube-scheduler` đều hoạt động theo hai bước: **Lọc** và **Chấm điểm**.

Trường hợp k8s deploy một pod vào cluster:

 - Bước **Lọc**: `kube-scheduler` sẽ liệt kê tất cả các node thỏa mãn điều kiện tối thiểu của workload (tức thỏa mãn lượng tài nguyên `request`). Nếu trong danh sách đó không có node nào, pod của bạn sẽ không bao giờ được deploy. Các pod chưa được deploy vẫn sẽ có cơ hội được chạy do `kube-scheduler` sẽ thực hiện việc chấm điểm này liên tục.
 - Bước **Chấm điểm**: `kube-scheduler` sẽ đánh giá các node thông qua nhiều tiêu chí khác nhau, từ đó đưa ra điểm số của node đó. `kube-scheduler` sẽ deploy vào node có điểm số cao nhất. Nếu có nhiều hơn 1 node có cùng một điểm số, pod sẽ được deploy ngẫu nhiên vào một trong các node đó.

Còn trường hợp k8s cần kill pod để thu hồi lại tài nguyên:

 - Bước **Lọc**: `kube-scheduler` sẽ liệt kê tất cả các pod đang hoạt động trong node đang bị quá tải.
 - Bước **Chấm điểm**: `kube-scheduler` sẽ đánh giá các pod đó thông qua độ ưu tiên của pod ([Pod Priority](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)). Pod nào có điểm ưu tiên thấp hơn sẽ bị kill, các pod có điểm ưu tiên ngang nhau sẽ được kill ngẫu nhiên. Việc này sẽ lặp đi lặp lại đến khi server đủ tài nguyên thì thôi. Nhưng thông thường chúng ta thường bỏ quên mất tính năng này, nên các pod sẽ có điểm ưu tiên ngang nhau, vì vậy các pod sẽ bị chấm điểm thông qua lượng tài nguyên sử dụng. Các pod vượt quá tài nguyên yêu cầu càng nhiều, thì pod đó càng có khả năng bị kill. Việc này cũng sẽ lặp đi lặp lại đến khi server đủ tài nguyên thì thôi.
 - Trong trường hợp bạn có đặt mức độ ưu tiên cho pod, nếu bạn đặt cho pod của mình có độ ưu tiên cao hơn các pod hệ thống như `kubelet`, k8s có thể kill luôn các pod đó để thu hồi bộ nhớ. Tất nhiên việc này vừa có lợi điểm và hại điểm.
	 - Điểm tốt: Node của bạn vẫn sẽ chạy và mọi thứ sẽ được deploy trở lại khi pod của bạn trả lại tài nguyên cho hệ thống.
	 - Hại điểm: Khiến bạn lo lắng khi node được thông báo là đã crash và không có thông tin nào được cập nhật về. Tệ hơn nữa, node fail quá lâu sẽ khiến cho hệ thống bên thứ ba tưởng node của bạn đã sập (node tained), node sẽ bị xoá đi và thay thế bằng node mới, mất toàn bộ những gì mà node của bạn đang thực hiện  (ví dụ như GKE autoscaler sẽ thay node đang sập bằng node mới sau một khoảng thời gian).

Việc **Lọc** và **Chấm điểm** sẽ được định đoạt bằng một trong hai quy cách: [Thông qua các quy chế đã quy định (Policies)](https://kubernetes.io/docs/reference/scheduling/policies/) hoặc [thông qua các profile quy chế (Profiles)](https://kubernetes.io/docs/reference/scheduling/config/#profiles) nhưng trong phạm vi bài viết này, chúng ta sẽ không đề cập kĩ đến hai quy cách phức tạp trên mà sẽ xoáy vào cơ chế tài nguyên CPU và RAM.

# Phân chia tài nguyên của cluster cho các namespace

Việc phân chia tài nguyên cho namespace (Resource Quota) được coi là **TỐI QUAN TRỌNG** đối với bất kì một người làm hệ thống nào. Thường một hệ thống sử dụng k8s sẽ không chỉ dành cho một mình DevOps hay SysAdmin sử dụng, mà sẽ được chia ra cho mỗi team (hiện tại đang trong một project) nắm một hoặc một vài namespace. Họ hoàn toàn có thể cung cấp quá ít tài nguyên cho workload để pod có thể hoạt động, hoặc đặt tài nguyên giới hạn quá cao khiến chúng chiếm hết tài nguyên hệ thống, vân vân. Tất cả đều sẽ dẫn đến một kết cục cuối cùng: sập server. Vậy nên với góc độ là người làm hệ thống, bạn cần phải phân chia tài nguyên của các namespace lại để đảm bảo server không bị quá tải. Khi họ vượt quá lượng tài nguyên yêu cầu, các pod nằm trong namespace sẽ bị kill nhưng sẽ không ảnh hưởng 

Ý tưởng của việc phân chia tài nguyên cũng rất đơn giản như sau:

 - Mặc định các namespace sẽ không được định nghĩa gì về phân chia tài nguyên, các pod trong namespace sẽ thoải mái đặt ra bất kì yêu cầu tài nguyên nào mà mình muốn.
 - Khi thiết đặt phân chia tài nguyên, các pod trong namespace đó chỉ có thể yêu cầu hoặc giới hạn lượng tài nguyên nhỏ hơn hoặc bằng số lượng tài nguyên phân chia, tương tự như cấp phát và giới hạn tài nguyên của pod, cũng có các thông số như sau:
	 - Request: Tổng lượng tài nguyên yêu cầu mà các pod có thể sử dụng khi deploy vào namespace. Vượt quá lượng tài nguyên trên pod sẽ không thể deploy
		 - Ví dụ: Một namespace được cấp phát `request: 4CPU, 8GiB RAM` có ý nghĩa tổng tất cả lượng tài nguyên request của các pod phải nhỏ hơn hoặc bằng 4CPU và 8192MiB RAM. Ví dụ bạn có thể deploy 4 pod yêu cầu `request: 1CPU, 2GiB RAM`, 2 pod yêu cầu `request: 2CPU, 4GiB RAM` hoặc một pod yêu cầu `request: 4CPU, 8GiB RAM`
	 - Limits: Tổng lượng tài nguyên giới hạn mà các pod có thể đạt được trong namespace. Các pod khi chạy mà tổng vượt quá giới hạn này, chúng sẽ bị kill theo [cơ chế đã được nêu ra ở mục trên](#killdeploymechanism).

Qua đây chúng ta có thể đảm bảo rằng, khi một team chẳng may gây ra sự cố ở namespace của họ, tất cả hệ thống vẫn sẽ chạy bình thường chứ không hề gây ra sự cố hỏng hóc gì cho toàn hệ thống. Đặc biệt trong các cluster dev, việc này là quan trọng để tránh một team sẽ phá hỏng tiến độ làm việc cho các team khác. Việc thiết đặt cấp phát tài nguyên này cũng là một phương pháp để các team có thể ước lượng được lượng tài nguyên họ sẽ sử dụng rồi đưa những thông số này vào áp dùng lên product.

# Kết thúc bài viết và các tài liệu tham khảo

Mong rằng thông qua bài viết này, mọi người có thể hiểu được cơ chế quản lý và deploy, kill pod của `kube-scheduler`, cũng như sự quan trọng của việc giới hạn và cấp phát tài nguyên trong hệ thống.

Các tài liệu tham khảo:

- [Setting Resource Requests and Limits in Kubernetes](https://www.youtube.com/watch?v=xjpHggHKm78)
- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)