---
title: "Xử lý vấn đề `Waiting for table metadata lock` khi `ALTER TABLE` trên InnoDB, MySQL 8"
author: "Aperture"
date: "2024-07-17T18:30:00+07:00"
categories:
    - MySQL
    - InnoDB
    - Database
tags:
    - rds
    - rdbms
    - aws
    - lambda
    - mysql
    - innodb
    - experience
---

> Lưu ý: Bài viết này áp dụng với MySQL 8, vậy nên không chắc mọi thứ vẫn sẽ hoạt động đúng với MySQL 5.7 trở xuống.

# Mở đầu câu chuyện

Vấn đề ở trên gặp phải khi tôi cần phải chạy lệnh sau để update Character Set và Collation từ `latin` sang `utf8` của một bảng trong MySQL. Đúng ra tôi nên đặt Collation và Character Set thành `utf8`/`utf8_unicode_ci` ngay từ đầu, nhưng do sơ suất nên giờ phải convert như thế này:

```sql
ALTER TABLE `some_table`
CONVERT TO CHARACTER SET `utf8`
COLLATE `utf8_unicode_ci`
```

Câu query này lâu một cách bất thường mãi không thấy xong trong khi bảng chỉ có khoảng 20k records. Khi chạy lệnh `SHOW FULL PROCESSLIST` thì ngoài câu query `ALTER TABLE` ở trên thì có một loạt câu query vào bảng con có foreign key tới bảng trên cùng bị trạng thái `Waiting for table metadata lock`:

{{< figure 
    src="/posts/alter-table-metadata-lock/waiting-for-table-metadata-lock.png"
    position="center"
    alt="Một loạt query đang đợi metadata lock"
    caption="Một loạt query đang đợi metadata lock" >}}

Tôi để câu query này 15 phút, nhưng càng ngày càng có nhiều query bị block lại. Tức là không chỉ mình bảng mà tôi đang thao tác bị lock, mà tất cả các bảng con có foreign key trỏ tới bảng này cũng bị lock theo luôn. Điều này rất không ổn với một hệ thống đang chạy production, nhất là khi bảng con đang bị liên luỵ đều là các bảng tối quan trọng, được sử dụng rất nhiều trong hệ thống. Locking 15 phút sẽ chết rất nhiều các function Lambda vốn chỉ có giới hạn thời gian sống tối đa 15 phút. Không còn cách nào khác, tôi phải ngừng câu query `ALTER TABLE` ở trên và tìm giải pháp khác.

# Các phương án không thành công

Để cố gắng giải quyết vấn đề trên, tôi đã thử các phương án sau:

## Tạm thời tắt `foreign_key_checks`

Vì các bảng con bị ảnh hưởng, tôi chạy lệnh sau để tắt `foreign_key_checks` với hi vọng khi chạy query sửa bảng ở trên sẽ không block các bảng con [theo document này](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-checks):

```sql
SET foreign_key_checks = 0;
```

Nhưng sau đó khi chạy `ALTER TABLE` ở trên, cả bảng cha lẫn bảng con vẫn bị lock. Lý do là vì câu query trên chỉ thay đổi việc check trong session hiện tại, còn các session khác vẫn sẽ bị ảnh hưởng. Vậy là phương án này thất bại.

## Nỗ lực sử dụng Online DDL

Online DDL là một tính năng giúp thay đổi cấu trúc bảng trong khi đang có tác vụ trong bảng đó. Theo như [document MySQL này](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations), khi đặt `ALGORITHM=INPLACE, LOCK=NONE` vào `ALTER TABLE`, câu query có thể chạy mà không gây locking bảng. Vậy nên tôi đã sửa câu query trên như sau:

```sql
ALTER TABLE `some_table`
CONVERT TO CHARACTER SET `utf8`
COLLATE `utf8_unicode_ci`,
ALGORITHM=INPLACE, LOCK=NONE
```

Nhưng ngay sau khi chạy câu query, lỗi sau đây xảy ra:

