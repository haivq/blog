---
title: "Một vài kinh nghiệm rút ra sau khi thi và đạt chứng chỉ RHCSA v10"
author: "Aperture"
date: "2026-05-09T23:00:00+07:00"
categories:
    - redhat
    - rhcsa
    - experience
tags:
    - redhat
    - rh
    - system administrator
    - infrastructure
    - sysadmin
    - rhel
    - exam
    - rhcsa
    - experience
---

# Hoàn cảnh

Câu chuyện bắt đầu khi sếp nhắc nhở tôi rằng rằng mỗi nhân viên Red Hat phải có tối thiểu 2 certificate Red Hat đang active, và bắt buộc phải có 1 certificate trước tháng 6 này. Vậy nên hiện tại trong tháng 5 tôi phải ôn và thi lấy một chứng chỉ. Dựa vào công việc của tôi đang làm, thì tôi cần phải thi lấy 3 chứng chỉ sau:

  * [RHCSA (EX200)](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam): Tối thiểu phải có để có nền kiến thức cơ bản để quản lý các máy RHEL
  * [RHCE (EX294)](https://www.redhat.com/en/services/training/ex294-red-hat-certified-engineer-rhce-exam-red-hat-enterprise-linux): Nâng cao hơn từ RHCSA, tự động hoá việc quản lý các máy RHEL thông qua [Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible)
  * [RHCOA (EX280)](https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam): Cũng là nền kiến thức tối thiểu để quản lý cluster OCP

Theo lộ trình tôi sẽ cần phải học toàn bộ các chứng chỉ này, tương lai sẽ phải học thêm 2 chứng chỉ nữa trong [danh sách này](https://www.redhat.com/en/services/certification/rhca#exams) để đạt được level [RHCA](https://www.redhat.com/en/services/certification/rhca).

Nhưng trước mắt vì deadline cuối tháng phải có certificate và công việc khá bận, nên tôi ôn và thi luôn RHCSA cho tiện. Dưới đây sẽ là kinh nghiệm mà tôi thu được từ lần thi RHCSA gần đây nhất.

# Kinh nghiệm khi đi thi

> Lưu ý: Do quy định của hãng nên tôi không được phép tiết lộ nội dung của đề, vui lòng đừng hỏi.

## Chuẩn bị trước khi thi

Vì thi là thi remote, nên bạn sẽ bắt buộc phải chuẩn bị môi trường thi theo yêu cầu của giám thị. Khi bạn đăng kí thi xong thì sẽ có email gửi về để mô tả sơ qua những gì bạn cần chuẩn bị. Nhưng tôi xin liệt kê một số yêu cầu quan trọng sau:

  * Chuẩn bị trước 1 webcam **góc rộng** (kể cả bạn thi bằng laptop và đã có webcam theo máy rồi), đặt ở vị trí có thể thấy được mặt, bàn phím và chuột của bạn. Không cần phải quay màn hình vì lúc thi đã có record màn hình.
  * Lúc thi chỉ cho dùng 1 màn hình, nên nếu bạn có 2 màn hình thì tháo xuống và bê ra chỗ khác. Kể cả việc tắt màn hình đi cũng không được chấp nhận, vì trước boot USB lên kiểm tra máy thì nó cũng sẽ detect được ra là bạn đang cắm 2 màn hình.
  * Không được hút thuốc (KHÁ BỰC), chỉ được uống nước, nhưng được đi vệ sinh (rời khỏi phòng). Tối đa bạn chỉ được đi vệ sinh 10 phút (và sẽ bị tính vào thời gian thi), nên cố giải quyết gánh nặng trong lòng trước cho xong trước khi thi, tránh việc đi quá 10 phút sẽ bị huỷ tư cách thi.
  * Nên tạo USB boot sớm trước khi thi, cắm máy vào check sớm trước hôm thi vài ngày, để đảm bảo máy móc của bạn đạt tiêu chuẩn đi thi, tránh để sát nút thi mới check ra không đạt chuẩn, phải đi mượn máy sẽ bất tiện, mà cũng có khả năng máy đi mượn của bạn không đủ khoẻ hay không đủ tiêu chuẩn để thi. Lưu ý rằng phải thi trên máy x86, máy ARM như máy Mac sẽ không dùng được.
  * Khi dự thi, bạn sẽ phải remote vào workstation của Red Hat để thi, vậy nên hãy đảm bảo internet ổn định trước khi thi. Tốt nhất là dùng mạng dây, tránh dùng mạng điện thoại kẻo bị đứt mạng hay hết tiền giữa chừng.

Xem thêm video sau để hiểu hình thù workstation bên trong khi đi thi cho đỡ bỡ ngỡ:
{{< youtube id=Me6Y12-sux8 caption="Inside a Red Hat Certification Exam: What you need to know" >}}

## Lưu ý version RHEL khi chọn bài thi

Mỗi một phiên bản RHEL sẽ có một chút khác biệt khi vận hành, và trong bài thi cũng phải ánh việc này. Nếu bạn đã quen vận hành RHEL 9 và muốn thi RHCSA v10, thì hãy chú ý tìm các ôn lại kiến thức cho RHEL 10. Tôi đã vận hành với RHEL 7, 8 và 9 khá lâu, nên tự tin thi luôn RHCSA v10 với suy nghĩ rằng, *"chắc chúng nó giống nhau"*, để rồi gặp một câu liên quan đến RHEL 10 mà kiến thức RHEL 9 không áp dụng được.

## Setup SSH key để SSH vào trong các node RHEL để quá trình thi được thuận tiện hơn

Việc này không bắt buộc, nhưng theo tôi, bạn nên tạo một SSH key từ workstation và add nó vào 2 node RHEL khi thi, lúc này bạn sẽ không cần phải nhập password nữa mà SSH phát là được luôn.

## Quản lý màn hình một cách thông minh

Việc sử dụng ChatGPT quá nhiều chắc hẳn đã khiến nhiều anh em ỷ lại vào công cụ trên mạng, cộng thêm việc sử dụng có một màn hình trong khi nhiều người có tới 2 - 3 cái màn hình sẽ đem lại trải nghiệm bí bách và bất lực trong lúc thi. Lúc này bạn cần phải phân chia màn hình thật thông minh để có thể tiện làm 3 việc sau:

  * Một terminal được SSH vào trong node, sẵn sàng gõ lệnh
  * Một terminal khác cũng được SSH vào trong node để sẵn sàng tra man page (bắt buộc thôi, vì làm gì có internet mà hỏi ChatGPT)
  * Một cửa sổ browser để tra cứu documentation của hãng. Documentation của RHEL được cung cấp sẵn dưới dạng [Offline Knowledge Portal](https://access.redhat.com/products/red-hat-offline-knowledge-portal/).

Bằng không bạn sẽ loạn cào cào lên và không biết phải làm gì ở đâu, dễ dẫn đến việc gõ lệnh sai ở một terminal khác, phá nát công sức bạn đã làm từ đầu đến giờ.

## Cẩn thận với câu liệt

Như thời điểm tôi thi, thì trong bài thi sẽ có ít nhất 2 câu điểm liệt. Nếu bạn không làm được câu này, thì toàn bộ các câu sau sẽ không làm được theo luôn. Lúc đi thi bạn sẽ hiểu ý của tôi nói, và các câu này là các câu rất rất rất cơ bản về quản trị hệ thống, mong rằng không có anh em nào vì câu liệt mà trượt luôn cả bài thi.

## Không có phương pháp nào là cố định để giải một bài thi

Khi bạn ôn các course hay xem tutorial ở trên mạng, có thể bạn sẽ thấy cái chuyên gia vẩy tay gõ ra những dòng lệnh dài ngoằng và trúc trắc để config một thứ gì đó, trong khi có một cách dễ hơn, có UI để hoàn thành công việc. Nhưng bạn có biết rằng thực tế giám khảo không quan tâm đến việc bạn hoàn thành bài thi như thế nào không? Hãy vận dụng toàn bộ kiến thức và kinh nghiệm của bạn để sử dụng các công cụ có sẵn để làm bài thi nhanh nhất có thể.

Ví dụ, để config network bạn có thể dùng `nmtui` thay vì `nmcli`. Nhiều tác vụ có thể thực hiện bằng `cockpit` thay vì phải gõ tay.

Tất cả đều là lựa chọn của bạn, sử dụng phương pháp nào bạn quen nhất để làm.

## Dành ra 10-15p cuối để kiểm tra lại toàn bộ bài thi

Kiểm tra lại một lần không chết ai cả. Nếu bài thi quá dễ và bạn hoàn thành nó trước hạn như một vị thần, hãy dành 10-15p để review lại đáp án, đảm bảo các yếu tố:
  * Không làm bài ở nhầm node, đề bài yêu cầu bạn làm node 1 này bạn lại làm ở trên node 2
  * Nếu trong đề bài yêu cầu bạn phải đảm bảo bài làm của bạn không bị mất khi reboot, bạn cũng phải chắc chắn rằng lúc reboot thì tất cả công sức của bạn của bạn không trôi tuột xuống cống

## QUAN TRỌNG NHẤT: THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH, THỰC HÀNH

Vì thi là thi thao tác thật, không phải khoanh trắc nghiệm, nên tôi khuyên rằng bạn nên thực hành nhiều lần trước khi thi, vì theo như kinh nghiệm của tôi, học lý thuyết suông không giải quyết được gì hết, bạn chỉ cần tắt điện thoại hay máy tính, leo lên giường lướt Facebook/TikTok hay đi ngủ là tất cả kiến thức bạn đã ôn cũng sẽ trôi tuột ra khỏi não.

# Kết luận

Ở trên là toàn bộ kinh nghiệm của tôi khi đi thi RHCSA. Theo như tôi đánh giá thì bài thi không quá khó và các câu hỏi đều rất thực tế, không phải dạng đánh đố hay bẫy đề, bẫy câu chữ để cố tình cho bạn trượt, nên nếu bạn đã làm SysAdmin một thời gian rồi thì bài thi này sẽ không thể nào làm khó được bạn. Chúc bạn thành công trong kì thi!

# Dẫn nguồn:

- [How to prepare for Red Hat certification exams | Red Hat](https://www.redhat.com/en/services/certification/exam-preparation)
- [Red Hat Certified System Administrator (RHCSA) exam | Red Hat](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)
- [Red Hat Certified Engineer (RHCE) exam | Red Hat](https://www.redhat.com/en/services/training/ex294-red-hat-certified-engineer-rhce-exam-red-hat-enterprise-linux)
- [Red Hat Certified OpenShift Administrator exam | Red Hat](https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam)
- [Red Hat Training & Certification space | Red Hat](https://access.redhat.com/community/learn)
- [Inside a Red Hat Certification Exam: What you need to know | YouTube](https://www.youtube.com/watch?v=Me6Y12-sux8)
- [How I Scored 300/300 on the RHCSA Exam – 8 Tips You Need to Know! | YouTube](https://www.youtube.com/watch?v=QlAwLcLz8CM)