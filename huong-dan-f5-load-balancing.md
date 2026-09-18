# Tài liệu đào tạo: Cấu hình Load Balancing trên F5 BIG-IP

> Phạm vi: áp dụng cho các hệ thống có backend là IIS (Windows), phục vụ chung cho nhiều loại dịch vụ/domain khác nhau đặt sau F5. Các nguyên tắc trình bày cũng áp dụng được cho các loại backend khác (Apache, Nginx, Tomcat...) với điều chỉnh tương ứng.

---

## 1. Kiến trúc tổng quan

Một Virtual Server trên F5 LTM gồm 4 khối chính, cần nắm rõ trước khi lựa chọn cấu hình:

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

## 2. Load Balancing Method — nguyên tắc lựa chọn thuật toán

LB Method quyết định **request tiếp theo được đưa vào member nào** trong pool.

| Method | Cơ chế | Tình huống áp dụng |
|---|---|---|
| **Round Robin** | Lần lượt xoay vòng qua từng member | Các server có cấu hình phần cứng tương đương, tải xử lý mỗi request tương tự nhau. Là lựa chọn mặc định phổ biến nhất, đơn giản, dễ dự đoán hành vi. |
| **Ratio (member/node)** | Chia tải theo trọng số (weight) đặt trước | Các server có cấu hình khác nhau (ví dụ một server có cấu hình mạnh gấp đôi) → đặt tỷ lệ 2:1 |
| **Least Connections** | Đưa request vào member đang có ít kết nối mở nhất | Ứng dụng có kết nối tồn tại lâu (long-lived), tải không đều theo thời gian, ví dụ WebSocket, các API xử lý nặng |
| **Fastest** | Chọn member có thời gian phản hồi nhanh nhất gần đây | Khi độ trễ xử lý giữa các server chênh lệch lớn (khác vùng địa lý, khác cấu hình phần cứng) |
| **Observed / Predictive** | Kết hợp số connection và thời gian phản hồi (Predictive có thêm yếu tố dự đoán xu hướng) | Hệ thống quy mô lớn, tải biến động mạnh, cần cân bằng thông minh hơn Round Robin thuần |
| **Least Sessions** | Dựa trên số session đang active (khác với connection ở tầng TCP) | Ứng dụng có dùng persistence, cần cân bằng theo số session thực tế thay vì số kết nối TCP |

**Khuyến nghị chung:** với các cụm backend có cấu hình đồng nhất, Round Robin là lựa chọn phù hợp cho phần lớn dịch vụ web thông thường. Chuyển sang Least Connections khi phát hiện tải bị dồn lệch do các kết nối tồn tại lâu.

---

## 3. Persistence Profile — xác định có cần giữ session hay không

Persistence quyết định: **cùng một client có luôn được đưa trở lại đúng server đã phục vụ trước đó hay không.**

### 3.1 Các loại persistence và ý nghĩa thực tế

| Loại | Cơ chế | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **None** | Không áp dụng persistence, mỗi request được phân phối tự do theo LB Method | Cân bằng tải tối ưu nhất, cấu hình đơn giản | Sẽ mất session nếu ứng dụng lưu session cục bộ trên từng server (In-Proc) |
| **Source Address Affinity** | Dựa vào địa chỉ IP nguồn của client | Đơn giản, không cần thay đổi gì phía ứng dụng | Nếu nhiều client đi qua chung một NAT/Proxy (IP nguồn giống nhau), toàn bộ các client đó sẽ bị dồn vào một server, gây mất cân bằng tải |
| **Cookie (Insert/Rewrite)** | F5 chèn thêm một cookie định danh member vào response | Chính xác theo từng client thực (trình duyệt), không bị ảnh hưởng bởi việc nhiều client dùng chung IP qua NAT | Chỉ hoạt động với HTTP/HTTPS; không dùng được cho traffic không phải web; cần bật HTTP profile trên Virtual Server |
| **Destination Address Affinity** | Dựa theo địa chỉ IP đích (VIP) | Gần như không có tác dụng phân biệt client trong trường hợp VIP chỉ có một địa chỉ cố định | Là loại **dễ bị chọn nhầm** trong thực tế triển khai — không tạo ra stickiness thực sự khi có nhiều client cùng gọi vào một VIP. Chỉ phù hợp với một số kịch bản NAT đặc thù, không khuyến nghị dùng mặc định |
| **SSL Session ID** | Dựa vào SSL Session ID trong quá trình bắt tay TLS | Hữu ích khi dùng mô hình SSL Passthrough (F5 không giải mã được nội dung để đọc cookie) | Session ID có thể thay đổi khi client renegotiate, độ ổn định thấp hơn Cookie |
| **Universal Persistence** | Dùng iRule để tự định nghĩa khóa persistence (ví dụ theo một header riêng, theo JSESSIONID của ứng dụng) | Linh hoạt tối đa, tùy biến theo yêu cầu nghiệp vụ | Cần viết iRule, độ phức tạp triển khai cao hơn |

