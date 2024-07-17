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
    title="Một loạt query đang đợi metadata lock"
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

Để xé lẻ vấn đề ra, ta có gạch đầu dòng sau cần phải thực hiện:
- `metadata lock` là gì?
- Việc gì ngăn cản `metadata lock` xảy ra?

## `metadata lock` là gì?

Theo như [document của MySQL](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-acquisition), thì `metadata lock` là một dạng mutex để khoá cấu trúc bảng lại, để đảm bảo chỉ có 1 session trong MySQL được can thiệp vào cấu trúc bảng. `metadata lock` xảy ra khi người dùng thực hiện các câu query DDL (Data Definition Language), và query `ALTER` được tính là DDL query. Cũng như trong [document này](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release), khi có bất kì transaction nào chưa hoàn thành đang thao tác trên bảng đều sẽ cản trở việc tạo ra `metadata lock`. Và khi query các bảng con có foreign key tới bảng cha đang cần `ALTER TABLE` mà đang bị block ở trên, tất cả các query đều sẽ lock theo do `foreign_key_checks = 1`

Vậy ta có sơ đồ đơn giản sau để mô tả vấn đề đợi dắt dây gặp phải khi `ALTER TABLE`:

{{< figure 
    src="/posts/alter-table-metadata-lock/alter-table-metadata-lock.drawio.png"
    position="center"
    alt="`ALTER TABLE` diagram"
    title="`ALTER TABLE` diagram"
    caption="`ALTER TABLE` diagram" >}}

## Việc gì ngăn cản `metadata lock` xảy ra?

Trước hết, tôi thử chạy `SHOW FULL PROCESSLIST` xem có câu query nào đang chạy hay không, và thực tế không có câu query nào đang chạy cả:

{{< figure 
    src="/posts/alter-table-metadata-lock/show-full-processlist.png"
    position="center"
    alt="SHOW FULL PROCESSLIST hiển thị không có query nào đang chạy cả"
    title="SHOW FULL PROCESSLIST hiển thị không có query nào đang chạy cả"
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
    title="Danh sách các transaction trong InnoDB"
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
CALL mysql.mysql.rds_kill(<session_id>);
```

Không có đường tắt nào cho bạn để xem nên kill hay nên bỏ qua session nào, vì chẳng có thông tin gì thêm về các session. Ta buộc phải chọc vào `performance_schema` của MySQL để xem query history của transaction. Để làm việc này ta có thể tham khảo [bài viết này của Percona](https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/).

Do mặc định RDS tắt `performance_schema` và tôi không có quyền bật nó lên, tôi đành phải kill từng session một rồi chạy thử `ALTER TABLE` xem có chạy không cho chắc.

Sau khi kill đúng session, cuối cùng tôi đã có thể chạy câu lệnh `ALTER TABLE` một cách thoải mái mà không cần phải đợi quá lâu.

# Ngăn ngừa

Như ở trên đã nêu, lý do xảy ra việc `Waiting for table metadata lock` là do transaction chưa được kết thúc khi không sử dụng. Việc để các transaction hanging còn gây tiêu tốn tài nguyên của MySQL, chiếm nhiều bộ nhớ để chứa transaction history và rollback log, chứ không chỉ mỗi gây block các DDL query (vốn hiếm khi xảy ra). Vậy để tránh vấn đề này, cách tôi giải quyết đơn giản là `ROLLBACK` hoặc `COMMIT` sau mỗi lần query DB.

Theo kinh nghiệm cua tôi thì dưới đây là 2 cách để quản lý transaction:

## Quản lý thuần bằng tay

Như trong trường hợp của tôi sử dụng AWS Lambda và dùng thư viện `PyMySQL`, tôi luôn mặc định `ROLLBACK` các transaction sau khi kết thúc request như ví dụ dưới đây:

```python
conn = get_connection() # function kết nối db, giữ connection warm
def lambda_handler(event, context):
    try:
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

Ta có thể nhét decorator này vào trong Lambda Layer và tái sử dụng ở nhiều function khác nhau

## Quản lý bằng thư viện/công cụ

Nếu không muốn quản bằng tay, ta có thể quản lý bằng việc sử dụng Connection Pool (như HikariCP chẳng hạn) hoặc dùng RDS Proxy để AWS quản lý hộ chúng ta. Tuỳ vào túi tiền và nhu cầu sử dụng.

# Kết luận

Mong rằng bài viết này sẽ giúp bạn giải quyết vấn đề, vì việc này đã tốn mất 1 ngày ngồi tra tài liệu MySQL và đào bới khắp StackOverflow để làm xong.

# Nguồn tham khảo
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/):
    * [15.1.20.5 FOREIGN KEY Constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking) / [Link archive.org](https://web.archive.org/web/20240706020716/https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html#foreign-key-locking)
    * [17.12.1 Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations) / [Link archive.org](https://web.archive.org/web/20240614132951/https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html#online-ddl-table-operations)
    * [10.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release) / [Link archive.org](https://web.archive.org/web/20240710093303/https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html#metadata-lock-release)
- [AWS Documentation - Amazon RDS](https://docs.aws.amazon.com/rds/):
    * [Common DBA tasks for MySQL DB instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End) / [Link archive.org](https://web.archive.org/web/20240713052429/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.CommonDBATasks.html#Appendix.MySQL.CommonDBATasks.End)
    * [RDS for MySQL stored procedure reference - Ending a session or query](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html) / [Link archive.org](https://web.archive.org/web/20240225060423/https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-stored-proc-ending.html)
- [StackOverflow](https://stackoverflow.com)
    * [MySQL 5.6 - table locks even when ALGORITHM=inplace is used](https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used) / [Link archive.org](https://web.archive.org/web/20240717161527/https://stackoverflow.com/questions/54667071/mysql-5-6-table-locks-even-when-algorithm-inplace-is-used)
- [Percona Blog](https://www.percona.com/blog/)
    * [Chasing a Hung MySQL Transaction: InnoDB History Length Strikes Back](https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/) / [Link archive.org](https://web.archive.org/web/20240522170813/https://www.percona.com/blog/chasing-a-hung-transaction-in-mysql-innodb-history-length-strikes-back/)