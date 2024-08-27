---
title: "Trải nghiệm khi xử lý ảnh trên Cloudflare Workers"
author: "Aperture"
date: "2024-08-21T16:13:00+07:00"
categories:
    - serverless
    - cloudflare
    - cloudflare workers
    - experience
tags:
    - cloudflare
    - workers
    - r2
    - cdn
    - cache
    - lambda
    - experience
    - cloud
    - image
    - optimization
draft: true
---

# Mở đầu câu chuyện

Gần đây tôi có 1 bài toán như sau:
- Lưu trữ ảnh trên R2 Bucket
- Ảnh gốc không được để lộ ra ngoài
- Khi khách lấy ảnh ra, tối ưu kích thước ảnh và thêm watermark

Có hai cách sau mà tôi đã cân nhắc, kèm theo ưu nhược điểm:

## Lưu thêm phiên bản ảnh đã tối ưu vào R2

Phương pháp này sẽ được thực hiện như sau:
- Chia 2 bucket, 1 bucket private để chứa ảnh gốc, 1 bucket public để chữa ảnh đã optimize và thêm watermark và đó.
- Với bucket public, cắm domain/CDN vào đó và server ảnh từ bucket này ra.
- Với mỗi ảnh được upload vào public bucket, trigger Cloudflare Workers để optimize, thêm watermark và lưu vào public bucket.

Phương pháp này có ưu và nhược điểm sau:
- Ưu: Bước tạo ảnh chỉ phải thực hiện 1 lần trong vòng đời của nó, có thể cache rất lâu trên CDN.
- Nhược: Không dễ thay đổi watermark của các ảnh cũ và ảnh bị lưu ra thành nhiều bản khác nhau cho các watermark khác nhau.

## Thêm watermark mỗi khi nào người dùng gọi vào ảnh

Phương pháp này có cách hoạt động khác hẳn so với phương pháp trên
- Chỉ sử dụng 1 private bucket duy nhất để lưu ảnh.
- Tạo một Cloudflare Workers có trách nhiệm lấy, optimize và cache ảnh trên CDN trên mỗi requeust thay vì lưu cố định thành nhiều phiên bản.
- Phụ thuộc vào URL lấy ảnh (ví dụ như domain, query string), kết quả ảnh sẽ trả ra khác nhau.

Phương pháp này tất nhiên cũng sẽ có các lợi thế và vấn đề của nó:
- Ưu: Lưu trữ rẻ hơn nhờ lưu ít hơn, nội dung của ảnh có thể thay đổi dễ dàng tuỳ ý.
- Nhược: Truy cập ảnh lần đầu tiên sẽ lâu, tốn tiền Cloudflare Workers.

# Lựa chọn của tôi

Tôi lựa chọn phương pháp sinh ảnh có watermark trên từng request thay vì tạo ảnh trước. Dưới đây là một vài lý do tôi lựa chọn sử dụng phương án này:

## Dễ dàng thay đổi watermark
Theo như yêu cầu, watermark cần phải được thay đổi tuỳ theo sự kiện đang diễn ra. Trong thực tế, ta nên sửa frontend dể dùng canvas API vẽ watermark lên ảnh, hoặc chồng đè 1 ảnh PNG watermark lên trên ảnh, nhưng như vậy khách hàng có thể dễ dàng tải được ảnh không có watermark về và theo như bài toán mà tôi gặp thì việc này là không được phép, vậy nên phương pháp này yêu cầu phải có watermark ghi đè lên ảnh.

## Logic lấy ảnh linh hoạt, không tốn chi phí lưu trữ
Do việc lấy ảnh được đảm nhiệm bằng 1 đoạn code, ta có thể dễ dàng thay đổi nó khi cần, giả dụ như thay đổi path lấy ảnh, đổi watermark theo domain mà không tốn thêm dung lượng lưu trữ.

### So sánh giữa phí lưu trữ và phí Workers

Giả sử ta có 500G ảnh, mỗi ảnh nặng trung bình 50kB, vậy ta có 10M ảnh. Giả sử mỗi tháng khách sẽ truy cập 8M ảnh (unique), ta có 2 chi phí sau là cố định hàng tháng (dựa theo [biểu giá của Cloudflare](https://developers.cloudflare.com/r2/pricing/#r2-pricing)):

Phí lưu trữ:
```
storage_cost = 500G * $0.015
             = $7.5
```

Phí GetObject:
> Do Cache của Cloudflare chỉ tồn tại ở trong 1 data center, nên số lượng request có thể bị nhân lên, ví dụ khách đã truy cập vào PoP Singapore và ảnh đã cache tại PoP của Singapore, thì khách khác gọi vào từ PoP Hong Kong thì Cloudflare sẽ lại gọi vào R2 lần nữa. Để đoán định chính xác thì rất khó, nên giả sử số lượng request ở tất cả các PoP sẽ gấp 3 lần số lượng request thực tế, bỏ qua request trùng:
```
get_object_cost = 8M * ($0.36/1M) * 3
                = 8 * $0.36 * 3 = $8.64
```

Vậy, tổng phí cố định hàng tháng sẽ là: `$16.14`

Dưới đây ta sẽ tính các biểu phí đặc trưng của từng phương án

## Phí lưu trữ một bản ảnh đã dán watermark ở bucket khác

Phí lưu trữ phiên bản ảnh đã dán watermark:
> Giả sử ảnh biến hết thành `webp` với kích thước nhỏ hơn khoảng 40%
```
watermarked_storage_cost = storage_cost * 0.7
                         = $7.5 * 0.6 = $4.5
```

Vậy tổng phí nếu lựa chọn phương án lưu thêm phiên bản khác là `$20.64`.
> Nếu lưu thêm các phiên bản khác nhau nữa, cần phải tính lại cả số lượng request của phiên bản mới để cộng lại, tạm thời ta chỉ tính đến phương án lưu 1 phiên bản

## Phí sử dụng Workers để convert ảnh:

Phí Cloudflare Workers Paid cố định: `$5` (do 8M request chưa vượt quá 10M request hàng tháng)

Vì Cloudflare có dịch vụ Cloudflare Image Transformation, ta tính thử phí như sau:
> Công thức tính sẽ gần giống như tính GetObject ở trên, nhưng nhân với [giá Image Transform](https://developers.cloudflare.com/images/pricing/#images-transformed)
```
cf_image_transform_cost = 8M * ($0.5 / 1k) * 3
                        = 8k * $0.5 * 3
                        = $12000 (?!!!)
```

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/shotgun-bill.gif"
    position="center"
    alt="Tận $12000 mỗi tháng???? Một số tiền quá lớn!"
    title="Tận $12000 mỗi tháng???? Một số tiền quá lớn!"
    caption="Tận $12000 mỗi tháng???? Một số tiền quá lớn!" >}}

Có vẻ hướng sử dụng Cloudflare Image Transformation là quá đắt... Ngay cả việc sử dụng Cloudflare Images cũng nói luôn rẳng việc chứa 20 variants [không bao gồm việc vẽ thêm watermark](https://developers.cloudflare.com/images/pricing/#images-stored), vậy nên ta nên chọn phương pháp khác rẻ tiền hơn: Sử dụng thư viện [@cf-wasm/photon](https://www.npmjs.com/package/@cf-wasm/photon) để tối ưu ảnh. Việc này sẽ ảnh hưởng tới việc sử dụng CPU time, và ta sẽ tính nó vào phần sau.