### 3.2 Quy trình lựa chọn

```
Ứng dụng có lưu session In-Proc trên server không?
├── KHÔNG (stateless, hoặc session được lưu tập trung ở SQL, Redis, State Server dùng chung)
│      → Chọn "None". Round Robin thuần, đạt hiệu quả cân bằng tải tối ưu.
│
└── CÓ (session mặc định của IIS lưu trong bộ nhớ từng server)
       ├── Traffic là HTTP/HTTPS thuần, không có nhiều client dùng chung NAT
       │      → Cookie persistence (khuyến nghị cho phần lớn ứng dụng web)
       │
       └── Có SSL Passthrough (F5 không giải mã) hoặc traffic không phải HTTP
              → SSL Session ID persistence hoặc Source Address
```

> Ghi chú kỹ thuật đối với IIS: khi ứng dụng dùng session In-Proc, giải pháp bền vững hơn việc chỉ dựa vào persistence trên F5 là chuyển session sang **ASP.NET Session State Server hoặc lưu tại SQL Server / Redis (out-of-process)**. Khi đó có thể loại bỏ hoàn toàn yêu cầu persistence và mở rộng cụm server một cách an toàn.

---

## 4. Health Check (Monitor) — nguyên tắc kiểm tra tình trạng backend

### 4.1 Các loại Monitor trên F5

| Loại Monitor | Tầng kiểm tra | Mô tả | Tình huống áp dụng |
|---|---|---|---|
| **ICMP (Gateway ICMP)** | Network (Layer 3) | Ping đến địa chỉ IP server | Chỉ nên dùng bổ sung, không đủ để xác định dịch vụ web có đang hoạt động hay không (server còn sống nhưng ứng dụng/site có thể đã dừng) |
| **TCP** | Transport (Layer 4) | Kiểm tra bắt tay TCP thành công tại cổng chỉ định | Backend không phải HTTP, hoặc chỉ cần xác định cổng dịch vụ có đang mở |
| **HTTP** | Application (Layer 7) | Gửi HTTP request dạng plaintext và kiểm tra response | Áp dụng khi kết nối giữa F5 và server là **plaintext HTTP** (không có TLS ở phía sau — tức chế độ SSL Offload/Termination) |
| **HTTPS** | Application (Layer 7) + TLS | Tương tự HTTP nhưng thực hiện bắt tay TLS trước khi gửi request | Áp dụng khi kết nối giữa F5 và server **yêu cầu TLS** (chế độ SSL Bridging) |
| **GET URI / HEAD URI** | Application | Biến thể của monitor HTTP/HTTPS, chỉ cần khai báo URI mà không cần soạn Send/Receive string thủ công | Dùng khi chỉ cần kiểm tra một URL trả về mã 200, không cần tùy chỉnh Host header |

### 4.2 Xử lý Health check khi backend dùng Host header binding

Trong các hệ thống có backend IIS lưu trữ nhiều site trên cùng một địa chỉ IP:Port, phân biệt bằng **Host header binding**, nếu monitor chỉ gửi `GET / HTTP/1.1` mà không kèm đúng `Host:` header khớp với cấu hình Binding trên IIS, server sẽ trả về **404** hoặc điều hướng sai site — dẫn đến monitor báo fail dù server đang hoạt động bình thường.

Cấu hình mẫu (loại monitor **HTTPS**, áp dụng khi kết nối F5–server là TLS):

