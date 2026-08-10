# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Thọ An
>
> Mã học viên: 2A202601883

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy ứng dụng lên production trên cloud, nếu kỹ sư quên thêm biến môi trường `AGENT_API_KEY`:  
> - Nếu để mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường nhưng khi có kẻ tấn công hoặc bot quét API với key `"changeme"`, họ sẽ gọi được vào LLM API làm tiêu hao ngân sách và lộ dữ liệu mà ta không hề hay biết.
> - Nhờ cơ chế Fail-fast (không có giá trị mặc định), ứng dụng crash ngay lập tức ở bước khởi động / health check fail, giúp đội ngũ nhận ra thiếu sót và bổ sung cấu hình ngay lập tức trước khi traffic người dùng đổ vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:48:37.000000+00:00", "user_id": "sv-test", "cost_usd": 0.0001, "tokens_in": 15, "tokens_out": 45}
> ```
> Hai việc làm được với dòng log JSON mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và tính toán số liệu định lượng tự động:** Các hệ thống giám sát (Datadog, CloudWatch, Loki) có thể parse JSON để tính tổng chi phí `cost_usd` theo từng `user_id` hoặc vẽ biểu đồ số token tiêu thụ theo thời gian thực.
> 2. **Tự động cảnh báo (Alerting):** Dễ dàng thiết lập rule cảnh báo tự động gửi về Slack/Telegram khi phát hiện log có `level == "error"` hoặc khi chi phí trong 5 phút vượt quá ngưỡng cho phép.

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
| 1 stage (bản đầu: `python:3.11` full) | ~1020 MB |
| Multi-stage (`python:3.11-slim` + 2 stages) | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) bao gồm: các công cụ build và biên dịch cấp hệ điều hành (trình biên dịch gcc/g++, make, git, header C/C++), tài liệu hướng dẫn man pages, và bộ nhớ cache tải về của pip. Multi-stage build kết hợp base image `python:3.11-slim` chỉ copy duy nhất thư mục package đã cài đặt sang image runtime cuối cùng, loại bỏ hoàn toàn các file thừa giúp image nhẹ hơn và deploy nhanh hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:** Các layer từ đầu đến trước lệnh `COPY app ./app` (bao gồm `FROM`, `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder`, `RUN useradd`) đều được dùng lại từ cache (`CACHED`) vì file `requirements.txt` không thay đổi. Chỉ có layer `COPY app ./app` và các chỉ thị phía sau phải chạy lại. Thời gian build chỉ mất 1–2 giây.
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`:** Mỗi khi sửa bất kỳ ký tự nào trong mã nguồn, layer `COPY . .` sẽ làm mất hiệu lực cache của tất cả các layer phía sau, buộc Docker phải tải và cài đặt lại toàn bộ thư viện Python từ đầu ở mỗi lần build, gây lãng phí thời gian và băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - **Chuỗi sự kiện:** Khi container chạy bằng root (UID 0), nếu ứng dụng Python có lỗ hổng thực thi mã từ xa (RCE) hoặc Path Traversal, mã độc của kẻ tấn công sẽ thực thi với quyền root bên trong container. Kết hợp với các lỗ hổng container breakout (hoặc mount thư mục nhạy cảm như Docker socket/host filesystem), kẻ tấn công sẽ leo thang chiếm quyền root tối cao trên toàn bộ máy host.
> - **Lệnh `USER appuser` cắt đứt chuỗi:** Chuyển tiến trình sang chạy với user không có đặc quyền (UID 10001). Khi đó, dù kẻ tấn công chiếm được tiến trình Python thì tiến trình này chỉ có quyền hạn người dùng thông thường, không thể sửa file hệ thống và không có quyền root trên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp (gấp đôi hạn mức 10/phút).
> - **Cách đạt được:** Người dùng gửi 10 request ở giây `10:00:59` (cuối phút 1). Đúng 1 giây sau (`10:01:00`), bộ đếm phút reset về 0 và người dùng gửi tiếp 10 request nữa. Như vậy trong 2 giây từ `10:00:59` đến `10:01:01`, người dùng gửi được 20 request mà không bị hệ thống fixed window chặn. Thuật toán Sliding Window (cửa sổ trượt 60 giây liên tục) loại bỏ hoàn toàn kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Điểm khác nhau:** Rate limit kiểm soát **tần suất/số lượng request** trong thời gian ngắn (ví dụ: 10 req/phút) để chống nghẽn hệ thống (DoS); Cost guard kiểm soát **tổng số tiền/ngân sách** (USD/tháng) dựa trên số lượng token LLM thực tế tiêu thụ.
> - **Rate limit cho qua nhưng Cost guard chặn:** Người dùng cả tháng chỉ gửi 1 request (tần suất rất thấp, rate limit cho qua), nhưng câu hỏi kèm văn bản 100.000 tokens làm chi phí vượt quá ngân sách 10 USD/tháng $\rightarrow$ Cost guard chặn với mã lỗi 402 Payment Required.
> - **Cost guard cho qua nhưng Rate limit chặn:** Người dùng mới còn nguyên ngân sách 10 USD nhưng gửi liên tiếp 15 request ngắn ("hi", "test") trong 5 giây $\rightarrow$ Chi phí token rất nhỏ (Cost guard cho qua), nhưng tần suất quá nhanh nên Rate limit chặn ở request thứ 11 với mã lỗi 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Thứ tự sự kiện:**
> 1. Redis mất kết nối 30 giây.
> 2. Cả 3 container đều kiểm tra Redis trong endpoint `/health` và đồng loạt trả về mã 503 / lỗi.
> 3. Hệ thống điều phối (Orchestrator như Docker / Kubernetes / Cloud) thấy liveness probe thất bại nên kết luận 3 container này bị hỏng và gửi lệnh `SIGKILL` restart cả 3 container cùng lúc.
> 4. Trong lúc 3 container đang restart, hệ thống hoàn toàn không có instance nào phục vụ, người dùng nhận lỗi 502 Bad Gateway / Connection Refused.
> 5. Khi các container khởi động lại mà Redis vẫn chưa xong (trong 30s), chúng lại tiếp tục fail liveness probe và bị restart lặp đi lặp lại (crash loop).
> 6. Sự cố mất kết nối tạm thời của Redis biến thành thảm họa sập toàn bộ hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Khi lưu trên Redis (Stateless):** Con số `history_length` tăng đều đặn và tuyến tính qua từng lượt hỏi (`0 -> 2 -> 4 -> 6 -> 8...`) dù các request liên tiếp rơi ngẫu nhiên vào các container khác nhau nhờ cơ chế chia sẻ state tập trung trên Redis.
> - **Nếu lưu bằng dict Python trong RAM của từng container (Stateful):** Con số `history_length` sẽ nhảy lung tung không theo thứ tự (ví dụ: `0 -> 0 -> 0 -> 2 -> 2 -> 2 -> 4...`) vì mỗi container chỉ lưu các tin nhắn được gửi trúng vào nó và không thấy dữ liệu của các container khác. Hậu quả là người dùng sẽ thấy Agent bị "mất trí nhớ" ngẫu nhiên giữa các câu hỏi liên tiếp.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Thông báo lỗi:** `Error: Invalid value for '--port': '$PORT' is not a valid integer.` khi Railway khởi động uvicorn.
> - **Nguyên nhân:** Lệnh `startCommand = "uvicorn ... --port $PORT"` trong `railway.toml` được Railway thực thi trực tiếp dạng exec không qua subshell, khiến `$PORT` không được nội suy thành số cổng mà bị truyền thành chuỗi ký tự thô `"$PORT"`.
> - **Cách sửa:** Đổi `startCommand` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` để kích hoạt subshell nội suy biến `$PORT` do Railway cấp phát thành số nguyên hợp lệ. Sau đó kết nối biến `REDIS_URL` với `REDIS_PUBLIC_URL` của Redis service để hoàn tất kết nối.
