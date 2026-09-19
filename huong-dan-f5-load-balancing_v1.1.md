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