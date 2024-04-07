---
title: "Những kinh nghiệm tôi có khi sử dụng SvelteKit làm fullstack webapp"
author: "Aperture"
date: "2024-04-07T00:20:00+07:00"
categories:
    - svelte
    - sveltekit
tags:
    - svelte
    - sveltekit
    - experience
    - fullstack
    - webapp
draft: true
---

# Đặt vấn đề

Với một người chỉ code backend, ngủ cạnh server, đêm canh cluster, sáng soi monitoring, thì việc code frontend không hề dễ dàng tí nào. Nếu cách đây 3 năm trước tôi vẫn hay bỉ bôi người code frontend, bảo frontend dễ bỏ mẹ, JS/TS là ngôn ngữ đồ chơi, thì hiện thực đã vả cho tôi một cú tát trời giáng khi solo code một cái webapp từ A đến Z. Hoá ra code frontend không hề dễ một chút nào cả, và câu "frontend dễ" là một cú lừa ngoạn mục của mấy ông code backend béo ị nghĩ rằng việc cắm cắm cái PostgreSQL vào một đống code Spring Boot rác rưởi mới là võ thuật thực sự.

Một vấn đề nữa là, khi chọn Cloudflare Workers để làm backend runtime, Cloudflare Pages làm frontend hosting, việc chọn JS/TS để code hết cả frontend lẫn backend là nên làm bởi:

- Cloudflare Workers support native JS, code JS/TS backend cùng frontend luôn cho tiện và cho dễ debug
- Tôi có nhiều thời gian để tìm hiểu kĩ về JS/TS hơn
- Tái sử dụng được thư viện và các component giữa frontend và backend
- Dễ dàng kiếm người code cùng về sau vì JS/TS khá phổ biến

Ngoài các yêu cầu ở trên, còn có các tiêu chí phụ dưới đây:

- Dễ dùng: Vì tôi không có nhiều thời gian để đi học kĩ càng một thư viện
- Có tính năng reactive: Làm vậy để tương tác các event dễ hơn

Dưới đây là một vài ý tưởng về frontend mà tôi đã tham khảo qua:

1. ReactJS: Thư viện frontend vô cùng phổ biến, nhà nhà học, người người biết. Khi tôi hỏi bất kì ai về việc chọn một library hay framework để code frontend, thì phần lớn mọi người đều gợi ý tôi ReactJS.
2. Code HTML chay + jQuery: Tôi từng có một thời gian sử dụng cách này để code JSP hồi còn code app enterprise cho một công ty viễn thông. Không còn nhiều người sử dụng công nghệ cổ lỗ sĩ này để code frontend, nhưng về API thì khá stable rồi.
3. Svelte: Một framework khá mới để code fullstack web, dù thời điểm tôi bắt đầu code là Svelte 3 nhưng cộng đồng vẫn còn khá vắng. Document và tutorial chủ yếu là từ chính [trang chủ của Svelte](https://svelte.dev).

Dưới đây là so sánh giữa các nền tảng này:

