---
title: "Những lý do để sử dụng Cloudflare"
author: "Aperture"
date: "2021-11-21T13:57:17+07:00"
categories:
    - cloudflare
    - cdn
    - ddos
tags:
    - cloudflare
    - cf
    - cloudflare cdn
    - cf cdn
    - cdn
    - load balancer
    - load balancing
    - ddos
---
Hiện tại như thiết kế điển hình của các hệ thống nằm trên AWS, Azure, ta đều có Load Balancer đứng trước chúng, không cho phép traffic chảy trực tiếp vào server. Như vậy, các server nằm phía sau tương đối an toàn khỏi các cuộc tấn công liên quan đến viêc lộ địa chỉ IP của server. Nhưng việc sử dụng Cloudflare sẽ giúp được các vấn đề sau đây:

# Tiết kiệm chí phí outbound traffic

Các dịch vụ Load Balancer luôn tính tiền thông qua các tiêu chí sau:

  * Số lượng request
  * Số lượng connection
  * Băng thông sử dụng

Ví dụ đây là thông tin giá trên AWS: [Elastic Load Balancing pricing](https://aws.amazon.com/elasticloadbalancing/pricing/)

Cloudflare là dịch vụ DNS sẽ kiêm luôn việc request, tránh việc tốn quá nhiều chi phí traffic, giúp giảm thiểu ba yếu tố trên.

Tất nhiên sử dụng Cloudflare sẽ không phủ định mức độ quan trọng của CDN ngoài, do Cloudflare tối đa chỉ cache được file có dung lượng tối đa 512MB (với gói dưới Enterprise), nên Cloudflare chỉ tốt nhất khi dùng cho các dịch vụ như load HTML, API, còn các dịch vụ serve các file lớn, ta phải dùng các dịch vụ như Akamai, Azure CDN hoặc Amazon S3. Bằng chứng cho việc này, Steam và Epic Games sử dụng Akamai làm CDN để tải game cho người dùng, còn Cloudflare dùng để serve API và các nội dung Web

# Chống các cuộc tấn công qua mạng

Cloudflare sẽ là đối tượng đứng trước để chịu tải, đồng thời giấu IP của các dịch vụ mà chúng ta cần bảo mật, vậy nên khi bị các cuộc tấn công qua mạng, Cloudflare sẽ là đối tượng đầu tiên đứng ra lọc các request.

Mặc dù Load Balancer đã đảm bảo cho ta rằng các địa chỉ IP của các máy sẽ luôn kín đáo, nhưng các cuộc tấn công sẽ khiến ta phải trả một khoản tiền lớn cho Load Balancer. Cloudflare phần nào cũng sẽ giúp chúng ta giảm các cuộc tấn công, qua đó tiết kiệm kinh phí cho hạ tầng mạng.

# Cung cấp giao diện theo dõi traffic

Cloudflare cung cấp giao diện theo dõi traffic rất đầy đủ. Theo dõi qua dashboard này sẽ tiện hơn nhiều thay vì việc phải xem nhiều dashboard ở nhiều cloud provider. Dashboard cung cấp đủ thông tin về phần trăm data được cache, số lượng request, các quốc gia request và thậm chí là bản đồ tấn công của bot, vân vân.

# Giá thành rẻ, nhiều chức năng hay

Cloudflare có giá rất rẻ so với tính năng mà nó mang lại, đặc biệt là không tính tiền traffic và băng thông. Hơn nữa, nó còn rất nhiều các tính năng tăng tốc tốc độ truy cập, ví dụ như HTTP/3, TCP Boost. Với giá thành rẻ, (20$ cho gói Pro và 200$ cho gói Business) thì không gì có thể so sánh với p/p của Cloudflare cả.

# Tổng kết

Tóm lại, hoặc chúng ta sẽ tốn công rất nhiều công sức để setup tất cả những tối ưu đã nêu ở trên bằng các dịch vụ cả AWS hoặc Azure với giá thành rất cao hơn và cài đặt sẽ phức tạp hơn rất nhiều. Nhưng tại sao lại không đơn giản là dùng Cloudflare và để họ tối ưu tất cả giúp cho chúng ta và để thời gian quý hoá của mình để làm những việc có ích hơn? Vậy nên làm ơn đừng khoan vào lỗ tai người làm hệ thống đã có nhiều kinh nghiệm những câu rằng: "Chú phải cho anh lý do vì sao lại dùng Cloudflare" hay "Mày phải phân tích kĩ lưỡng cho anh thiệt hơn khi sử dụng Cloudflare" sau khi họ đã tốn rất nhiều công phân tích. Rất có khả năng họ sẽ mệt mỏi và không đủ tỉnh táo để đưa ra những câu trả lời đầy đủ lý lẽ và phang luôn câu "Tôi thích, được chưa?" để kết thúc nhanh cuộc nói chuyện.

Đối với người làm hệ thống sắp tới, hãy đưa họ bài viết này để họ ngừng đưa ra những câu hỏi sẽ mất vô hạn thời gian để trả lời.