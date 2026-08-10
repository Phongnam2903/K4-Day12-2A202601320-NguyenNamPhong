# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Nam Phong  Mã học viên: 2A202601320

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử bạn deploy ứng dụng lên public cloud mà quên cấu hình biến môi trường API_TOKEN. Nếu có giá trị mặc định "changeme", app vẫn khởi động bình thường. Kẻ xấu có thể dò ra hoặc biết giá trị này, dùng nó để gọi API tốn phí của bạn một cách miễn phí. Việc "chết sớm" (báo lỗi ngay khi start) buộc bạn phải khai báo token hợp lệ ngay từ đầu, tránh mất tiền oan.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:00:00+00:00", "client_id": "sv01", "prompt_tokens": 10, "completion_tokens": 20, "usd_cost": 0.0001}`
Hai việc làm được: 1. Truy vấn và lọc log theo `client_id` để biết khách hàng nào tiêu tốn nhiều `usd_cost` nhất trong ngày. 2. Có thể set cảnh báo tự động trên hệ thống monitor (như Datadog, GCP Logging) khi `usd_cost` hoặc tỷ lệ lỗi tăng vọt.

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
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | ~150-300 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch chủ yếu là bộ công cụ biên dịch (compiler, header files) và môi trường base cồng kềnh của image `python:3.11`. Trong multi-stage, các công cụ này chỉ dùng ở stage `builder` để cài dependencies, sau đó ta chỉ copy kết quả sang stage `runtime` (dùng `python:3.11-slim`), nên image cuối cùng rất nhỏ gọn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, layer `COPY requirements.txt .` và `RUN pip install` được dùng lại (cached) vì dependencies không đổi; Docker chỉ chạy lại từ lệnh `COPY app ./app`. Nếu đặt `COPY . .` lên trước `pip install`, bất kỳ thay đổi nào trong source code (như 1 ký tự trong main.py) cũng làm invalid cache của toàn bộ thư mục, khiến Docker phải tải và cài lại toàn bộ thư viện Python từ đầu, cực kỳ tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: (1) Mã Python có lỗi bảo mật (như RCE, deserialize lỗi). (2) Hacker khai thác lỗi để thực thi lệnh shell trong container. (3) Nếu app chạy bằng root, hacker có quyền root bên trong container, từ đó có thể khai thác tiếp để thoát ra ngoài (container breakout) và lấy quyền root trên host.
Lệnh `USER appuser` cắt đứt chuỗi này ở bước 3: hacker chỉ có quyền của user thường, không thể cài mã độc, thay đổi file hệ thống hay dễ dàng leo thang đặc quyền.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

Trả kèm `WWW-Authenticate: Bearer` là quy định bắt buộc của chuẩn HTTP (RFC 6750) để báo cho client biết server đang yêu cầu kiểu xác thực nào.
Việc gom chung một thông báo lỗi giúp ngăn ngừa rò rỉ thông tin. Nếu ta nói "sai scheme" hay "sai token", kẻ tấn công (đang dò quét) sẽ lợi dụng thông tin này để thu hẹp phạm vi dò tìm.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Nó gửi được tối đa 10 request trước khi bị 429 vì xô chứa được tối đa 10 token.
Nếu bỏ `min(capacity, ...)`, số token sẽ không bị giới hạn trần. Sau 10 phút (600 giây), xô sẽ tích được 100 token, lúc đó client có thể xả cùng lúc 100 request. Hệ thống sẽ mất khả năng chống chịu bạo phát lưu lượng (burst traffic).

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Với $30/tháng: Thiệt hại tối đa là $30 ngay trong hôm đó. Service của client đó sập phần còn lại của cả tháng (29 ngày tiếp theo) cho đến khi nạp thêm tiền hoặc sang tháng mới.
Với $1/ngày: Thiệt hại tối đa chỉ là $1 cho riêng hôm bị sự cố. Service tự động khôi phục vào ngay nửa đêm khi sang ngày mới, bảo vệ được $29 ngân sách còn lại.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện: (1) Redis mất kết nối. (2) Orchestrator (K8s/Docker) gọi /healthz (gộp) và nhận lỗi 503 vì không ping được Redis. (3) Orchestrator cho rằng toàn bộ 3 process đã chết nên tự động kill và khởi động lại cả 3 container. (4) Khi Redis hồi phục sau 30s, các container vẫn đang trong quá trình restart nên hệ thống mất dịch vụ hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi: App khởi động được nhưng Health check timeout và platform báo fail.
Nguyên nhân: Trong Dockerfile hoặc lệnh chạy app, port bị hardcode là 8000 và host bị bind vào `127.0.0.1` thay vì `0.0.0.0` và cổng động.
Cách sửa: Đổi cờ khởi động Uvicorn thành `--host 0.0.0.0` và dùng port lấy từ biến môi trường `$PORT` (ví dụ: `--port ${PORT:-8000}`).
