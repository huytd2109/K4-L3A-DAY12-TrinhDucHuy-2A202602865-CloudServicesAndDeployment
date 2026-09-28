# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trịnh Đức Huy  Mã học viên: 2A202602865

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên đặt `AGENT_API_KEY`, `Settings` làm
> container dừng ngay và log báo thiếu cấu hình. Tôi phát hiện lỗi trước khi
> Railway đưa service ra Internet. Nếu có mặc định `"changeme"`, service vẫn
> báo healthy nhưng bất kỳ ai đoán được khóa mặc định đều có thể gọi `/ask`,
> tiêu quota và ngân sách của tôi. Fail fast biến một lỗi bảo mật âm thầm thành
> lỗi deploy nhìn thấy ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật tôi lấy từ container:
>
> ```json
> {"event":"ask_completed","level":"info","timestamp":"2026-09-28T08:20:06.853003+00:00","user_id":"cp4-live-634a333aa3a44de692c59e8a57d4aab5","tokens_in":3,"tokens_out":41,"cost_usd":0.00002505}
> ```
>
> Từ các field có cấu trúc, tôi có thể: (1) lọc theo `event`/`user_id` và cộng
> `cost_usd` để biết user nào tiêu nhiều tiền; (2) nhóm theo `timestamp` để đếm
> request, token và đặt cảnh báo khi lưu lượng hoặc chi phí tăng bất thường.
> Chuỗi `print("đã trả lời xong")` không mang đủ dữ liệu để làm hai việc đó.

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
| 1 stage (bản đầu) | 1.728,4 MB (Docker hiển thị khoảng 1,73 GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile gốc từ commit khởi tạo thành `agent:single` và build
> Dockerfile hiện tại thành `agent:multi`. Chênh lệch khoảng 1.457 MB chủ yếu
> đến từ image `python:3.11` đầy đủ: nhiều package hệ điều hành, compiler, header
> và công cụ build; bản một stage còn giữ cả cache của pip. Bản multi-stage dùng
> `python:3.11-slim` ở runtime, cài bằng `--no-cache-dir` và chỉ copy kết quả
> dependency từ builder, nên không mang môi trường build sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer `FROM`, `WORKDIR`,
> `COPY requirements.txt`, `RUN pip install` và
> `COPY --from=builder /install /usr/local` vẫn dùng cache vì dependency không
> đổi. Layer `COPY app ./app` bị invalid cache; các layer đứng sau nó phải được
> xét/build lại. Trong lần build không đổi source, output thực tế cho thấy các
> layer dependency đều là `CACHED`. Nếu đặt `COPY . .` trước
> `RUN pip install`, chỉ một ký tự source đổi cũng làm layer `COPY` đổi và buộc
> cài lại toàn bộ dependency, khiến build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: request khai thác lỗ hổng Python để thực thi lệnh trong
> container; tiến trình đang là root nên kẻ tấn công có toàn quyền trong
> container; nếu container còn được cấp quyền cao, mount thư mục nhạy cảm,
> mount Docker socket hoặc có lỗ hổng kernel/runtime, quyền đó có thể được dùng
> để sửa host hoặc tạo container đặc quyền. `USER appuser` cắt chuỗi ngay sau
> bước thực thi lệnh: mã bị chiếm chỉ có UID thường, không thể tùy ý sửa file hệ
> thống hay thực hiện thao tác cần root. Đây là giảm quyền, không thay thế việc
> tránh privileged mode và mount nguy hiểm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là **20 request trong khoảng 2 giây**: gửi 10 request ở cuối phút, ví
> dụ từ `10:00:59`, rồi ngay sau khi bộ đếm reset ở `10:01:00` gửi thêm 10
> request. Mỗi phút riêng vẫn chỉ có 10 request nhưng tải dồn thực tế là 20
> request sát nhau. Sliding window 60 giây vẫn nhìn thấy cả hai nhóm nên không
> cho phép cách lách này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất/số request trong 60 giây**, còn cost guard giới
> hạn **tổng tiền theo user trong tháng**. Một user chỉ gửi 1 request/phút nhưng
> đã tiêu gần hết ngân sách hoặc request đó rất đắt: rate limit cho qua nhưng
> cost guard phải chặn `402`. Ngược lại, user còn nguyên ngân sách nhưng gửi 11
> request rất rẻ trong 60 giây: cost guard vẫn cho phép về tiền, còn rate limiter
> chặn request thứ 11 bằng `429`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện sẽ là: (1) Redis mất kết nối; (2) probe gộp của cả ba container
> gọi Redis và cùng trả `503`; (3) orchestrator hiểu nhầm cả ba process đã chết
> và restart đồng thời; (4) container mới cùng khởi động, vẫn chưa nối được Redis
> nên tiếp tục fail probe và rơi vào restart loop; (5) cụm mất toàn bộ instance
> phục vụ dù code web vẫn sống. Khi tách endpoint, `/health` vẫn trả `200` nên
> container không bị restart, còn `/ready` trả `503` để load balancer tạm ngừng
> gửi traffic. Redis trở lại thì readiness tự phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Compose hiện publish cố định `8000:8000`, nên không thể scale ba replica trực
> tiếp trên cùng host port nếu chưa thêm load balancer/port động. Tôi kiểm tra
> tính stateless bằng hai `ConversationStore` riêng dùng chung Redis và bằng cách
> restart container agent thật: request đầu có `history_length=0`, sau restart
> request thứ hai vẫn thấy `history_length=2`. Với nhiều replica dùng chung
> Redis, chuỗi sẽ tiếp tục tăng `0, 2, 4, ...` dù request vào instance nào. Nếu
> dùng dict Python, mỗi replica có bản riêng nên số liệu sẽ nhảy hoặc reset như
> `0, 0, 2, 0, 2...` tùy request được chuyển tới container nào; restart còn làm
> mất toàn bộ lịch sử của instance đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế tôi gặp ở bước chuẩn bị deploy là:
> `railway: could not find the CLI binary ... The @railway/cli install step did
> not complete`. Tôi đọc output của npm và thấy postinstall của
> `@railway/cli@5.62.1` bị chặn bởi chính sách `allowScripts`, nên file
> `railway.exe` chưa được tải. Tôi cài lại đúng gói với lệnh
> `npm install -g --allow-scripts=@railway/cli @railway/cli --foreground-scripts`,
> kiểm tra `railway --version`, đăng nhập lại rồi deploy.
> Sau đó deployment `8acbbc46-1c21-433c-bf38-fbb18622a393` báo `SUCCESS`;
> `/health` và `/ready` đều trả `200`.
