---
title: "Kinh nghiệm sau khi dành gần một năm chạy code xử lý ảnh trên Cloudflare Workers"
author: "Aperture"
date: "2025-05-27T14:37:11+07:00"
categories:
    - serverless
    - cloudflare
    - cloudflare workers
    - experience
    - image processing
tags:
    - cloudflare
    - workers
    - r2
    - cdn
    - cache
    - lambda
    - experience
    - lesson learned
    - cloud
    - image
    - optimization
---

> Đọc phần trước của bài viết này tại đây: [Trải nghiệm khi xử lý ảnh trên Cloudflare Workers](/posts/image-transform-with-cf-workers-experience)

# Mở đầu câu chuyện

Tiếp tục nội dung của bài viết [cũ](/posts/image-transform-with-cf-workers-experience), sau một năm sử dụng Cloudflare Workers để xử lý ảnh bằng thư viện [Photon](https://github.com/silvia-odwyer/photon), tôi đã rút ra được một vài kinh nghiệm và giới hạn của Cloudflare Workers khi sử dụng nó để làm các tác vụ nặng về tính toán. 

Lưu ý: Bài viết này hơi khô khan và chán, nhưng được cái không hề có nội dụng sinh ra bởi AI.

# Những giới hạn của Cloudflare Workers khi làm các tác vụ nặng

Dưới đây là những giới hạn của riêng môi trường Cloudflare Workers

## Kích thước bộ nhớ của Workers khá giới hạn

Như trong [document này](https://developers.cloudflare.com/workers/platform/limits/#worker-limits) của Cloudflare, mỗi Worker chỉ có tối đa 128MB RAM để thực hiện request. Nhưng điều này không đồng nghĩa với việc chúng ta sẽ có thể xử lý ảnh đến 128MB, mà sẽ phụ thuộc rất lớn vào số lượng pixel của ảnh. Coi như ta hào phóng để dành 28MB để chứa bộ nhớ cho code, thì cũng không thể nào xử lý ảnh lên tới 100MB được.

Để giải thích lý do vì sao, ta cần hiểu rằng ảnh đầu vào là định dạng đã được được nén, như JPEG hay WEBP, nhưng khi bung ảnh ra để xử lý thì ảnh được lưu trữ dưới dạng bitmap, tức một ma trận có kích thước ngang bằng với kích thước ảnh, mỗi pixel sẽ đại diện cho một vector RGBA. Giả sử như chúng ta dùng hệ màu 8 bit, thì 1 pixel (gồm 4 channel như ở trên) đã tốn đến 32 bits, tức là 4 bytes, chia 128M cho 4 ta được 32 megapixels. Nếu quy ra ảnh vuông, ta sẽ có kích thước mỗi cạnh của ảnh chỉ khoảng 5600 pixel. Dù ảnh JPEG của bạn có kích thước bé, chỉ khoảng 80 - 300kb, nhưng nếu số lượng pixel quá lớn vẫn sẽ khiến Workers chết giãy đành đạch vì thiếu bộ nhớ. Đó là chưa kể các thể loại ảnh có độ sâu màu cao hơn như 10 bit, 12 bit, hoàn toàn có thể gây crash worker. Thường các ảnh có độ phân giải cao, màu sắc sặc sỡ sẽ có kích thước lớn. Dù quy ước này không hề đúng, vì một cái ảnh JPEG trắng toát có thể có kích thước rất nhỏ, nhưng nếu số lượng pixel lớn vẫn gây crash như thường, nhưng để đơn giản hoá ta sẽ chỉ xử lý các ảnh có kích thước nhỏ hơn 3MB.

## CPU time của Worker bị giới hạn

Cũng như đã nêu trong [document ở trên](https://developers.cloudflare.com/workers/platform/limits/#worker-limits), nếu ta dùng Workers free plan thì chỉ có 10ms CPU để xử lý 1 tấm ảnh như thêm hiệu ứng, chèn watermark. Cloudflare sẽ dựa trên giá trị median CPU time để tính xem worker có đang dùng quá 10ms CPU hay không để đưa ra lỗi `Exceeded CPU Time Limits`.

{{< figure 
    src="/posts/one-year-of-img-proc-on-cf-worker/cf-metric-1.png"
    position="center"
    alt="Metric trên Cloudflare trong 2 tuần"
    caption="Metric trên Cloudflare trong 2 tuần" >}}

Như thống kê ở trên thì median CPU time của code chạy worker tận 148ms, vượt quá 10ms của Workers Free Plan. Vì thế bắt buộc ta phải mua Cloudflare Workers Paid Plan ($5/tháng) thì mới có đủ CPU mà chạy các job xử lý ảnh. Nhưng để an toàn thì ta nên giới hạn lại CPU Time limit lại trong `wrangler.jsonc` để tránh các trường hợp 1 worker dùng quá nhiều CPU:

```jsonc
{
	"$schema": "node_modules/wrangler/config-schema.json",
    //...       
	"limits": {
		"cpu_ms": 750
	},
    //...
}
```

Như vậy đảm bảo được việc các request đến sẽ không dùng quá nhiều CPU nhưng vẫn đảm bảo đủ giới hạn CPU để Worker có thể chạy trơn tru mà không bị tốn tiền, vì Cloudflare chỉ bill theo số lượng CPU đã dùng chứ không tính phí hết cả 750ms như cách AWS Lambda tính với RAM.

## Lỗi `unreachable` khi thao tác trên ảnh

Thỉnh thoảng sẽ có những tấm ảnh khi xử lý trên Cloudflare Workers sẽ trả ra lỗi `unreachable`:

{{< figure 
    src="/posts/one-year-of-img-proc-on-cf-worker/unreachable-wasm-error.png"
    position="center"
    alt="Lỗi `unreachable` do code WASM gây ra"
    caption="Lỗi `unreachable` do code WASM gây ra" >}}

Vì lỗi năm trong code WASM đã compile ra, nên việc debug đòi hỏi tôi phải build lại `Photon-rs` từ đầu và debug trong code thư viện Rust. Nhưng vì số lượng ảnh bị lỗi này cũng ít, thay vì trả ra lỗi 500, ta có thể handle nó thông qua exception chung của WASM như sau:

```typescript
try {
    // xử lý ảnh tại đây
} catch (e) {
    
    if (!(e instanceof WebAssembly.RuntimeError)) {
        // không phải WASM thì xử lý tại đây
    }
    
    if (!e.message.includes('unreachable')) {
        // không phải lỗi unreachable thì xử lý ở đây
    }
    // còn lại lỗi unreachable xử lý ở đây
}
```

Thông qua cách này ta có thể bắt được các ảnh bị lỗi và lấy về để debug sau, bằng cách nhét chúng vào Cloudflare Queue cho nhanh. Hoặc tuỳ mục đích sử dụng, khi gặp các ảnh không xử lý nổi ta có thể trả về luôn ảnh gốc nếu cần

# Giới hạn nằm trên hạ tầng Cloudflare CDN

Dưới đây là những giới hạn của hạ tầng CDN của Cloudflare, khác so với Cloudflare Workers

## Cache data trên CDN của Cloudflare chỉ tồn tại ở 1 PoP và Cloudflare Workers chạy ở mọi PoP, gây đội giá dù đã dùng Cache API
 
Cache trên CDN ở một Point of Presence (PoP) Cloudflare, như nhiều CDN khác, chỉ tồn tại ở PoP đó chứ không tự động replicate sang các PoP khác. Khi mỗi khi có connect tới PoP bị miss cache thì Cloudflare sẽ gọi vào vào Origin Server. Nhưng vấn đề là Cloudflare có tận 300 PoP, theo lý thuyết thì 300 PoP mà miss cache sẽ gọi vào origin server 300 lần, nhiều khi sẽ không cải thiện tốc độ được nhiều. Để khắc phục vấn đề này, Cloudflare có một giải pháp gọi là [`Tiered Cache`](https://developers.cloudflare.com/cache/how-to/tiered-cache/), thay vì tất cả các PoP đều gọi vào Origin server thì PoP ở tier thấp sẽ gọi vào PoP tier cao để tối ưu hoá Cache và tốc độ truy cập mà không phải gọi vào Origin server quá nhiều.

Map vấn đề này sang Cloudflare Workers, thì tất cả các PoP của Cloudflare đều chạy được Workers. Nhưng rất tiếc Cache API của Cloudflare Workers chỉ có thể tương tác với Cache của PoP đó mà thôi, không dùng được với Tiered Cache. Nhắc lại rằng Cloudflare có tới 300 PoP, tức là mỗi ảnh đi tới Cloudflare Workers ở một PoP mới chưa cache sẽ phải chạy lại code xử lý ảnh từ đầu. Đây không phải là một tin vui với tôi, vì chỉ riêng khu vực Bắc Mỹ đã có tới gần 60 PoP. Việc gọi vào PoP mà convert ảnh lại từ đầu thực sự đang đốt tiền CPU.

Trước mắt do giá của Cloudflare Workers đang rất rẻ, nên chưa có vấn đề gì xảy ra cả. Nhưng trong tương lai chắc chắn tôi phải tìm ra một cách nào đó để giải quyết vấn đề kinh tế này.

# Kết luận

Ở trên là những giới hạn mà tôi gặp phải khi sử dụng Cloudflare Workers để xử lý ảnh. Những giới hạn này không chỉ gói gọn trong việc xử lý ảnh, mà còn là giới hạn cho việc xử lý các tác vụ data nặng trên Cloudflare Workers. Mong độc giả có thể rút ra được kinh nghiệm từ bài viết của tôi.