```
Type: HTTPS
Interval: 5
Timeout: 16
Send String:  GET / HTTP/1.1\r\nHost: <ten-mien-dich-vu>\r\nConnection: Close\r\n\r\n
Receive String: HTTP/1.1 200
```

**Nguyên tắc bắt buộc:** mỗi domain/dịch vụ chạy trên hạ tầng IIS dùng chung địa chỉ IP cần có **một monitor riêng** (khác Send String theo Host header tương ứng) và được gắn vào **một pool riêng** của dịch vụ đó, ngay cả khi các pool này chia sẻ chung các pool member (server vật lý).

### 4.3 Công thức xác định Interval và Timeout

Quy tắc khuyến nghị: `Timeout ≥ (Interval × 3) + 1`

Ví dụ: Interval = 5 giây → Timeout tối thiểu = 16 giây.

- Interval quá ngắn (ví dụ 1–2 giây) làm tăng tải không cần thiết lên cả server và F5.
- Timeout quá dài khiến khi server thực sự gặp sự cố, F5 mất nhiều thời gian hơn để đánh dấu down, kéo dài thời gian gián đoạn dịch vụ đối với người dùng.

### 4.4 Lựa chọn endpoint cho Health check

- **Không nên** trỏ vào trang chủ (`/`) nếu trang này có nhiều truy vấn nặng (nhiều lượt gọi cơ sở dữ liệu, tải nhiều tài nguyên) — dễ gây báo lỗi sai (false negative) khi server chỉ đang xử lý chậm tạm thời.
- **Nên** xây dựng một endpoint riêng dạng `/healthcheck.aspx` hoặc `/health`, phản hồi nhanh (không truy vấn cơ sở dữ liệu, hoặc chỉ kiểm tra kết nối ở mức tối thiểu) — phản ánh đúng tình trạng "ứng dụng có hoạt động được không" mà không phụ thuộc vào tải của trang chủ.
- Đối với hệ thống nhiều tầng (Application + Database), có thể xây dựng custom EAV monitor (external script) để kiểm tra sâu hơn, bao gồm cả tình trạng kết nối đến tầng cơ sở dữ liệu.

---

## 5. SSL trên F5 — ba mô hình cần phân biệt rõ

Việc phân biệt đúng mô hình SSL quyết định trực tiếp loại Monitor và loại Persistence cần sử dụng.

### 5.1 SSL Offload / Termination

```
Client --[HTTPS]--> F5 (giải mã) --[HTTP plaintext]--> Server (cổng 80)
```

- F5 xử lý toàn bộ TLS, server nhận HTTP thuần.
- **Ưu điểm:** giảm tải xử lý mã hóa cho server backend, F5 có thể đọc được nội dung (header, cookie) để áp dụng iRule/persistence Cookie.
- **Cấu hình liên quan:** chỉ cần **Client SSL Profile**, không cần Server SSL Profile. Monitor sử dụng loại **HTTP**.

### 5.2 SSL Bridging (Re-encrypt)

```
Client --[HTTPS]--> F5 (giải mã) --[HTTPS mã hóa lại]--> Server (cổng 443)
```

- F5 vẫn đọc được nội dung ở giữa (phục vụ iRule, cookie persistence...) nhưng dữ liệu tới server backend vẫn được mã hóa, đáp ứng yêu cầu bảo mật end-to-end.
- **Cấu hình liên quan:** cần cả **Client SSL Profile** (giữa client và F5) và **Server SSL Profile** (giữa F5 và server).
- **Monitor bắt buộc sử dụng loại HTTPS**, không dùng loại HTTP, vì server yêu cầu bắt tay TLS trước khi nhận request.

### 5.3 SSL Passthrough

```
Client --[HTTPS]--> F5 (không giải mã, chỉ chuyển tiếp gói TCP) --[HTTPS]--> Server
```

- F5 không đọc được nội dung, không thể áp dụng iRule dựa trên nội dung HTTP, không sử dụng được Cookie persistence.
- Chỉ áp dụng khi có yêu cầu bảo mật đặc biệt (F5 không được lưu giữ private key của server), hoặc khi client cần xác thực chứng chỉ trực tiếp với server (mutual TLS đầu cuối).
- **Persistence:** chỉ có thể dùng SSL Session ID hoặc Source Address.
- **Monitor:** dùng loại TCP, do không đọc được nội dung tầng HTTP.

