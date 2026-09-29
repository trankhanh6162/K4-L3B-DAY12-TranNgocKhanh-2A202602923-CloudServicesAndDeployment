# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Ngọc Khánh  Mã học viên: 2A202602923

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi tôi deploy lên Railway nhưng quên đặt
> `AGENT_API_KEY`. Nếu code có khóa mặc định `"changeme"`, service vẫn báo
> healthy và người lạ có thể đoán khóa rồi gọi `/ask`, làm phát sinh chi phí.
> Khi trường này không có mặc định, Pydantic báo `ValidationError` ngay lúc
> khởi động nên deployment không nhận traffic và tôi biết phải bổ sung secret.
> Tôi đã thấy đúng hành vi này khi chạy lệnh tạo `Settings` với
> `_env_file=None`: chương trình dừng tại trường `agent_api_key` và báo
> `Field required`. Theo tôi đây là lỗi dễ xử lý hơn nhiều so với việc service
> vẫn chạy âm thầm bằng một khóa không an toàn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:39:46.804829+00:00", "user_id": "cp4-check", "tokens_in": 2, "tokens_out": 40, "cost_usd": 2.43e-05}`.
> Từ log này tôi có thể lọc và cộng chi phí theo `user_id`, đồng thời thống kê
> token hoặc số request theo khoảng thời gian dựa trên `timestamp`. Dòng
> `print("đã trả lời xong")` không có các trường có cấu trúc để máy truy vấn và
> tổng hợp như vậy. Khi xem `docker compose logs agent`, mỗi event nằm gọn trên
> một dòng nên tôi vẫn đọc được bằng mắt, còn hệ thống thu thập log cũng có thể
> parse trực tiếp. Nếu cần điều tra một user dùng nhiều token, tôi có thể lọc
> `user_id` rồi xem `tokens_in`, `tokens_out` và `cost_usd` thay vì phải đoán từ
> một câu log tự do.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile một-stage từ commit ban đầu và đo được `1.73GB`, còn
> image multi-stage là `271MB`. Phần chênh lệch chủ yếu đến từ base image
> `python:3.11` đầy đủ và các thành phần không cần cho runtime. Bản multi-stage
> dùng `python:3.11-slim` và chỉ copy dependency đã cài từ builder sang runtime,
> nên image cuối không mang toàn bộ môi trường build và file thừa. Chênh lệch
> khoảng 1.46GB cũng ảnh hưởng rõ tới thời gian tải image: lần build bản cũ phải
> tải base image hơn 200MB ở một layer và mất khá lâu, trong khi các lần build
> bản slim sau đó nhanh hơn và phù hợp hơn để deploy lại nhiều lần.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và
> `RUN pip install` vẫn được lấy từ cache; layer copy source và các layer phía
> sau nó phải tạo lại. Tôi quan sát log build hiển thị `CACHED` cho các bước cài
> dependency. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source
> sẽ làm layer copy đổi, kéo theo layer pip mất cache và phải tải/cài toàn bộ
> thư viện lại. Trong lần `docker compose up -d --build` sau khi sửa CP4, phần
> requirements không đổi nên Docker bỏ qua bước cài package và chỉ cần cập nhật
> phần source. Vì vậy tôi hiểu thứ tự lệnh không chỉ để Dockerfile đẹp hơn mà
> ảnh hưởng trực tiếp tới thời gian build mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công sẽ chạy lệnh
> với quyền của process trong container. Nếu process là root và container còn
> được cấp mount hoặc capability nguy hiểm, họ có thể sửa file hệ thống, đọc
> dữ liệu được mount và tìm cách tác động tới host. Lệnh `USER appuser` làm
> Uvicorn chạy với UID 10001, nên quyền có được sau khai thác chỉ là quyền user
> thường. Tôi kiểm tra bằng `docker run --entrypoint id` và nhận
> `uid=10001(appuser)`, không phải `uid=0(root)`. Việc này không biến container
> thành an toàn tuyệt đối, nhưng nó cắt chuỗi tấn công ngay tại bước process bị
> chiếm quyền: kẻ tấn công không mặc nhiên có quyền root để sửa mọi thứ bên
> trong container hoặc tận dụng các quyền nhạy cảm được cấp nhầm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây: gửi 10 request ở giây 59 của phút trước,
> sau đó gửi tiếp 10 request ở giây 00 của phút mới. Bộ đếm theo phút đã reset
> nên cả hai nhóm đều hợp lệ dù thực tế có 20 request sát nhau. Sliding window
> 60 giây vẫn nhìn thấy nhóm trước nên sẽ chặn nhóm sau khi đạt hạn mức. Trong
> bài tôi lưu timestamp vào Redis Sorted Set, xóa các entry cũ hơn `now - 60`
> rồi mới đếm. Khi test public URL, 10 request đầu trả 200 và request 11, 12 trả
> 429, đúng với giới hạn đã cấu hình.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây, còn cost guard giới hạn tổng số
> tiền một user đã tiêu trong tháng. Một user gửi ít request nhưng mỗi prompt
> rất dài có thể vẫn nằm dưới rate limit nhưng bị cost guard chặn. Ngược lại,
> một user đã gửi 10 câu rất ngắn trong vài giây có thể còn ngân sách tháng
> nhưng request thứ 11 vẫn bị rate limiter trả 429. Tôi thấy hai lớp này cần
> kiểm tra trước khi gọi LLM: rate limit bảo vệ hệ thống khỏi một đợt request
> dồn dập, còn cost guard bảo vệ hóa đơn trong thời gian dài hơn. Nếu kiểm tra
> sau `ask_llm()` thì dù trả lỗi cho client, chi phí của request đó đã phát sinh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai probe, Redis mất kết nối làm cả ba container trả 503 cho liveness.
> Orchestrator hiểu nhầm rằng ba process đã chết và restart tất cả gần như cùng
> lúc. Trong lúc restart không còn instance nào phục vụ; nếu Redis vẫn chưa
> hồi phục, các container mới lại tiếp tục fail và tạo vòng lặp restart. Tách
> `/ready` giúp load balancer chỉ rút traffic khỏi instance, còn `/health` vẫn
> 200 nên process không bị restart vô ích. Khi Redis hoạt động, tôi gọi
> `/ready` trên Railway và nhận `{"status":"ready","redis":true}`. Còn
> `/health` chỉ trả tên và phiên bản service, hoàn toàn không tạo kết nối Redis;
> đây là điểm khác nhau tôi thấy rõ nhất giữa hai endpoint.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong test stateless, hai `ConversationStore` khác nhau cùng dùng một Redis
> vẫn đọc được dữ liệu của nhau; request đầu có `history_length=0`, request kế
> tiếp thấy hai message trước đó nên có `history_length=2`. Khi nhiều container
> dùng chung Redis, độ dài tiếp tục tăng nhất quán dù request đổi instance. Nếu
> dùng dict Python, mỗi container có một bản riêng nên `history_length` sẽ lúc
> tăng, lúc quay về 0 hoặc một giá trị thấp hơn tùy request rơi vào container
> nào; restart container còn làm mất toàn bộ lịch sử của nó. Tôi chưa bật Nginx
> để quan sát round-robin trực tiếp vì đó là phần mở rộng, nhưng test dùng hai
> object store riêng đã mô phỏng đúng hai container cùng trỏ tới một Redis. Tôi
> cũng kiểm tra Redis thật sau một request: `LLEN history:cp4-check` bằng 2 và
> TTL bằng 604800 giây, tức lịch sử được giữ 7 ngày thay vì nằm trong RAM.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy Railway đầu tiên của tôi build và chạy thành công. Lỗi production
> tôi gặp ngay trước khi deploy là container `agent` thoát với thông báo
> `NotImplementedError: TODO (CP4): cài đặt install` tại
> `lifecycle.install()`. Tôi tìm nguyên nhân bằng `docker compose logs agent`,
> thấy FastAPI gọi hàm này trong lifespan lúc startup. Tôi triển khai handler
> cho `SIGTERM`/`SIGINT`, chuyển tiếp handler cũ của Uvicorn và thêm `exec` vào
> `CMD` để Uvicorn nhận signal trực tiếp. Sau khi build lại, log có
> `Application startup complete`; khi stop còn có `Application shutdown
> complete`. Cùng image đó được Railway deploy thành công, `/health` và
> `/ready` đều trả 200. Trên Railway tôi tạo riêng service `agent` và Redis,
> truyền `REDIS_URL` bằng tham chiếu nội bộ thay vì chép mật khẩu vào repo.
> Railway cấp `PORT=8080`; log cho thấy Uvicorn thực sự chạy tại
> `0.0.0.0:8080`, nên phần đọc `$PORT` trong cấu hình đã hoạt động đúng. Sau
> cùng tôi gọi public URL qua HTTPS, thử cả trường hợp thiếu key (401), đúng key
> (200) và gọi quá giới hạn (429) trước khi kết luận bản deploy hoạt động.
