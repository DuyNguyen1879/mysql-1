# System Administration

## Thiết lập chung và security

**Thiết lập mật khẩu cho tài khoản root**

- Thiết lập secure với mysql_secure_installation script

`mysql_secure_installation`

Khi đó remove tất cả anonymous account; Remove test database.

Có thể sử dụng tool **mysqladmin** khi chạy tạo root password lần đầu tiên

`mysqladmin -u root password 'new-password'`

- Đổi root password với các cách sau

`mysqladmin -u root -pold-password password 'new-password'`

hoặc

```
mysql> USE mysql;
mysql> UPDATE user SET Password=PASSWORD('new-password') WHERE user='root';
mysql> FLUSH PRIVILEGES;
```

**Note**: Chỉ sử dụng `flush privileges` khi các trường hợp sử dụng tham số như UPDATE, INSERT, DELETE

**Thiết lập quyền để bảo vệ tệp tin cấu hình**

Thiết lập quyền quyền read write chỉ cho người dùng mà chạy mysql server (ví dụ mysql).

Khi đó thiết lập tệp tin my.cnf với quyền 600 cho mysql user

**Show database size**

- Show size mỗi database trong toàn bộ databases

Tính theo MB (làm tròn)

`SELECT table_schema AS "DB Name", ROUND(SUM(data_length + index_length) / 1024 / 1024 , 1) AS "Size (MB)" FROM information_schema.TABLES GROUP BY table_schema;`

Tính theo GB (làm tròn)

`SELECT table_schema AS "DB Name", ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024 , 1) AS "Size (GB)" FROM information_schema.TABLES GROUP BY table_schema;`

- show size của một database

`SELECT table_schema AS "DB Name", ROUND(SUM(data_length + index_length) / 1024 / 1024 , 1) AS "Size (MB)" FROM information_schema.TABLES WHERE table_schema = "vnn";`

Thay vnn bằng tên database cần query

- Show size của tables

```
SELECT table_name AS "Table Name",
ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size in (MB)"
FROM information_schema.TABLES
WHERE table_schema = "vnn"
ORDER BY (data_length + index_length) DESC;
```

Thay vnn với tên database cần query

- Ngoài ra, có thể xem kích thước bằng dung lượng trên disk

`sudo du -sh /var/lib/mysql`

**Kiểm tra cú pháp cho tệp tin cấu hình mysql**

`/usr/sbin/mysqld — verbose — help`

## Quản lý tài khoản người dùng

- Tạo tài khoản mới

syntax:

`CREATE USER 'username'@'hostname' IDENTIFIED BY 'Password';`

Ở đây hostname có thể là tên máy hoặc địa chỉ IP.

Các ký tự đặc biệt '%' hoặc '_' có thể được sử dụng cho tên máy như 'user'@'%'. Trong đó '%' cho phép tất cả hosts, không gồm localhost :D

Với địa chỉ IP, chúng ta có gán với dải mạng như 'user'@'192.168.1.0/255.255.255.0'

ex:

`CREATE USER 'user1'@'192.168.1.%' IDENTIFIED BY 'P@ssword';`

- Gán quyền

user mới đã tạo ở trên mặc định chưa có quyền được gán. Để gán đặc quyền cho user, chúng ta sử dụng lệnh GRANT

Các quyền có sẵn như sau:

-- Object Rights: SELECT, INSERT, UPDATE, DELECT, EXECUTE, SHOW VIEW.

-- Others: ALL, ALL WITH GRANT OPTION, GRANT, CREATE USER, CREATE TEMPORARY TABLES, LOCK TABLES.

-- Gán tất cả quyền trên all databases (ngoại trừ quyền GRANT)

`GRANT ALL ON *.* TO 'username'@'hostname';`

-- Gán tất cả quyền, gồm cả quyền GRANT trên all databases

`GRANT ALL ON *.* TO 'username'@'hostname' WITH GRANT OPTION;`

-- Gán một số quyền được chỉ định trên tất cả databases

ex: chỉ gán quyền SELECT và INSERT

`GRANT SELECT, INSERT ON *.* TO 'username'@'hostname';`
   
## Tối ưu MySQL Query Cache

Kích hoạt chế độ cache MySQL Query để tăng khả năng truy vấn database mysql

