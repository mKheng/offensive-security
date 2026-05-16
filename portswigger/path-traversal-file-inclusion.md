# Path Traversal & File Inclusion

Source: https://www.notion.so/31e4d37069a88075b192cfe16190866d

## Path Traversal

Path traversal, hay directory traversal, xảy ra khi ứng dụng nhận đường dẫn file từ người dùng nhưng không kiểm soát hoặc chuẩn hóa đầu vào đúng cách. Kẻ tấn công có thể dùng chuỗi như `../` hoặc `..\` để thoát khỏi thư mục dự kiến và đọc file nhạy cảm trên server.

Ví dụ payload:

```bash
../../../../etc/passwd
..\..\..\windows\system.ini
```

## Filter Bypass

Một số kỹ thuật bypass phổ biến:

- URL encoding: `%2e%2e%2f`, `%2e%2e%5c`
- Double URL encoding: `%252e%252e%252f`
- Unicode encoding: `%u002e`, `%u2215`
- Nested traversal: `....//....//`
- Null byte trong một số môi trường cũ: `../../../etc/passwd%00.png`

## File Inclusion

File inclusion xảy ra khi ứng dụng cho phép người dùng chỉ định file để include hoặc load vào trang mà không kiểm soát đường dẫn. Có hai nhóm chính:

- LFI: Local File Inclusion, đọc hoặc thực thi file cục bộ trên server.
- RFI: Remote File Inclusion, include file từ server bên ngoài do attacker kiểm soát.

Ví dụ PHP dễ lỗi:

```php
<?php
  $file = $_GET['page'];
  include($file);
?>
```

Payload LFI:

```text
https://example.com/index.php?page=../../../../etc/passwd
```

RFI thường phụ thuộc cấu hình như `allow_url_include = On`.