```
ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported.
Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

Hoá ra trong document cũng đề cập rằng, việc thay đổi character encoding sang một loại khác là phải build bảng lại từ đầu. Việc này cũng sẽ dẫn đến blocking như ở trên nếu để `ALGORITHM=COPY`.

Kể cả việc không dùng `CONVERT TO` như sau cũng gây ra cùng một lỗi:

```sql
ALTER TABLE `some_table`
CHARACTER SET `utf8`
COLLATE `utf8_unicode_ci`,
ALGORITHM=INPLACE, LOCK=NONE
```

## Các phương pháp can thiệp đến data
 
Có một vài phương pháp khác mà tôi đã thử mà can thiệp vào data, nhưng tất cả đều lỗi như nhau:

- `SET foreign_key_column=NULL` cho các cột foreign key bảng con rồi chạy `ALTER TABLE`, vẫn gây locking.
- Backup data xong xoá trắng bảng cha rồi `ALTER TABLE`, vẫn bị lock.
- Thử drop foreign key ở bảng con: mất quá nhiều thời gian và vẫn gây lock, rủi ro lại cao.

Sau một loạt các nỗ lực "giải quyết nhanh" không thành công, tôi quyết định đào sâu hơn vào gốc của vấn đề gặp phải thay vì tiếp tục tìm những phương án chỉ tiếp cận phần ngọn như ở trên.

# Lý do xảy ra

Thực tế còn một phương án cuối là restart lại MySQL, nhưng việc restart MySQL trong môi trường production là rất không nên (thực tế tôi cũng không có quyền restart RDS MySQL), nên chúng ta đào sâu kĩ hơn vấn đề. 

## Về `metadata lock`

Vấn đề đầu tiên ta gặp phải, chính là nội dung Query status ở trên: `Waiting for table metadata lock`. Vậy câu query này đang đợi bảng `some table` được lock lại, nhưng có vẻ đang có một operation nào đó chặn việc tạo `metadata lock`.

Để xé lẻ vấn đề ra, ta có 2 gạch đầu dòng sau cần phải thực hiện:
- `metadata lock` là gì?
- Việc gì ngăn cản `metadata lock` xảy ra?

## `metadata lock` là gì?

Theo như [document của MySQL](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-acquisition), thì `metadata lock` là một dạng mutex để khoá cấu trúc bảng lại, để đảm bảo chỉ có 1 session trong MySQL được can thiệp vào cấu trúc bảng. `metadata lock` xảy ra khi người dùng thực hiện các câu query DDL (Data Definition Language), và query `ALTER` được tính là DDL query. Cũng như trong [document này](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release), khi có bất kì transaction nào chưa hoàn thành đang thao tác trên bảng đều sẽ cản trở việc tạo ra `metadata lock`. Và khi query các bảng con có foreign key tới bảng cha đang cần `ALTER TABLE` mà đang bị block ở trên, tất cả các query đều sẽ lock theo do `foreign_key_checks = 1`

Vậy ta có sơ đồ đơn giản sau để mô tả vấn đề đợi dắt dây gặp phải khi `ALTER TABLE`:

{{< figure 
    src="/posts/alter-table-metadata-lock/alter-table-metadata-lock.drawio.png"
    position="center"
    alt="`ALTER TABLE` diagram"
    caption="`ALTER TABLE` diagram">}}

## Việc gì ngăn cản `metadata lock` xảy ra?

Trước hết, tôi thử chạy `SHOW FULL PROCESSLIST` xem có câu query nào đang chạy hay không, và thực tế không có câu query nào đang chạy cả:

{{< figure 
    src="/posts/alter-table-metadata-lock/show-full-processlist.png"
    position="center"
    alt="SHOW FULL PROCESSLIST hiển thị không có query nào đang chạy cả"
    caption="SHOW FULL PROCESSLIST hiển thị không có query nào đang chạy cả" >}}

Nhưng một transaction tồn tại không có nghĩa là phải có query xảy ra trong nó. Có thể transaction đang được để rảnh nhưng chưa có ai `ROLLBACK`, `COMMIT` transaction hay session chứa transaction đó chưa bị kill. Do đó ta cần query vào bảng `information_schema.innodb_trx`, liệt kê tất cả các transaction đang chạy:


```sql
SELECT * FROM information_schema.innodb_trx
```

Ta sẽ có kết quả như sau:

{{< figure 
    src="/posts/alter-table-metadata-lock/innodb-trx.png"
    position="center"
    alt="Danh sách các transaction trong InnoDB"
    caption="Danh sách các transaction trong InnoDB" >}}

Như ở bảng trên, ta thấy có rất nhiều các transaction đang chạy. Nếu một trong các transaction này đã/đang can thiệp vào bảng cha đang cần `ALTER TABLE` hoặc một trong các bảng con của nó, các query về sau mà liên quan tới bảng cha và các bảng con trong lúc đang `ALTER TABLE` bảng cha cũng sẽ bị block lại. Vậy để cho câu query `ALTER TABLE` có thể chạy được, ta phải kết thúc transaction đang bị block.

> Tham chiếu cột `trx_mysql_thread_id` ở bảng `innodb_trx` với cột `id` trong `PROCESSLIST`, ta thấy tất cả các ID ở trong `trx_mysql_thread_id` tồn tại trong `PROCESSLIST`, và trong `PROCESSLIST` thì các session trong đó đều đang `sleep`, điều đó củng cố luận điểm "transaction tồn tại không có nghĩa là phải có query xảy ra trong nó".

# Giải pháp

Để kết thúc một transaction, ta cần chạy lệnh `ROLLBACK` hoặc `COMMIT` ở trong session đang chạy transaction đó. Nhưng MySQL là một tài nguyên chung, chúng ta không thể can thiệp vào 1 session đang chạy ở một nơi khác, mà ta chỉ có thể cancel transaction đó đi mà thôi.

Hiện tại tôi chưa có cách cancel nào hay hơn việc kill session đang chứa transaction đó.

Để kill một session trong MySQL, ta chạy lệnh `KILL` với `<session_id>` chính là `trx_mysql_thread_id`:

```sql
KILL <session_id>
```

Nhưng trong RDS, AWS không cho chúng ta chạy lệnh `KILL`, mà cung cấp cho chúng ta procedure để thực hiện việc đó. Tôi đang dùng RDS, nên chạy lệnh sau để kill session:

```sql
CALL mysql.rds_kill(<session_id>);
```

Không có đường tắt nào cho bạn để xem nên kill hay nên bỏ qua session nào, vì chẳng có thông tin gì thêm về các session. Ta buộc phải chọc vào `performance_schema` của MySQL để xem query history của transaction. Để làm việc này ta có thể tham khảo [bài viết này của Percona](https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/).

Do mặc định RDS tắt `performance_schema` và tôi không có quyền bật nó lên, tôi đành phải kill từng session một rồi chạy thử `ALTER TABLE` xem có chạy không cho chắc.

Sau khi kill đúng session, cuối cùng tôi đã có thể chạy câu lệnh `ALTER TABLE` một cách thoải mái mà không cần phải đợi quá lâu.

# Ngăn ngừa

Như ở trên đã nêu, lý do xảy ra việc `Waiting for table metadata lock` là do transaction chưa được kết thúc khi không sử dụng. Do DB này là DB OLTP nên việc có các transaction lâu tới vài phút như vậy là không ổn. Việc để các transaction hanging còn gây tiêu tốn tài nguyên của MySQL, chiếm nhiều bộ nhớ để chứa những thứ đại loại như transaction history và rollback log, chứ không chỉ mỗi gây block các DDL query (vốn hiếm khi xảy ra). Vậy để tránh vấn đề này, cách tôi giải quyết đơn giản là `ROLLBACK` hoặc `COMMIT` sau mỗi lần query DB. Percona Blog đã có một [bài viết khá hay](https://www.percona.com/blog/small-changes-impact-complex-systems-mysql-example/) về vấn đề này (dù nội dung về MySQL 5.6 đã tương đối cũ nhưng vẫn còn đúng với MySQL 8+ do về bản chất vấn đề vẫn không thay đổi).

Theo kinh nghiệm của tôi thì dưới đây là 2 + 1 cách để quản lý transaction:

## Quản lý transaction bằng thư viện/công cụ

Ngoài việc đóng/mở transaction một cách có trách nhiệm, ta có thể quản lý bằng việc sử dụng Connection Pool có kèm theo quản lý transaction (như HikariCP + JOOQ chẳng hạn) hoặc dùng RDS Proxy để AWS quản lý hộ chúng ta. Sau một khoảng thời gian transaction không có động tĩnh gì, thì ta `ROLLBACK` hoặc `COMMIT`.

## Định kì kill các connection chứa transaction chạy quá lâu

Như mục [Việc gì ngăn cản `metadata lock` xảy ra?](#việc-gì-ngăn-cản-metadata-lock-xảy-ra), ta có thể query ra các transaction đang ngồi lại quá lâu và dễ gây blocking cho CSDL. Giải pháp là `SELECT` các transaction sống lâu hơn 1 khoảng thời gian, rồi kill connection chứa nó đi. 

Một phương pháp _ngây thơ_ để `SELECT` các transaction đang sống quá 45 phút lâu mà tại thời điểm đó đang không làm gì là:

```sql
SELECT trx_mysql_thread_id, TIMESTAMPDIFF(SECONDtrx_started, NOW()) AS living_seconds
FROM information_schema.innodb_trx
WHERE living_seconds > 2700
  AND trx_operation_state IS NULL
  AND trx _query IS NULL
  AND trx_tables_in_use = 0
