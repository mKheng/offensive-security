# SQL Injection

# 1. Định nghĩa

**SQL Injection** là một lổ hổng bảo mật web cho phép kẻ tấn công có thể can thiệp vào câu truy vấn mà ứng dụng web thực hiện đối với DB của nó. Đây là lổ hổng bảo mật khá phổ biến, cho phép kẻ tấn công thực hiện các câu truy vấn SQL tuỳ ý. Điều này có thể dẫn đến truy cập dữ liệu trái phép, thao tác này kia.

---

# 2. Impact

Lổ hổng này có ảnh hưởng đến cả 3 mặt trong tam giác CIA:

**Về tính bí mật:**

- Database thường chứa dữ liệu nhạy cảm: thông tin cá nhân, mật khẩu, thẻ tín dụng, y tế, tài chính…
- SQL Injection cho phép attacker:
    - Đọc dữ liệu không được phép (bypass logic, **`UNION SELECT`**, blind SQLi…).
    - Xuất toàn bộ bảng, backup, hoặc dump data ra file rồi tải về.

Từ đó làm lộ dữ liệu người dùng, bí mật kinh doanh, dữ liệu thẻ, dẫn đến vụ việc data breach.

Tính toàn vẹn:

- **Sửa đổi dữ liệu**: **`UPDATE`** tài khoản, số dư, quyền hạn, giá sản phẩm…
- **Xóa dữ liệu**: **`DELETE`** bảng quan trọng (orders, users, logs…).

Tính sẵn sàng:

- Gây **DoS** bằng các truy vấn nặng, **`SLEEP()`**, lock table, hay xóa dữ liệu quan trọng.
- Trong một số trường hợp, escalation lên OS (nếu DB có quyền cao) có thể làm tê liệt cả server.

---

# 3. Phân loại SQL Injection

Có 3 nhóm chính: In-band, Blind, Out-of-band

## 3.1. In-band

Kẻ tấn công dùng **cùng một kênh** để gửi payload và nhận kết quả (ví dụ: gửi qua form, nhìn kết quả trên chính trang đó).

Hai kiểu phổ biến:

1. **Error-based SQLi**
    - Dựa vào thông báo lỗi của DBMS để lấy thông tin cấu trúc DB (tên bảng, cột, version…).
    - Ví dụ: **`' OR 1=1 --`** hoặc payload gây lỗi cú pháp để error message lộ thông tin.
        
        acunetix
        
2. **Union-based SQLi**
    - Dùng toán tử **`UNION`** để ghép kết quả từ bảng khác vào result set.
    - Ví dụ: **`' UNION SELECT username, password FROM users --`** để lấy toàn bộ bảng users.

## 3.2. Blind

Response không hiện trực tiếp dữ liệu hoặc lỗi của DB, nhưng vẫn bị inject. Từ đó attacker phải suy luận qua hành vi của ứng dụng web (thời gian trả response, phản hồi, nội dung có sự khác biệt,….)

1. **Boolean-based blind**
    - Inject điều kiện đúng/sai: **`AND 1=1`**, **`AND 1=2`**.
    - Quan sát sự khác biệt trong response (trang có/không có dữ liệu, nội dung khác nhau, redirect khác nhau…).
2. **Time-based blind**
    - Dùng các lệnh gây delay (ví dụ **`SLEEP`**, **`WAITFOR DELAY`**, **`pg_sleep`**) để quan sát thời gian phản hồi.
    - Nếu có delay, điều kiện inject là đúng.
3. **Error-based blind (inferential errors)**
    - Gây lỗi có điều kiện, nhưng error message không hiện trực tiếp; vẫn phải suy luận qua nội dung response.

## 3.3. Out-of-band

Kẻ tấn công **không thể dùng cùng một kênh** để gửi payload và nhận kết quả; thay vào đó, dữ liệu được “gửi ra ngoài” qua kênh khác (DNS, HTTP request, email…)
Ví dụ: dùng **`xp_dirtree`** (MSSQL) hoặc **`UTL_HTTP`** (Oracle) để DBMS gửi HTTP/DNS request đến server do kẻ tấn công kiểm soát, mang theo dữ liệu cần lấy.

Ngoài ra, còn có:

- **Second-order SQLi**: payload không được thực thi ngay, mà được lưu lại (ví dụ vào DB) và sau đó được dùng trong một câu truy vấn khác, gây injection sau đó.
- **SQLi trong stored procedures / ORM / NoSQL**: vẫn có thể xảy ra nếu dynamically build SQL/NoSQL từ input không tin cậy
- Stacked Querry nữa cái này có thể dẫn đến RCE nếu đáp ứng đủ yếu tố.

---

# 4. Detect

## 4.1. **Test thủ công**

- Bước 1: Xác định các **entry point** (input fields, URL parameters, headers, cookies).
- Bước 2: Thử các **payload đơn giản**:
    - **`'`** (dấu nháy đơn) → gây lỗi cú pháp SQL.
    - **`' OR '1'='1`**, **`1 OR 1=1`**, **`" OR "1"="1`**.
    - **`; DROP TABLE users; --`** (chỉ test môi trường an toàn).
- Bước 3: Quan sát:
    - Error messages của DBMS hiện ra → có khả năng lỗi-based SQLi.
    - Thay đổi nội dung trang, số dòng trả về, hay thời gian xử lý → có thể là blind SQLi.
    - Response khác biệt rõ rệt khi thay đổi điều kiện (true/false) → boolean-based blind.
    - Response bị delay rõ rệt khi dùng **`SLEEP`**/**`WAITFOR`** → time-based blind.
- Bước 4: Thử **`UNION`**:
    - Xác định số cột bằng **`ORDER BY`** hoặc **`UNION SELECT NULL, NULL, ...`**.
    - Thử **`UNION SELECT`** để lấy dữ liệu từ bảng khác.
- Bước 5: Nếu không rõ kết quả trong response:
    - Sử dụng **out-of-band** (DNS/HTTP request đến server kiểm soát) để kiểm tra.

## 4.2. Dùng tools

SQLmap là bestchoice.

# 5. Phòng chống

**Tránh nối chuỗi input vào câu lệnh SQL.**

**Prepared Statements**

- Định nghĩa câu lệnh SQL trước, dùng placeholder (**`?`** hoặc **`:name`**), sau đó bind giá trị.
- Ví dụ (Java):
    
    String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
    
    PreparedStatement ps = connection.prepareStatement(sql);
    
    ps.setString(1, username);
    
    ps.setString(2, password);
    
- Hầu hết ngôn ngữ hiện đại đều hỗ trợ: Java, C#, PHP, Python, Node.js,

**ORM**

**Validate & Whitelist input**

- Chỉ chấp nhận định dạng hợp lệ (ví dụ: ID là số, username là alphanumeric, …).
- Dùng whitelist cho các giá trị có thể (ví dụ **`ORDER BY`** chỉ cho phép tên cột hợp lệ).
- Tuy nhiên, **không bao giờ** chỉ dựa vào validate/filter mà không dùng parameterized queries

**Escape / Encoding (ít khuyến nghị hơn)**

- Escape ký tự đặc biệt (như **`'`** thành **`''`** hoặc **`\'`**) tùy DBMS.
- OWASP khuyến cáo: escape chỉ là lớp phòng thủ thứ yếu, không thay thế parameterized queries.

Blind khai thác lấy từng kí tự thông qua SELECT SUBSTRING(password,1,1),….

---

Cross Site Scripting (XSS)

**Cross-Site Request Forgery**