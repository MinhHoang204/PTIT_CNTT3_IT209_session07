# Bài 2: Quản trị tường lửa UFW cho cụm dịch vụ multi-port
## Các bước thực hiện:
### Bước 1: Thiết lập chính sách mặc định
Chạy theo đúng thứ tự dưới đây để tránh mất kết nối SSH:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
* deny incoming: Chặn mọi kết nối từ ngoài vào máy chủ theo mặc định.
* allow outgoing: Cho phép máy chủ kết nối ra bên ngoài bình thường.

### Bước 2: Mở các cổng dịch vụ cần thiết
Quan trọng: Luôn cho phép SSH trước khi bật UFW, nếu không bạn sẽ bị khóa ngoài máy chủ:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 8082/tcp
```

Giải thích:
* 22/tcp - SSH để quản trị.
* 80/tcp - Nginx phục vụ web.
* 8082/tcp - Ứng dụng Spring Boot.

### Bước 3: Kích hoạt tường lửa
```bash
sudo ufw enable
```

Hệ thống sẽ hỏi Command may disrupt existing ssh connections. Proceed with operation (y|n)? - gõ y và Enter. Vì bạn đã allow
SSH ở bước trước, kết nối hiện tại không bị ngắt.

###  Bước 4: Kiểm tra trạng thái
Chạy lệnh kiểm tra theo yêu cầu đề bài:
```bash
sudo ufw status verbose
```

Kết quả mong đợi sẽ hiển thị:
```code
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
8082/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
8082/tcp (v6)              ALLOW IN    Anywhere (v6)
```