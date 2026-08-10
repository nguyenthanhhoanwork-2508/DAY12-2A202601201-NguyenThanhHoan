# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thanh Hoàn  Mã học viên: 2A202601201

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu tôi quên cấu hình `AGENT_API_KEY` trên cloud mà ứng dụng có khóa mặc
> định `"changeme"`, service vẫn khởi động và bất kỳ ai biết khóa mẫu cũng có
> thể gọi `/ask`, làm phát sinh chi phí. Khi trường này không có mặc định,
> Pydantic ném `ValidationError` ngay lúc deploy. Health check thất bại và tôi
> thấy lỗi khi còn đang theo dõi log, trước khi service nhận traffic công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được là:
> `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T03:10:12.135312+00:00","user_id":"cp4-runtime-check","tokens_in":45,"tokens_out":46,"cost_usd":3.435e-05}`.
> Với log này, tôi có thể lọc và cộng `cost_usd` theo `user_id` để tìm người
> dùng tiêu nhiều nhất. Tôi cũng có thể đếm event theo thời gian hoặc level để
> theo dõi lưu lượng và tạo cảnh báo. Dòng `print("đã trả lời xong")` không có
> trường dữ liệu ổn định để máy thực hiện hai việc đó.

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
| 1 stage (bản đầu) | 1.17 GB |
| Multi-stage | 183 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage chứa base image Python đầy đủ, công cụ hệ thống và toàn bộ môi
> trường cài package nên đạt 1.17 GB. Bản multi-stage chỉ đưa package đã cài từ
> builder sang runtime `python:3.11-slim`, không mang builder và các thành phần
> thừa, nên còn 183 MB. Chênh lệch khoảng 987 MB chủ yếu đến từ base image đầy
> đủ và công cụ chỉ cần trong quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa source rồi build lại, các layer base image, tạo user,
> `COPY requirements.txt` và `RUN pip install` đều hiện `CACHED`. Docker chỉ
> chạy lại các layer `COPY app ./app`, `COPY utils ./utils` và export image.
> Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source làm layer
> `COPY` đổi; Docker phải chạy lại `pip install`, khiến build chậm dù dependency
> không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước tiên có
> quyền của process trong container. Khi process chạy root, quyền đó là root
> trong container; nếu runtime/container hoặc mount host còn lỗ hổng, họ có thể
> đọc file nhạy cảm, sửa volume hoặc leo sang host với quyền cao. Lệnh
> `USER appuser` làm process chạy bằng UID 10001 không đặc quyền. Vì vậy bước
> chiếm process chỉ cho quyền hạn chế, cắt chuỗi tấn công trước quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong khoảng 2 giây: gửi 10 request ở cuối phút,
> ví dụ 10:00:59, rồi gửi tiếp 10 request ngay sau khi bộ đếm reset ở 10:01:00.
> Cả hai nhóm đều hợp lệ với bộ đếm theo phút nhưng tạo burst 20 request. Cửa
> sổ trượt 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau đến nên chặn burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ request trong 60 giây, còn cost guard giới hạn
> tổng tiền của từng user trong tháng. Một user gửi dưới 10 request/phút nhưng
> mỗi request có prompt rất lớn vẫn qua rate limit; cost guard phải chặn khi
> tổng chi phí vượt 10 USD. Ngược lại, user gửi 11 request nhỏ liên tiếp khi
> mới tiêu rất ít tiền vẫn còn ngân sách, nhưng request thứ 11 bị rate limiter
> chặn. Khi kiểm tra thật, 10 request đầu trả 200 và 5 request sau trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint chung kiểm tra Redis, lúc Redis mất kết nối cả ba container đều
> trả 503. Orchestrator xem đó là lỗi liveness và restart cả ba gần như cùng
> lúc. Trong khi Redis chưa phục hồi, container mới tiếp tục fail health check
> và có thể bị restart lặp lại. Khi Redis hoạt động lại, cụm vẫn có thể chưa có
> instance sẵn sàng nên toàn bộ traffic bị gián đoạn. Tách `/health` giúp
> process vẫn sống; `/ready` chỉ yêu cầu load balancer tạm ngừng gửi traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong lần kiểm tra runtime với Redis, request đầu trả `history_length=0` và
> request thứ hai trả `history_length=2`, vì lượt user và assistant trước đó đã
> được lưu. Test stateless còn tạo hai `ConversationStore` khác nhau dùng chung
> Redis và instance thứ hai vẫn đọc được dữ liệu instance thứ nhất. Nếu dùng
> dict Python, mỗi container có lịch sử riêng; qua load balancer tôi sẽ thấy
> số lúc tăng, lúc quay về 0 hoặc lặp lại tùy request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy Railway đầu tiên build thành công nhưng health check thất bại. Log
> deploy báo `Invalid value for '--port': '$PORT' is not a valid integer`.
> Tôi đọc build log và thấy image đã tạo xong, sau đó đọc deploy log nên xác
> định lỗi nằm ở lệnh khởi động chứ không phải Docker build. `startCommand`
> trong `railway.toml` được Railway chạy trực tiếp nên `$PORT` không được shell
> expand. Tôi xóa override đó để Railway dùng `CMD` trong Dockerfile với
> `sh -c` và `${PORT:-8000}`. Lần redeploy sau thành công; `/health` và
> `/ready` đều trả 200.