```

Sau đó ta có thể sử dụng `KILL <thread_id>` hoặc `CALL mysql.rds_kill(<thread_id>)` để kết thúc connection đang treo CSDL lại. Đưa code trên vào cronjob ta sẽ có 1 job định kì quét các transaction để trống quá lâu.

Nên nhớ rằng phương pháp nay quá đơn giản để giải quyết vấn để transaction lock. Để đưa ra quyết định chính xác hơn nên kill transaction nào, hãy xem [bài viết này của Planet MySQL](https://planet.mysql.com/entry/?id=5988591).

## QUAN TRỌNG NHẤT: Sử dụng transaction có trách nhiệm, luôn kết thúc transaction ngay sau khi không sử dụng nữa

Dù có bao nhiêu phương pháp hay bao nhiêu công cụ đi nữa, tất cả chỉ là phao cứu hộ để giúp chúng ta cầm máu sau khi tự bắn vào chân mình, chứ không phải vì có _thuốc_ chữa rồi thì cứ mở transaction xả láng rồi mặc kệ nó sống chết ra sao thì ra. Phòng bệnh thì luôn tốt hơn chữa bệnh, và chữa bệnh lúc nào cũng bung bét và dễ có tác dụng phụ hoặc gây di chứng. Dưới đây là hai vấn đề dễ thấy nhất khi dùng connection pool và kill định kì transaction:
- Với connection pool, lợi thế chính của nó là multiplexing (mux) nôm na là trộn nhiều câu query vào với nhau rồi chạy, nhưng không phải các query đều có thể mux. Việc ta query lẫn lộn vào với nhau hoàn toàn có thể khiến việc mux không thể thực hiện được. Ví dụ như `INSERT` vào CSDL xong `SELECT` lại nó ra, khiến Connection Pool phải hold riêng một Connection để `INSERT` còn câu `SELECT` sẽ phải chạy sau. Nếu dùng không khôn ngoan thì cuối cùng vẫn gây ra vấn đề hết connection như ở trên. Vấn đề này đã được nêu trong [document của RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-managing.html#rds-proxy-pinning). Mặc dù không khiến Database lăn ra chết nhưng cũng sẽ khiến hiệu suất query giảm đi đáng kể do bị bottleneck ở connection pool.
- Với việc kill transaction định kì thực sự rất tiêu cực, không khác kết án kịch khung cho tội nhẹ chỉ vì trạng thái của thẩm phán không vui. Tưởng tưởng có một transaction nào đó thực sự có thời gian idle cao hơn bình thường mà ta kill mất, thì sẽ làm mất hết quá trình thực thi của transaction đó, gây ra những lỗi rất khó debug trong (lỗi ở CSDL luôn khó debug hơn lỗi code).

Vậy nên hãy luôn sử dụng transaction có trách nhiệm, kết thúc nó khi không dùng đến nữa. Như trong trường hợp của tôi sử dụng AWS Lambda và dùng thư viện `PyMySQL`, tôi luôn mặc định `ROLLBACK` các transaction sau khi kết thúc request như ví dụ dưới đây:

```python
conn = get_connection() # function kết nối db, giữ connection warm
def lambda_handler(event, context):
    try:
        conn.begin()
        with conn.cursor() as cur:
            cur.execute('INSERT SOMETHING HERE')
        # nếu muốn commit kết quả query vào db thì COMMIT ở đây
        conn.commit() 
        return {"statusCode": "200"}
    finally:
        # luôn ROLLBACK nếu có thể, kể cả khi code ở trên có xảy ra lỗi
        conn.rollback() 
