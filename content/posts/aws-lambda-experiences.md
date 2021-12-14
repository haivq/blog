---
title: "Một vài kinh nghiệm của tôi khi sử dụng AWS Lambda"
date: "2021-12-11T15:29:17+07:00"
author: "Aperture"
draft: true
---

Một trong những xu hướng mà tôi nắm bắt được khi sang làm DevOps ở công ty mới chính là công nghệ Serverless. Project đầu tiên mà tôi nhận được chính là migrate hệ thống từ sử dụng server vật lý cổ điển lên sử dụng AWS Lambda. Đây là một trải nghiệm vô cùng thú vị khi tôi có cơ hội rất tốt để có thêm một mindset mới để thiết kế một hệ thống dựa hoàn toàn vào một nhà cung cấp hạ tầng và không phải lo lắng về những lỗi về server cổ điển như trước. Tất nhiên, không có bữa trưa nào là miễn phí, việc chuyển giao không chỉ đơn giản là port các method trong code cũ thành các function và cứ thế mà nó chạy, tôi đã tốn khá nhiều thời gian để re-engineer lại hệ thống và dưới đây là một vài kinh nghiệm tôi thu thập được trong quá trình chuyển giao

# Thiết kế một API thuần sử dụng các dịch vụ của AWS

Một API có rất nhiều thứ chuyển động, trong đó sẽ sẽ có các thành phần sau:
- Lưu trữ
    - File tĩnh: Lưu trữ các file gần như không bao giờ thay đổi, (có thể) thường xuyên được lấy ra bởi người dùng nhưng gần như không bao giờ được lấy ra và xử lý tiếp API
    - File động: Bao gồm các file thường xuyên được truy xuất bởi API dùng cho các mục đích tính toán hoặc thay đổi
- CSDL
    - SQL: Lưu các dữ liệu có tính cấu trúc cao, quan hệ với nhau rất lằng nhằng, đảm bảo tính ACID, lưu lượng truy vấn tương đối nhiều và rất phức tạp
    - NoSQL: Lưu trữ các dữ liệu phi cấu trúc, lưu lượng truy vấn trung bình đến cao, truy vấn có thể đơn giản hoặc phức tạp tuỳ vào mục đích sử dụng, yêu cầu tốc độ truy vấn phải rất nhanh
    - Cache: Chứa các dữ liệu có thời gian sống ngắn, dùng để chứa các dữ liệu thường xuyên được truy vấn trong một thời gian ngắn
- Bảo mật
    - Mật khẩu các tài nguyên như mật khẩu DB
    - Các key giải mã dữ liệu

Thông qua các thành phần trên, tôi có thể liệt kê sơ qua các tài nguyên sẽ sử dụng trong AWS
- File tĩnh: S3, một lựa chọn khá hiển nhiên và đơn giản. Tuỳ thuộc vào mục đích và tần suất truy xuất dữ liệu, ta có thể lưu trữ chúng trong các bucket với tiering khác nhau
- File động: EFS, AWS hỗ trợ việc mount EFS vào Lambda, qua đó ta có thể dễ dàng thao tác với dữ liệu giống như ta đang thao tác với file trong ổ cứng của các server cổ điển
- SQL: Aurora RDS Serverless với giá thành rẻ và khả năng scale tốt hơn
- NoSQL: DynamoDB với key được sinh ra dựa trên các rule có sẵn dữa vào dữ liệu của SQL hoặc key từ dữ liệu từ người dùng gửi lên
- Cache: ElastiCache Redis nhờ giá thành rẻ, khá quen thuộc khi sử dụng
- Bảo mật: Secret Manager do AWS không hỗ trợ Vault của HashiCorp

Để minh hoạ (bao gồm cả thiết kế mạng), tôi có hình sau:

![Design topology](/aws-lambda-experiences/design-topology.jpg)

Về cơ bản, thiết kế như trên (không bao gồm các tài nguyên nằm ngoài AWS) đảm bảo gần như đầy đủ các yêu cầu của một API server với chi phí tương đối rẻ 

# Tận dụng tối đa tài nguyên trong vòng đời sống của Lambda Runtime

Dưới đây là biểu đồ trạng thái sơ lược flow chạy Lambda khi được invoke (ví dụ như API Gateway invoke lambda):

![Lambda Execution FLow](/aws-lambda-experiences/lambda-execution-flow.jpg)

Với những ai chưa biết, thay vì tạo mới một runtime cho mỗi lần request, AWS sẽ tạo ra một môi trường chạy code và giữ nó sống (hoặc gọi là `warm`) một lúc và tắt nó đi khi không có request tới nó nữa trong một khoảng thời gian đợi (ta không quyết định được khoảng thời gian đợi này). Sau khi khởi tạo, các đối tượng không nằm trong các function/method (như các global variables trong code hay kể cả trong các layer) sẽ vẫn sống chứ không bị giải phóng như các biến nằm trong các function.

Để hiểu rõ hơn, ta sẽ sử dụng một ví dụ python dưới đây:

```python
# file name: lambda_function
import pymysql
from contextlib import closing
conn = pymysql.connect(host="somehost", user="root", passwd="password", db="db_name", connect_timeout=5)

# lambda will call this function on every requests
def lambda_handler(event, context):
    get_all_usernames()

    return {
        "statusCode": 200,
        "message": "success",
        "body": {}
    }

def get_all_usernames():
    with closing(conn.cursor()) as cur:
        cur.execute("select id, username from user")
        result = cur.fetchall()

    return {uid:username for uid, username in result.items()}
```

Như ở ví dụ trên, việc `Tạo môi trường code` chính là import các thư viện và chạy code không nằm trong các function, như việc tạo connection `conn` tới mysql. Cũng ví dụ trên, sau khi thực hiện các request tới, biến `conn` sẽ không bị giải phóng nên các . Ưu điểm của phương pháp này là tiết kiệm được thời gian khởi tạo các tài nguyên khởi tạo lâu (việc khởi tạo một connection mới vào RDS có thể tốn từ 30ms đến 100ms), nhờ vậy làm giảm thời gian chạy của lambda, khiến API response nhanh hơn và là giảm chi phí vận hành.

Ngoài ra, ngoài các biến dữ liệu trong code được giữ lại (như trong ví dụ trên), giữa các lần chạy, Lambda còn giữ lại dữ liệu trong thư mục `/tmp` với kích thước tối đa là 512M. Vậy nên thông qua việc lưu trữ dữ liệu lâu dài trong các global variable, chúng ta hoàn toàn có thể lưu dữ liệu vào trong thư mục này như một dạng cache để đẩy nhanh quá trình xử lý. Tất nhiên, việc lựa chọn cách sử dụng nằm ở bạn, đây chỉ là một vài ví dụ về cách tận dụng vòng đời của Lambda, hãy tìm thật nhiều cách để tìm ra các phương pháp sử dụng tối ưu tài nguyên.

# Tinh chỉnh lượng tài nguyên cấp cho Lambda Runtime

Một hệ thống API tốt cần phải đảm bảo cân bằng giữa hiệu năng và chi phí bỏ ra, một API lệch sang một trong hai cán cân trên đều gây ra thiệt hại về tiền bạc. Nếu API quá chậm sẽ phá hoại trải nghiệm của người dùng và làm giảm tính nhiệm của người dùng với sản phẩm. Nếu bỏ quá nhiều tiền cho hệ thống API, có thể nó sẽ không đem lại hiệu năng mà bạn mong muốn và số lượng tiền bỏ sẽ trở thành hoang phí. Dưới đây là một vài đặc tính về sức mạnh tính toán và chi phí của Lambda:

### Chi phí

Công thức tính chi phí vận hành của Lambda có thể tính sơ qua như sau (Áp dụng với vùng Virginia)

```
Chi phí vận hành của Lambda ($) = Thời gian chạy function (ms) * Dung lượng RAM (M) * $0.0000000021
(Công thức trên tính dung lượng RAM với bội của 128M)
```

### Năng lực tính toán

CPU time được cấp cho một Lambda function tỉ lệ thuận với số lượng RAM cấp cho function đó. Một cách hiển nhiên rằng, thời gian vận hành sẽ giảm nếu được cấp nhiều CPU time hơn. Nói cách khác, thời gian chạy tỉ lệ nghịch với lượng RAM. RAM càng nhiều thì code chạy càng nhanh. Đồng thời việc truy cập filesystem, xử lý mạng cũng sẽ nhanh hơn.

Đến đây, chúng ta phải chú ý để chọn CPU time hợp lý để đảm bảo thời gian chạy phải thấp, nhưng cũng không quá đắt. Rất tiếc, không có một *viên đạn bạc* nào có thể đưa ra một công thức tính thần kì giúp bạn đưa ra một số lượng RAM cần thiết. Cách làm tối ưu hiện tại chỉ là thử liên tục để chọn lấy một mức RAM hợp lý. Tất nhiên trong blog post của Lambda có để cập đến một phương pháp để bạn có thể chọn một mức RAM hợp lý để chạy function Lambda, [đường dẫn ở đây](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-2/).

Nhưng sau một thời gian làm việc với Lambda, cũng có một vài kinh nghiệm tôi rút ra được khi chọn mức RAM sử dụng Lambda:
- Tốc độ thực hiện function không tăng tuyến tính do [luật Ahmdal](https://en.wikipedia.org/wiki/Amdahl%27s_law)
- Các tác vụ truy cập các API network phức tạp (như truy xuất file từ EFS) tốn kha khá CPU, cấp ít nhất 512M RAM
- Các tác vụ truy xuất database không tốn quá nhiều CPU, có thể dùng 128 đến 256M RAM


# Giới hạn số lượng Lambda chạy cùng lúc



# Nguồn tham khảo
[AWS Lambda execution environment](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-context.html#runtimes-lifecycle-ib])
[Operating Lambda: Performance optimization – Part 1](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)
[Operating Lambda: Performance optimization – Part 2](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-2/)
[Memory and computing power](https://docs.aws.amazon.com/lambda/latest/operatorguide/computing-power.html)