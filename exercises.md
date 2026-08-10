# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Trọng Việt Dũng  Mã học viên: 2A202601746

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định `"changeme"`, ứng dụng vẫn sẽ khởi động bình thường trên production. Người quản trị sẽ không phát hiện ra là chưa cấu hình secret thật, và bất kỳ ai có token mặc định `"changeme"` đều có thể truy cập hệ thống trái phép, dẫn tới rò rỉ dữ liệu hoặc bùng nổ chi phí LLM. Việc "chết sớm" (fail-fast) buộc hệ thống dừng ngay từ khâu khởi động, ngăn chặn việc đưa một service chưa được bảo vệ lên môi trường production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON mẫu:
> `{"ts": "2026-08-10T15:30:00.123456Z", "severity": "INFO", "event": "chat_completed", "client_id": "user-demo", "usd_cost": 0.0000375, "turns_before": 2}`
> 
> Hai việc log JSON làm được mà print() thông thường không làm được:
> 1. **Phân tích và truy vấn tự động (Structured Querying):** Các hệ thống thu thập log (như Datadog, Kibana, Grafana Loki) có thể tự động parse các trường key-value (`client_id`, `usd_cost`, `severity`) để thống kê tổng chi phí theo từng client hoặc cảnh báo tự động khi chi phí tăng bất thường.
> 2. **Dễ dàng lọc ngữ cảnh (Traceability & Context):** Nhờ có cấu trúc JSON kèm ISO timestamp và client_id, ta có thể nhanh chóng lọc ra toàn bộ hành trình request của một user cụ thể mà không phải viết regex phức tạp để parse chuỗi văn bản tự do.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 950 MB |
| Multi-stage | 180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~770 MB) bao gồm toàn bộ các trình biên dịch (gcc, g++), bộ công cụ build C-extensions, header files, wheel build tools và cache của `pip` nằm ở stage `builder` đã được loại bỏ hoàn toàn, chỉ giữ lại các thư viện runtime Python tinh gọn ở stage `production`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (đặt `COPY requirements.txt .` và `RUN pip install` trước `COPY app/ app/`), khi sửa `app/main.py`, Docker engine sẽ dùng lại (cache hit) toàn bộ các layer từ base image cho đến layer `pip install`. Chỉ từ layer `COPY app/ app/` trở đi mới bị invalidate và phải chạy lại, giúp build cực nhanh (< 2s). Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi trong code sẽ làm vô hiệu hóa cache của lệnh `COPY . .`, buộc Docker phải chạy lại lệnh `pip install` tốn hàng phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python có lỗ hổng RCE hoặc Path Traversal. Kẻ tấn công gửi payload độc hại để thực thi lệnh shell. Nếu container chạy dưới quyền root, tiến trình shell có đặc quyền root trong container, cho phép kẻ tấn công thực hiện container escape (leo thang chiếm quyền root của máy host thông qua Docker socket, mount point hoặc lỗ hổng kernel). Lệnh `USER appuser` chuyển tiến trình sang user thường (non-root), cắt đứt chuỗi tấn công ngay tại container vì tiến trình không còn quyền root để thao tác với tài nguyên hệ thống nhạy cảm.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Theo chuẩn RFC 6750 (OAuth 2.0 Authorization Framework), khi trả về HTTP 401 Unauthorized bắt buộc phải bao gồm header `WWW-Authenticate: Bearer` để thông báo cho HTTP Client/Browser biết phương thức xác thực chuẩn mà API yêu cầu. Việc trả về cùng một thông báo lỗi nhằm ngăn chặn kỹ thuật thám mã (Enumeration Attack) của kẻ tấn công; nếu trả về thông báo chi tiết, kẻ tấn công có thể lợi dụng để đoán định format token hoặc xác định xem token nào tồn tại trên hệ thống.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10`, client im lặng 10 phút thì xô nạp đầy tối đa 10 token. Nó chỉ gửi được đúng **10 request liên tiếp** trước khi bị HTTP 429. Nếu bỏ `min(capacity, ...)`, số token sẽ tích lũy thành $10 + 10 \times 10 = 110$ token. Khi đó client có thể gửi một đợt bùng nổ (burst) 110 request liên tiếp, làm mất tác dụng bảo vệ của thuật toán Token Bucket.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng: khi gặp sự cố từ 2h sáng, client có thể tiêu hết toàn bộ $30 chỉ trong vài phút. Thiệt hại tối đa là **$30**, và service bị khóa đối với client đó đến tận đầu tháng sau (chờ 30 ngày). Với hạn mức $1/ngày: client chỉ có thể tiêu tối đa **$1** trước khi bị chặn bởi 402 Payment Required. Thiệt hại tối đa là **$1**, và service tự động hồi phục vào 00:00 UTC ngày hôm sau (chờ tối đa 24h).

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Khi Redis mất kết nối 30s, endpoint gộp lập tức trả lỗi HTTP 503.
> 2. Orchestrator (Kubernetes/Docker) tưởng rằng tiến trình Python đã chết (Liveness Failure).
> 3. Orchestrator lập tức kill và restart lại cả 3 container `chat`.
> 4. Việc khởi động lại container liên tục trong khi Redis vẫn ngắt kết nối gây ra vòng xoáy Restart Loop (Cascading Failure), làm sập toàn bộ hệ thống và tiêu tốn vô ích CPU/RAM.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi: `/readyz trả về 500 Internal Server Error` khi deploy lên Render. Nguyên nhân: Kết nối Redis trên Cloud sử dụng URL SSL (`rediss://`), khiến thư viện `redis-py` trong Python yêu cầu thêm tùy chọn SSL `ssl_cert_reqs=None`. Ngoài ra, khởi tạo client thiếu khối try-except dẫn tới unhandled exception trong FastAPI dependency. Cách sửa: Thêm hỗ trợ `rediss://` với `ssl_cert_reqs=None` và bao bọc hàm khởi tạo `get_redis_client` bằng try-except để trả về HTTP 503 thay vì 500.

