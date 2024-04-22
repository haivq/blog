---
title: "Phương pháp ma quỷ để gộp chung Pandas, NumPy và SciPy vào chung một layer lambda để chạy trên python 3.10 mà không quá dung lượng 250M"
author: "Aperture"
date: "2024-04-22T00:20:00+07:00"
categories:
    - lambda
    - aws
    - serverless
    - experience
tags:
    - lambda
    - aws
    - experience
    - webapp
    - devops
    - python3.10
    - python
    - numpy
    - pandas
    - scipy
draft: true
---

# Mở đầu câu chuyện

Mọi thứ bắt đầu khi tôi được giao một task yêu cầu phải migrate tất cả function lambda của hệ thống từ Python 3.8 lên một phiên bản mới hơn do Python 3.8 sẽ kết thúc vòng đời (EOL) vào 10/2024. Sau một hồi cân nhắc thì tôi quyết định update lên 3.10. Mọi người có thể hỏi vì sao không update thằng lên 3.11+, câu trả lời đơn giản là vì nhiều thư viện Python hiện tại chưa support các bản mới như vậy, điển hình là PyTorch. Hơn nữa, các phiên bản mới có nhiều tính năng mới không dùng và có thể gây ra bug tiềm tàng. Dùng bản cũ ít tính năng thì sẽ bớt entropy để gây ra lỗi.

Công việc này bao gồm 3 phần:

1. Kiểm tra các breaking change về library và syntax từ 3.8 sang 3.10 có thay đổi gì không (tức phải xem thay đổi của cả 3.9 và 3.10)
2. Update các layer library có sẵn từ Python 3.8 lên 3.10
3. Update các function và để bên tester làm việc

Việc review các breaking change xảy ra khá suôn sẻ, vì phần lớn các update từ 3.8 đến 3.10 chủ yếu là thêm tính năng chứ không có thay đổi lớn gì về cú pháp. Các thư viện có cải tiến mà tôi hay dùng là `concurrent.futures` cũng không có breaking change nào cả. Việc update phần lớn các layer library cũng đơn giản, ngay cả thư viện hay gặp vấn đề nhất là `cffi` cũng chạy luôn không phải sửa gì cả, nhờ vậy phần lớn các layer đều migrate lên Python 3.10 khá nhanh chóng và chạy không có vấn đề gì. Tôi đã nghĩ mọi thứ như vậy là ổn, cho đến khi gặp phải vấn đề nan giải: Cho NumPy, SciPy và Pandas chạy được cùng với nhau.

# Vấn đề kích thước với NumPy, SciPy và Pandas

Nghiệp vụ mà tôi gặp phải yêu cầu phải có cả 3 thư viện SciPy, NumPy và Pandas. Nếu bạn dùng SciPy, NumPy và Pandas thì cũng hiểu các thư viện này khi tải về rất nặng. Ngoài code ra, các thư viện trên còn chứa cả thư viện phụ trợ kèm theo, các thư viện `.so` đã build ra (`OpenBLAS` và `GFortran` cho NumPy và SciPy, mỗi thư viện dùng một phiên bản) nên tổng cả thư mục này có kích thước lên tới 278M, trong khi AWS Lambda chỉ cho phép tổng TẤT CẢ các layer lại trong một function là 250M, tức là tôi đã quá 28M so với giới hạn, đấy là chưa kể cần các layer khác cho code và các thư viện khác.

Ở runtime Python 3.8, AWS cung cấp sẵn 2 layer sau:

- `AWSLambda-Python38-SciPy1x`: Bao gồm SciPy và NumPy
- `AWSSDKPandas-Python38`: Bao gồm Pandas và NumPy

Và việc nhét một lúc cả 2 layer này vào một function cũng gây ra lỗi quá kích thước nốt. Tôi đã exploit việc Pandas không có thư viện rời, nên đã tải xuống một layer chỉ có một mình Pandas mà không có NumPy:

```bash
pip install pandas --target python
rm -r python/numpy*/
```

Vậy là layer mới này chỉ chứa mỗi Pandas và các dependency khác của nó nên nhỏ hơn, chỉ còn khoảng 70M. Khi dùng thì chọn cả layer này vào cùng với `AWSLambda-Python38-SciPy1x`, thì cả kết quả ta được 1 function có đầy đủ Pandas, SciPy và NumPy cùng một chỗ. Như ở trên theo lý thuyết thì không thể, nhưng có lẽ bản build custom của AWS đã tối ưu việc import 2 thư viện `OpenBLAS` và `GFortran` của SciPy và NumPy, nên tiết kiệm được khoảng 25M.

Nhưng khi lên runtime Python 3.10, AWS chỉ cung cấp một layer `AWSSDKPandas-Python310` chỉ chứa mỗi Pandas và NumPy mà không cung cấp layer nào chứa SciPy cả, mà như ở trên, việc tạo layer chứa cả 3 thư viện là quá cỡ, mà tạo mỗi layer chứa mỗi Pandas không là không thể do không có layer mặc định nào chứa SciPy và NumPy. Để giải quyết vấn đề này, tôi đưa ra 4 phương án:

1. Giữ nguyên các function cần 3 thư viện trên ở Python 3.8
2. Nhét tất cả code vào Docker do AWS Lambda có thể chạy 1 image Docker lên tới 10G, lúc đó install thư viện nào, bao nhiêu cũng được
3. Mount EFS vào function và cài thư viện vào đó, sau đó dùng một vài trick để Python có thể detect được vị trí thư viện và lôi ra sử dụng
4. Xoá bớt code trong các thư viện đi cho nhỏ hơn 250M (MA QUỶ 💀💀💀)

