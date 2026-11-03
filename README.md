# BÁO CÁO PHÂN TÍCH SỰ CỐ MẠNG & WEB (SESSION 18 - BÀI 1)
## 1. Yêu cầu & Bối cảnh kỹ thuật

### Chức năng 1: Phân tích chi tiết tiến trình DNS Lookup
Khi Nam nhập địa chỉ `phong-dieu-phoi.greenlog.vn` vào thanh URL trình duyệt và nhấn Enter:
1. **Kiểm tra Bộ nhớ tạm (DNS Cache):**
   * **Browser Cache:** Trình duyệt kiểm tra lịch sử phân giải đã lưu trước đó.
   * **OS Cache:** Hệ điều hành kiểm tra tệp `hosts` và bộ nhớ cache DNS của máy (`ipconfig /displaydns`).
   * **Router Cache:** Kiểm tra cache tại Modem/Router Wi-Fi nội bộ.
2. **Hỏi Local DNS Server (DNS Resolver):** Gửi yêu cầu tới Resolver của nhà mạng (ISP) hoặc DNS công cộng (`8.8.8.8`). Nếu cache miss, tiến trình đệ quy (Recursive Resolution) bắt đầu:
   * **Root Nameserver:** Resolver hỏi cụm Root Server (`.`), nhận về địa chỉ IP của TLD Nameserver quản lý đuôi `.vn`.
   * **TLD Nameserver (`.vn`):** Resolver gửi truy vấn tiếp đến TLD Server, nhận về IP của Authoritative Nameserver phụ trách `greenlog.vn`.
   * **Authoritative Nameserver:** Máy chủ nắm giữ bản ghi DNS gốc tra cứu bản ghi (`A`/`CNAME`) của `phong-dieu-phoi` và trả về địa chỉ IP công khai của máy chủ đích.
3. **Phản hồi và Kết nối:** Local DNS Resolver trả IP về cho trình duyệt. Trình duyệt lưu cache và bắt đầu bắt tay TCP (TCP 3-way Handshake) và TLS để gửi request HTTP/HTTPS tới máy chủ.

### Chức năng 2: Chẩn đoán lỗi DNS vs Lỗi 500 Internal Server Error
Nhận định của Nam *"do máy chủ công ty bị sập nguồn"* là **sai**.
* **Bản chất lỗi DNS (`DNS_PROBE_FINISHED_NXDOMAIN`):** Lỗi xảy ra ở tầng phân giải tên miền (Non-Existent Domain). Trình duyệt không tìm ra địa chỉ IP nên gói tin **chưa bao giờ chạm tới được máy chủ đích**. Nguyên nhân nằm ở phía Client (gõ sai tên miền) hoặc DNS Provider chưa cấu hình bản ghi.
* **Bản chất lỗi 500 Internal Server Error:** Mã trạng thái HTTP phía Server. Gói tin đã chạm tới Web Server thành công nhưng server gặp lỗi crash code/database/cấu hình khi xử lý.

### Ràng buộc: Hậu quả gõ nhầm tên miền (`phong-dieu-phoi.grenlog.vn`)
* **Không tìm thấy bản ghi:** Tên miền sai chính tả không tồn tại trong danh bạ DNS → Trình duyệt ngắt kết nối và báo lỗi `DNS_PROBE_FINISHED_NXDOMAIN`.
* **Nguy cơ an ninh (Typosquatting/Phishing):** Kẻ tấn công có thể đăng ký trước tên miền sai để dựng trang giả mạo, đánh cắp mật khẩu đăng nhập của nhân viên điều phối.

---

## 2. Yêu cầu đầu ra

### Phần 1: Phân tích luồng dữ liệu DNS
* **Input:** Tên miền `phong-dieu-phoi.greenlog.vn`
* **Output:** Địa chỉ IP máy chủ (ví dụ: `203.162.x.x`)
* **Sơ đồ luồng:**
```text
[Trình duyệt của Nam]
        │ (Nhập phong-dieu-phoi.greenlog.vn)
        ▼
 [Kiểm tra Cache: Browser -> OS -> Router]
        │ (Miss)
        ▼
[DNS Resolver / ISP] ──1. Hỏi Root (.)──────────► [Root Server]
        │            ◄──Trả về IP của .vn───────┘
        │
        ├────────────2. Hỏi TLD (.vn)──────────► [TLD Server (.vn)]
        │            ◄──Trả về Auth Server──────┘
        │
        └────────────3. Hỏi Record phòng-điều-phối─► [Authoritative Server]
                     ◄──Trả về IP máy chủ───────┘
        │
        ▼ (IP Address)
[Trình duyệt của Nam] ──HTTP/HTTPS Request────────► [Máy chủ GreenLogistics]
```

### Phần 2: Triển khai quy trình kiểm tra HTTP Status Code
1. Nhấn `F12` mở DevTools, chọn tab **Network**.
2. Bật nút tròn đỏ ghi nhận và tick chọn **Preserve log**.
3. Tải lại trang:
   * Nếu hiển thị `(failed)` hoặc `Status: net::ERR_NAME_NOT_RESOLVED`: Yêu cầu chưa chạm tới máy chủ.
   * Nếu nhận bất kỳ mã số HTTP nào ($2xx, 3xx, 4xx, 5xx$): Yêu cầu đã chạm tới máy chủ thành công.

### Phần 3: Kiểm thử tình huống lỗi 404 Not Found
Khi sửa đúng tên miền nhưng nhận mã **404 Not Found**:
* Phân giải DNS và kết nối mạng đã thành công 100% (đã tới máy chủ).
* Nguyên nhân: Đường dẫn URL/route trên web server không tồn tại, chưa cấu hình Virtual Host / Reverse Proxy trên NGINX/Apache cho subdomain, hoặc file tài nguyên đã bị xóa/đổi tên.
