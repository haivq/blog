---
title: "Tạm biệt Sử dụng `libopenshot` để render video"
author: "Aperture"
date: "2025-03-12T14:37:11+07:00"
categories:
    - video
    - video rendering
tags:
    - video
    - video rendering
    - moviepy
    - openshot
    - libopenshot
    - experience
    - python
    - cpp
---

# Mở đầu câu chuyện

Một trong những job gần đây nhất của tôi là làm một renderer để render số lượng lớn các video ngắn [`brain rot`](https://en.wikipedia.org/wiki/Brain_rot) theo định dạng có sẵn để upload chúng lên TikTok, Instagram Reels hay Youtube Shorts, hoặc các video dài dạng tin tức phóng sự kèm TTS để đọc văn bản. Và tất nhiên tôi chọn thư viện phổ biến nhất để làm việc này: [`moviepy`](https://github.com/Zulko/moviepy)

## Trải nghiệm mệt mỏi với `moviepy`

Bản chất `moviepy` là một cái wrapper [`ffmpeg CLI`](https://ffmpeg.org), kèm theo các thư viện xử lý ảnh để xử lý từng frame. Nguyên lý hoạt động của `moviepy` đại loại như sau:

- Đọc cả video vào trong memory, rồi đọc từng frame về dạng `numpy` array, sau đó sử dụng thư viện có thể modify `numpy` array đó để sửa nội dung từng frame. 
- Gửi frame đó thành dạng buffer vào `stdin` của process `ffmpeg` CLI đã được bật từ trước thông qua IPC của hệ điều hành.
- Tiếp tục đọc cho đến khi hết video hoặc đến một khoảng thời gian đã định, sau đó gửi `EOF` để tắt `ffmpeg` và video của chúng ta ra đời.

Cách tiếp cận này khá hay khi ta có thể sử dụng `numpy` và các thư viện có thể thao tác trên `numpy` để chỉnh sửa video. Người dùng chỉ cần biết mỗi Python để có thể làm việc. Nhưng sau một thời gian sử dụng `moviepy`, tôi gặp phải vấn đề quan trọng nhất: **Overhead quá lớn**. Vấn đề này nằm ở bản chất hoạt động của `moviepy`. Việc convert từng frame sang `numpy` array, xong lại từ `numpy` array thành stream gửi ngược vào `ffmpeg` CLI thông qua IPC sẽ mất rất nhiều thời gian. Từ đó việc render video sẽ bị chậm đi nhiều chỉ vì convert data xong serialize chúng thành các định dạng của các môi trường khác nhau.

Ngoài ra còn có một số lý do khác như nhiều effect tôi dùng bị lỗi khi lên `moviepy` v2, nhưng lý do quan trọng nhất khiến tôi bỏ `moviepy` chính là tốc độ. Với yêu cầu ngày càng cao thì việc render chậm là không thể chấp nhận được.

Trong quá trình tìm kiếm giải pháp thay thế cho `moviepy`, tôi tìm ra một thư viện mới sử dụng `libopenshot`: