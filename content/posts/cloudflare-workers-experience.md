---
title: "Trải nghiệm thiết kế web server sử dụng giải pháp của Cloudflare"
author: "Aperture"
date: "2024-01-23T14:14:00+07:00"
---

# Đặt vấn đề

Sau một thời gian dài, rất dài làm việc với AWS, Azure, Oracle Cloud, tôi chợt nhận ra tất cả những nhà cung cấp trên có một đặc điểm chung: Khá rẻ khi đang dev, giá vẫn còn gồng được được khi bắt đầu serve lượng nhỏ người dùng và khi phải chạy một hệ thống khủng để gánh một lượng lớn khách thì giá phải trả sẽ ở trên trời. Với một người có kinh tế eo hẹp, không được funding vốn từ bất kì đối tượng nào mà phải tự gồng gánh từ hạ tầng đến lương nhân viên, thì việc vỡ nợ trước khi sản phẩm ra tiền là viễn cảnh không hề xa.

{{< figure 
    src="/cloudflare-workers-experience/9999-up-time-fireship.png"
    alt="Khi bạn cố gắng xây hệ thống có uptime cao trên AWS"
    title="Khi bạn cố gắng xây hệ thống có uptime cao trên AWS"
    caption="Khi bạn cố gắng xây hệ thống có uptime cao trên AWS"
    attr="Fireship.io"
    attrlink="https://youtu.be/ZzI9JE0i6Lc?t=22"
    link="https://youtu.be/ZzI9JE0i6Lc?t=22">}}

Để đủ trang trải cho cả dự án, cũng như đảm bảo tính ổn định của hệ thống, bắt buộc tôi phải lựa chọn một giải pháp nào đó thoả mãn các tiêu chí sau:

* Giá rẻ: Tiền hạ tầng nên được cắt giảm để đầu tư vào những thứ khác có ích hơn, ví dụ như đồ để ăn, nước để uống, điện để sạc máy tính và thuốc lá để hút.
* Công sức vận hành thấp: Nếu phải dành quá 50-60% thời gian trong một ngày chỉ để theo dõi một hạ tầng đang sống hay chết trong khi nhân lực không đủ thì sẽ là quá phí phạm công sức
* Tương đối dễ sử dụng và vận hành: Tốn quá nhiều thời gian để xây dựng một hạ tầng, khắc phục sự cố cho nó thì cũng chẳng còn thời gian đâu để phát triển thêm tính năng với một đội ít người
* Đủ khoảng thở để phát triển: Không ai muốn code plugin cho WordPress cả, cũng chẳng ai muốn dùng những công nghệ cũ kĩ như Drupal hay Joomla, hoặc những thứ khổng lồ như Magento làm gì cho nhọc công.

Dựa vào nhưng tiêu chí ở trên, tôi có hai giải pháp như sau:

- AWS Lambda + API Gateway + RDS/Dynamo + S3: Giải pháp thường thấy bởi bất kì những người từ già trẻ lớn bé biết đến cloud service.
- Cloudflare Workers + Pages + D1 + R2: Giải pháp mới đưa ra của Cloudflare, gần tương tự với giải pháp của AWS.
- Heroku: Giải pháp sử dụng một web và nhiều background worker khá hay của Salesforces

Dưới đây là bảng so sánh các tính năng giữa các giải pháp

|            | Ưu điểm                                                                                                                                                                                                                                | Nhược điểm                                                                                                                                                                                                                                                                                                                                                                                              |
|------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AWS        | - Nền tảng rất phổ biến, nhiều người dùng và được chứng thực bởi nhiều khách hàng<br>- Được thiết kế để scale<br>- Nhiều khoảng thở để phát triển ứng dụng, vì AWS Lambda sử dụng môi trường Linux và hỗ trợ native rất nhiều ngôn ngữ | - Giá thành không hề rẻ, ngoài phải trả tiền cho các sản phẩm sử dụng, ta phải trả các khoản phí không tên khác như tiền băng thông, tiền VPC, vân vân                                                                                                                                                                                                                                                  |
| Cloudflare | - Giá rẻ như bùn<br>- Scale rất tốt với cold start thấp<br>- Hỗ trợ native một số framework fullstack như SvelteKit, NuxtJS                                                                                                            | - Chỉ hỗ trợ native JS/TS, muốn dùng ngôn ngữ khác phải compile ra WASM (rất khó và trúc trắc)<br>- Nền tảng Workers chạy trong sandbox Google V8 nên rất nhiều giới hạn về bộ nhớ, năng lực tính toán và tính năng thua xa AWS Lambda<br>- Các giải pháp xung quanh Workers (như Queues, KV và D1) khá khó dùng và có nhiều hạn chế<br>- Việc connect tới các dịch vụ sử dụng TCP gần như không hỗ trợ |
| Heroku     | - Thân thiện với người dùng, nhiều add-ons có sẵn<br>- Hỗ trợ nhiều ngôn ngữ, đặc biệt là cho phép dùng Docker                                                                                                                         | - Giá vẫn còn đắt (để có option 1GB RAM cần tới 50$ 1 tháng)<br>- Không có region nào ngoài US và EU M<br>- Tiền add-ons tính vào thì đắt cũng chẳng kém AWS                                                                                                                                                                                                                                                                                                    |

