# Tài liệu đào tạo: Cấu hình Load Balancing trên F5 BIG-IP & Thiết kế Health Check Backend

> Phạm vi: Áp dụng chung cho tất cả các hệ thống Backend (Nginx, Apache, Tomcat, IIS, Node.js, Spring Boot, Go...) phục vụ các loại dịch vụ/domain khác nhau đặt sau F5 BIG-IP.

---

## 1. Kiến trúc tổng quan

Một Virtual Server trên F5 LTM gồm các khối chính, cần nắm rõ trước khi lựa chọn cấu hình:

| Thành phần | Vai trò |
|---|---|
| **Virtual Server** | Địa chỉ IP:Port mà client kết nối vào (VIP) |
| **Pool** | Nhóm các server thật (Pool Member/Node) xử lý traffic |
| **Monitor (Health check)** | Kiểm tra pool member còn sống/khỏe hay không |
| **Persistence Profile** | Quyết định một client có luôn được đưa về cùng một server hay không |
| **LB Method** | Thuật toán chọn server nào trong pool để xử lý request tiếp theo |
| **SSL Profile (Client/Server)** | Xử lý mã hóa TLS ở chiều client→F5 và F5→server |
| **iRule** | Script TCL can thiệp vào luồng traffic để xử lý logic nâng cao |

Thứ tự xử lý một request: `Client → SSL Client Profile (giải mã) → iRule (nếu có) → kiểm tra Persistence → LB Method chọn member → SSL Server Profile (mã hóa lại nếu bridging) → Monitor quyết định member có "available" hay không`.

---

## 2. Load Balancing Method — Nguyên tắc lựa chọn thuật toán

LB Method quyết định **request tiếp theo được đưa vào member nào** trong pool.

| Method | Cơ chế | Tình huống áp dụng |
|---|---|---|
| **Round Robin** | Lần lượt xoay vòng qua từng member | Các server có cấu hình phần cứng tương đương, tải xử lý mỗi request tương tự nhau. Là lựa chọn mặc định phổ biến nhất, đơn giản, dễ dự đoán hành vi. |
| **Ratio (member/node)** | Chia tải theo trọng số (weight) đặt trước | Các server có cấu hình khác nhau (ví dụ một server có cấu hình mạnh gấp đôi) → đặt tỷ lệ 2:1 |
| **Least Connections** | Đưa request vào member đang có ít kết nối mở nhất | Ứng dụng có kết nối tồn tại lâu (long-lived), tải không đều theo thời gian, ví dụ WebSocket, các API xử lý nặng |
| **Fastest** | Chọn member có thời gian phản hồi nhanh nhất gần đây | Khi độ trễ xử lý giữa các server chênh lệch lớn (khác vùng địa lý, khác cấu hình phần cứng) |
| **Observed / Predictive** | Kết hợp số connection và thời gian phản hồi (Predictive có thêm yếu tố dự đoán xu hướng) | Hệ thống quy mô lớn, tải biến động mạnh, cần cân bằng thông minh hơn Round Robin thuần |
| **Least Sessions** | Dựa trên số session đang active (khác với connection ở tầng TCP) | Ứng dụng có dùng persistence, cần cân bằng theo số session thực tế thay vì số kết nối TCP |

**Khuyến nghị chung:** Với các cụm backend có cấu hình đồng nhất, Round Robin là lựa chọn phù hợp cho phần lớn dịch vụ web thông thường. Chuyển sang Least Connections khi phát hiện tải bị dồn lệch do các kết nối tồn tại lâu.

---

## 3. Persistence Profile — Xác định có cần giữ session hay không

Persistence quyết định: **Cùng một client có luôn được đưa trở lại đúng server đã phục vụ trước đó hay không.**

### 3.1 Các loại persistence và ý nghĩa thực tế

