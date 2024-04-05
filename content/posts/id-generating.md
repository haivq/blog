---
title: "Các phương pháp đặt ID trong CSDL"
author: "Aperture"
date: 2021-12-16T15:21:33+07:00
tags:
    - id
    - database
tags:
    - id
    - id generating
    - snowflake
    - uuid
    - ulid
    - database sorting
    - experience
---

# Đặt vấn đề

Một trong những việc quan trọng nhất khi lưu dữ liệu xuống bất kì đâu chính là đặt cho chúng một cái tên. Theo tôi một cách đặt tên được gọi là tốt cần phải thoả mãn các điều kiện sau:

1. Đảm bảo tính độc lập và duy nhất. Nhập nhằng về tên sẽ khiến việc truy xuất dữ liệu khó hơn rất nhiều
2. Kích thước không quá lớn vì tên quá lớn khi lưu vào, tìm kiếm và lấy ra sẽ mất nhiều thời gian
3. Cách đặt tên đơn giản. Quy tắc đặt tên quá phức tạp khi gặp lỗi sẽ rất khó sửa, cũng như khó bảo trì

Nhưng để một cách đặt tên được gọi là tốt để làm UID trong CSDL nên có thêm các yếu tố sau:

4. Quy tắc nên sinh ra UID dạng có thể sắp xếp được, do các RDBMS thường sử dụng cấu trúc dữ liệu dạng cây BTREE, đưa thành dạng sắp xếp được sẽ đảm bảo tốc độ đọc và ghi luôn cao. Trong trường hợp khác, ID sắp xếp được sẽ khiến việc query đơn giản hơn ta thường truy vấn dữ liệu theo chiều thời gian tăng dần
5. Quy tắc nên sinh ra UID đơn điệu tăng, tức là UID sau sinh ra luôn được so sánh lớn hơn UID trước, việc này làm cho quá trình truy vấn, sắp xếp đơn giản hơn. Hơn nữa, việc để ID dạng tăng dần được sẽ giúp cho việc truy vấn và dàn đều dữ liệu trong một cluster CSDL dễ hơn (nhờ cơ chế replicating và sharding dựa vào ID của các CSDL hỗ trợ chúng như Mongo, Cassandra hay MySQL)
6. ID sinh ra hoàn toàn độc lập đến dữ liệu mà nó đại diện, đảm bảo tính bảo mật cho dữ liệu, tránh để đối thủ hiểu ra cách vận hành để lấy dữ liệu của bạn về

Tóm lại việc sinh ra một UID hoàn hảo phải đảm bảo 3 yếu tố: không trùng lặp, đơn giản và an toàn. Tất nhiên, không có một *viên đạn bạc* nào có thể giải quyết tất cả 6 vấn đề trên, trong các trường hợp cụ thể ta có thẻ bỏ qua một vài yếu tố ở trên mà vẫn đảm bảo rằng hệ thống của chúng ta không lâm nguy. Dưới đây là một vài phương pháp sinh UID nổi tiếng và cách sử dụng chúng **theo ý kiến của tôi** là hợp lý nhất

# Sử dụng hệ thống sinh ID mặc định trong CSDL

Lựa chọn đầu tiên và đơn giản nhất với tất cả những ai sử dụng CSDL chính là sử dụng `AUTO INCREMENT` cho cột ID trong bảng MySQL, hoặc đối với các NoSQL hỗ trợ đánh ID cho bản ghi (ví dụ như MongoDB), người dùng thậm chí còn không phải quan tâm đến việc ID của nó là gì. Dưới đây ta sẽ đánh giá ưu điểm và nhược điểm của phương pháp này:

Ưu điểm:
- Thiết lập đơn giản, bạn không cần phải lo lắng việc sinh ID nằm trong code
- Đối với `AUTO INCREMENT`, việc insert trùng sẽ không xảy ra do trước khi insert, tất cả các node RDBMS sẽ check xem ID hiện tại là bao nhiêu rồi sau đó mới insert

