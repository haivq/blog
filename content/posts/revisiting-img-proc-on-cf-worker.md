---
title: "Quay lại câu chuyện xử lý ảnh trên Cloudflare Workers sử dụng Photon WASM"
author: "Aperture"
date: "2026-07-09T14:37:11+07:00"
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

> Đọc các phần trước của bài viết này tại đây:
> - [Trải nghiệm khi xử lý ảnh trên Cloudflare Workers](/posts/image-transform-with-cf-workers-experience)
> - [Kinh nghiệm sau khi dành gần một năm chạy code xử lý ảnh trên Cloudflare Workers](/posts/one-year-of-img-proc-on-cf-worker)

# Mở đầu câu chuyện

Sau bài viết [cũ về kinh nghiệm xử lý ảnh trên Workers](/posts/one-year-of-img-proc-on-cf-worker), gần đây Cloudflare cho ra mắt một tính năng rất quan trọng, ảnh hưởng trực tiếp đến hiệu năng và ví tiền tới công việc này: [Giờ đây đã có Cache đứng trước Workers](https://blog.cloudflare.com/workers-cache/). Tính năng khủng bố này không chỉ giải quyết vấn đề cache cho ảnh trên Cloudflare Edge, ngoài ra còn giải quyết rất nhiều vấn đề khác mà nhiều năm nay tôi gặp phải

Lưu ý: Bài viết không sử dụng AI.

# Giới hạn trước kia trên Workers khi chưa có Cache đứng trước

Tôi xin liệt kê lại 2 giới hạn lớn trước khi có Cache đứng trước.

## Cache trong Worker là Regional

Cache của Worker là regional theo PoP, tức là nếu Worker đã serve ảnh cho khách ở Hong Kong - do PoP HKG handle, thì khách ở Singapore muốn lấy ảnh, PoP SIN serve ảnh cho họ, thì không thể tái sử dụng cache ở Hong Kong mà phải transform ảnh từ đầu và lưu ở cache trong PoP SIN. Điều này đã được nhắc tới trong [doc của Cloudflare](https://developers.cloudflare.com/workers/reference/how-the-cache-works/#interact-with-the-cloudflare-cache).

Vậy mỗi khi có khách request ảnh ở 1 PoP mới, thì từng này bước sẽ thực hiện:
- Worker sẽ lấy ảnh trong R2 (tốn 1 lần tiền GET request)
- Worker tốn CPU time để transform ảnh (tốn tiền CPU time)
- Worker put ảnh vào regional cache (không replicate được sang PoP khác)

Tóm lại là sẽ bị tốn tiền với mỗi request ảnh mới cho mỗi PoP 

## Tốn tiền Worker invocation mặc dù đã cache, lại còn tăng thời gian response ảnh

Kể cả khi ảnh đã vào cache, thì mỗi lần khách request ảnh, sẽ tốn 1 lần invoke Worker. Dù [giá Workers invocation đã rất rẻ](https://developers.cloudflare.com/workers/platform/pricing/#workers), nên dù không tốn CPU để transform lại ảnh, thì cũng tốn tiền invocation để serve lại ảnh trong cache. Đây là thiết kế ban đầu của Cloudflare, Worker đứng trước, Cache đứng sau:

{{< figure 
    src="/posts/revisiting-img-proc-on-cf-worker/worker-in-front-of-cache.png"
    position="center"
    alt="Minh hoạ Worker đứng trước Cache"
    caption="Minh hoạ Worker đứng trước Cache"
    attr="Minh hoạ Worker đứng trước Cache"
    attrlink="https://blog.cloudflare.com/workers-cache/"
    link="https://blog.cloudflare.com/workers-cache/" >}}

Đặc biệt, asset ảnh ngoài người dùng lấy ra khi truy cập web thì còn bị các web crawler (của các Search Engine, AI Crawler, vân vân) hoặc các đối tượng xấu muốn phá hoại, DDoS kiểu làm lủng ví người vận hành, khiến cho vấn đề còn trầm trọng hơn.

Ngoài ra, thời gian xử ảnh cũng không hề nhanh. Khi khách hàng truy cập vào trang web, với số lượng PoP khổng lồ của Cloudflare, thì rất có khả năng họ sẽ phải chống nạnh đợi cái ảnh nó load từ từ về máy. Việc này phá hoại UX của trang web và giảm trải nghiệm người dùng xuống rất thấp.

# Worker Cache xuất hiện, giải quyết cả 2 vấn đề trên

Ơn giời, [Cache đứng trước Workers](https://blog.cloudflare.com/workers-cache/) đã giải quyết gọn ghẽ 2 vấn đề đốt tiền ở trên.

{{< figure 
    src="/posts/revisiting-img-proc-on-cf-worker/cache-in-front-of-worker.png"
    position="center"
    alt="Minh hoạ Cache đứng trước Workers"
    caption="Minh hoạ Cache đứng trước Workers"
    attr="Minh hoạ Cache đứng trước Workers"
    attrlink="https://blog.cloudflare.com/workers-cache/"
    link="https://blog.cloudflare.com/workers-cache/" >}}

Thực tế ra tính năng này đã được request [từ cách đây 6 năm trước](https://community.cloudflare.com/t/cache-in-front-of-worker/171258), và mãi đến bây giờ Cloudflare mới cho ra mắt giải pháp. Theo như [lời trả lời](https://community.cloudflare.com/t/cache-in-front-of-worker/171258/8?) của [Kenton Varda](https://www.linkedin.com/in/kenton-varda-5b96a2a4/) - chủ trương, chủ trì lẫn chủ nhân của dự án Cloudflare Workers - đã nói rằng đây là "pricing issue". Nhưng vật đổi sao rời, giờ Workers đã có cache đứng trước. Có lẽ đúng như anh [Lê Sỹ Cường](https://www.facebook.com/sycule/) đã [chia sẻ](https://www.facebook.com/groups/cloudflarevn/posts/1011147971655596/?comment_id=1011545278282532):

> *"Cloudflare mỗi ngày đều release một tính năng khiến doanh thu của họ có khả năng giảm 🫣"*

Tất cả những gì chúng ta cần làm để bật Cache ở trước Workers là:

- Bật cache lên trong `wrangler.jsonc`:
```jsonc
{
  // ...các config trước...
  "compatibility_date": "2026-07-06", // Đặt compatibility date lên cao để đảm bảo tính năng này được nhận diện
  "cache": {
    "enabled": true
  }
  // ...các config sau...
}
```
- Sau đó, set `Cache-Control` header cho response (lấy ví dụ code đã có ở bài viết trước):
```typescript
    // ... Code đang có

    export default {
        async fetch(request, env, ctx): Promise<Response> {
            const url = new URL(request.url)
            const imageURL = url.searchParams.get("url")
            if (!imageURL) return new Response("url parameter is required", { status: 400 })
            
            // Code mới để kiểm tra cache
            const cache = caches.default
            let response = await cache.match(imageURL, { ignoreMethod: true })
            if (response !== undefined) {
                console.log('cache hit')
                return response
            }
            console.log('cache missed')
        
            response = await getWatermarkedImageResponse(imageURL)

            // Set header để lưu cache 1 năm trên cả browser và trên Cloudflare
            response.headers.set("Cache-Control", "public, max-age=31536000, s-maxage=31536000")

            // Thêm stale-while-revalidate nếu ảnh không cần thiết phải luôn là mới nhất
            // mà chấp nhận một khoảng thời gian outdate
            // response.headers.set("Cache-Control", "public, max-age=3600, s-maxage=3600, stale-while-revalidate=3600")

            // Thêm logic cache tag ở đây để dễ invalidate response
            // sử dụng `ctx.cache.purge({ tags: ["your-tag"] })`;
            response.headers.set("Cache-Tag", "your-tag")

            // Lưu kết quả vào cache nhưng không block việc trả ảnh cho user
            ctx.waitUntil(cache.put(imageURL, response.clone()))

            return response
        },
    } satisfies ExportedHandler<Env>;
```

Quá là dông dài với màn tiểu sử và giải thích rồi, giờ ta sẽ đi đến tiết mục đánh giá những lợi ích mà tính năng này mang lại

## Tiết kiệm tiền Worker invocation khi ảnh được lưu lại trong cache đứng trước Worker

Giờ đây không cần phải tốn mất 1 invocation của worker chỉ để query vào cache của Cloudflare nữa, mà cache đứng trước Workers sẽ trả ngay ra kết quả luôn cho chúng ta trước nếu match cache. Như vậy chỉ tốn một (vài) requets đầu để ghi dữ liệu vào cache, là từ đó không cần phải invoke lại Workers để lấy ảnh từ trong cache ra nữa, tiết kiệm tiền invocation.

## Tiered Cache - Thứ thực sự giải quyết vấn đề cache ở nhiều PoP

Như đã phân tích ở trên, Tiered Cache sẽ là thứ giải quyết vấn đề cache chỉ tồn tại ở 1 PoP - giới hạn cố hữu của worker cache - vấn đề khiến ta đốt rất nhiều CPU time để transform hình ảnh ở nhiều PoP khác nhau, lại còn huỷ hoại trải nghiệm người dùng vì load ảnh chậm.

Nhờ có Tiered Cache, số lượt transform ảnh sẽ giảm đi đáng kể. Thay vì request đến PoP nào sẽ transform ảnh lại từ đầu ở PoP đó, thì khi request đến các lower-tier cache, Cloudflare sẽ ưu tiên tra lại thông tin trên upper-tier cache trước trước khi đấm mồm workers. Như vậy đồng nghĩa với việc ảnh đã được trên 1 PoP sẽ có cơ hội cao được cache tiếp ở 1 PoP khác. Đây chính là selling point khiến ta tiết kiệm tiền CPU time của Workers.

{{< figure 
    src="/posts/revisiting-img-proc-on-cf-worker/tiered-cache.png"
    position="center"
    alt="Minh hoạ Tiered Cache"
    caption="Minh hoạ Tiered Cache"
    attr="Minh hoạ Tiered Cache"
    attrlink="https://blog.cloudflare.com/workers-cache/"
    link="https://blog.cloudflare.com/workers-cache/" >}}


# Những vấn đề còn tồn đọng chưa giải quyết được

Dù giải quyết được vấn đề 
Vẫn còn một vài vấn đề tồn động mà tôi đang đợi các kĩ sư ở Cloudflare giải quyết (dù tôi cũng không rõ việc xử lý ảnh trên Cloudflare có phải là một lựa chọn đúng đắn hay không nữa...)

- Kích thước bộ nhớ của Workers vẫn là 128M, chưa thể xử lý các ảnh có kích thước lớn
- Lỗi `unreachable` khi chạy code WASM xử lý ảnh. Đây thực tế không phải là lỗi của Cloudflare, mà là lỗi của code WASM, nhưng rất khó debug.

# Tóm cái váy lại

Việc có cache đứng trước workers thực sự là tin vui với tôi, là điều tôi đã mong đợi từ lâu tới bây giờ mới được "giải khát". Mong bài viết tào lao này của tôi cho bạn một ví dụ về việc tận dụng Cache đứng trước Worker, cũng như phá vỡ giới hạn của Cloudflare Workers bằng việc xử lý ảnh trên WASM V8 Runtime.

# Dẫn nguồn:

  - [Your Worker can now have its own cache in front of it | Cloudflare](https://blog.cloudflare.com/workers-cache/)