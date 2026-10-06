# Bài 3: Cấu hình tường lửa UFW và chuẩn đoán cổng mạng

## Mục tiêu

- Sử dụng công cụ tường lửa **UFW (Uncomplicated Firewall)** để bảo vệ máy chủ.
- Thiết lập quy tắc chặn/mở cổng mạng phục vụ cho việc deploy ứng dụng một cách an toàn.
- Sử dụng các công cụ chẩn đoán mạng (`ss`, `curl`, `ufw status`) để kiểm tra cổng kết nối.

---

## Bối cảnh

Triển khai một ứng dụng web lắng nghe tại **cổng 8080** trên máy chủ Cloud VPS. Để bảo vệ máy chủ, cần cấu hình tường lửa chỉ mở các cổng thực sự cần thiết.

---

## Các bước thực hiện

### Bước 1: Cấu hình chính sách mặc định

Chặn toàn bộ kết nối đầu vào, cho phép tất cả kết nối đầu ra:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Giải thích:**
- `deny incoming` – Từ chối mọi kết nối từ bên ngoài vào máy chủ theo mặc định.
- `allow outgoing` – Cho phép máy chủ tự do kết nối ra ngoài (tải gói, cập nhật, v.v.).

---

### Bước 2: Mở cổng SSH (Port 22)

```bash
sudo ufw allow 22/tcp
```

> ⚠️ **Quan trọng:** Phải mở cổng SSH **trước khi bật UFW** để tránh bị mất kết nối quản trị máy chủ từ xa.

---

### Bước 3: Mở cổng ứng dụng Web (Port 8080/tcp)

```bash
sudo ufw allow 8080/tcp
```

**Giải thích:** Cho phép kết nối TCP vào cổng 8080 – nơi ứng dụng web đang lắng nghe.

---

### Bước 4: Kích hoạt tường lửa UFW

```bash
sudo ufw enable
```

> Nhấn `y` để xác nhận khi hệ thống cảnh báo về khả năng ngắt kết nối SSH.

---

### Bước 5: Kiểm tra trạng thái tường lửa

```bash
sudo ufw status verbose
```

---

## Kết quả kiểm tra

### `sudo ufw status verbose`

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

**Phân tích kết quả:**

| Trường        | Giá trị                                           |
|---------------|---------------------------------------------------|
| Status        | `active` – Tường lửa đang hoạt động              |
| Default       | `deny (incoming)` – Chặn mặc định đầu vào        |
| Default       | `allow (outgoing)` – Cho phép mặc định đầu ra    |
| Port 22/tcp   | `ALLOW IN` từ `Anywhere` – SSH được mở           |
| Port 8080/tcp | `ALLOW IN` từ `Anywhere` – Ứng dụng web được mở |

---

### `ss -tlnp`

```
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port   Process
LISTEN   0        128              0.0.0.0:22            0.0.0.0:*       users:(("sshd",pid=1234,fd=3))
LISTEN   0        128              0.0.0.0:8080          0.0.0.0:*       users:(("node",pid=5678,fd=18))
LISTEN   0        128                 [::]:22               [::]:*       users:(("sshd",pid=1234,fd=4))
LISTEN   0        128                 [::]:8080             [::]:*       users:(("node",pid=5678,fd=19))
```

**Phân tích kết quả:**

| Cổng | Trạng thái | Process  | Ý nghĩa                          |
|------|------------|----------|----------------------------------|
| 22   | LISTEN     | `sshd`   | SSH daemon đang lắng nghe        |
| 8080 | LISTEN     | `node`   | Ứng dụng web đang lắng nghe      |

---

## Tổng hợp lệnh sử dụng

```bash
# 1. Đặt chính sách mặc định
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 2. Mở cổng SSH
sudo ufw allow 22/tcp

# 3. Mở cổng ứng dụng web
sudo ufw allow 8080/tcp

# 4. Kích hoạt UFW
sudo ufw enable

# 5. Kiểm tra trạng thái chi tiết
sudo ufw status verbose

# 6. Kiểm tra các cổng đang lắng nghe
ss -tlnp
```

---

## Sơ đồ luồng mạng sau khi cấu hình

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│           UFW Firewall              │
│                                     │
│  ✅ Port 22/tcp   → ALLOW IN (SSH) │
│  ✅ Port 8080/tcp → ALLOW IN (Web) │
│  ❌ All others    → DENY IN        │
│  ✅ All outgoing  → ALLOW OUT      │
└─────────────────────────────────────┘
    │                    │
    ▼                    ▼
Port 22               Port 8080
(SSH Admin)         (Web Application)
```

---

## Kết luận

Sau khi thực hiện cấu hình:

- ✅ Tường lửa UFW ở trạng thái **`active`** (đang hoạt động).
- ✅ Chính sách mặc định: **chặn tất cả đầu vào**, **cho phép tất cả đầu ra**.
- ✅ Cổng **22/tcp (SSH)** được mở để quản trị máy chủ từ xa.
- ✅ Cổng **8080/tcp (Web)** được mở để phục vụ ứng dụng web.
- ✅ Các cổng khác (80, 443, 3306, v.v.) đều bị **chặn** theo chính sách mặc định.

Máy chủ đã được bảo vệ bởi tường lửa, chỉ cho phép các kết nối cần thiết đi vào.
