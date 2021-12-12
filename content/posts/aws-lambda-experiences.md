---
title: "Một vài kinh nghiệm của tôi khi sử dụng AWS Lambda"
date: "2021-12-11T15:29:17+07:00"
author: "Aperture"
draft: true
---

Một trong những xu hướng mà tôi nắm bắt được khi sang làm DevOps ở Maxflowtech chính là công nghệ Serverless. Project đầu tiên mà tôi nhận được chính là migrate hệ thống từ sử dụng server vật lý cổ điển lên sử dụng AWS Lambda. Đây là một trải nghiệm vô cùng thú vị khi tôi có cơ hội rất tốt để có thêm một mindset mới để thiết kế một hệ thống dựa hoàn toàn vào một nhà cung cấp hạ tầng và không phải lo lắng về những lỗi về server cổ điển như trước. Tất nhiên, không có bữa trưa nào là miễn phí, việc chuyển giao không chỉ đơn giản là port các method trong code cũ thành các function và cứ thế mà nó chạy, tôi đã tốn khá nhiều thời gian để re-engineer lại hệ thống và dưới đây là một vài kinh nghiệm tôi thu thập được trong quá trình chuyển giao

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

Với những ai chưa biết, thay vì tạo mới một runtime cho mỗi lần request, AWS sẽ tạo ra một môi trường chạy code và giữ nó sống (hoặc gọi là `warm`) một lúc và tắt nó đi khi không có request tới nó nữa trong một khoảng thời gian đợi (ta không quyết định được khoảng thời gian đợi này). Chi tiết hơn, đây là Automata sơ lược flow chạy Lambda khi có một request đi vào API Gateway:

![Lambda Execution FLow](/aws-lambda-experiences/lambda-execution-flow.jpg)

Để hiểu rõ hơn, ta sẽ sử dụng một ví dụ python dưới đây:

```python
# file name: 
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

Tất cả các đối tượng không nằm trong các function sẽ được giữ lại. Tức là như ví dụ trên, object `conn` chứa connection của mysql sẽ được giữ lại. Ưu điểm của phương pháp này là tiết kiệm được thời gian khởi tạo các tài nguyên khởi tạo lâu (việc khởi tạo một connection mới vào RDS có thể tốn từ 30ms đến 100ms), nhờ vậy làm giảm thời gian chạy của lambda, khiến API response nhanh hơn và là giảm chi phí vận hành.

# Tinh chỉnh lượng tài nguyên cấp cho Lambda Runtime

Một hệ thống API tốt cần phải đảm bảo cân bằng giữa hiệu năng và chi phí bỏ ra, một API lệch sang một trong hai cán cân trên đều gây ra thiệt hại về tiền bạc. API chậm sẽ đem lại trải nghiệm tệ hại cho người dùng. Nhưng hiệu năng của API lại không tăng tuyến tính dựa trên sức mạnh phần cứng, do vậy đến một ngưỡng nào đó số 

# Giới hạn số lượng Lambda Runtime



# Nguồn tham khảo
[Operating Lambda: Performance optimization – Part 1](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)