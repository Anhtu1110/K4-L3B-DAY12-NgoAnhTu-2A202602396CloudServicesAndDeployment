# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Anh Tú  Mã học viên: 2A202602396

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử tôi deploy lên Railway và quên set biến `AGENT_API_KEY`. Nếu để mặc định `"changeme"`, app vẫn khởi động bình thường — load balancer thấy `/health` trả 200 nên coi như ổn. Mọi người có thể gọi API bằng key `"changeme"` và tốn token của tôi mà tôi không hay biết cho đến khi nhìn hóa đơn cuối tháng. Ngược lại, không có mặc định thì app ném `ValidationError` ngay lúc khởi động, Railway báo deploy failed, tôi biết ngay và fix trước khi bất kỳ request nào được xử lý. Lỗi xuất hiện lúc deploy khi tôi đang nhìn màn hình, không phải lúc cuối tháng khi tiền đã bay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON khi gọi `/ask`:
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T08:15:30.123456+00:00", "user_id": "sv01", "tokens_in": 120, "tokens_out": 85, "cost_usd": 0.000041}
> ```
>
> Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và thống kê theo field**: Tôi có thể query "user nào tiêu nhiều tiền nhất hôm nay?" bằng cách lọc theo `user_id` và tính tổng `cost_usd`. Với `print("đã trả lời xong")` không có field nào để lọc cả.
> 2. **Cảnh báo tự động**: Có thể cấu hình alert khi `cost_usd` của một user vượt ngưỡng, hoặc đếm tỷ lệ lỗi 5 phút qua bằng cách đếm các dòng có `level: error`. `print()` không có cấu trúc nên không parse được.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | ~210 MB |

> Phần dung lượng chênh lệch (~840 MB) là các công cụ build không cần thiết ở runtime: trình biên dịch C (gcc, build-essential) được pip dùng để build một số package native, toàn bộ header file của Python, file `.a` static library, và các tool dev khác. Multi-stage build loại bỏ chúng hoàn toàn — stage `builder` dùng xong rồi bị vứt đi, chỉ thư mục `/install` chứa kết quả được copy sang stage `runtime`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile multi-stage hiện tại, khi sửa `app/main.py`:
> - **Dùng lại cache**: `FROM python:3.11-slim`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install` — vì `requirements.txt` không đổi nên toàn bộ layer cài thư viện được cache lại.
> - **Phải chạy lại**: `COPY app ./app` và các bước sau — vì Docker phát hiện nội dung thư mục `app/` thay đổi.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: mỗi lần sửa bất kỳ file nào (kể cả 1 dấu phẩy trong `main.py`), Docker hủy cache từ layer `COPY . .` trở đi, buộc phải chạy lại `pip install` toàn bộ — mất 2-5 phút mỗi lần build thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
> 1. Kẻ tấn công tìm thấy lỗ hổng RCE (Remote Code Execution) trong endpoint `/ask` — ví dụ injection qua tham số `question`.
> 2. Họ thực thi lệnh shell bên trong container. Vì container chạy bằng root (uid 0), họ có toàn quyền bên trong container.
> 3. Nếu Docker daemon đang chạy với quyền root trên host (mặc định), và có volume mount hoặc Docker socket `/var/run/docker.sock` bị expose, kẻ tấn công leo thang lên host với quyền root.
> 4. Từ đó họ có thể đọc file `/etc/shadow`, cài malware, hay kiểm soát toàn bộ máy.
>
> Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: dù RCE thành công, kẻ tấn công chỉ có quyền của user `appuser` (uid 10001) — không thể ghi vào `/usr/local`, không thể chạy lệnh cần root, và khó leo thang lên host hơn nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với đếm theo phút đồng hồ, người dùng có thể gửi **20 request** trong 2 giây mà vẫn "đúng luật":
> - Gửi 10 request từ 10:00:58 đến 10:00:59 (phút 10:00 — dùng hết quota).
> - Đến 10:01:00 bộ đếm reset về 0.
> - Gửi thêm 10 request từ 10:01:00 đến 10:01:01 (phút 10:01 — quota mới).
>
> Tổng: 20 request trong khoảng 3 giây, mà hệ thống không phát hiện được. Sliding window không có lỗ hổng này vì nó luôn nhìn 60 giây **gần nhất** tính từ request hiện tại — không có ranh giới cứng để khai thác.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát **tốc độ** (số request/phút), còn cost guard kiểm soát **chi phí** (tổng tiền/tháng). Chúng bổ trợ cho nhau.
>
> **Rate limit cho qua, cost guard chặn**: User `sv01` gửi đúng 10 request/phút (trong hạn mức). Nhưng mỗi request hỏi một đoạn văn 50.000 token, dùng model đắt nhất. Sau 5 request đã tiêu hết \$10 ngân sách tháng — cost guard chặn, rate limit không thể biết vì nó chỉ đếm số lượng.
>
> **Cost guard cho qua, rate limit chặn**: User `sv02` đầu tháng, chưa tiêu đồng nào, ngân sách còn nguyên. Nhưng họ gửi 50 request trong 60 giây (câu hỏi ngắn, rẻ) — rate limit chặn ở request thứ 11, cost guard không can thiệp vì chi phí chưa vượt budget.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Chuỗi sự kiện nếu gộp `/health` và `/ready` thành một và cho kiểm tra Redis:
> 1. Redis mất kết nối (ví dụ restart để vá lỗi).
> 2. Ba container gọi endpoint duy nhất → cả 3 đều trả 503.
> 3. Orchestrator (Kubernetes/Docker Swarm) thấy liveness probe 503 → đánh giá "container bệnh, cần restart".
> 4. Orchestrator **restart cả 3 container cùng lúc** — đây là hành động đúng với liveness probe.
> 5. Trong khi 3 container đang khởi động lại, Redis đã quay lại bình thường.
> 6. Nhưng không còn container nào đang chạy để nhận request → hệ thống down hoàn toàn.
>
> Đúng ra: `/health` (liveness) không được kiểm tra Redis — Redis chết thì chỉ `/ready` (readiness) trả 503, load balancer ngừng đẩy traffic vào nhưng **không restart** container. Khi Redis phục hồi, `/ready` xanh trở lại và traffic tự động vào lại — không down một giây nào.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless): `history_length` tăng đều 0, 1, 2, 3, 4... dù mỗi request vào container khác nhau. Mọi container đều đọc từ cùng một Redis key `history:sv01`.
>
> Nếu dùng dict Python trong RAM: `history_length` nhảy loạn — ví dụ 0, 1, 0, 1, 2, 0... Vì mỗi container có dict riêng. Request 1 vào Container A → dict của A có 1 item. Request 2 vào Container B → dict của B vẫn rỗng, trả về 0. Request 3 lại vào A → trả về 1 (chỉ nhớ câu hỏi 1, quên câu 2 đã vào B). Agent "mất trí nhớ" ngẫu nhiên tùy theo load balancer quyết định route vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải**: Health check timeout sau khi deploy lên Railway — dashboard báo "Service unhealthy", log hiện `Connection refused`.
>
> **Thông báo lỗi**: `curl: (7) Failed to connect to 0.0.0.0 port 8000: Connection refused` trong health check log.
>
> **Nguyên nhân**: CMD trong Dockerfile ban đầu cố định cổng 8000 (`--port 8000`) thay vì đọc biến `$PORT`. Railway tự gán cổng khác (ví dụ 3000) qua biến môi trường `PORT`, nhưng uvicorn lắng nghe ở 8000 — không ai gọi vào được.
>
> **Cách sửa**: Đổi CMD thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Shell form (`sh -c`) cần thiết để biến `${PORT:-8000}` được expand, exec form JSON array không expand shell variable. Sau khi push commit này, Railway rebuild và health check pass.
