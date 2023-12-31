Hi guys, 

Phần repo này mình muốn chia sẽ một chút về phần cải thiện performance cho tính năng "upsert" của laravel - đã có từ phiên bản laravel v8.

Về cách upsert của laravel thực thi, thì họ đang dùng câu SQL: 

`
INSERT INTO ... ON DUPLICATE KEY UPDATE ...
`

đại loại sẽ gọi câu insert, nếu trùng key (khóa chính hoặc cặp khóa chính) sẽ bắt event "DUPLICATE KEY" rồi update record - thay vì thường phải viết 1 câu insert và update, thì dùng câu này để gộp cho gọn

**Vậy vấn đề gặp phải là gì ?

- Giả sử bạn upsert với lượng lớn data, 10k record chẳng hạn, thì sẽ có 10k query => vậy không ổn tí nào nhỉ?

**Giải pháp (tạm thời)

- Thay vì phải tạo ra 10k query, mình sẽ cố gắng giảm bớt nó thành 2 câu query (nhưng không hẳn sẽ nhanh hơn trong trường hợp update)
  + 1 câu dùng batch insert: logic là sẽ tìm những id không có trong db rồi gộp chúng lại, phần còn lại là để update
  
  + 1 câu dùng batch update: update...case...when

- Use case hay dùng: import data từ file

** Vậy upsert của laravel dùng ổn trong trường hợp nào?

- Khi bạn nghĩ luồng logic của bạn đang làm có ít dữ liệu (chỉ có 5-10 records) thì dùng có sẵn của laravel cho nhanh, cũng không đáng kể
- và câu upsert (SQL trên) có hổ trợ nhiều key (khóa chính và cặp khóa chính) 

** Một vài lưu ý
- Hiện tại code của mình chỉ support 1 field, bạn có thể fork rồi code thêm nhé :D
- Đã work trên MYSQL, còn lại mình chưa test :D
- Bạn có thể chỉ dùng trait SqlBulkUpdatable nếu bạn chỉ muốn dùng cho bulk update, hoặc wantsUpsertQuery cho cả update và insert

*Diễn giải khác

Update record bằng nhiều câu query (1)

hay một câu query thì tốt hơn (2) ?

---------------------------------------------------------------------------------------------

query (1)
<br>
<br>
  update table

  set some_field = val_1

  where id = 1

  <br>

  update table

  set some_field = val_2

  where id = 2

  ...

  update table

  set some_field = val_n

  where id = 3

---------------------------------------------------------------------------------------------

query (2)

`
  update table
  set some_field = CASE WHEN ....
`

---------------------------------------------------------------------------------------------
Khi dùng (1), bạn phải check n điều kiện trong n lần update<br>
=> n^2 lần
<br>
Khi dùng (2), bạn chỉ check n điều kiện chỉ trong 1 lần gọi table<br>
=> n lần
<br><br>


Cập nhật bài viết 

[06/11]

- Mình test thiếu trường hợp nên nghĩ upsert chưa hổ trợ bulk update, bulk insert

- upsert chỉ hổ trợ primary key hoặc unique key

=> upsert của laravel chỉ chạy một câu cho toàn bộ câu insert và update

=> upsert của laravel sau khi dùng một cách đúng đắn và test lại thì nhanh gấp chục lần cách dùng "UPDATE CASE WHEN" và

gấp đôi với cách dùng "dynamic temporary table"

`(
  UPDATE date_test
  join
    SELECT *
    FROM (
        VALUES
        ROW....
      ) AS temp_table(id, name, code)
    ) as temp_data
  on ...
  set ...
)`

https://dev.mysql.com/.../8.0/en/insert-on-duplicate.html

[8/11]

- Mình vừa có viết thêm package để dùng cho bulk update, nếu bạn cần có thể xem tại

https://github.com/quanggpv/fast-batch-update


Thanks for reading, happy coding