| Loại | Cơ chế | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **None** | Không áp dụng persistence, mỗi request được phân phối tự do theo LB Method | Cân bằng tải tối ưu nhất, cấu hình đơn giản | Sẽ mất session nếu ứng dụng lưu session cục bộ trên bộ nhớ RAM của từng server (In-Memory Session) |
| **Source Address Affinity** | Dựa vào địa chỉ IP nguồn của client | Đơn giản, không cần thay đổi gì phía ứng dụng | Nếu nhiều client đi qua chung một NAT/Proxy (IP nguồn giống nhau), toàn bộ các client đó sẽ bị dồn vào một server, gây mất cân bằng tải |
| **Cookie (Insert/Rewrite)** | F5 chèn thêm một cookie định danh member vào response | Chính xác theo từng client thực (trình duyệt), không bị ảnh hưởng bởi việc nhiều client dùng chung IP qua NAT | Chỉ hoạt động với HTTP/HTTPS; không dùng được cho traffic không phải web; cần bật HTTP profile trên Virtual Server |
| **Destination Address Affinity** | Dựa theo địa chỉ IP đích (VIP) | Gần như không có tác dụng phân biệt client trong trường hợp VIP chỉ có một địa chỉ cố định | Là loại **dễ bị chọn nhầm** trong thực tế triển khai — không tạo ra stickiness thực sự khi có nhiều client cùng gọi vào một VIP. |
| **SSL Session ID** | Dựa vào SSL Session ID trong quá trình bắt tay TLS | Hữu ích khi dùng mô hình SSL Passthrough (F5 không giải mã được nội dung để đọc cookie) | Session ID có thể thay đổi khi client renegotiate, độ ổn định thấp hơn Cookie |
| **Universal Persistence** | Dùng iRule để tự định nghĩa khóa persistence (ví dụ theo một header riêng, theo JSESSIONID, Bearer token) | Linh hoạt tối đa, tùy biến theo yêu cầu nghiệp vụ | Cần viết iRule, độ phức tạp triển khai cao hơn |

### 3.2 Quy trình lựa chọn

```text
Ứng dụng có lưu session cục bộ trên từng server không?
├── KHÔNG (stateless, JWT token, hoặc session lưu tập trung tại Redis, Memcached, Database dùng chung)
│      → Chọn "None". Round Robin thuần, đạt hiệu quả cân bằng tải tối ưu.
│
└── CÓ (session mặc định lưu trong bộ nhớ RAM của server backend)
       ├── Traffic là HTTP/HTTPS thuần, không có nhiều client dùng chung NAT
       │      → Cookie persistence (khuyến nghị cho phần lớn ứng dụng web)
       │
       └── Có SSL Passthrough (F5 không giải mã) hoặc traffic không phải HTTP
              → SSL Session ID persistence hoặc Source Address
```
---

## 4. Health Check (Monitor) — Nguyên tắc kiểm tra tình trạng backend

### 4.1 Các loại Monitor trên F5

| Loại Monitor | Tầng kiểm tra | Mô tả | Tình huống áp dụng |
|---|---|---|---|
| **ICMP (Gateway ICMP)** | Network (Layer 3) | Ping đến địa chỉ IP server | Chỉ nên dùng bổ sung, không đủ để xác định dịch vụ web/API có đang hoạt động hay không |
| **TCP** | Transport (Layer 4) | Kiểm tra bắt tay TCP thành công tại cổng chỉ định | Backend không phải HTTP (Database, LDAP, Mail...), hoặc chỉ cần xác định cổng dịch vụ có đang mở |
| **HTTP** | Application (Layer 7) | Gửi HTTP request dạng plaintext và kiểm tra response | Áp dụng khi kết nối giữa F5 và server là **plaintext HTTP** (chế độ SSL Offload/Termination) |
| **HTTPS** | Application (Layer 7) + TLS | Tương tự HTTP nhưng thực hiện bắt tay TLS trước khi gửi request | Áp dụng khi kết nối giữa F5 và server **yêu cầu TLS** (chế độ SSL Bridging) |
| **GET URI / HEAD URI** | Application | Biến thể của monitor HTTP/HTTPS, chỉ cần khai báo URI đơn giản | Dùng khi chỉ cần kiểm tra một URL trả về mã 200, không cần tùy chỉnh Host header phức tạp |

### 4.2 Xử lý Health check khi backend dùng Host header (Virtual Host / Server Name Binding)

Trong các hệ thống backend chạy nhiều domain/dịch vụ trên cùng một IP:Port (như VirtualHost trên Nginx/Apache, IIS Binding), nếu monitor chỉ gửi `GET / HTTP/1.1` mà không kèm đúng header `Host:` khớp với cấu hình backend, server sẽ trả về **404**, **500** hoặc route sai site — dẫn đến monitor báo fail dù server vẫn chạy bình thường.

Cấu hình mẫu (loại monitor **HTTPS**, áp dụng khi kết nối F5–server là TLS):

    Type: HTTPS
    Interval: 5
    Timeout: 16
    Send String:  GET /health HTTP/1.1\r\nHost: <ten-mien-dich-vu>\r\nConnection: Close\r\n\r\n
    Receive String: HTTP/1.1 200

**Nguyên tắc bắt buộc:** Mỗi domain/dịch vụ riêng biệt cần có **một monitor riêng** (khác Send String theo Host header tương ứng) và được gắn vào **một pool riêng** của dịch vụ đó, ngay cả khi các pool này chia sẻ chung các server vật lý.

### 4.3 Công thức xác định Interval và Timeout

Quy tắc khuyến nghị: `Timeout >= (Interval x 3) + 1`

Ví dụ: Interval = 5 giây -> Timeout tối thiểu = 16 giây.

- Interval quá ngắn (1–2 giây) làm tăng tải không cần thiết lên cả server và F5.
- Timeout quá dài khiến khi server thực sự gặp sự cố, F5 mất nhiều thời gian hơn để đánh dấu DOWN, kéo dài thời gian gián đoạn dịch vụ đối với người dùng.

### 4.4 Hướng dẫn thiết kế endpoint `/health` chuẩn mực cho ứng dụng Backend

Trong thực tế triển khai, nhiều hệ thống cấu hình health check bằng cách tạo một file tĩnh (như `health.html`, `index.md`) hoặc một endpoint đơn giản chỉ trả về mã HTTP 200 OK. Cách làm này **chỉ xác nhận được web server/reverse proxy còn sống**, nhưng **không phản ánh đúng ứng dụng có thực sự phục vụ được người dùng hay không**.

#### A. Phân biệt Endpoint `/health` chuẩn mực vs. Trả về File tĩnh / Check HTTP 200 đơn thuần

| Tiêu chí | Trả về File tĩnh / HTTP 200 đơn thuần | Endpoint `/health` chuẩn mực (Readiness Check) |
|---|---|---|
| **Cơ chế** | Trả về nội dung tĩnh hoặc mã 200 ngay khi server nhận kết nối. | Chạy bài "kiểm tra nhanh" các dịch vụ phụ thuộc cốt lõi trước khi phản hồi. |
| **Đánh giá Database/Cache** | **Không kiểm tra** — Nếu DB sập hay kẹt connection pool, F5 vẫn tưởng server "khỏe" và liên tục đẩy traffic vào. | **Có kiểm tra** — Thực hiện query siêu nhẹ (như `SELECT 1;` với DB, `PING` với Redis). |
| **Độ tin cậy** | **Thấp (False Positive):** Server sống nhưng ứng dụng bị sập logic (deadlock, mất kết nối DB). | **Cao:** Phản ánh chính xác khả năng xử lý nghiệp vụ thực tế của ứng dụng. |
| **Sử dụng tài nguyên F5** | F5 phải tốn CPU để parse chuỗi string matching trong đống HTML/Text tĩnh nạp về. | F5 xử lý tối ưu bằng cách kiểm tra **HTTP Status Code** (200 = UP, 503 = DOWN) và JSON gọn nhẹ. |

#### B. Phân loại Health Check theo kiến trúc Microservices / Cloud Native

Một hệ thống chuẩn mực cần phân biệt 2 khái niệm endpoint health check:
1. **Liveness Check (`/health/live` - Ứng dụng có đang chạy không?):** Kiểm tra tiến trình (process) ứng dụng có bị treo hoặc rơi vào trạng thái deadlock hay không. 
   - *Xử lý:* Phản hồi ngay `200 OK` mà không gọi sang các service con. Tần suất check nhanh (mỗi 2-5 giây).