- Check trạng thái query cache hiện tại

```
mysql -u root -p
show variables like 'query_cache_%';
```

+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_type             | OFF     |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
5 rows in set (0,00 sec)

- Thêm nội dung sau vào tệp tin my.cnf trong phần [mysqld]

```
query_cache_type = 1
query_cache_size = 4096M ; Kích thước tối đã query được cache, mặc định 1048576 B
query_cache_limit = 2M ; Kích thước tối của một kết quả query riêng lẻ có thể cache, mặc định là 1M
```

- Sau đó restart mysqld service

`systemctl restart mysqld`

## Logs trong MySQL

MySQL có thể duy trì với nhiều định dang log như Error Log, Binary Log, General Query Log, Slow Query Log và DDL log (metadata log)

- Error log: ghi lại các vấn đề về starting, running và stoping của mysql service

- Binary log

Ghi lại các thông tin mà thao tác liên quan đến thay đổi database như CREATE TABLE, INSERT, UPDATE, DELETE. Nó không ghi lại các thông tin query như SELECT, SHOW, DESCRIBE.

Binary log cũng được sử dụng trong quá trình mysql replication.

Hệ thống sẽ tạo một tệp log mới khi server start hoặc log bị flush. Ngoài ra hệ thống tạo một tệp index dùng để track tất cả thông tin binary log.

Để đọc nội dung tệp binary log, chúng ta sử dụng tool **mysqlbinlog** để convert định dạng binary sang text

- General Query Log

Ghi thông tin về các kết nối từ clients và tất cả thông tin SQL

Thực hiện thao tác sau để enable general query log

Sửa tệp tin cấu hình chính của mysql (mặc định /etc/my.cnf trên RHEL/CentOS; /etc/mysql/my.cnf trên Ubuntu/Debian)
và thêm nội dung sau vào khối [mysqld]

```
general_log = on
general_log_file=/var/log/mysql-general.log
```

Tạo tệp log

`touch /var/log/mysql-general.log`

Gán quyền cho mysql user trên tệp general log file

`chown -R mysql:msql /var/log/mysql-general.log && chmod 640 /var/log/mysql-general.log`

Restart mysql service

`systemctl restart mysqld`

- Slow Query Log

Ghi lại những thông tin SQL mà time để query mất nhiều thời gian (mặc định là 10s)

Để enable mysql slow query log chúng ta sẽ đề cập ở phần dưới.

## MySQL Slow Query Log

**Sửa nội dung tệp tin cấu hình mysql/mariadb**

Thêm nội dung sau vào tệp tin /etc/my.cnf, mở rộng trong phần [mysqld]

```
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /var/log/mysql/slow-query.log
log_queries_not_using_indexes=ON
```

Trong đó:

- slow_query_log = 1 : Dùng để enable slow query log

- long_query_time = 2 : Thiết lập thời gian một truy vấn SQL sẽ thực hiện (tính bằng giây)

- slow_query_log_file = /var/log/mysql-slow.log : Chỉ định tên tệp cho slow query log

- log_queries_not_using_indexes = ON : Chỉ định ON/OF log query sẽ được indexes

**Tạo slow query logfile**

```
sudo touch /var/log/mysql/slow-query.log
sudo chown -R mysql:mysql /var/log/mysql/slow-query.log
```

**Restart mysql/mariadb**

`sudo systemctl restart mysqld`

**Xem, phân tích thông tin slow query log**

- Xem nội dung tệp tin slow query log

`sudo tail -f /var/log/mysql/slow-query.log`

- Sử dụng mysqldumpslow để phân tích và xuất slow query logfile

`mysqldumpslow -a /var/log/mysql/slow-query.log`

- Sử dụng công cụ pt-query-digest để phân tích query log

**pt-query-digest** là một công cụ của Percona dùng để phân tích các query mysql từ slow, general, và binary log files

Cài đặt công cụ Percona trên CentOS 7

`sudo yum install https://www.percona.com/downloads/percona-toolkit/3.0.13/binary/redhat/7/x86_64/percona-toolkit-3.0.13-1.el7.x86_64.rpm`

Phân tích slow query log file

`pt-query-digest /var/log/mysql/slow-query.log`
