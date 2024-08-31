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
> Do Cache của Cloudflare chỉ tồn tại ở trong 1 data center, nên số lượng request có thể bị nhân lên, ví dụ khách đã truy cập vào PoP Singapore và ảnh đã cache tại PoP của Singapore, thì khách khác gọi vào từ PoP Hong Kong thì Cloudflare sẽ lại gọi vào R2 lần nữa. Để đoán định chính xác thì rất khó, nên giả sử số lượng request ở tất cả các PoP sẽ gấp 3 lần số lượng request thực tế, ta có:
```
get_object_cost = 8M * ($0.36/1M) * 3
                = 8 * $0.36 * 3 = $8.64
```

Vậy, tổng phí cố định hàng tháng sẽ là: `$16.14`

Dưới đây ta sẽ tính các biểu phí đặc trưng của từng phương án

#### Phí lưu trữ một bản ảnh đã dán watermark ở bucket khác

Phí lưu trữ phiên bản ảnh đã dán watermark:
> Giả sử ảnh biến hết thành `webp` với kích thước nhỏ hơn khoảng 40%
```
watermarked_storage_cost = storage_cost * 0.7
                         = $7.5 * 0.6 = $4.5
```

Vậy tổng phí nếu lựa chọn phương án lưu thêm phiên bản khác là `$20.64`.
> Nếu lưu thêm các phiên bản khác nhau nữa, cần phải tính lại cả số lượng request của phiên bản mới để cộng lại, tạm thời ta chỉ tính đến phương án lưu 1 phiên bản

#### Phí sử dụng Workers để convert ảnh:

Phí Cloudflare Workers Paid cố định: `$5` (do 8M request chưa vượt quá 10M request hàng tháng)

Vì Cloudflare có dịch vụ Cloudflare Image Transformation, vừa cache được ảnh đã transform mà vừa tiết kiệm được CPU time, ta sẽ tính thử phí để convert và đầy vào cache như sau:
> Công thức tính sẽ gần giống như tính GetObject ở trên, nhưng nhân với [giá Image Transform](https://developers.cloudflare.com/images/pricing/#images-transformed)
```
cf_image_transform_cost = 8M * ($0.5 / 1k) * 3
                        = 8k * $0.5 * 3
                        = $12000 (Siêu đắt ?!!!)
```

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/shotgun-bill.gif"
    position="center"
    alt="$12000 mỗi tháng?? Một số tiền quá lớn!"
    title="$12000 mỗi tháng???? Một số tiền quá lớn!"
    caption="$12000 mỗi tháng???? Một số tiền quá lớn!" >}}

Có vẻ hướng sử dụng Cloudflare Image Transformation là quá đắt... Ngay cả việc sử dụng Cloudflare Images cũng nói luôn rẳng việc chứa 20 variants [không bao gồm việc vẽ thêm watermark](https://developers.cloudflare.com/images/pricing/#images-stored), vậy nên ta nên chọn phương pháp khác rẻ tiền hơn: Sử dụng thư viện [@cf-wasm/photon](https://www.npmjs.com/package/@cf-wasm/photon) để tối ưu ảnh. Việc này sẽ ảnh hưởng tới việc sử dụng CPU time, và ta sẽ tính nó vào phần sau. 

# Thử nghiệm và đánh giá phương án dùng Cloudflare Workers và Photon

Ta sẽ khảo sát tính khả thi của phương án thông qua:
- Thời gian xử lý của function (lấy p95 hoặc p99)
- Lương CPU tiêu tốn

## Implement của tôi

Trước khi chạy, ta tạo project workers và cài [@cf-wasm/photon](https://www.npmjs.com/package/@cf-wasm/photon):

```sh
npm create cloudflare@latest -- cf-workers-watermark # Lưu ý sử dụng typescript
cd cf-workers-watermark
npm i @cf-wasm/photon
```
Sau đó, trong `src/index.ts`, ta tạo file có nội dung như sau:

```typescript
import { PhotonImage, watermark } from "@cf-wasm/photon";

let WATERMARK_IMAGE: PhotonImage | null = null

const _getWatermarkImage = async () => {
	if (!WATERMARK_IMAGE) {
		// NOTE: 
		// Temporally use lorem picsum.
		// In reality I use a watermark image from a base64 string and use
		// `PhotonImage.new_from_base64` to create the image.
		const resp = await fetch('https://picsum.photos/id/237/200/200')
		console.log('watermark loaded')
		WATERMARK_IMAGE = PhotonImage.new_from_byteslice(new Uint8Array(await resp.arrayBuffer()))
	}
	return WATERMARK_IMAGE
}

const getWatermarkedImageResponse = async (url: string): Promise<Response> => {
	const imageResponse = await fetch(url)
	const image = PhotonImage.new_from_byteslice(new Uint8Array(await imageResponse.arrayBuffer()))
	const watermarkImage = await _getWatermarkImage()
	const xOffset = BigInt(image.get_width() - watermarkImage.get_width())
	const yOffset = BigInt(image.get_height() - watermarkImage.get_height())
	watermark(image, watermarkImage, xOffset, yOffset)
	const response = new Response(image.get_bytes_webp(), imageResponse)
	response.headers.set("Content-Type", "image/webp")
	image.free()
	return response
}

export default {
	async fetch(request, env, ctx): Promise<Response> {
		const url = new URL(request.url)
		const imageURL = url.searchParams.get("url")
		if (!imageURL) 
			return new Response("url parameter is required", { status: 400 })

		let response = await getWatermarkedImageResponse(imageURL)

		return response
	},
} satisfies ExportedHandler<Env>;


```

Sau đó ta chạy lệnh sau để bật server dev:
```sh
npm run dev
```

Để dùng thử, gọi vảo URL có định dạng như sau: `http://localhost:8787/?url={link ảnh bạn muốn}`, giả sử: `http://localhost:8787/?url=https://picsum.photos/1000/1000`

Sau khi gọi liên tục vào server dev, tôi đo được mỗi request sẽ mất khoảng 0.5s tới 2s.

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/request-time.png"
    position="center"
    alt="Thời gian cho một request"
    title="Thời gian cho một request"
    caption="Thời gian cho một request" >}}