Tất nhiên, dù có nhiều nhược điểm nhưng vấn đề tiền bạc đã chiến thắng tất cả, tôi lựa chọn Cloudflare Workers. Giải pháp mà tôi lựa chọn là fullstack SvelteKit deploy trên Cloudflare Pages + Cloudflare Workers làm backend, D1 + KV làm database và R2 làm nơi lưu trữ dữ liệu.

# Những vấn đề gặp phải khi thiết kế

Tất nhiên, đã chọn CF Workers là xác định đối mặt với những nhược điểm của nó, và dưới đây là những vấn đề tôi gặp phải khi sử dụng giải pháp của Cloudflare

## Chỉ support JS/TS native, các ngôn ngữ khác phải build ra WASM
Việc này không hề thân thiện một chút nào với bất kì ai, do khi build ra như vậy khiến việc debug/testing gặp rất nhiều trở ngại. Hơn nữa không phải ngôn ngữ nào cũng có thể build ra WASM, mà chỉ có các ngôn ngữ compile mới có thể dùng được. Mặc dù theo lý thuyết ta có thể code hoặc compile các thư viện viết bằng Rust, C/C++ để tự tạo các thư viện mà trong môi trường Cloudflare Workers không có, nhưng TẠI SAO???

## Cloudflare D1 thực chất là SQLite và migration trên đó thực sự là thảm hoạ

SQLite là database nằm trên file, vậy nên phải xác định rằng nó có những giới hạn nhất định của nó. Giới hạn đầu tiên mà tôi đụng phải chính là việc thay đổi cấu trúc bảng trong nó. Việc sửa đổi bảng, ngoài việc thêm cột và thêm index, thực sự rất khó khăn, và đều đi đến một quyết định khá đau thương là tạo bảng mới với cấu trúc mới và drop bảng cũ đi. Việc này rất dễ dẫn sai sót và khiến việc migration trở nên lằng nhằng vô cùng.

Dù sao thì SQLite vẫn là DB chỉ dùng trong một node và dù có implement như nào đi chăng nữa thì nó vẫn là SQLite, hoặc những kĩ sư trong Cloudflare siêu thông minh và họ có thể implements các tính năng của D1 siêu đỉnh, đến mức có thể vượt qua những giới hạn trong thiết của SQLite như database-locking khi có transaction, thì việc đối mặt với vấn đề cấu trúc bảng vẫn là một ác mộng.

## Chạy các tác vụ liên quan đến crypto khá cùi

Cùi ở đây có hai điểm:

1. Môi trường chạy khá yếu, 128M RAM với 10ms CPU time (nếu trả 5$ để có thêm 40ms CPU time nữa) là tương đối ít. Nếu phải sử dụng Workers Unbound cho cả project thì lại phí. Do đó các tác vụ liên quan đến mã hoá/giải mã dữ liệu sẽ không đủ RAM và CPU time để chạy.
2. API WebCrypto hoá khá nhiều giới hạn, đặc biệt password hasing chỉ có PBKDF2 là support bởi Cloudflare Workers, và việc implements chúng vào code cũng không dễ dàng gì, [và đây là một ví dụ](https://timtaubert.de/blog/2015/05/implementing-a-pbkdf2-based-password-storage-scheme-for-firefox-os/)

Vậy nên nếu để sử dụng các thuật toán mã hoá BCrypt hay Argon2, bắt buộc phải gọi ra một service ngoài, ví dụ như một function Lambda hoặc Unbound worker đề offload việc sang, tạo thêm một infrastructure dependency không đáng có, khiến việc maintain hệ thống trở nên khó hơn. Hơn nữa kể cả khi đã chạy PBKDF2, việc hashing vẫn có nguy cơ hết RAM/CPU giữa chừng, gây ra khó chịu nhất định.

## Logging khá rối rắm

Khi bạn không tốn 5$ một tháng cho Cloudflare, tất cả những gì bạn có thể làm là dùng `wrangler tail` để xem log của Workers. Ngay cả khi sử dụng Cloudflare Logpush, mọi thứ chợt trở nên phức tạp phát khiếp so với việc mở AWS CloudWatch để xem log, [xem document này để thấy độ phức tạp](https://developers.cloudflare.com/logs/get-started/enable-destinations/elastic/)

## Không thể dùng nhiều package của NodeJS

Kể cả có bật `node_compat` lên thì vẫn sẽ có khả năng code của bạn sẽ không thể sử dụng kha khá các thư viện của NodeJS, phần vì không thể implements hết các API của NodeJS, phần vì giới hạn 1M file code của Cloudflare, khi webpack có thể tạo file lớn hơn giới hạn.

## Connect vào database sử dụng TCP khá giới hạn

Sẽ có lúc bạn cần connect vào các database ở ngoài để lấy dữ liệu từ chúng. Hiện tại chỉ có PostgreSQL là được hỗ trợ thông qua Hyperdrive, còn lại chúng ta không thể làm gì để connect vào MySQL, Redis, ngoại trừ việc phải sử dụng các bên có support endpoint HTTPS như Upstash hay PlanetScale và tất nhiên, giá sẽ cao vút nếu bạn bắt đầu phải phục vụ lượng người dùng lớn. 

# Tóm lại
Ở trên là những trải nghiệm của tôi khi sử dùng CF Workers. Dù nó có nhiều giới hạn, nhưng tiêu chí rẻ đã khiến tôi tin tưởng Cloudflare Workers. Mong là tôi sẽ không bỏ của chạy lấy người giữa như ông bạn [liftosaur](http://liftosaur.com) ở đây mà kịp sống dai và kiên nhẫn để đợi CF worker close beta.