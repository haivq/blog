---
title: "Trải nghiệm thiết kế web server sử dụng giải pháp của Cloudflare"
author: "Aperture"
date: "2024-01-23T14:14:00+07:00"
categories:
    - cloudflare
    - serverless
    - cloudflare workers
    - cloudflare pages
tags:
    - cloudflare
    - cloudflare workers
    - cloudflare pages
    - cloudflare d1
    - cloudflare kv
    - cloudflare r2
    - cloudflare cdn
    - wrangler
    - cdn
    - system design
    - edge computing
    - serverless
    - svelte
    - sveltekit
    - optimization
    - experience
    - scaling
    - cloud
---

# Đặt vấn đề

Sau một thời gian dài, rất dài làm việc với AWS, Azure, Oracle Cloud, tôi chợt nhận ra tất cả những nhà cung cấp trên có một đặc điểm chung như sau:
- Khá rẻ khi đang dev
- Giá tăng theo cấp số nhân khi bắt đầu có khách sử dụng, do thiết kế phải sử dụng rất nhiều các dịch vụ built in của nền tảng
- Vẫn còn gồng được được khi bắt đầu serve lượng nhỏ người dùng nhưng khi phải gánh một hệ thống khủng để gánh một lượng lớn khách thì giá sẽ cao vọt lên.

Với một người có kinh tế eo hẹp, không được funding vốn từ bất kì đối tượng nào mà phải tự gồng gánh từ hạ tầng đến lương nhân viên, thì việc vỡ nợ trước khi sản phẩm ra tiền là viễn cảnh không hề xa.

{{< figure 
    src="/posts/cloudflare-workers-experience/9999-up-time-fireship.png"
    position="center"
    alt="Khi bạn cố gắng xây hệ thống có uptime cao trên AWS"
    caption="Khi bạn cố gắng xây hệ thống có uptime cao trên AWS"
    attr="Fireship.io"
    attrlink="https://youtu.be/ZzI9JE0i6Lc?t=22"
    link="https://youtu.be/ZzI9JE0i6Lc?t=22">}}

Để đủ trang trải cho cả dự án, cũng như đảm bảo tính ổn định của hệ thống, bắt buộc tôi phải lựa chọn một giải pháp nào đó thoả mãn các tiêu chí sau:

* Giá rẻ: Tiền hạ tầng nên được cắt giảm để đầu tư vào những thứ khác có ích hơn, ví dụ như đồ để ăn, nước để uống, điện để sạc máy tính và thuốc lá để hút.
* Công sức vận hành thấp: Nếu phải dành quá 50-60% thời gian trong một ngày chỉ để theo dõi một hạ tầng đang sống hay chết trong khi nhân lực không đủ thì sẽ là quá phí phạm công sức
* Tương đối dễ sử dụng và vận hành: Tốn quá nhiều thời gian để xây dựng một hạ tầng, khắc phục sự cố cho nó thì cũng chẳng còn thời gian đâu để phát triển thêm tính năng với một đội ít người. Hơn nữa một công cụ/nền tảng phức tạp trong sử dụng sẽ yêu người có trình độ cao, do đó cần tuyển ai cũng có trình độ cao để sử dụng/vận hành.
* Đủ khoảng thở để phát triển: Nền tảng này phải đủ mở để về sau thêm được nhiều tính năng của mình mà không phải bị cản trở bởi nền tảng quá nhiều. Không ai muốn code plugin cho WordPress cả, cũng chẳng ai muốn dùng những công nghệ cũ kĩ như Drupal hay Joomla, hoặc những thứ khổng lồ như Magento làm gì cho nhọc công.

Bài toán của tôi còn có những yêu cầu sau:

* Cần nơi lưu trữ các blob có dung lượng lớn, khoảng 10% trong đó là hot access
* Serve các blob có dung lượng lớn đó tới khách hàng

Dựa vào nhưng tiêu chí ở trên, tôi có ba giải pháp như sau:

- AWS Lambda + API Gateway + RDS/Dynamo + S3 + CloudFront: Giải pháp thường thấy bởi bất kì những người từ già trẻ lớn bé biết đến cloud service.
- Cloudflare Workers + Pages + D1 + R2 + Cloudflare CDN: Giải pháp mới đưa ra của Cloudflare, gần tương tự với giải pháp của AWS.
- Heroku + S3 Compatible Storage + CDN khác: Giải pháp sử dụng một web và nhiều background worker khá hay của Salesforces
- On-prem/VM + S3 Compatible Storage + CDN khác: Sử dụng runtime là máy ảo của AWS hoặc các provider nhỏ hơn. Giải pháp tự dựng 100%, làm chủ tất cả các công nghệ, tự dựng stack công nghệ của riêng mình

> Ở đây có thể các bạn thắc mắc rằng tại sao lại không dùng CDN Cloudflare cùng với các giải pháp khác, thì nên lưu ý rằng trong [Điều khoản sử dụng CDN của Cloudflare](https://www.cloudflare.com/en-gb/service-specific-terms-application-services/), việc serve các nội dung không phải HTML qua mạng của Cloudflare là vi phạm chính sách của họ, trừ khi bạn serve các nội dung đó từ các dịch vụ của chính Cloudflare (như R2, Images hay Pages). Trừ khi bạn dùng gói Enterprise, còn đâu thì việc cache một lượng blob lớn lên CDN của Cloudflare có thể dẫn đến kết quả tài khoản của bạn bị terminate. 

Dưới đây là bảng so sánh các tính năng giữa các giải pháp

|            | Ưu điểm                                                                                                                                                                                                                                | Nhược điểm                                                                                                                                                                                                                                                                                                                                                                                              |
|------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AWS        | - Nền tảng rất phổ biến, nhiều người dùng và được chứng thực bởi nhiều khách hàng<br>- Được thiết kế để scale<br>- Nhiều khoảng thở để phát triển ứng dụng, vì AWS Lambda sử dụng môi trường Linux và hỗ trợ native rất nhiều ngôn ngữ | - Giá thành không hề rẻ, ngoài phải trả tiền cho các sản phẩm sử dụng, ta phải trả các khoản phí không tên khác như tiền băng thông, tiền VPC, vân vân                                                                                                                                                                                                                                                  |
| Cloudflare | - Giá rẻ như bùn<br>- Scale rất tốt với cold start thấp<br>- Hỗ trợ native một số framework fullstack như SvelteKit, NuxtJS                                                                                                            | - Chỉ hỗ trợ native JS/TS, muốn dùng ngôn ngữ khác phải compile ra WASM (rất khó và trúc trắc)<br>- Nền tảng Workers chạy trong sandbox Google V8 nên rất nhiều giới hạn về bộ nhớ, năng lực tính toán và tính năng thua xa AWS Lambda<br>- Các giải pháp xung quanh Workers (như Queues, KV và D1) khá khó dùng và có nhiều hạn chế<br>- Việc connect tới các dịch vụ sử dụng TCP gần như không hỗ trợ |
| Heroku     | - Thân thiện với người dùng, nhiều add-ons có sẵn<br>- Hỗ trợ nhiều ngôn ngữ, đặc biệt là cho phép dùng Docker                                                                                                                         | - Giá vẫn còn đắt (để có option 1GB RAM cần tới 50$ 1 tháng)<br>- Không có region nào ngoài US và EU<br>- Tiền add-ons tính vào thì đắt cũng chẳng kém AWS                                                                                                                                                                                                                                                                                                    |
| On-prem   | - Làm chủ toàn bộ hoàn toàn về công nghệ | - Tất cả những vấn đề như quản lý server, scaling, trục trặc hạ tầng sẽ do mình lo lắng toàn bộ<br>- Công sức để engineer ra toàn bộ stack công nghệ, sử dụng tối ưu máy móc không hề dễ dàng<br>- Chi phí cũng không giảm hơn nhiều nếu không tối ưu được mức sử dụng |



Dù có nhiều nhược điểm nhưng vấn đề kinh tế đã chiến thắng tất cả, tôi lựa chọn Cloudflare Workers. Giải pháp mà tôi lựa chọn là fullstack SvelteKit deploy trên Cloudflare Pages + Cloudflare Workers làm backend, D1 + KV làm database và R2 làm nơi lưu trữ dữ liệu. Ngoài ra, việc trữ dữ liệu ở R2 cũng free tiền băng thông egress ra ngoài internet, tiết kiệm kha khá tiền CDN.

# Những vấn đề gặp phải khi thiết kế

Tất nhiên, đã chọn CF Workers là xác định đối mặt với những nhược điểm của nó, và dưới đây là những vấn đề tôi gặp phải khi sử dụng giải pháp của Cloudflare

## Chỉ support JS/TS native, các ngôn ngữ khác phải build ra WASM
Việc này không hề thân thiện một chút nào với nhiều người, do khi build ra như vậy khiến việc debug/testing gặp rất nhiều trở ngại. Hơn nữa không phải ngôn ngữ nào cũng có thể build ra WASM, mà chỉ có các ngôn ngữ compile mới có thể dùng được. Mặc dù theo lý thuyết ta có thể code hoặc compile các thư viện viết bằng Rust, C/C++ để tự tạo các thư viện mà trong môi trường Cloudflare Workers không có, nhưng so với việc sử dụng Lambda thì CF Workers kém thân thiện hơn rất nhiều.

## Cloudflare D1 thực chất là SQLite và migration trên đó thực sự là thảm hoạ

SQLite là database nằm trên file, vậy nên phải xác định rằng nó có những giới hạn nhất định của nó. Giới hạn đầu tiên mà tôi đụng phải chính là việc thay đổi cấu trúc bảng trong nó. Việc sửa đổi bảng, ngoài việc thêm cột và thêm index, thực sự rất khó khăn, và đều đi đến một quyết định khá đau thương là tạo bảng mới với cấu trúc mới và drop bảng cũ đi. Việc này rất dễ dẫn sai sót và khiến việc migration trở nên lằng nhằng vô cùng.

Dù sao thì SQLite vẫn là DB chỉ dùng trong một node và dù có implement như nào đi chăng nữa thì nó vẫn là SQLite, hoặc những kĩ sư trong Cloudflare siêu thông minh và họ có thể implements các tính năng của D1 siêu đỉnh, đến mức có thể vượt qua những giới hạn trong thiết của SQLite như database-locking khi có transaction, thì việc đối mặt với vấn đề cấu trúc bảng vẫn là một ác mộng.

## Chạy các tác vụ liên quan đến crypto khá cùi

Cùi ở đây có hai điểm:

~~1. Môi trường chạy khá yếu, 128M RAM với 10ms CPU time (nếu trả 5$ để có thêm 40ms CPU time nữa) là tương đối ít. Nếu phải sử dụng Workers Unbound cho cả project thì lại phí. Do đó các tác vụ liên quan đến mã hoá/giải mã dữ liệu sẽ không đủ RAM và CPU time để chạy.~~
1. Luận cứ trên không còn đúng nữa do Cloudflare đã thay đổi phương thức hoạt động của Workers vào 2024/03/01, giờ ta có 30 triệu ms CPU time và mỗi fetch worker có thể sử dụng tối đa 30s CPU time (nếu sử dụng Cron Trigger thì còn được dùng tới 15 phút CPU time). Nhưng câu chuyện 128M RAM thì vẫn còn. Nếu bạn sử dụng global var để lưu thông tin giữa các request, thì khi lưu quá nhiều thì worker của bạn vẫn sẽ lăn ra chết. Vậy nên vấn đề thời gian của Workers không còn đáng lo, mà chủ yếu là vấn đề memory, nếu là về memory thì đây là bệnh chung của tất cả các nền tảng serverless.
2. API WebCrypto hoá khá nhiều giới hạn, đặc biệt password hasing chỉ có PBKDF2 là support bởi Cloudflare Workers, và việc implements chúng vào code cũng không dễ dàng gì, [và đây là một ví dụ](https://timtaubert.de/blog/2015/05/implementing-a-pbkdf2-based-password-storage-scheme-for-firefox-os/), nhân tiện tôi cũng đang sử dụng phương pháp này để mã hoá password, tất nhiên là đã thêm salt và pepper để an toàn hơn.

~~Vậy nên nếu để sử dụng các thuật toán mã hoá BCrypt hay Argon2, bắt buộc phải gọi ra một service ngoài, ví dụ như một function Lambda hoặc Unbound worker đề offload việc sang, tạo thêm một infrastructure dependency không đáng có, khiến việc maintain hệ thống trở nên khó hơn. Hơn nữa kể cả khi đã chạy PBKDF2, việc hashing vẫn có nguy cơ hết RAM/CPU giữa chừng, gây ra khó chịu nhất định.~~

Vậy luận điểm đã nêu trước kia chỉ còn là vấn đề về RAM. Có nhiều CPU time thì ta hoàn toàn có thể sử dụng Argon2 hoặc BCrypt nếu muốn, vì trong thực tế số lượng request dùng đến password hashing không nhiều, chủ yếu dùng lúc tạo tài khoản và login. Nhưng để không phải install nhiều thư viện và bật `node_compat` thì tôi vẫn sử dụng PBKDF2.


## Logging khá rối rắm

Khi bạn không tốn 5$ một tháng cho Cloudflare, cách duy nhất để xem log `wrangler tail` ở máy local để stream log của Workers real time về. Ngay cả khi đã tốn tiền thì Cloudflare không cung cấp dịch vụ nào tương tự như Cloudwatch để lưu trữ và query log trực tiếp, mà ta phải sử dụng Cloudflare Logpush để gửi log vào một dịch vụ nào đó để xem, [như trong xem document này](https://developers.cloudflare.com/logs/get-started/enable-destinations/elastic/). Mọi thứ chợt trở nên kém tiện lợi hơn so với AWS, nhưng với giá thành rẻ như vậy thì ta có thể tạm chấp nhận được. Một trong những cách mà ta có thể tiết kiệm được tiền lưu log chính là tạo một con ElasticSearch ở nhà và cho Cloudflar Logpush đầy log vào đó, thế là tiết kiệm được tiền logging. Trong bài toán thực tế, tôi sử dụng New Relic để lưu và xem log vì hạn mức free khá cao (thực ra vì setup lưu vào Elastic mất thời gian quá nên đưa vào New Relic cho tiện).

Ở thời điểm hiện tại (2024/05/20) thì Cloudflare vẫn chưa cho phép Pages function có thể push log thông qua Logpush. Vậy nên để log cho Pages function trong Cloudflare, ta bắt buộc phải implements Pages plugin trong code để push log đi, [như trong document này](https://developers.cloudflare.com/pages/functions/plugins/sentry/). Quá phức tạp cho việc logging.

## Không thể dùng nhiều package của NodeJS

Kể cả có bật `node_compat` lên thì vẫn sẽ có khả năng code của bạn sẽ không thể sử dụng kha khá các thư viện của NodeJS, phần vì không thể implements hết các API của NodeJS, phần vì giới hạn 1M (10M nếu mất 5$ để dùng Cloudflare Paid Plan) file code của Cloudflare, khi webpack có thể tạo file lớn hơn giới hạn.

## Connect vào database sử dụng TCP khá giới hạn

Sẽ có lúc bạn cần connect vào các database ở ngoài để lấy dữ liệu từ chúng. Hiện tại chỉ có PostgreSQL là được hỗ trợ thông qua Hyperdrive, còn lại chúng ta không thể làm gì để connect vào MySQL, Redis, ngoại trừ việc phải sử dụng các bên có support endpoint HTTPS như Upstash hay PlanetScale và tất nhiên, giá sẽ cao vút nếu bạn bắt đầu phải phục vụ lượng người dùng lớn. 

# Tóm lại

Ở trên là những trải nghiệm của tôi khi sử dùng CF Workers. Dù nó có nhiều giới hạn, nhưng tiêu chí rẻ đã khiến tôi tin tưởng giải pháp của Cloudflare. Mong là tôi sẽ không bỏ của chạy lấy người giữa như ông bạn [liftosaur](http://liftosaur.com) ở đây mà kịp sống dai, kiếm đủ tiền ra lãi và đợi để nền tảng Cloudflare Workers có thêm nhiều tính năng.