2. **Readiness Check (`/health/ready` - Ứng dụng đã sẵn sàng nhận traffic chưa? - *Khuyên dùng cho F5*):** Kiểm tra xem ứng dụng có đủ điều kiện để xử lý request của người dùng hay không.
   - *Xử lý:* Tự động gọi kiểm tra các thành phần phụ thuộc bắt buộc: Database (`SELECT 1`), Redis/Cache (`PING`), dung lượng ổ cứng, kết nối Message Queue...

#### C. Thiết kế định dạng phản hồi (Response Format)

- **Trạng thái UP (Tất cả ổn định):** Trả về **HTTP 200 OK** kèm JSON gọn nhẹ:

    {
      "status": "UP",
      "timestamp": "2026-09-19T04:15:00Z",
      "components": {
        "database": { "status": "UP", "duration_ms": 2 },
        "redis": { "status": "UP", "duration_ms": 1 },
        "diskSpace": { "status": "UP", "free_bytes": 45192837120 }
      }
    }

- **Trạng thái DOWN (Thành phần Core sập):** Bắt buộc trả về mã **HTTP 503 Service Unavailable** (hoặc **500 Internal Server Error**) để F5 chủ động ngắt traffic sang member này:

    {
      "status": "DOWN",
      "timestamp": "2026-09-19T04:15:02Z",
      "components": {
        "database": { "status": "DOWN", "error": "Connection timeout" },
        "redis": { "status": "UP", "duration_ms": 1 }
      }
    }

#### D. Các nguyên tắc "xương máu" khi viết trang `/health`

1. **Cấu hình Timeout cực ngắn cho kết nối con (1–2 giây):** Khi kiểm tra DB/Redis trong trang `/health`, hãy đặt timeout của truy vấn là 1-2s. Nếu DB bị treo mà không đặt timeout, chính trang `/health` sẽ bị nghẽn và kéo sập toàn bộ luồng xử lý của Web Server.
2. **Không check quá sâu (Đừng phụ thuộc Third-party):** Không gọi sang các API bên thứ 3 không bắt buộc (SMS, Tỷ giá, Payment gateway...). Nếu các dịch vụ này sập mà bạn đánh dấu DOWN ứng dụng, bạn sẽ tự đánh sập toàn cụm server trong khi các chức năng cốt lõi khác vẫn phục vụ tốt.
3. **Bảo mật trang `/health`:** Trang `/health` chứa thông tin nhạy cảm về hạ tầng.
   - *Giải pháp 1:* Đặt Rule Restrict IP chỉ cho phép IP của Load Balancer (Self IP của F5) truy cập trang `/health`.
   - *Giải pháp 2:* Với truy cập Public, chỉ trả về mã 200/503 kèm `{"status": "UP"}` gọn nhẹ, ẩn toàn bộ chi tiết `components` nếu không có Bearer Token/Secret Header.
4. **Tận dụng thư viện chuẩn của các Framework (Không tự code từ đầu):**
   - **Java Spring Boot:** Sử dụng `Spring Boot Actuator` (endpoint `/actuator/health`).
   - **.NET Core:** Sử dụng thư viện gốc `Microsoft.Extensions.Diagnostics.HealthChecks` (endpoint `/health`).
   - **Node.js (Express):** Sử dụng thư viện `@cloudnative/health-connect` hoặc `express-actuator`.
   - **Go (Golang):** Sử dụng các module như `github.com/hellofresh/health-go`.

---

## 5. SSL trên F5 — Ba mô hình cần phân biệt rõ

Việc phân biệt đúng mô hình SSL quyết định trực tiếp loại Monitor và loại Persistence cần sử dụng.

### 5.1 SSL Offload / Termination

    Client --[HTTPS]--> F5 (giải mã) --[HTTP plaintext]--> Backend Server

