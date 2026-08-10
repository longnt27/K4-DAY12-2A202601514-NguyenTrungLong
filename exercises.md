# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trung Long  Mã học viên: 2A202601514

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu quên set token trên cloud, app dừng ngay lúc deploy thay vì chạy với
> `changeme` và để người lạ gọi API bằng mật khẩu đoán được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T08:20:00+00:00","client_id":"sv-test","usd_cost":0.00003}`.
> Tôi có thể lọc log theo client và cộng chi phí theo ngày; một câu `print`
> chung chung không có đủ dữ liệu để làm hai việc đó.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu là base Python đầy đủ và các layer build không cần
> ở runtime. Multi-stage chỉ mang dependency đã cài sang image slim cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer cài dependency vẫn lấy từ cache; `COPY app`
> và các layer sau nó chạy lại. Nếu `COPY . .` đứng trước `pip install`, mỗi
> lần sửa code Docker phải cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app bị khai thác, tiến trình root trong container có nhiều quyền hơn để
> đụng tới mount hoặc lợi dụng lỗi runtime nhằm thoát ra host. `USER appuser`
> giới hạn tiến trình bị chiếm quyền ngay từ trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` cho client biết cơ chế đăng nhập cần dùng. Cùng
> một thông báo lỗi giúp tránh tiết lộ token đúng một phần hay scheme nào đã
> được chấp nhận cho người đang dò.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Nó gửi được 10 request rồi nhận 429. Nếu bỏ `min`, sau 10 phút xô có thể có
> tới 110 token (10 ban đầu và 100 token nạp thêm), nên giới hạn burst mất tác dụng.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức tháng có thể mất đủ $30 trong một sự cố và chỉ tự mở lại tháng sau.
> Hạn mức ngày giới hạn thiệt hại ở $1 và service tự dùng lại được vào ngày
> UTC kế tiếp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm cả ba container báo unhealthy; orchestrator restart
> cả ba gần như cùng lúc. Khi Redis trở lại thì không còn instance sẵn sàng,
> nên một lỗi dependency ngắn biến thành outage của cả service.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi dùng nhầm `day12-chat.onrender.com`, nên request có token luôn trả 401.
> Tôi đối chiếu URL thật trong Render, thấy hostname có hậu tố `-4186`, rồi
> sửa `PUBLIC_URL` và `DEPLOYMENT.md`; chạy lại CP5 thì 9 test đều pass.