### 5.4 SNI (Server Name Indication) — khi nhiều domain dùng chung một Virtual Server

Khi nhiều domain/dịch vụ dùng chung một Virtual Server IP:443 nhưng sử dụng chứng chỉ SSL khác nhau, cần cấu hình:

- Nhiều **Client SSL Profile**, mỗi profile gắn một chứng chỉ riêng tương ứng với từng domain.
- Trên Virtual Server, gán đồng thời nhiều Client SSL Profile — F5 sẽ tự động chọn profile phù hợp dựa trên thông tin SNI mà client gửi trong bước `ClientHello` khi bắt tay TLS.
- Nếu sử dụng chứng chỉ dạng wildcard hoặc SAN cert dùng chung cho nhiều domain, không cần cấu hình SNI riêng, chỉ cần một Client SSL Profile.

Lưu ý khi triển khai SNI: cần đảm bảo toàn bộ client hỗ trợ SNI. Trình duyệt hiện đại đều hỗ trợ, tuy nhiên một số client thế hệ cũ hoặc thiết bị IoT có thể không hỗ trợ, cần đánh giá trước khi áp dụng.

---

## 6. Xác định địa chỉ IP thật của client tại backend

Đây là tình huống rất thường gặp khi triển khai backend sau F5: ứng dụng trên server (IIS) cần biết địa chỉ IP thực của client (ví dụ để ghi log, chống brute-force, phân quyền theo IP, thống kê truy cập...), nhưng nếu không cấu hình đúng, IIS chỉ nhìn thấy **địa chỉ IP của F5 (Self IP/SNAT)** thay vì IP thật của client.

### 6.1 Nguyên nhân

Khi F5 sử dụng **SNAT (Source Network Address Translation)** — cấu hình phổ biến và gần như bắt buộc trong hầu hết các Virtual Server để đảm bảo traffic trả về đi qua đúng F5 — địa chỉ IP nguồn trong gói tin gửi tới server sẽ bị thay thế bằng địa chỉ SNAT (Self IP hoặc SNAT Pool của F5), không còn là IP gốc của client.

### 6.2 Các phương án khắc phục

| Phương án | Cơ chế | Điều kiện áp dụng | Ghi chú |
|---|---|---|---|
| **X-Forwarded-For (XFF) header** | F5 tự động hoặc qua iRule chèn thêm header `X-Forwarded-For: <IP client>` vào HTTP request trước khi gửi tới server | Traffic là HTTP/HTTPS (không dùng SSL Passthrough thuần) | Phương án phổ biến và khuyến nghị nhất cho các ứng dụng web. IIS/ứng dụng cần được cấu hình để đọc header này thay vì lấy IP từ kết nối TCP (`Request.ServerVariables["REMOTE_ADDR"]` mặc định sẽ không có IP thật nếu không xử lý XFF) |
| **Bật "Insert X-Forwarded-For" trên HTTP Profile** | Trong cấu hình HTTP Profile gắn vào Virtual Server, bật tùy chọn tự động chèn XFF, không cần viết iRule | HTTP/HTTPS, khi không có nhu cầu tùy biến thêm | Cách đơn giản nhất, F5 tự động thêm mà không cần can thiệp thủ công |
| **iRule chèn XFF thủ công** | `HTTP::header insert X-Forwarded-For [IP::client_addr]` | Khi cần tùy biến thêm định dạng, hoặc chèn thêm các header khác đi kèm (ví dụ `X-Forwarded-Proto`, `X-Forwarded-Port`) | Linh hoạt hơn tùy chọn có sẵn trên HTTP Profile |
| **Disable SNAT / dùng chế độ Auto Map có kiểm soát** | Không thực hiện SNAT, giữ nguyên IP nguồn của client trong gói tin gửi tới server | Yêu cầu server backend có route trả về (return route) trỏ đúng qua F5 | Phức tạp hơn về mặt định tuyến mạng, thường yêu cầu cấu hình lại default gateway của server trỏ về F5 thay vì router mạng, cần đội hạ tầng mạng phối hợp. Không phải phương án được khuyến nghị mặc định do rủi ro sai định tuyến |
| **Proxy Protocol** | Chèn thông tin IP client ở tầng TCP (trước dữ liệu ứng dụng), không phụ thuộc vào việc đọc được header HTTP | Áp dụng được kể cả khi không phải HTTP thuần (ví dụ TCP passthrough) | Yêu cầu backend (web server) phải hỗ trợ đọc Proxy Protocol; IIS mặc định không hỗ trợ trực tiếp, cần module hoặc cấu hình bổ sung, ít phổ biến với hạ tầng Windows/IIS hơn so với XFF |

