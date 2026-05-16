# Linux

OffSec Linux practice notes exported from Notion.

| Lab Name | Status | Priority | Note |
| --- | --- | --- | --- |
| Twiggy | Done | Intermediate | nmap scan → nhận ra Salt API trên port 8000 → tìm được CVE-2020-11651 → exploit qua ZeroMQ → nhận được shell root trực tiếp |
| Exfiltrated | Done | Intermediate | scan → thu thap được cred admin:admin → exploit qua vul Subrion CMS 4.2.1 → Authenticated File Upload Bypass → RCE (Reverse Shell) → Root Cron Job (ExifTool) → CVE-2021-22204 (DjVu payload) → Root Shell. |
| Pelican | Done | Intermediate | nmap → truy cap http:8080 → Exhibitor for ZooKeeper exploit tim duoc module exploit → upgrade shell → leo quyen dua vao /usr/bin/gcore NOPASSWD. Tim them process nao chay quyen root roi lay pid sau do thuc thi sudo/usr/bin/gcore |
| Astronuat | Done | Easy | scan → vul tạo file/chèn không cần xác thực vào .yaml → RCE → Tìm thấy uuid bất thường php chạy dưới quyền user thường → khai thác leo root |
| Blackgate | Done | Advanced | scan → lổ hổng nằm ở dịch vụ Redis → RCE → upload linpeas → phát hiện lỗi ở pwnkit → upload pwnkit → exploit và lấy được root |
| Boolean | Done | Intermediate | scan → vul nằm ở port 80 → đăng kí account bình thường và login → lỗi ở một request cho phép thêm 1 param bật cờ confirm email = true (qua được vul1). Tiếp theo lợi dung path traversal đọc được /etc/passwd → Lợi dụng cơ chế pair:key . Tự gen keyssh và upload lên user kemi/.ssh (đổi tên public key cho đúng)→ Login SSH → lộ private key của root (biết đăng nhập ssh sử dụng key chỉ định và bật cờ dùng đúng 1 key có trong folder) → Done |
| Clue | Done | Advanced | Làm lại |
| Codo | Done | Easy | scan → dùng gobuster scan ra /admin → đăng nhập account admin:admin → research tiêu đề codoforum exploit → tìm hiểu về cách khai thác bằng cách đọc exploit → poC có sẵn không sài được tự làm manual rồi lấy được shell → leo root bằng linpeas → lộ passwd của root → lụm |
| Crane | Done | Easy | scan → tìm được cred admin:admin ở port 80 → research 1 tí thì phát hiện cve → download về thực hiện PoC và thành công RCE → leo root thử bằng sudo -l thì phát hiện www-data có thể chạy /usr/sbin/service mà không cần password → sudo /usr/sbin/service ../../bin/bash → service sẽ chạy ở /etc/init.d/ khi truyền đói số như vậy thì shell sẽ thành /etc/init.d/../../bin/sh → sudo /bin/sh lụm |
| Levram | Done | Easy | scan → tìm được port 8000 → login bằng admin:admin → gerapy → khai thác poc và leo root bằng /usr/bin/python3.10 đc cap_setuid=ep |
| Extplorer | Done | Intermediate |  |
| Hub | In progress | Easy |  |
| Image | Not started | Easy |  |
| Law | Not started | Easy |  |
| Lavita | Not started | Easy |  |
| PC | Not started | Easy |  |
| Fired | Not started | Easy |  |
| Press | Not started | Easy |  |
| Scrutiny | Not started | Easy |  |
| RubyDome | Not started | Easy |  |
| Zipper | Not started | Easy |  |
| Flu | Not started |  |  |
| Workaholic | Not started |  |  |
| PyLoader | Not started |  |  |
| Plum | Not started |  |  |
| SPX | Not started |  |  |
| Jordak | Not started |  |  |
| BitForge | Not started |  |  |
| Vmdak | Not started |  |  |
| Ochima | Not started |  |  |
| Nibbles | Not started |  |  |
| CVE-2023-6019 | Not started |  |  |
| Sea | Not started |  |  |
| Payday | Not started |  |  |
| Snookums | Not started |  |  |
| SpiderSociety | In progress | Intermediate |  |