```

ta có thể gom hết đống `try`/`finally` xấu xí kia thành một decorator:

```python
conn = get_connection()

def db_rollback():
    def decorating_handler(lambda_handler):
        def wrapper(event, context):
            try:
                conn.begin()
                return lambda_handler(event, context)
            finally:
                if conn.open:
                    conn.rollback()
        return wrapper
    return decorating_handler

@db_rollback
def lambda_function(event, context):
    with conn.cursor() as cur:
        cur.execute('INSERT SOMETHING')
        conn.commit()

    return {"statusCode": "200"}
```

Ta có thể nhét decorator này vào trong Lambda Layer và tái sử dụng đoạn code này ở nhiều function khác nhau.

# Kết luận

Việc thay đổi cấu trúc bảng (hay chạy các query DDL nói chung) không phải là một công việc thường xuyên xảy ra, vì hiếm khi một cái CSDL đang chạy ngon lành tự dưng lại lôi ra để chỉnh sửa cả, mà có muốn sửa cũng phải chọn khung thời gian và chiến lược hợp lý (và có một chút mê tín). Việc quản lý các transaction càng trở nên quan trọng hơn khi các câu query chồng chéo nhau trên các bảng ở các transaction khác nhau sẽ gây blocking, kéo tụt hiệu năng của cả hệ thống xuống. Để giải quyết các vấn đề về chiến lược, tư duy khi query, thiết kế CSDL thì phải dành cho một người có trình độ cao hơn và kinh nghiệm dày dặn hơn, ví dụ như anh [Trần Quốc Huy](https://www.youtube.com/@tranquochuywecommit) với kênh YouTube rất nhiều kiến thức bổ ích về CSDL ở đây.

Mong rằng bài viết này sẽ giúp bạn đọc giải quyết vấn đề, vì việc này đã tốn mất 1 ngày ngồi tra tài liệu MySQL và đào bới khắp StackOverflow để hiểu rõ ngọn nguồn,

> Ghi chú: Một lần nữa, ChatGPT hay bất kì công cụ AI nào đều không giúp sức được cho tôi trong quá trình sửa lỗi này. Có thể do tôi sử dụng sai cách nên mong các chuyên gia _prompt engineer_ giúp tôi đưa ra những câu prompt hào sảng để nó chói qua tim mấy cái model LLM mà sinh ra những câu trả lời có thực sự hữu ích (đã thử phương pháp thêm _"please"_ và _"làm ơn"_ sau mỗi câu hỏi nhưng không được).

# Nguồn tham khảo
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/):
    * [15.1.20.5 FOREIGN KEY Constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking) / [archive](https://web.archive.org/web/20240706020716/https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking)
    * [17.12.1 Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations) / [archive](https://web.archive.org/web/20240614132951/https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations)
    * [10.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release) / [archive](https://web.archive.org/web/20240710093303/https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release)
- [AWS Documentation - Amazon RDS](https://docs.aws.amazon.com/rds/):
    * [Common DBA tasks for MySQL DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End) / [archive](https://web.archive.org/web/20240713052429/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End)
    * [RDS for MySQL stored procedure reference - Ending a session or query](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html) / [archive](https://web.archive.org/web/20240225060423/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html)
    * [Managing an RDS Proxy - Avoiding pinning](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-managing.html#rds-proxy-pinning) / [archive](https://web.archive.org/web/20240704212205/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-managing.html#rds-proxy-pinning)
- [StackOverflow](https://stackoverflow.com)
    * [MySQL 5.6 - table locks even when ALGORITHM=inplace is used](https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used) / [archive](https://web.archive.org/web/20240717161527/https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used)
- [Percona Blog](https://www.percona.com/blog/)
    * [Chasing a Hung MySQL Transaction: InnoDB History Length Strikes Back](https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/) / [archive](https://web.archive.org/web/20240522170813/https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/)
    * [How small changes impact complex systems – MySQL example](https://www.percona.com/blog/small-changes-impact-complex-systems-mysql-example/) / [archive](https://web.archive.org/web/20240718070236/https://www.percona.com/blog/small-changes-impact-complex-systems-mysql-example/)
- [Planet MySQL](https://planet.mysql.com/)
    * [Tracking MySQL query history in long running transactions](https://planet.mysql.com/entry/?id=5988591) / [archive](https://web.archive.org/web/20240814092726/https://planet.mysql.com/entry/?id=5988591)