### 6.3 Cấu hình phía IIS để đọc được IP thật từ XFF

Do IIS mặc định không tự động lấy IP từ header `X-Forwarded-For`, cần bổ sung một trong các cách sau:

- Cài đặt module **Application Request Routing (ARR)** hoặc module tương đương hỗ trợ đọc XFF và ghi đè `REMOTE_ADDR`.
- Với ứng dụng .NET, xử lý trực tiếp trong code: đọc `Request.Headers["X-Forwarded-For"]` thay vì `Request.ServerVariables["REMOTE_ADDR"]` khi ghi log hoặc kiểm tra IP.
- Nếu dùng IIS Logging, cấu hình thêm trường log tùy chỉnh (Custom Field) để ghi nhận giá trị header XFF song song với trường IP mặc định.

### 6.4 Khuyến nghị triển khai

- Phương án được khuyến nghị áp dụng mặc định cho các dịch vụ HTTP/HTTPS: **bật Insert X-Forwarded-For trên HTTP Profile**, kết hợp cập nhật phía ứng dụng/IIS để đọc đúng header này.
- Trường hợp có nhiều lớp proxy/thiết bị trung gian phía trước F5 (ví dụ WAF, CDN đặt trước F5), cần lưu ý header XFF có thể chứa danh sách nhiều IP nối tiếp nhau (IP client thực tế thường là giá trị **đầu tiên** trong danh sách) — cần xử lý đúng logic parse tại tầng ứng dụng để tránh lấy nhầm IP của thiết bị trung gian.

### 6.5 Trường hợp backend là giao thức TCP thuần (LDAP, SMTP, FTP, RDP...)

Đối với các dịch vụ chạy trên giao thức TCP thuần — không phải HTTP — như **LDAP (389/636), SMTP (25), FTP (21), RDP (3389)**, khái niệm "chèn header" hoàn toàn không áp dụng được, vì các giao thức này không có cấu trúc header dạng text như HTTP. Đây là điểm khác biệt căn bản so với các dịch vụ web đã trình bày ở trên.

**Vì sao XFF không dùng được:** `X-Forwarded-For` là một HTTP header, chỉ tồn tại trong luồng dữ liệu HTTP/HTTPS. Với LDAP hay các giao thức nhị phân/TCP thuần khác, không có vị trí nào trong luồng dữ liệu để F5 chèn thêm thông tin IP mà không làm hỏng cấu trúc giao thức gốc.

Các phương án khả thi trong trường hợp này:

| Phương án | Cơ chế | Điều kiện áp dụng | Ghi chú |
|---|---|---|---|
| **Disable SNAT (khuyến nghị chính)** | Đặt SNAT = None trên Virtual Server, giữ nguyên IP nguồn của client khi chuyển gói tin tới server | Bắt buộc phải đảm bảo route trả về (return route) của server đi ngược qua F5 | Là phương án duy nhất giúp backend **thực sự nhận được** IP thật ở tầng TCP/IP. Yêu cầu cấu hình lại **default gateway của server trỏ về Self IP của F5** (thay vì trỏ về router mạng thông thường), cần phối hợp với đội hạ tầng mạng để tránh định tuyến bất đối xứng (asymmetric routing) |
| **Proxy Protocol** | Chèn thông tin IP client vào đầu luồng TCP, trước dữ liệu ứng dụng, không phụ thuộc cấu trúc giao thức phía sau | Chỉ áp dụng được nếu **server đích hỗ trợ đọc Proxy Protocol** | Các LDAP server phổ biến (Active Directory/AD DS, OpenLDAP) **không hỗ trợ đọc Proxy Protocol theo mặc định** — nếu bật mà server không hiểu, dữ liệu ở đầu luồng sẽ bị coi là rác, làm hỏng phiên kết nối. Chỉ khả thi khi có middleware/proxy trung gian hỗ trợ giao thức này |
| **Ghi log IP tại F5 bằng iRule (giải pháp thay thế khi không đổi được SNAT)** | Dùng iRule log lại IP client tại sự kiện `CLIENT_ACCEPTED`, gửi log qua High-Speed Logging (HSL) về hệ thống Syslog/SIEM tập trung | Áp dụng khi không thể thay đổi định tuyến mạng để Disable SNAT | Không giúp bản thân server (LDAP/SMTP/FTP/RDP) tự nhận biết IP client, nhưng đáp ứng được nhu cầu **giám sát, audit, tra cứu sau này** ai đã kết nối vào dịch vụ tại thời điểm nào |

