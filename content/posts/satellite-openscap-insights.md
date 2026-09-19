---
title: "Tự động hoá việc audit bảo mật và giảm tải việc bảo trì hệ thống với Red Hat Lightspeed (Insights) và OpenSCAP trên Satellite"
author: "Aperture"
date: "2026-04-18T14:07:01+07:00"
categories:
    - redhat
    - infrastructure
    - sysadmin
    - satellite
    - rhel
    - security
tags:
    - satellite
    - redhat
    - infrastructure
    - openscap
    - insights
    - lightspeed
    - ansible
    - sysadmin
    - maintenance
    - rhel
---

# Hoàn cảnh

Một công việc khá là cực nhọc và tốn nhiều công sức mà sysadmin phải làm hàng ngày (như tôi được biết khi tiếp xúc với nhiều khách hàng) đó là:

  * Duy trì đảm bảo các cấu hình hạ tầng phải theo một quy chuẩn nhất định (ở mức độ platform, team hạ tầng không nên can thiệp vào ứng dụng đang chạy sau khi cấp phát máy)
  * Nhận báo cáo bảo mật từ team security quét ra và khắc phục các lỗ hổng/bad configuration

Việc này nhiều khi đem lại rất nhiều cực nhọc và ức chế, tôi có thể liệt kê một vài dưới đây:

  * Không có một nguồn rule (cả về bảo mật lẫn khuyến nghị vận hành) cụ thể để biết rằng một server có đang sử dụng config được khuyến nghị hay không
  * Sau khi có danh sách rule rồi, thì phải viết báo cáo tường trình tường tận, gồm máy nào dính rule nào, khuyến nghị sửa ra sao, sau đó đi trình cho các team khác đang sử dụng server rồi trình lại cho cấp trên.
  * Sau khi đánh giá hết rồi, thì lại phải lội xuống từng server để kiểm tra và vá lỗi

Công việc này rất mệt và gây tụt mood rất nhanh, đi lùng sục tìm best practices ở đâu đã mất thời gian, xong lúc thao tác trên server nữa, tầm 10-20 con server đã thấy oải, nữa là có tận 1k-2k con máy cần phải đánh giá và bảo trì thì còn mệt đến thế nào. Vì thế ai cũng muốn có một giải pháp:
  * Có một nguồn rule tin cậy để đưa ra đánh giá bảo mật và khuyến nghị vận hành
  * Tự động hoá việc đánh giá và xuất báo cáo
  * Có thêm khoản đưa ra câu lệnh và tự động động sửa lỗi dựa trên báo cáo đã có nữa thì quá tuyệt

Rất may rằng, Red Hat đã cung cấp một công cụ tuyệt vời để xử lý việc này, đó chính là Red Hat Lightspeed trên Red Hat Satellite để đưa ra lời khuyên vận hành và OpenSCAP scanner để quét các máy theo chuẩn bảo mật.
  
# Phương thức vận hành của Red Hat Lightspeed và OpenSCAP

Satellite nói một cách đơn giản, là một 