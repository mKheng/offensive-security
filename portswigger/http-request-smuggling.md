# HTTP Request Smuggling

Source: https://www.notion.so/3584d37069a880c19411cb45451fee2a

## Tổng Quan

HTTP request smuggling là kỹ thuật lợi dụng việc front-end server và back-end server diễn giải ranh giới request HTTP khác nhau. Trong kiến trúc hiện đại, request thường đi qua CDN, WAF, load balancer, reverse proxy rồi mới tới back-end. Nếu các thành phần này không thống nhất cách xác định điểm kết thúc request, attacker có thể nhét một request ẩn vào kết nối dùng chung.

Nguyên nhân thường nằm ở sự mơ hồ giữa hai header:

- `Content-Length`
- `Transfer-Encoding: chunked`

Theo chuẩn, nếu cả hai cùng xuất hiện thì `Content-Length` phải bị bỏ qua. Nhưng khi nhiều server trong pipeline xử lý khác nhau, request smuggling xuất hiện.

## Biến Thể Chính

### CL.TE

Front-end dùng `Content-Length`, back-end dùng `Transfer-Encoding`.

```http
POST / HTTP/1.1
Host: website.com
Content-Length: 26
Transfer-Encoding: chunked

0

GET /404 HTTP/1.1
Foo: x
```

Back-end coi chunk `0` là kết thúc request, phần `GET /404...` còn lại bị giữ trong buffer và có thể gắn vào request tiếp theo.

### TE.CL

Front-end dùng `Transfer-Encoding`, back-end dùng `Content-Length`.

```http
POST / HTTP/1.1
Host: website.com
Content-Length: 4
Transfer-Encoding: chunked

15
GET /404 HTTP/1.1
0
```

### TE.TE

Cả hai đều hỗ trợ `Transfer-Encoding`, nhưng attacker làm mờ header để một bên không nhận ra, ví dụ biến thể casing hoặc spacing lạ.

## Phát Hiện

Dấu hiệu thường thấy:

- Request bị treo hoặc timeout.
- Response bất thường ở request kế tiếp.
- Back-end trả lỗi khác front-end.
- Socket bị poison.

Một kỹ thuật xác nhận là differential response: gửi request độc hại để đầu độc kết nối, sau đó gửi request bình thường và quan sát phản hồi.

## Lab Notes

Các lab PortSwigger thường yêu cầu:

- Bypass front-end controls để truy cập `/admin`.
- Tìm header nội bộ do front-end thêm vào request.
- Capture request của user khác bằng cách smuggle request có `Content-Length` lớn.
- Kết hợp reflected XSS với smuggling để kích hoạt payload trên user kế tiếp.

Ví dụ payload xóa user qua CL.TE:

```http
POST / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 71
Transfer-Encoding: chunked

0

GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
X-Ignore: X
```

## Phòng Chống

- Dùng HTTP/2 end-to-end.
- Reject request có cả `Content-Length` và `Transfer-Encoding`.
- Không reuse TCP connection giữa FE và BE nếu không cần.
- Normalize request tại front-end.
- Scan bằng Burp Suite HTTP Request Smuggler.

