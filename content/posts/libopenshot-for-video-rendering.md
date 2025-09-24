---
title: "Sử dụng `libopenshot` thay cho `moviepy` để render video"
author: "Aperture"
date: "2025-03-12T14:37:11+07:00"
categories:
    - video
    - video rendering
    - libopenshot
    - moviepy
tags:
    - video
    - video rendering
    - moviepy
    - ffmpeg
    - openshot
    - libopenshot
    - experience
    - python
    - cpp
---

> Blog ngắn chia sẻ câu chuyện là chính, nếu bạn thấy tò mò tôi sẽ viết thêm một bài ví dụ về sử dụng `libopenshot`.

# Mở đầu câu chuyện

Một trong những job gần đây nhất của tôi là làm một renderer để render số lượng lớn các video ngắn [`brain rot`](https://en.wikipedia.org/wiki/Brain_rot) theo định dạng có sẵn để upload chúng lên TikTok, Instagram Reels hay Youtube Shorts, hoặc các video dài dạng tin tức phóng sự kèm TTS để đọc văn bản. Và tất nhiên việc đầu tiên tôi làm là lên Google tìm xem có cái thư viện nào code sẵn hay chưa rồi dùng thẳng luôn cho khoẻ. Cuối cùng tôi tìm thấy thư viện phổ biến nhất để làm việc này: [`moviepy`](https://github.com/Zulko/moviepy)

## Trải nghiệm mệt mỏi với `moviepy`

Bản chất `moviepy` là một cái wrapper [`ffmpeg` CLI](https://ffmpeg.org), kèm theo các thư viện xử lý ảnh để xử lý từng frame. Nguyên lý hoạt động của `moviepy` đại loại như sau:

- Dùng `ffmpeg` để đọc video vào trong RAM, rồi đọc từng frame về dạng `numpy` array, sau đó sử dụng các library có thể modify `numpy` array đó để sửa nội dung từng frame (ví dụ dùng `scipy`, `opencv` hay chính `numpy` luôn, về cơ bản thì đưa về `numpy` rồi thì trăm ngàn cách sửa)
- Gửi frame đó thành dạng buffer vào `stdin` của process `ffmpeg` CLI đã được bật từ trước thông qua IPC của hệ điều hành.
- Tiếp tục chạy cho đến khi hết frame (thường là do hết thời gian của video đã định trước), sau đó gửi `EOF` để tắt `ffmpeg` và video của chúng ta ra đời.

Chính việc biến từng frame thành `numpy` array khiến MoviePy có một thứ vô cùng quý giá: Sự đơn giản và mạnh mẽ. Người dùng chỉ cần biết mỗi Python + 2 neurone não và 1 quả tim vẫn đập để bơm máu là có thể vào việc được rồi. Nhưng gì cũng phải có cái giá của nó, đã đơn giản và mạnh mẽ thì khó đạt được tốc độ cao. Chính điểm mạnh của MoviePy là convert video frame thành `numpy` rồi gửi ngược vào `ffmpeg` tạo ra một overhead cực lớn cho ứng dụng. Tưởng tượng bạn phải dùng 1 process `ffmpeg` để đọc từng frame, rồi gửi nó qua IPC tới process `python` đang chạy, rồi `moviepy` bên trong đọc cái frame đó, convert nó thành `numpy` array, rồi từ `numpy` array lại serialize ra định dạng mà `ffmpeg` có thể đọc được, gửi cho 1 process `ffmpeg` ở ngoài, đến lúc đó `ffmpeg` mới cho ra video. Việc convert qua lại quá nhiều định dạng, convert từng frame một khiến cho nhiều lúc thời gian gửi qua gửi lại cái frame còn tốn hơn là process frame.

Vậy khi yêu cầu của khách hàng lớn, mỗi phút phải render rất nhiều video, thì tôi không thể sử dụng `moviepy` được nữa. Bắt buộc tôi phải tìm tới một giải pháp nào nhanh hơn, ít overhead hơn. Tôi đã thử sử dụng `libavcodec`, nhưng có vẻ overkill quá, tôi phải implement code đọc, handle số luồng, tất cả mọi thứ từ đầu trong khi tiền của project đó không đủ để tôi làm vậy. Từ `libavcodec` tôi tìm ra một thư viện mới wrap lại nó, chính là `libopenshot`

# `libopenshot` là gì?

Project [libopenshot](https://github.com/OpenShot/libopenshot) thực chất là thư viện lõi của một project lớn hơn: [openshot-qt](https://github.com/OpenShot/openshot-qt) hay là ứng dụng edit video open source [OpenShot](https://www.openshot.org). Bản chất `libopenshot` là một thư viện C++, nhưng điều may mắn với tôi là, [Johnathan](https://github.com/jonoomph) đã làm binding cho các ngôn ngữ như Java, Ruby và, thật may mắn làm sao, Python. Chính project `openshot-qt` thực chất là một app Python Qt wrap lại `libopenshot`.

## Ưu điểm:

- C++, nhanh, mạnh
- Cấu trúc code ít overhead đi rất nhiều. Sau khi data được đọc là hoàn toàn xử lý trong in-memory, không còn phải sử dụng IPC nữa
- Ý tưởng code sử dụng Clip, Timeline khá hay ho và dễ hiểu
- Có hẳn một ví dụ to khổng lồ chính là project `openshot-qt` cho bạn tha hồ tìm hiểu.

## Nhược điểm:

- Bindng của Python hoàn toàn không có document, bạn phải đọc doc của C++ và đối chiếu sang bên Python, việc này gây khó cho người nào không quen.
- Sử dụng SWIG binding nên GC của Python không play nice với data binding của C++. Giả dụ bạn tạo ra Clip, thêm nó vào Timeline mà không save cái Clip đó vào đây khi thoát scope function, thư viện sẽ bắn ra SegFault. Tôi đã tốn rất nhiều thời gian để hiểu ra vấn đề này, cùng với đó tôi cũng hiểu binding SWIG mà `libopenshot` sử dụng
- Vấn đề đóng mở các stream một cách chính xác là rất quan trọng. Việc bạn mở file và load nó vào memory, nhưng không Close nó đúng cách sẽ khiến video render ra đen thui mà không có bất kì một chỉ báo nào cả
- Không thể modify frame một cách đơn giản được nữa. `libopenshot` không expose frame data ra chỗ nào để bạn có thể modify nó trong Python. Bạn bắt buộc phải fork thư viện ra, tạo một Effect mới, và build lại thư viện. Việc này có thể không vui vẻ gì vì thực tế số lượng Effect trong `libopenshot` không nhiều lắm, chính tôi cũng phải contribute 2 effect mới là `Shadow` và `Outline` (`Outline` đã được merge vào code chính nhưng `Shadow` thì chưa).

Với nhiều người thì những nhược điểm của `libopenshot` có thể không đáng cho họ thay đổi, nhưng với tôi thì reward của `libopenshot` quá lớn so với những bất tiện nó mang lại, nên là tôi quyết định thay đổi.

# Tóm lại

Sau khi deliver ứng dụng render cho khách hàng, họ rất bất ngờ khi code `libopenshot` render rất nhanh (chỉ mất 8-20s trong khi dùng `moviepy` sẽ mất 10-120s). Nhưng khi họ thấy code của tôi thì có vẻ họ không hài lòng cho lắm, dù nhanh hơn nhiều, giúp họ giảm TCO nhưng có vẻ nhanh và rẻ thôi là chưa đủ với họ, mà cần phải dễ hiểu, phổ biến và dễ extend nữa, mà những điều này thì cần phải có một chút hiểu biết về C++ mới được. Kết thúc project thì họ có thể render cả đống video trên mạng và tích hợp vào hạ tầng render video Shorts/Reels/TikTok. Câu chuyện này làm tôi suy nghĩ, liệu nhanh hơn có tốt, hay ưu tiên feature trước mới là điều quan trọng nhất?