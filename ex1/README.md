# Bài 1: Quản lý người dùng giới hạn và Truyền tải dữ liệu qua SFTP
## Các bước thực hiện trên VPS
### Bước 1: Cấu hình trên máy chủ Linux (VPS)
Thực hiện tất cả các lệnh sau trên terminal kết nối VPS của bạn:

**1. Tạo tài khoản người dùng hệ thống chuyên dụng cho SFTP:**
```bash
sudo adduser sftp-user
```

Lệnh này sẽ tạo một tài khoản hệ thống mới hoàn toàn độc lập, không có cấu hình mặc định nào liên quan đến quyền quản trị. 
Khi được yêu cầu nhập mật khẩu, hãy đặt một mật khẩu mạnh, có độ dài tối thiểu 12 ký tự kết hợp chữ hoa, chữ thường, số và 
ký tự đặc biệt theo yêu cầu bảo mật. Các thông tin bổ sung như tên đầy đủ, số điện thoại bạn có thể bỏ qua hoàn toàn bằng
cách nhấn Enter liên tục, không cần điền. Sau khi hoàn tất bước này, hệ thống sẽ tự động tạo thư mục nhà người dùng 
/home/sftp-user và shell đăng nhập mặc định cho tài khoản.

**2. Xác nhận ngay tài khoản không có quyền sudo**
Sau khi tạo xong tài khoản, chạy lệnh kiểm tra quyền đầu tiên:
```bash
id sftp-user
```

Kết quả trả về sẽ hiển thị toàn bộ các nhóm mà tài khoản sftp-user thuộc về. Bạn cần xác nhận rằng trong danh sách nhóm này 
không xuất hiện tên nhóm sudo hay wheel - đây là các nhóm có quyền chạy lệnh với đặc quyền quản trị

**3. Tạo thư mục lưu trữ log và tệp kiểm tra giả lập**
Tiếp theo chúng ta tạo cấu trúc thư mục lưu trữ tệp backup log theo đúng đường dẫn yêu cầu của bài tập:
```bash
sudo mkdir -p /var/log/app-backup/
sudo touch /var/log/app-backup/backup-check.log
sudo bash -c 'echo "Backup status: SUCCESS at $(date)" > /var/log/app-backup/backup-check.log'
```

* Lệnh mkdir -p sẽ tự động tạo toàn bộ các thư mục cha còn thiếu trong đường dẫn /var/log/app-backup/ mà không báo lỗi nếu 
thư mục đã tồn tại trước đó.
* Lệnh touch tạo tệp rỗng backup-check.log tại đường dẫn chỉ định.
* Lệnh cuối cùng sẽ ghi nội dung kiểm tra vào tệp, bao gồm dòng trạng thái thành công và thời gian hệ thống hiện tại lúc 
bạn chạy lệnh, để sau này dễ dàng đối chiếu nội dung sau khi tải về máy Windows.

**4. Cấu hình quyền truy cập chính xác cho tài khoản SFTP**
Đây là bước quan trọng nhất quyết định tài khoản có thể truy cập và đọc tệp log hay không, bạn cần chạy đúng các lệnh sau 
theo thứ tự:

```bash
sudo chown -R root:sftp-user /var/log/app-backup
sudo chmod 750 /var/log/app-backup
sudo chmod 640 /var/log/app-backup/backup-check.log
```

**5. Xác nhận lại cấu hình quyền trước khi chuyển bước tiếp theo**
Chạy hai lệnh kiểm tra cuối cùng trên VPS để đảm bảo mọi cấu hình đã chính xác:
```bash
# Xác nhận lại thông tin tài khoản và nhóm
id sftp-user

# Kiểm tra chi tiết quyền của tệp log
ls -l /var/log/app-backup/backup-check.log
```

Kết quả mong đợi hiển thị quyền của tệp là -rw-r-----, chủ sở hữu là root, nhóm sở hữu là sftp-user.

### Bước 2: Cấu hình giới hạn nâng cao (khuyến nghị bắt buộc)
Để đáp ứng yêu cầu "người dùng giới hạn" một cách hoàn chỉnh, bạn nên cấu hình thêm chroot jail trong tệp cấu hình SSH để
khóa hoàn toàn tài khoản sftp-user trong phạm vi thư mục được phép truy cập, ngăn chặn họ điều hướng đến các thư mục hệ 
thống quan trọng khác hay chạy bất kỳ lệnh nào trên VPS:

**1. Mở tệp cấu hình dịch vụ SSH bằng trình soạn thảo nano:**
```bash
sudo nano /etc/ssh/sshd_config
```

**2. Tìm đến cuối tệp, xóa bỏ mọi cấu hình liên quan đến người dùng sftp-user nếu đã tồn tại trước đó, sau đó thêm khối cấu
hình sau vào cuối tệp:**
```code
Match User sftp-user
  ChrootDirectory /var/log/app-backup
  ForceCommand internal-sftp
  AllowTcpForwarding no
  X11Forwarding no
  PermitTunnel no
```