Ví dụ iRule ghi log tại tầng kết nối TCP (áp dụng chung cho LDAP, SMTP, FTP, RDP):

```tcl
when CLIENT_ACCEPTED {
    log local0. "Ket noi tu [IP::client_addr]:[TCP::client_port] toi pool member [LB::server addr]:[LB::server port]"
}
```

**Nguyên tắc lựa chọn:**

- Nếu yêu cầu nghiệp vụ là **bản thân server backend phải tự nhận biết và xử lý theo IP client** (ví dụ Active Directory áp dụng policy theo IP, ghi nhận log đăng nhập kèm IP nguồn ngay tại AD) → **bắt buộc Disable SNAT**, không có phương án thay thế tương đương ở tầng ứng dụng như HTTP.
- Nếu yêu cầu chỉ là **giám sát tập trung, phục vụ tra cứu/audit** mà không cần server tự xử lý theo IP → dùng iRule log qua HSL, không cần thay đổi mô hình SNAT hiện tại, tránh rủi ro về định tuyến mạng.
- Trường hợp hạ tầng mạng không cho phép thay đổi định tuyến (ví dụ server dùng chung gateway cho nhiều Virtual Server, nhiều dịch vụ khác không qua F5), cần chấp nhận giới hạn là server chỉ thấy IP của F5, và bù lại bằng giải pháp log tập trung.

---

## 7. iRule — các tình huống sử dụng nâng cao phổ biến

iRule là script (ngôn ngữ TCL) chạy trực tiếp trên F5 để can thiệp vào luồng traffic tại các sự kiện (event) như `CLIENT_ACCEPTED`, `HTTP_REQUEST`, `HTTP_RESPONSE`... Chỉ nên sử dụng khi các cấu hình chuẩn (Pool/Persistence/Profile) không đáp ứng được yêu cầu.

### 7.1 Định tuyến theo Host Header về đúng Pool

Khi một Virtual Server dùng chung cho nhiều domain nhưng cần định tuyến linh hoạt đến các pool khác nhau theo domain, có thể sử dụng iRule thay vì tạo nhiều Virtual Server riêng biệt:

```tcl
when HTTP_REQUEST {
    switch -glob [string tolower [HTTP::host]] {
        "domain-a.example.vn*" { pool pool_domain_a_443 }
        "domain-b.example.vn*" { pool pool_domain_b_443 }
        default                { pool pool_default }
    }
}
```

> Chỉ cần thiết khi nhiều domain dùng chung một Virtual Server. Khi mỗi domain đã có Virtual Server/Pool riêng theo địa chỉ IP:Port riêng biệt, không bắt buộc sử dụng iRule này.

### 7.2 Redirect HTTP sang HTTPS

```tcl
when HTTP_REQUEST {
    HTTP::redirect "https://[HTTP::host][HTTP::uri]"
}
```

### 7.3 Universal Persistence theo giá trị tùy chỉnh

Áp dụng khi cần persistence theo một giá trị riêng của ứng dụng (ví dụ JSESSIONID) thay vì dùng cookie do F5 tự chèn thêm:

```tcl
when HTTP_REQUEST {
    persist uie [HTTP::cookie value "JSESSIONID"]
}
```

### 7.4 Kiểm soát truy cập theo điều kiện

