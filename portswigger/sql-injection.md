# SQL Injection

Source: https://www.notion.so/31e4d37069a8801e90a3fbd4a8a0db06

SQL Injection là lỗ hổng cho phép attacker can thiệp vào truy vấn mà ứng dụng gửi tới cơ sở dữ liệu.

## Impact

SQLi thành công có thể dẫn đến:

- Đọc dữ liệu nhạy cảm của user khác.
- Lộ thông tin thẻ, mật khẩu, dữ liệu nội bộ.
- Ghi, sửa, hoặc xóa dữ liệu.
- Trong một số trường hợp có thể dẫn tới RCE hoặc persistence/backdoor.

## Detect

Các payload/dấu hiệu thường dùng khi kiểm tra:

- Dấu nháy đơn: `'`
- Điều kiện boolean: `AND 1=1`, `AND 1=2`
- Payload gây delay/time-based.
- Payload OAST để tạo tương tác out-of-band.
- Cú pháp SQL đặc thù để xem response thay đổi có hệ thống hay không.

## Ghi Chú

Khi test SQLi, ưu tiên quan sát sự khác biệt có kiểm soát giữa response đúng/sai thay vì chỉ dựa vào lỗi. Với blind SQLi, dùng boolean-based, time-based hoặc OAST để xác nhận.