- F5 xử lý toàn bộ TLS, backend nhận HTTP thuần.
- **Ưu điểm:** Giảm tải xử lý mã hóa cho backend server, F5 đọc được nội dung (header, cookie) để áp dụng iRule/Cookie persistence.
- **Cấu hình liên quan:** Chỉ cần **Client SSL Profile**, không cần Server SSL Profile. Monitor sử dụng loại **HTTP**.

### 5.2 SSL Bridging (Re-encrypt)

    Client --[HTTPS]--> F5 (giải mã) --[HTTPS mã hóa lại]--> Backend Server

- F5 vẫn đọc được nội dung ở giữa (phục vụ iRule, cookie persistence...) nhưng dữ liệu truyền tới backend vẫn được mã hóa, đáp ứng tiêu chuẩn bảo mật end-to-end.
- **Cấu hình liên quan:** Cần cả **Client SSL Profile** (giữa client và F5) và **Server SSL Profile** (giữa F5 và server).
- **Monitor:** Bắt buộc dùng loại **HTTPS**, không dùng HTTP.

### 5.3 SSL Passthrough

    Client --[HTTPS]--> F5 (không giải mã, chỉ chuyển tiếp gói TCP) --[HTTPS]--> Backend Server

- F5 không đọc được nội dung, không thể áp dụng iRule dựa trên tầng HTTP, không dùng được Cookie persistence.
- Áp dụng khi có yêu cầu bảo mật đặc biệt (F5 không được giữ private key), hoặc khi client cần xác thực mTLS trực tiếp với backend.
- **Persistence:** Chỉ dùng SSL Session ID hoặc Source Address.
- **Monitor:** Dùng loại **TCP**.

### 5.4 SNI (Server Name Indication)

Khi nhiều domain/dịch vụ dùng chung một Virtual Server IP:443 nhưng sử dụng các chứng chỉ SSL khác nhau:
- Tạo nhiều **Client SSL Profile**, mỗi profile gắn một chứng chỉ riêng.
- Gán đồng thời các Client SSL Profile lên Virtual Server — F5 sẽ tự chọn profile phù hợp dựa trên thông tin SNI trong gói `ClientHello` TLS.

---

## 6. Xác định địa chỉ IP thật của client tại Backend (Client IP Pass-Through)

Khi F5 bật **SNAT (Source Network Address Translation)**, IP nguồn của các gói tin đến Backend sẽ bị đổi thành IP của F5 (Self IP). Backend sẽ không thấy IP thực của người dùng nếu không cấu hình đúng.

### 6.1 Các phương án xử lý cho ứng dụng Web (HTTP/HTTPS)

1. **Chèn Header `X-Forwarded-For` (Khuyến nghị cao nhất):**
   - **Phía F5:** Bật tùy chọn `Insert X-Forwarded-For` trong **HTTP Profile** gắn với VIP (hoặc dùng iRule chèn thủ công).
   - **Phía Backend Server:** Cấu hình Web Server / Middleware đọc header `X-Forwarded-For` để ghi log hoặc nhận diện IP:
     - *Nginx:* Dùng module `realip` (`set_real_ip_from <F5_IP>; real_ip_header X-Forwarded-For;`).
     - *Apache:* Dùng module `mod_remoteip` (`RemoteIPHeader X-Forwarded-For`).
     - *IIS:* Dùng module *Application Request Routing (ARR)* hoặc tùy chỉnh IIS Logging Custom Fields.
     - *Node.js/Spring Boot/ASP.NET:* Đọc từ `req.headers['x-forwarded-for']` hoặc bật cấu hình `trust proxy`.

2. **Lưu ý với Multi-Proxy (WAF/CDN đứng trước F5):**
   - Header `X-Forwarded-For` sẽ chứa một chuỗi danh sách các IP dạng: `Client_IP, Proxy1_IP, Proxy2_IP`.
   - Ứng dụng Backend cần lấy giá trị IP **đầu tiên** trong chuỗi danh sách đó để xác định đúng IP thực của người dùng.

### 6.2 Các phương án xử lý cho ứng dụng TCP thuần (LDAP, Database, SMTP, FTP, RDP...)

Với các giao thức không phải HTTP, khái niệm HTTP Header hoàn toàn không tồn tại.

