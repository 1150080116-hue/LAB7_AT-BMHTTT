# LAB7_AT-BMHTTT
# Họ và tên: Hoàng Minh Thắng
# MSSV:1150080116
## 1. PHIÊN BẢN MÔI TRƯỜNG & HỆ THỐNG
| Máy ảo / Thành phần | Hệ điều hành & Phiên bản | Dịch vụ chính & Phiên bản | Vai trò trong mô hình |
|---|---|---|---|
| **Hypervisor** | Windows 11 Host | Oracle VM VirtualBox 7.1.x | Nền tảng ảo hóa mạng nội bộ |
| **pfSense** | pfSense 2.7.2-RELEASE (amd64) | Packet Filter, pfSense WebGUI | Firewall gateway phân chia LAN - DMZ - WAN |
| **DC01** | Windows Server 2025 Standard | Active Directory Domain Services, DNS Server | DNS Server cho zone `lab7.test` & Client kiểm thử |
| **WEB-PROXY** | Ubuntu Server 24.04 LTS (Noble) | Squid 6.6, E2Guardian 5.3.5, ClamAV 1.3.x | Gateway Web Filter, Cache & Content Antivirus Scanner |
| **DMZ-WEB** | Ubuntu Server 24.04 LTS (Noble) | Nginx 1.24.0 | Web Server hosting 13 domain ảo (`*.lab7.test`) |
## 2. KIẾN TRÚC MẠNG & CÁCH DỰNG MÔ HÌNH
Bước 1: Cấu hình pfSense (Firewall & Rule Chống Né Proxy)
1. Cấu hình gán interface: WAN (`em0`), MGMT (`em1`), LAN (`em2`), DMZ (`em3`).
2. Tạo các **Aliases** trên pfSense WebGUI (`Firewall > Aliases`):
   - `LAB7_CLIENTS`: `10.6.0.10`
   - `LAB7_PROXY`: `10.6.0.30`
   - `WEB_PORTS`: `80`, `443`
   - `MAIL_PORTS`: `25`, `465`, `587`, `993`, `995`, `110`
3. Thiết lập **Firewall Rules** trên tab LAN theo nguyên tắc **First-Match** nghiêm ngặt (từ trên xuống dưới):
   - **Rule 1 (Pass DNS)**: IPv4 UDP/TCP | Source: `LAN net` | Port: `*` | Dest: `*` | Port: `53 (DNS)`.
   - **Rule 2 (Pass NTP)**: IPv4 UDP | Source: `LAN net` | Port: `*` | Dest: `*` | Port: `123 (NTP)`.
   - **Rule 3 (Pass Proxy Web Outbound)**: IPv4 TCP | Source: `LAB7_PROXY` | Port: `*` | Dest: `*` | Port: `WEB_PORTS`.
   - **Rule 4 (Block Direct Web - Anti-Bypass)**: IPv4 TCP | Source: `LAB7_CLIENTS` | Port: `*` | Dest: `*` | Port: `WEB_PORTS` *(Action: Reject/Block)*.
   - **Rule 5 (Pass Client to Proxy)**: IPv4 TCP | Source: `LAB7_CLIENTS` | Port: `*` | Dest: `LAB7_PROXY` | Port: `8080`.
   - **Rule 6 (Pass Mail)**: IPv4 TCP | Source: `LAB7_CLIENTS` | Port: `*` | Dest: `*` | Port: `MAIL_PORTS`.
   - **Rule 7 (Block All Other)**: IPv4 * | Source: `LAB7_CLIENTS` | Port: `*` | Dest: `*` | Port: `*` *(Action: Reject/Block)*.
4. Trên tab **DMZ**: Thêm Rule cho phép DMZ ra Internet (WAN) để cài gói Nginx (`Source: DMZ net -> Dest: Any`).
#### Bước 2: Cấu hình DNS trên DC01 (Windows Server 2025)
1. Mở DNS Manager, tạo mới một **Forward Lookup Zone**: `lab7.test`.
2. Tạo 13 bản ghi **Host (A)** cùng trỏ về IP của `DMZ-WEB` (`172.16.6.10`):
   - `normal`, `blockedgame`, `china`, `vietnam`, `weapons`, `drugs`, `bomb`, `violence`, `jobs`, `education`, `download`, `portal`, `admin`.
3. Cấu hình Proxy trên Client (`inetcpl.cpl` > LAN settings):
   - Check *Use a proxy server for your LAN*: Address `10.6.0.30`, Port `8080`.
#### Bước 3: Cấu hình WEB-PROXY (Squid + E2Guardian + ClamAV)
1. Cài đặt các gói phần mềm:
   ```bash
   sudo apt update && sudo apt install -y squid e2guardian clamav clamav-daemon
   ```
2. Cấu hình **Squid** (`/etc/squid/squid.conf`):
   - Thiết lập chỉ lắng nghe localhost: `http_port 127.0.0.1:3128` (chặn người dùng kết nối trực tiếp bỏ qua bộ lọc).
   - Khởi động lại dịch vụ: `sudo systemctl restart squid`.
3. Cấu hình **ClamAV**:
   - Cập nhật cơ sở dữ liệu chữ ký: `sudo freshclam`
   - Bật và chạy daemon quét: `sudo systemctl enable --now clamav-daemon`
4. Cấu hình **E2Guardian** (`/etc/e2guardian/e2guardian.conf`):
   - `filterip = 0.0.0.0`
   - `filterport = 8080`
   - `proxyip = 127.0.0.1`
   - `proxyport = 3128`
   - Kích hoạt Antivirus Scanner:
     ```ini
     contentscanner = '/etc/e2guardian/contentscanners/clamdscan.conf'
     ```
5. Cấu hình chính sách lọc (`/etc/e2guardian/e2guardianf1.conf`):
   - `reportinglevel = 3` (Hiển thị trang cảnh báo HTML chi tiết với link Bypass).
   - `naughtynesslimit = 50`
   - `bypass = 300` (Thời gian tạm thời cho phép vượt rào là 300 giây).
   - `bypasskey = 'lab7key'`
6. Thiết lập các danh sách Rule (`/etc/e2guardian/lists/`):
   - `bannedsitelist`: Thêm các domain cấm cứng:
     ```text
     blockedgame.lab7.test
     china.lab7.test
     jobs.lab7.test
     ```
   - `exceptionsitelist`: Thêm domain danh sách trắng:
     ```text
     education.lab7.test
     ```
   - Kích hoạt danh sách từ khóa cấm vũ khí (`phraselists/weapons/weighted`).
7. Khởi động lại dịch vụ: `sudo systemctl restart e2guardian`.
#### Bước 4: Cấu hình DMZ-WEB (Nginx Server)
1. Cài đặt Nginx: `sudo apt update && sudo apt install -y nginx`.
2. Tạo thư mục web và 13 Virtual Host tương ứng với 13 domain `*.lab7.test`.
3. Sinh các file dữ liệu thử nghiệm trong `/var/www/lab7/download/files/`:
   - `10MB.bin`, `100MB.bin`, `110MB.bin`.
   - File mã độc kiểm thử chuẩn quốc tế: `eicar.com.txt` chứa chuỗi EICAR test string:
     ```text
     X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
     ```
