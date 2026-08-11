# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Kiều Thế Hiệp  Mã học viên: 19205

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để "changeme", ứng dụng sẽ khởi động bình thường mà không báo lỗi gì. Nhưng khi deploy lên public cloud, ai cũng có thể dùng key "changeme" để cào API, xài sạch tiền ngân sách LLM của mình. "Chết sớm" giúp server từ chối khởi động ngay lập tức để mình nhận ra đang quên cấu hình biến môi trường quan trọng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"timestamp": "2026-08-11T12:00:00Z", "level": "info", "message": "ask_completed", "user_id": "sv-test", "tokens_in": 10, "tokens_out": 20, "cost_usd": 0.0015}`. Hai việc làm được:
> 1. Đẩy vào các tool như Datadog/Elasticsearch để truy vấn bằng câu lệnh (SQL-like) hoặc filter những request có `cost_usd > 0.001`.
> 2. Vẽ biểu đồ Dashboard tự động thống kê tổng chi phí tiêu thụ theo `user_id` qua từng ngày nhờ dữ liệu được bóc tách sẵn.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch dung lượng chính là do các công cụ build (gcc, make...), bộ nhớ đệm pip (pip cache), các file mã nguồn phụ. Multi-stage giúp giữ lại tất cả đống rác này ở stage builder, và chỉ copy bộ thư viện đã build gọn gàng sang stage production cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Lệnh `COPY requirements.txt .` và `RUN pip install` sẽ được dùng lại cache. Lệnh `COPY app/ app/` trở về sau sẽ chạy lại. Nếu đảo `COPY . .` lên trước, mỗi lần sửa 1 dấu phẩy trong main.py, Docker sẽ coi layer COPY đó đã thay đổi, khiến toàn bộ các layer bên dưới (bao gồm `RUN pip install`) mất cache và phải download thư viện lại từ đầu (rất chậm).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Hacker gửi payload qua /ask khai thác lỗi Remote Code Execution trong app Python -> Lệnh thực thi được chạy với quyền người dùng hiện tại (là root) bên trong container -> Hacker có quyền root container, từ đó lợi dụng volume mount hoặc lỗ hổng kernel để chiếm quyền máy chủ host. Lệnh `USER` giáng cấp process xuống thành một người dùng hạn chế quyền, không thể đục thủng container kể cả khi đã RCE thành công.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> 20 request. Người dùng gọi 10 request ở 10:00:59, lúc này vẫn đang trong phút số 0. Ngay giây tiếp theo (10:01:00), phút mới bắt đầu nên bộ đếm được reset về 0, người dùng gọi tiếp 10 request nữa. Tổng cộng họ đã gửi 20 request chỉ trong 2 giây, vượt xa mong muốn ban đầu của ta. Cửa sổ trượt ngăn được điều này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit chặn dồn dập (số req/phút). Cost guard chặn lạm chi (USD/tháng).
> 1. Rate cho qua, Cost chặn: User hỏi đúng 1 câu mỗi phút nhưng đính kèm file 500 trang tốn siêu nhiều token. Request thứ 2 sẽ bị chặn vì hết budget tháng dù không gọi nhanh.
> 2. Cost cho qua, Rate chặn: User viết tool spam 30 request "hi" chỉ trong 5 giây. Số tiền hao hụt rất nhỏ không bị Cost chặn, nhưng Rate sẽ lập tức trả lỗi 429 vì tốc độ gọi vượt ngưỡng 10req/phút.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis chết, endpoint gộp báo 503 (Lỗi kết nối).
> 2. Orchestrator (Loadbalancer/Docker) ping health check thấy báo lỗi, tưởng process Uvicorn bị treo.
> 3. Orchestrator lập tức SIGKILL (giết) và restart lại toàn bộ 3 container liên tục (CrashLoop) dù thực chất Uvicorn vẫn hoạt động bình thường, làm sập hoàn toàn hệ thống. Tách riêng giúp Orchestrator chỉ tạm thời ngưng đẩy traffic chứ không giết app.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Con số sẽ nhảy lung tung (1, 2, 1, 1, 3...). Do Nginx load balancer điều phối request round-robin ngẫu nhiên vào 1 trong 3 container. Nếu dùng dict RAM, mỗi container có bộ nhớ riêng. Nếu request rơi vào container mới, app không thấy lịch sử trước đó -> coi như bắt đầu lại từ đầu (length = 1). Dùng chung Redis thì cả 3 container đều đọc được chung lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi "curl: (7) Failed to connect to localhost port 8000: Connection refused". Nguyên nhân là trong uvicorn tôi để `--host 127.0.0.1`, khiến nó chỉ nhận traffic từ chính nó. Traffic từ bên ngoài container (hoặc LB) sẽ bị từ chối. Cách khắc phục: chuyển thành `--host 0.0.0.0` trong CMD của Dockerfile.