Giải thích từng dòng cấu hình:
* Match User sftp-user: Áp dụng toàn bộ các quy tắc bên dưới chỉ riêng cho tài khoản sftp-user, không ảnh hưởng đến các tài 
khoản quản trị khác trên hệ thống.
* ChrootDirectory /var/log/app-backup: Nhốt hoàn toàn tài khoản sftp-user vào trong thư mục /var/log/app-backup/, sau khi kết
nối SFTP thành công họ sẽ không thể thấy hay điều hướng đến bất kỳ thư mục nào khác bên ngoài đường dẫn này.
* ForceCommand internal-sftp: Buộc tài khoản chỉ có thể sử dụng tính năng SFTP tích hợp sẵn của OpenSSH, từ chối hoàn toàn mọi
yêu cầu đăng nhập shell thông thường. Nghĩa là không ai có thể dùng tài khoản sftp-user để mở terminal chạy lệnh trên 
ngay cả khi biết mật khẩu.
* Các dòng cuối cùng vô hiệu hóa toàn bộ tính năng chuyển tiếp cổng, chuyển tiếp giao diện đồ họa X11 để thu hẹp bề mặt tấn 
công của tài khoản giới hạn.

**3. Lưu tệp cấu hình bằng tổ hợp phím Ctrl + O, xác nhận ghi đè và thoát nano bằng Ctrl + X. Sau đó khởi động lại dịch vụ
SSH để các thay đổi có hiệu lực:**
```bash
sudo systemctl restart sshd
```

### Bước 3: Cài đặt và kết nối từ máy tính Windows bằng SFTP Client
Trên máy tính chạy Windows của bạn, bạn có thể lựa chọn một trong các phần mềm SFTP Client được đề xuất trong yêu cầu bài 
tập, tất cả đều miễn phí và tương thích hoàn toàn với cấu hình bạn vừa thiết lập trên VPS:

**Hướng dẫn chi tiết với Bitvise SSH Client (phần mềm được khuyên dùng cao nhất)**
1. Tải và cài đặt phiên bản mới nhất của Bitvise SSH Client từ trang chủ chính thức, đây là phần mềm nhẹ, không yêu cầu cài
đặt phức tạp và được đánh giá rất cao về độ ổn định.
2. Mở phần mềm, bạn sẽ thấy giao diện kết nối chính với các ô thông tin cần điền:
*    Host: Điền địa chỉ IP công cộng của VPS bạn đang sử dụng
*    Port: Để giá trị mặc định là 22 - đây là cổng chuẩn cho kết nối SSH/SFTP
*    Username: Nhập chính xác tên tài khoản bạn vừa tạo là sftp-user
*    Authentication method: Chọn Password từ danh sách thả xuống
*    Password: Nhập mật khẩu mạnh bạn đã đặt cho tài khoản sftp-user ở bước trước
3. Bạn có thể bỏ qua toàn bộ các thiết lập nâng cao khác ở phần bên phải, giữ nguyên giá trị mặc định. Nhấn nút Log in màu
xanh để bắt đầu kết nối.
4. Lần kết nối đầu tiên, Bitvise sẽ hiển thị cửa sổ xác nhận dấu vân tay khóa máy chủ - hãy nhấn Accept and Save để lưu lại
xác nhận cho các lần kết nối sau, đây là cơ chế xác nhận bạn đang kết nối đúng với máy chủ thật, không phải máy chủ giả mạo.
5. Sau khi kết nối xác thực thành công, giao diện Bitvise sẽ hiển thị thông tin trạng thái kết nối thành công. Nhấn nút New
SFTP Window trên thanh công cụ để mở trình quản lý tệp đồ họa hai ngăn.
6. Ở khung bên phải của cửa sổ SFTP là danh sách tệp trên máy chủ từ xa, bạn sẽ thấy ngay thư mục /var/log/app-backup/ (hoặc
nếu bạn đã cấu hình chroot jail, bạn sẽ được đặt ngay bên trong thư mục này ngay lập tức sau khi kết nối). Nhấp đúp để mở
thư mục, bạn sẽ thấy tệp backup-check.log xuất hiện trong danh sách.
7. Ở khung bên trái là danh sách tệp trên máy tính Windows cục bộ, điều hướng đến thư mục bạn muốn lưu tệp log tải về. Chỉ
cần nhấp giữ và kéo thả tệp backup-check.log từ khung bên phải sang khung bên trái, Bitvise sẽ tự động thực hiện quá trình
truyền tải an toàn qua kết nối mã hóa SFTP.

### Bước 4: Xác minh kết quả và kiểm tra tính toàn vẹn dữ liệu
Sau khi quá trình tải tệp hoàn tất, bạn cần thực hiện các bước kiểm tra sau để đảm bảo đáp ứng toàn bộ yêu cầu của bài tập:
1. Mở tệp backup-check.log vừa được tải về trên máy tính Windows bằng trình soạn thảo văn bản bất kỳ (Notepad, VS Code,...). 
Quan sát nội dung tệp, ghi lại dòng trạng thái và thời gian trong tệp.
2. Quay lại cửa sổ terminal kết nối VPS, chạy lệnh hiển thị nội dung tệp gốc trên máy chủ:
```bash
cat /var/log/app-backup/backup-check.log
```
3. So sánh nội dung hiển thị trên terminal với nội dung tệp bạn vừa mở trên Windows. Toàn bộ ký tự, dòng trạng thái và thời 
gian phải khớp 100% - điều này chứng tỏ quá trình truyền tải không bị lỗi hay làm hỏng dữ liệu.
4. Thử thực hiện một vài thao tác để kiểm tra giới hạn quyền của tài khoản sftp-user: thử tạo một tệp mới trong thư mục từ 
xa, thử chỉnh sửa nội dung tệp log, thử tải một tệp lên VPS - tất cả các thao tác này phải bị từ chối, đây là bằng chứng 
quyền truy cập đã được cấu hình đúng theo nguyên tắc đặc quyền tối thiểu.