```tcl
when HTTP_REQUEST {
    if { [HTTP::uri] starts_with "/admin" } {
        if { not ([IP::addr [IP::client_addr] equals 10.0.0.0/8]) } {
            HTTP::respond 403 content "Forbidden"
        }
    }
}
```

### 7.5 Chèn thêm header nhận diện cho backend

```tcl
when HTTP_REQUEST {
    HTTP::header insert X-Forwarded-Proto "https"
    HTTP::header insert X-Forwarded-For [IP::client_addr]
}
```

> Khi áp dụng SSL Offload, cần cấu hình bổ sung phía ứng dụng/IIS để đọc header `X-Forwarded-Proto`, tránh trường hợp ứng dụng tự động redirect sai (gây vòng lặp redirect giữa HTTP và HTTPS).

---

## 8. Bảng tổng hợp khuyến nghị áp dụng chung

| Hạng mục | Khuyến nghị |
|---|---|
| LB Method | Round Robin cho backend đồng nhất cấu hình; chuyển sang Least Connections nếu phát hiện lệch tải do kết nối dài |
| Persistence | None nếu session lưu ngoài (SQL/Redis/State Server); Cookie nếu ứng dụng dùng session In-Proc |
| Monitor Type | HTTP khi SSL Offload; **HTTPS** khi SSL Bridging; TCP khi SSL Passthrough hoặc backend không phải HTTP |
| Send String | Cần khai báo đúng Host header riêng cho từng domain/dịch vụ khi backend dùng Host header binding |
| Interval/Timeout | Tuân theo công thức Timeout ≥ (Interval × 3) + 1 |
| Pool | Tách riêng theo từng domain/dịch vụ, có thể chia sẻ chung pool member (server vật lý) |
| SSL Profile | SSL Bridging cần cả Client SSL Profile và Server SSL Profile; nhiều domain chung VIP cần cấu hình SNI hoặc dùng SAN/wildcard cert |
| Client IP thật (backend HTTP/HTTPS) | Bật Insert X-Forwarded-For trên HTTP Profile, cấu hình bổ sung phía IIS/ứng dụng để đọc đúng header |
| Client IP thật (backend TCP thuần: LDAP/SMTP/FTP/RDP) | Disable SNAT kèm cấu hình route trả về qua F5 nếu server cần tự nhận biết IP; nếu chỉ cần audit thì dùng iRule log qua HSL, không cần đổi SNAT |
| iRule | Sử dụng khi cần định tuyến theo domain trên chung một VIP, redirect, custom persistence, chèn thêm header cho backend, hoặc ghi log kết nối cho các dịch vụ TCP thuần |

---

## 9. Checklist trước khi triển khai

- [ ] Xác định đúng mô hình SSL đang áp dụng (Offload / Bridging / Passthrough) trước khi chọn loại Monitor
- [ ] Mỗi domain/dịch vụ có Send String với Host header riêng, khớp đúng Binding trên IIS
- [ ] Health check endpoint phản hồi nhanh, không phụ thuộc vào các truy vấn nặng
- [ ] Xác nhận IIS đã bind đúng chứng chỉ và tên site khớp với Host header mà monitor gửi lên
- [ ] Xác định ứng dụng có lưu session In-Proc hay không để quyết định loại persistence phù hợp
- [ ] Nếu nhiều domain dùng chung một VIP: đã cấu hình SNI hoặc chứng chỉ SAN/wildcard phù hợp
- [ ] Đã cấu hình chèn X-Forwarded-For (hoặc phương án tương đương) nếu backend HTTP/HTTPS cần biết IP thật của client
- [ ] Đã kiểm tra/điều chỉnh phía IIS hoặc ứng dụng để đọc đúng header X-Forwarded-For thay vì dùng IP mặc định từ kết nối TCP
- [ ] Với backend TCP thuần (LDAP/SMTP/FTP/RDP): đã xác định rõ yêu cầu là server tự nhận biết IP (cần Disable SNAT + kiểm tra route trả về) hay chỉ cần audit (dùng iRule log qua HSL)
- [ ] Đã kiểm thử thực tế: tắt một server trong pool để xác nhận F5 phát hiện down đúng theo thời gian Interval/Timeout đã cấu hình