Những yếu tố ảnh hưởng tới thời gian request ở đây bao gồm:
1. Việc tải ảnh từ host ngoài
2. Load ảnh và gắn water
3. Tạo response gửi về cho user

Trong 3 yếu tố trên, ta có thể giảm thiểu 2 yếu tố `1` và `2` nếu ta cache được request ảnh gửi về. Ta sẽ sử dụng Cache API có sẵn của Cloudflare để giản lược 2 bước trên:

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

        // Lưu kết quả vào cache nhưng không block việc trả ảnh cho user
		ctx.waitUntil(cache.put(imageURL, response.clone()))

		return response
	},
} satisfies ExportedHandler<Env>;

```

Sau khi thực hiện bước cache, ta thử gọi lại vào URL đã nêu ở trên và kết quả khá lúc này rất nhanh:

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/request-time-cached.png"
    position="center"
    alt="Thời gian sau khi cache"
    title="Thời gian sau khi cache"
    caption="Thời gian sau khi cache" >}}

Giờ, ta deploy lên Cloudflare. Để tận dụng Cache API, ta cần phải có 1 custom domain cho nó. Mở file `wrangler.toml` và thêm dòng sau để bật custom domain của bạn (bắt buộc domain phải đang có nameserver của Cloudflare):

```toml
routes = [
  { pattern = "some.domain.com", custom_domain = true }
]
```

Sau đó chạy lệnh sau để deploy

```sh
npm run deploy
```

Để sinh nhiều request test random, tôi sử dụng đoạn script python đơn giản sau:

```python
import requests
import random
from time import sleep, perf_counter

size_list = list(range(1000, 2100, 100))

for i in range(int(1e4)):
    random_width = random.choice(size_list)
    random_height = random.choice(size_list)
    
    url = f'https://some.domain.com/?url=https://picsum.photos/{random_width}/{random_height}'
    start_time = perf_counter()
    resp = requests.get(url)
    end_time = perf_counter()
    print(url, resp.status_code, end_time-start_time)
    sleep(0.05)
