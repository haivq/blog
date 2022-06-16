---
title: "Thiết kế Serverless Backend sử dụng Cloudflare Workers"
author: "Aperture"
date: "2022-06-16T15:29:17+07:00"
draft: true
---

# Đặt vấn đề

Với xu hướng hiện nay, khi làn sóng Server trên các nền tảng đám mây như AWS, Azure đã trở nên vô cùng phổ biến và bão hoà, các nền tảng quản lý và điều khiển như HashiStack, Kubernetes đã rất lớn mạnh và nổi tiếng, lâu dần xu hướng công nghệ đám mây này bắt đầu bộc lộ những vấn đề:

## Rất khó tối ưu chi phí vận hành
Mặc dù đã khắc phục được các vấn đề như scaling và giảm thiểu chi phí system administrating so với thời kì máy móc cổ điển, giúp giải phóng nhân công nhưng chi phí để vận hành máy móc vẫn khá cao. Trong trường hợp giả tưởng, một máy EC2 `m5.large` trên sẽ phục vụ được 1.000 khách hàng, khi thiết lập auto scaling, khi hệ thống đã tải tới 990 khách, AWS sẽ tạo thêm một máy `m5.large` thứ hai để đảm bảo hệ thống không bị quá tải. Nhưng vấn đề đặt ra là, khi có 1.001 khách hàng, ta vẫn phải tạo 2 máy `m5.large` để phân tải, do đó chi phí sẽ nhân lên gấp đôi trong khi hai máy ở trên không chạy hết công suất.

Do tính chất phân phối của số lượng khách hàng sử dụng sản phẩm có thể được minh hoạ bằng một quá trình Poisson [^1], nhưng do tính Memoryless, không thể hoàn toàn phụ thuộc vào quá trình này để tạo ra một hệ thống thông minh đến mức có thể dự đoán và chủ động scale up hệ thống.

Tóm lại, vấn đề về upfront cost của việc vận hành máy móc vẫn là một bài toán đau đầu cho người điều hành hệ thống, đặc biệt là với hệ thống cần máy móc tính toán hiệu năng cao, với chi phí upfront rất cao. Hệ thống Serverless sẽ khắc phục vấn đề này bằng cách đưa code thực thi vào một môi trường chung cùng với rất nhiều người khác, sử dụng các công nghệ cô lập môi trường (như AWS Lambda sử dụng CGroups, Oracle Function sử dụng Containerd) để đảm bảo môi trường thực thi độc lập với nhau.

## Tốn nhân công vận hành
Một hệ thống server sẽ luôn cần tới DevOps để quản lý và vận hành, nhất là khi hệ thống lớn và xuyên nhiều khu vực sẽ cần tới một đội DevOps duy trì độ ổn định. Đây sẽ là một chi phí nhân sự tương đối cao dành cho các doanh nghiệp lớn, đặc biệt là với các công ty vừa và nhỏ (nhất là các công ty startup), việc thuê một đội DevOps sẽ tốn một khoản lớn vào chi phí vốn đã hạn hẹp của các công ty này.

# Những lợi thế Serverless đối với Server truyền thống

Serverless có các đặc trưng như sau:
- Code sẽ được thực thi vào một hệ thống chung cùng với rất nhiều khách hàng khác, trong các môi trường cô lập [^2] [^3]
- Chỉ trả tiền dựa vào số lượng request
- Đối tượng cung cấp nền tảng sẽ chịu trách nhiệm vận hành hệ thống

Nhờ các yếu tố cơ bản trên, việc sử dụng nền tảng Serverless sẽ khắc phục được các vấn đề đã nêu ở trên như sau:

## Chí phí vận hành được tính chính xác dựa trên số khách hàng
So với server truyền thống, khi hệ thống có 990 khách hàng, nền tảng Serverless sẽ chỉ thu đúng lượng tiền tương ứng với các request do 990 khách hàng trên sinh ra, tương tự 1000 khách hay 2000 khách, nền tảng chỉ thu đúng lượng tương ứng. Do đó ta sẽ giảm thiểu được chi phí upfront khi scale out dịch vụ của mình. Hơn nữa, do hệ thống Serverless luôn bật, luôn sẵn sàng để chạy code do đó tốc độ scale sẽ rất nhanh, chứ không phải đợi vài phút để tạo máy ảo, khởi tạo môi trường như server truyền thống.

## Giải phóng nhân công và chi phí cho nhân công vận hành hệ thống
Do đối tượng cung cấp nền tảng đã chịu trách nhiệm vận hành hệ thống, ta sẽ không phải tốn tiền để thuê một số lượng lớn DevOps để vận hành hệ thống như server truyền thống nữa, qua đó giải phóng nhân công và giảm thiểu chi phí nhân công. Hệ thống thực thi Serverless được thiết kế và vận hành bởi các kĩ sư có kinh nghiệm, nên phần nào ta cũng giảm thiểu những vấn đề do lỗi con người gây ra.

# Nhược điểm của hệ thống Serverless so với server truyền thống

Tất nhiên không có một *viên đạn bạc* nào có thể giúp bạn xây dựng nên một hệ thống lớn mà chỉ tốn có vài xu lẻ, mà làm được mọi thứ bạn muốn mà không có downtime. Dưới đây là các nhược điểm mà của các nền tảng Serverless nói chung:

## Môi trường thực thi có các giới hạn
Nền tảng Serverless được tạo ra để phục vụ các nhu cầu phổ thông và cơ bản của khách hàng, vì vậy với những yêu cầu phức tạp và nâng cao từ phía người dùng, ví dụ như sử dụng GPU, sử dụng các thư viện yêu cầu đến tập lệnh CPU, sử dụng Serverless sẽ không phải là một ý tưởng hay

## Phụ thuộc vào đối tượng cung cấp nền tảng
Bạn sẽ không thể đơn giản mang code của mình từ AWS sang Azure Function hay Oracle Function mà không phải sửa nội dung. Mỗi một nền tảng sẽ có những đặc trưng riêng, việc để hệ thống của mình nằm trên một bên cung cấp dịch vụ sẽ là ví dụ điển hình cho một bad practice: "Cho hết trứng vào một giỏ". Một vấn đề xảy ra và hệ thống của bạn sẽ gặp downtime hay tệ hơn là mất dữ liệu. Ví dụ về sự kiện [sập hệ thống mạng tại vùng US-EAST-1 của Amazon vào 10/12/2021](https://aws.amazon.com/message/12721/), khiến một loạt các dịch vụ dựa trên AWS chết theo, là một bài học cho việc để hết dịch vụ tại một nơi. Càng sử dụng dịch vụ của một bên cung cấp càng lâu, sự phụ thuộc vào các dịch vụ của họ sẽ càng sâu sắc và khiến việc di chuyển khỏi nền tảng của bên cung cấp dịch vụ càng khó khăn hơn

# Chú thích
- [^1] [The Poisson Distribution and Poisson Process Explained](https://towardsdatascience.com/the-poisson-distribution-and-poisson-process-explained-4e2cb17d459)
- [^2] [Lambda Isolation Technologies](https://docs.aws.amazon.com/whitepapers/latest/security-overview-aws-lambda/lambda-isolation-technologies.html)
- [^3] [Isolates | How Workers works](https://developers.cloudflare.com/workers/learning/how-workers-works/#isolates)

# Nguồn tham khảo
- [Why use serverless computing? | Pros and cons of serverless](https://www.cloudflare.com/en-gb/learning/serverless/why-use-serverless/)
- [AWS Lambda execution environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)