| Phương án | Cơ chế | Điều kiện & Ghi chú |
|---|---|---|
| **Disable SNAT (Khuyến nghị)** | Đặt SNAT = None trên Virtual Server, giữ nguyên IP nguồn gốc của client | **Bắt buộc:** Route mặc định (Default Gateway) của Backend Server phải trỏ trực tiếp về Self IP của F5 để đảm bảo luồng traffic trả về đi qua F5. |
| **Proxy Protocol** | F5 chèn thông tin IP client ở đầu luồng TCP trước dữ liệu ứng dụng | Backend (như Nginx, HAProxy, MariaDB) phải hỗ trợ bật tính năng đọc Proxy Protocol. |
| **Ghi Log tại F5 via HSL** | Dùng iRule ghi nhận log kết nối từ F5 và đẩy qua Syslog/SIEM | Áp dụng khi không đổi được Gateway của backend và backend không hỗ trợ Proxy Protocol. Phục vụ mục đích audit tra cứu. |

---

## 7. iRule — Các mẫu script TCL nâng cao phổ biến

### 7.1 Định tuyến theo Host Header về đúng Pool

    when HTTP_REQUEST {
        switch -glob [string tolower [HTTP::host]] {
            "api.example.com*" { pool pool_api_backend }
            "web.example.com*" { pool pool_web_backend }
            default            { pool pool_default_backend }
        }
    }

### 7.2 Redirect HTTP sang HTTPS tự động

    when HTTP_REQUEST {
        HTTP::redirect "https://[HTTP::host][HTTP::uri]"
    }

### 7.3 Chèn Header bảo mật và thông tin Protocol cho Backend

    when HTTP_REQUEST {
        HTTP::header insert X-Forwarded-Proto "https"
        HTTP::header insert X-Forwarded-For [IP::client_addr]
    }

---

## 8. Bảng tổng hợp khuyến nghị cấu hình chuẩn

| Hạng mục | Cấu hình khuyến nghị |
|---|---|
| **LB Method** | **Round Robin** cho cụm server đồng nhất; **Least Connections** cho ứng dụng có kết nối dài/tải nặng. |
| **Persistence** | **None** nếu ứng dụng là Stateless hoặc dùng External Session Store (Redis); **Cookie Persistence** nếu dùng In-Memory Session. |
| **Monitor Type** | **HTTP** khi SSL Offload; **HTTPS** khi SSL Bridging; **TCP** khi SSL Passthrough hoặc dịch vụ non-HTTP. |
| **Health Check URI** | Trỏ đến endpoint `/health` (Readiness Check), **không** check trang chủ `/` hoặc các trang có query nặng. |
| **Client IP (HTTP)** | Bật **Insert X-Forwarded-For** trên HTTP Profile, cấu hình Nginx/Apache/IIS/App đọc đúng Header này. |
| **Client IP (Non-HTTP)** | **Disable SNAT** (Cần đổi Default Gateway backend về F5) hoặc dùng **Proxy Protocol**. |

---

## 9. Checklist triển khai Production

- [ ] Xác định đúng mô hình SSL (Offload / Bridging / Passthrough) để chọn loại Monitor tương ứng.
- [ ] Xây dựng trang `/health` riêng biệt trên Backend, có check kết nối DB/Cache nhẹ (timeout <= 2s), trả về mã HTTP 200 (UP) hoặc 503 (DOWN).
- [ ] Khai báo đúng `Host:` header trong Send String của Monitor nếu Backend chạy nhiều Virtual Host / Binding.
- [ ] Đặt tỷ lệ thời gian `Timeout >= (Interval x 3) + 1` cho Monitor.
- [ ] Xác nhận ứng dụng là Stateful hay Stateless để chọn loại Persistence Profile thích hợp.
- [ ] Đã cấu hình chèn và đọc `X-Forwarded-For` trên cả F5 và Web Server / Middleware Backend.
- [ ] Kiểm thử thực tế: Thử dừng dịch vụ Backend hoặc ngắt kết nối Database thử nghiệm để đảm bảo F5 đánh dấu node DOWN chính xác và không điều hướng traffic vào node lỗi.