```

Ta sẽ được các output kết quả đại loại thế này khi chạy (theo thứ tự: url status_code elapsed_time):

```
https://some.domain.com/?url=https://picsum.photos/1700/1200 200 2.2869445839987748
https://some.domain.com/?url=https://picsum.photos/1400/1600 200 3.347536041999774
https://some.domain.com/?url=https://picsum.photos/1400/1800 200 1.1148227079993376
https://some.domain.com/?url=https://picsum.photos/1200/1900 200 1.0733135840000614
https://some.domain.com/?url=https://picsum.photos/1800/1600 200 3.2671381249983824
https://some.domain.com/?url=https://picsum.photos/1100/1000 200 3.0311982500006707
https://some.domain.com/?url=https://picsum.photos/1400/1300 200 2.766320209000696
https://some.domain.com/?url=https://picsum.photos/1300/1800 200 3.537811291000253
...
```

Theo như code trên, ta sẽ có 100 variant ảnh khác nhau (chọn 1 kích thước lấy chiều dài và chiều rộng của ảnh). Theo lý thuyết, ta sẽ phải lấy mẫu là 1600 request để đạt được phân bố chuẩn cho các thông số trong request (để tính p99, p75 và p50). Nhưng khi tính trừ hao khi Cloudflare miss cache, ta nên lấy nhiều hơn, vừa đảm bảo các trường hợp lỗi, hư hao, vừa đảm bảo biến ngẫu nhiên `CPU TIME` hội tụ về phân bố chuẩn.

Sau khoảng 2 tiếng chạy với 4k requests, ta thu được thống kê như sau:

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/cf-dashboard-result.png"
    position="center"
    alt="Kết quả thu được trên Cloudflare"
    title="Kết quả thu được trên Cloudflare"
    caption="Kết quả thu được trên Cloudflare" >}}

Mặc dù p50, p75 và p99 có vẻ khá cao, nhưng ta phải xét đến trường hợp ban đầu mất thời gian để cache ảnh vào Cache API. Sau một thời gian khi tất cả ảnh đã nằm trong cache, các giá trị p50, p79, p99 sẽ hội tụ xấp xỉ 1ms. Với giá thành hiện tại $0.02 cho mỗi 1M mCPU, ta có thể tạm tính theo giả định nhau sau (gấp 3 số lượng request do Cache của Cloudflare không sync qua các PoP):

```
request_caching_cost = 400ms * 8M * $0.02 / 1M * 3
                     = $192
```

Như vậy, đây là một kết quả khá là khả quan, khi cũng resize và optimize 8M ảnh thì chỉ mất có `$192`, tức là chỉ bằng 1.6% nếu so với việc dùng Cloudflare Images Transform.

## Bất cập

Tất nhiên, không có bữa trưa nào là miễn phí cả. Bây giờ chúng ta sẽ bắt đầu đánh giá đến những bất cập mà nó mang tới.

### Thời gian tạo ảnh ban đầu rất lâu

{{< figure 
    src="/posts/image-transform-with-cf-workers-experience/cf-dashboard-wall-time.png"
    position="center"
    alt="p50, p75 và p99 khá cao"
    title="p50, p75 và p99 khá cao"
    caption="p50, p75 và p99 khá cao" >}}

Như ta thấy, khoảng thời gian ban đầu khi đưa optimize ảnh và đưa vào cache, tới 50% số request sẽ mất tới 400ms. Nếu trang của người dùng chỉ có 4 5 cái ảnh thì không sao, nhưng nếu trang của họ phần lớn là ảnh thì sẽ tạo ra trải nghiệm rất tệ cho người dùng.

### Cache chỉ tồn tại ở 1 PoP

Điều này là điều khiến tôi cân nhắc có nên sử dụng Cloudflare Workers nhiều nhất. [Theo như document này](https://developers.cloudflare.com/workers/runtime-apis/cache/), mỗi khi người dùng request tới PoP của Cloudflare, code Workers ở trên sẽ lại tốn CPU để tạo lại ảnh. Thực sự chưa có cái gì đo đạc được vấn đề này cả, và đây là điều mà tôi lấn cấn nhất vì nếu có đông khách ở nhiều nơi, tiền có thể bị đội lên cao vọt lên nữa mà không có cách nào ngăn chặn được cả.

# Tóm lại

Đây là bài viết nói về suy nghĩ của tôi khi cần resize ảnh ở trên hệ thống serverless của Cloudflare. Mặc dù về giá không được rẻ cho lắm, nhưng về độ tiện lợn thì tốt hơn các giải pháp mà tôi đã biết. Có lẽ về khoản watermark, ta nên code 1 dòng JS để cho browser tự dán lên ảnh và mặc kệ đời ra sao thì ra, giống như cách mà Shoppee đang dán ảnh quảng cáo sale vào ảnh sản phẩm, rồi tập trung não để xây một sản phẩm có giá trị cốt lõi to hơn chứ không chỉ nằm trên vài cái ảnh.

# Tư liệu tham khảo

- [Cloudflare Docs](https://developers.cloudflare.com)
    - [R2 / Pricing](https://developers.cloudflare.com/r2/pricing/#r2-pricing) / [archive](https://web.archive.org/web/20240827210539/https://developers.cloudflare.com/r2/pricing/#r2-pricing)
    - [Cloudflare Image Optimization / Pricing](https://developers.cloudflare.com/images/pricing/#images-transformed) / [archive](https://web.archive.org/web/20240810001916/https://developers.cloudflare.com/images/pricing/#images-transformed)
    - [Workers / Runtime APIs / Cache](https://developers.cloudflare.com/workers/runtime-apis/cache/) / [archive](https://web.archive.org/web/20240822091425/https://developers.cloudflare.com/workers/runtime-apis/cache/)
- [GitHub](https://github.com)
    - [fineshopdesign/cf-wasm](https://github.com/fineshopdesign/cf-wasm)