Nhược điểm:
- Đối với `AUTO INCREMENT` của MySQL:
    - Sử dụng chúng tạo ra quy luật tuần tự, đối thủ có thể crawl sạch dữ liệu của bạn về
    - Khi sử dụng với một cluster MySQL lớn, insert bản ghi vào một bảng sẽ khiến bảng đó bị lock, đồng nghĩa với việc tốc độ insert sẽ giảm, đặc biệt khi tần suất insert nhiều hoặc có nhiều lệnh insert số lượng lớn. Việc sử dụng `AUTO_INCREMENT` sẽ khiến RDBMS phải sync giữa các node với nhau và kéo dài thêm thời gian insert. Có một lựa chọn khác cho phép bạn insert nhiều bản ghi cùng một lúc nhưng điều này đồng nghĩa với việc hi sinh tính "duy nhất" với ID bản ghi.
    - Điều này vô tình để lộ số lượng bản ghi trong CSDL, có thể là một bất lợi đối với an toàn thông tin
- Đối với hệ thống sinh ID trong các NoSQL
    - Thường việc các NoSQL không có cơ chế tự sinh ID
    - Trong trường hợp của MongoDB, không có gì đảm bảo cho việc ObjectID của chúng là duy nhất

Khá nhiều nhược điểm chết người so với số lượng ưu điểm ít ỏi, nhưng không phải vì vậy mà hệ thống sinh ID mặc định không có giá trị gì. Dưới đây là một vài trường hợp sử dụng hệ thống sinh ID là hợp lý:

- Hệ thống CSDL chỉ gồm một node duy nhất
- Bảng sử dụng `AUTO INCREMENT` không có tần suất insert cao
- Trường ID không xuất hiện ra ngoài

# Sử dụng UUIDv1-5

Một lựa chọn rất phổ biến khác được sử dụng để làm ID của dữ liệu. Ý tưởng của UUID rất đơn giản, đầu vào là một khối dữ liệu và đầu ra là một chuỗi số 128 bit. Tính chất đặc trưng của UUIDv1-5 chính là sự ngẫu nhiên và hỗn loạn, nói cách khác là các UUID không có ràng buộc gì đến nhau (thực ra cũng không chính xác lắm nếu nói như vậy do ở UUIDv1 và v2 dùng các yếu tố như địa chỉ MAC, IP của máy để làm entropy tạo ra ID, nhưng ý tôi nói ở đây là khi kể cả chung một nguồn, các ID sinh ra không thể kiểm tra được quan hệ giữa chúng). Do độ thông dụng và nổi tiếng của UUID, tôi không cần phải nhắc thêm nhiều nữa, mà để cập đến thuật toán sinh UUID sẽ rất dài dòng. Vậy nên ở dưới đây tôi sẽ nêu ra các ưu điểm và nhược điểm của nó:

Ưu điểm:
- Thư viện sinh UUD được implement ở tất cả các ngôn ngữ
- Khả năng trùng lặp gần như bằng không
- Các ID sinh ra không có quy luật gì, do đó có thể tránh việc crawl dữ liệu từ đối thủ

Nhược điểm:
- Các ID không có sắp xếp theo thứ tự, do đó không thể truy vấn sắp xếp theo thời gian một cách đơn giản
- Kích thước rất lớn (128 bit), lưu trữ khá tốn dung lượng
 
Do nhược điểm trên, UUIDv1-5 không phù hợp để làm ID cho các bản ghi cần phải sắp xếp theo thời gian, hay cho các bài toán phải insert một lượng lớn dữ liệu trong một thời gian ngắn, ví dụ như hệ thống chat, khi mỗ dòng chat là một record trong CSDL, số lượng record cần được lấy ra rất lớn và cần phải được sắp xếp theo thứ tự thời gian. Vì vậy, UUID, theo tôi, sẽ phù hợp khi làm ID định danh cho các đối tượng thường được gọi đích danh bằng ID, ví dụ như User ID, và việc sắp xếp không phải yếu tố tiên quyết.

# Sử dụng ID dạng Snowflake

Khởi đầu từ một project năm 2010 từ sinh ID của Twitter, phục vụ bài toán sinh ID với khối lượng lớn, giờ Snowflake được áp dụng rất nhiều hệ thống chịu tải cao nổi tiếng khác như Discord, WeChat, Sony. Ý tưởng sinh của Snowflake ID rất đơn giản: Là một số nguyên dương 64 bit, khi biến thành số nhị phân sẽ thấy rõ chúng được cấu thành bởi 4 thành phần:

- bit 64: luôn bằng 0 (quy ước định dạng số nguyên dương)
- bit 63 - 22: UTC Timestamp (milli giây), giá trị này có thể khác nhau tuỳ vào người implement
- bit 21 - 0: Tuỳ thuộc vào cách implement mà phần này sẽ khác nhau với mỗi bên sử dụng Snowflake ID, nhưng tựu chung là định danh để khiến các snowflake được sinh trong cùng một thời điểm sẽ khác nhau

Lấy ví dụ như Snowflake của Discord sẽ như sau:

![Discord Snowflake](/id-generating/discord-snowflake.png)

Như ở trên từ bit 21 - 0 sẽ chia tiếp ra làm 3 phần:
- bit 21 - 17: ID của máy
- bit 16 - 12: Process ID (pid) của thread sinh ra Snowflake ID
- bit 11 - 0: Đánh số tuần tự của Snowflake ID được sinh ra từ cùng một máy và một thread

Hoặc như Snowflake của Baidu:

![Baidu Snowflake](/id-generating/baidu-snowflake.png)

Baidu implement bit 21 - 0 theo một cách khác:
- bit 21 - 13: ID của máy sinh Snowflake
- bit 12 - 0: Đánh số tuân tự của snowflake ID sinh ra từ cùng một worker sinh Snowflake

Đây là một trong các số ít các thuật toán sinh ID đơn giản mà có thể mô tả như thế này. Ở dưới đây, tôi sẽ đề cập ra các ưu và nhược điểm của chúng:

Ưu điểm:
- Cấu trúc ID đơn giản, dễ hiểu
- Khả năng trùng lặp bằng không (nếu implement đúng)
- Kích thước hợp lý để có thể lưu gọn trong CSDL (số nguyên 64 bit)
- ID không có quy luật tuần tự một cách bình thường, nên cũng không lo vấn đề bị crawl dữ liệu

Nhược điểm:
- Để đạt hiệu năng cao, thuật toán sinh Snowflake ID sẽ rất phức tạp (xem [UidGenerator](https://github.com/baidu/uid-generator/blob/master/README.md) của Baidu và xem cách họ tìm cách để sinh Snowflake ID vượt thời gian), còn thuật toán Snowflake ID cổ điển (như Twitter) sẽ khá chậm
- Nhiều phương pháp implement (như đề xuất của Twitter) sẽ yêu cầu phải có một server riêng để sinh ID, việc lấy ID sẽ thông qua RPC, và điều này có thể ảnh hưởng tới hiệu năng, tạo single point failure cho hệ thống microservice

Mặc dù không đảm bảo được tính đơn giản, và tốc độ sinh sẽ phụ thuộc lớn vào cách implements, Snowflake ID vẫn được sử dụng bởi tính chất tối ưu của nó khi sắp xếp. Đặc biệt nếu bạn có một loạt node CSDL, Snowflake ID sẽ khiến việc sharding và truy vẫn liên server sẽ đơn giản hơn nhiều vì nó hoàn toàn tuyến tính tăng dần. Đây là lựa chọn phù hợp cho các hệ thống yêu cầu lấy dữ liệu theo thời gian nói chung, ví dụ như hệ thống chat (như WeChat, Discord) hay hệ thống log

# Tóm lại

Còn rất nhiều các thuật toán sinh ID khác mà tôi chưa để cập tới, như sinh ID dựa trên hash của dữ liệu (md5, sha), ticket server (như [cách Flickr implements](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)), vân vân. Nhưng trên đây là các cách sinh ID khá độc đáo, nổi tiếng và DỄ HIỂU mà tôi ấn tượng. Có thể trong tương lai, tôi sẽ đề cập đến các ý tưởng khác. Bạn có thể xem những cách sinh khác ở [đây](https://www.fatalerrors.org/a/9-kinds-of-distributed-id-generation-methods-there-is-always-one-for-you.html)

# Tài liệu tham khảo

- [AUTO_INCREMENT Handling in InnoDB - MySQl 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)
- [Snowflakes - Discord Developer Portal](https://discord.com/developers/docs/reference#snowflakes)
- [Forget snowflake and feel the global unique ID generation algorithm with 587 times higher performance - Develop Paper](https://developpaper.com/forget-snowflake-and-feel-the-global-unique-id-generation-algorithm-with-587-times-higher-performance/)