4. Kiểm tra cấu hình và nạp lại dịch vụ: `sudo nginx -t && sudo systemctl reload nginx`.
---
## 3. CÁC TÌNH HUỐNG (SCENARIOS) ĐÃ THỰC HIỆN
1. **Kịch bản 1 (Baseline - Truy cập hợp lệ qua Proxy)**: Client đặt cấu hình Proxy `10.6.0.30:8080`, mở trình duyệt truy cập `http://normal.lab7.test`.
2. **Kịch bản 2 & 3 (Hard Block Domain)**: Client truy cập các trang web có tên miền nằm trong danh mục cấm: `http://blockedgame.lab7.test` và `http://china.lab7.test`.
3. **Kịch bản 4 (Warning & Content Filtering)**: Client truy cập trang có nội dung nhạy cảm / vũ khí `http://weapons.lab7.test` bị bộ lọc từ khóa nhận diện và hiển thị cảnh báo.
4. **Kịch bản 5 (Whitelist Exception)**: Client truy cập trang học tập `http://education.lab7.test` (dù chứa các từ khóa học thuật có thể trùng với từ cấm nhưng được đưa vào danh sách ngoại lệ).
5. **Kịch bản 6 (Anti-Bypass Firewall Protection)**: Thử nghiệm hành vi bypass bằng cách tắt Proxy trong Internet Settings của Client hoặc truy cập trực tiếp port 80/443 của Web Server.
6. **Kịch bản 7 (Temporary Bypass Verification)**: Trên trang cảnh báo của `weapons.lab7.test`, người dùng nhấp vào đường link cho phép tạm thời vượt rào truy cập trong thời hạn quy định (300 giây).
7. **Kịch bản 8 (Chặn trang tìm việc)**: Client truy cập trang `http://jobs.lab7.test`.
8. **Kịch bản 9 (Antivirus Gateway)**: Client cố gắng tải file mã độc thử nghiệm tại địa chỉ `http://download.lab7.test/files/eicar.com.txt`.
---
## 4. KẾT QUẢ KIỂM THỬ (PASS / FAIL)
| STT | Kịch bản / Mục tiêu kiểm thử | URL / Thao tác thực hiện | Hành vi kỳ vọng | Kết quả thực tế | Trạng thái |
|:---:|---|---|---|---|:---:|
| **KB1** | Truy cập Web thông thường (Baseline) | `http://normal.lab7.test` qua Proxy `10.6.0.30:8080` | Trang hiển thị bình thường, mã trạng thái HTTP 200 | Trang Web "Normal Site" hiển thị thành công | **PASS** |
| **KB2** | Chặn Game cấm (Hard Block) | `http://blockedgame.lab7.test` | E2Guardian chặn ngay lập tức, báo vi phạm Site List | Hiển thị màn hình chặn E2Guardian Access Denied | **PASS** |
| **KB3** | Chặn Domain quốc tế cấm | `http://china.lab7.test` | E2Guardian chặn, không cho phép truy cập | Hiển thị màn hình chặn E2Guardian Access Denied | **PASS** |
| **KB4** | Cảnh báo nội dung vũ khí | `http://weapons.lab7.test` | E2Guardian bắt theo Weighted Phrase, hiện trang Warning | Hiển thị cảnh báo kèm thông tin điểm Naughtiness & link Bypass | **PASS** |
| **KB5** | Danh sách trắng ngoại lệ | `http://education.lab7.test` | Bỏ qua các rule lọc nội dung, cho phép tải trang | Trang "Education Site" tải bình thường không bị chặn | **PASS** |
| **KB6** | Chống người dùng né Proxy (Anti-Bypass) | Bỏ cấu hình Proxy trên Client, truy cập thẳng Web | pfSense Rule 4 kích hoạt, Drop toàn bộ kết nối ra 80/443 | Trình duyệt báo `ERR_CONNECTION_TIMED_OUT` (Chặn triệt để) | **PASS** |
| **KB7** | Cơ chế Vượt rào tạm thời (Bypass) | Nhấp link Bypass trên trang cảnh báo `weapons.lab7.test` | Cho phép truy cập trang web trong thời hạn 300 giây | Truy cập thành công vào trang `weapons.lab7.test` sau khi click | **PASS** |
| **KB8** | Chặn trang tìm kiếm việc làm | `http://jobs.lab7.test` | E2Guardian chặn theo danh sách bannedsitelist | Hiển thị màn hình chặn E2Guardian Access Denied | **PASS** |
| **KB9** | Diệt virus tại Gateway (ClamAV) | Tải `http://download.lab7.test/files/eicar.com.txt` | ClamAV phát hiện chữ ký virus, E2Guardian ngắt tải và báo Virus Found | E2Guardian hiển thị trang cảnh báo đỏ: *Virus or bad content detected: Eicar-Signature* | **PASS** |
---
## 5. LỖI GẶP PHẢI (TROUBLESHOOTING)
### Lỗi 1: VirtualBox Unattended Installation can thiệp sai cấu hình Ubuntu Server
- **Hiện tượng**: Khi cài đặt Ubuntu Server 24.04, máy ảo tự động chạy tiến trình cài đặt tự động, tự thiết lập thông tin đăng nhập ngẫu nhiên và nhận IP động (DHCP) thay vì cho phép cấu hình mạng tĩnh theo thiết kế ban đầu.
### Lỗi 2: Không truy cập được pfSense WebGUI qua giao thức HTTPS
- **Hiện tượng**: Máy Host mở trình duyệt truy cập địa chỉ `https://192.168.56.254` bị báo lỗi kết nối không phản hồi (Connection timed out).

### Lỗi 3: Đặt sai thứ tự Firewall Rule trên pfSense làm mất kết nối toàn bộ mạng LAN
- **Hiện tượng**: Sau khi bổ sung rule chặn né proxy, Client (DC01) hoàn toàn không ping được Gateway pfSense, mất khả năng phân giải DNS và không thể kết nối tới Proxy.

### Lỗi 4: Máy DMZ-WEB không thể phân giải tên miền để cài đặt gói (`apt update`)
- **Hiện tượng**: Đứng trên máy DMZ-WEB thực hiện lệnh `apt update` gặp lỗi `Temporary failure resolving 'archive.ubuntu.com'`, không thể cài đặt Nginx.

### Lỗi 5: E2Guardian không cập nhật danh sách domain cấm do lỗi cú pháp ghi đè
- **Hiện tượng**: Client truy cập thử `http://blockedgame.lab7.test` vẫn tải được bình thường, E2Guardian không thực hiện hành vi chặn theo danh mục cấm.

### Lỗi 6: E2Guardian hiển thị trang cấm cứng thay vì hiển thị cảnh báo có nút Bypass
- **Hiện tượng**: Khi Client truy cập `http://weapons.lab7.test`, hệ thống hiển thị màn hình Access Denied cấm hoàn toàn, không xuất hiện liên kết vượt rào tạm thời (Bypass URL).

### Lỗi 7: Truy cập tải file test virus EICAR bị lỗi `404 Not Found` từ Nginx
- **Hiện tượng**: Client mở trình duyệt truy cập `http://download.lab7.test/files/eicar.com.txt` nhận về thông báo lỗi `404 Not Found nginx/1.24.0 (ubuntu)`.
