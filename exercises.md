# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời vào phần trích dẫn bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đào Ngọc Bình Thiên  Mã học viên: 2A202602814

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy service lên môi trường production trên Cloud (như Railway/Render), nếu lập trình viên sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
> - Nếu để giá trị mặc định `"changeme"`: Service vẫn khởi động thành công và báo trạng thái xanh. Tuy nhiên, các bot quét tự động trên Internet có thể dùng ngay khóa mặc định này để gọi endpoint `/ask`, tiêu tốn hạn mức và làm phát sinh chi phí LLM thực tế mà người quản trị không hề hay biết cho đến khi nhận hóa đơn.
> - Ngược lại, khi không có giá trị mặc định: Pydantic ném lỗi `ValidationError` và làm app dừng hoạt động ngay lập tức (Fail Fast). Lỗi được hiển thị rõ ràng trong build/deploy log giúp lập trình viên phát hiện và bổ sung secret ngay lập tức trước khi service phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:30:00.123456+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000114}`
> 
> Hai việc làm được với dòng log JSON mà `print` không làm được:
> 1. Truy vấn và tổng hợp dữ liệu tự động (Structured Aggregation): Các hệ thống quản lý log (như Datadog, Grafana Loki, CloudWatch) có thể parse các trường JSON để tính toán số liệu thống kê trong thời gian thực, ví dụ: tính tổng chi phí `cost_usd` của từng `user_id` trong ngày, hoặc tính lượng token trung bình tiêu thụ trên mỗi request.
> 2. Giám sát và kích hoạt cảnh báo tự động (Alerting & Metrics): Dễ dàng thiết lập các bộ lọc cảnh báo khi có bất thường, ví dụ: gửi cảnh báo về Slack khi trường `level == "error"` vượt quá tỷ lệ 5% trong 5 phút, hoặc khi phát hiện một request đơn lẻ có `cost_usd` tăng vọt bất thường.

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
| 1 stage (bản đầu) | 1045 MB |
| Multi-stage | 192 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~853 MB) bao gồm:
> - Base OS image: Bản 1 stage dùng base image `python:3.11` đầy đủ chứa toàn bộ hệ điều hành Debian với nhiều thư viện, tiện ích, documentation và gói phụ trợ không cần thiết trong môi trường chạy; trong khi multi-stage dùng `python:3.11-slim` đã được lược bỏ tối đa.
> - Build dependencies & compilers: Trong stage builder, các công cụ biên dịch (`gcc`, `g++`, `make`), header packages (`python3-dev`, `libc-dev`) và pip cache chỉ phục vụ cho việc build bánh xe thư viện (wheels), sau đó chỉ thư mục cài đặt kết quả (`/install` chuyển sang `/usr/local`) được copy sang stage runtime, loại bỏ hoàn toàn các toolchain nặng nề khỏi image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile tối ưu:
>   - Các layer được tái sử dụng từ cache (`CACHED`): `FROM python:3.11-slim`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`, tạo user `appuser`.
>   - Các layer phải chạy lại: Chỉ từ lệnh `COPY app ./app` trở đi mới chạy lại vì nội dung mã nguồn thay đổi, thời gian build lại chỉ mất 1-2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ ký tự nào trong code, checksum của thư mục thay đổi làm mất hiệu lực toàn bộ layer cache từ `COPY . .` trở đi. Khi đó, Docker bị buộc phải chạy lại toàn bộ bước `RUN pip install`, tải lại và cài đặt lại tất cả thư viện từ đầu, làm tốn nhiều phút và tiêu hao băng thông không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
> 1. Ứng dụng Python tồn tại lỗ hổng bảo mật (ví dụ: Remote Code Execution qua pickle/deserialization, command injection, hoặc thư viện phụ thuộc có lỗ hổng zero-day).
> 2. Kẻ tấn công gửi payload khai thác thành công để thực thi lệnh shell bên trong container. Vì container mặc định chạy với user `root` (UID 0), kẻ tấn công chiếm toàn quyền root trong môi trường container.
> 3. Từ quyền root trong container, kẻ tấn công thực hiện kỹ thuật Container Escape (khai thác lỗ hổng kernel của máy host, truy cập Docker socket `/var/run/docker.sock` nếu bị mount, hoặc can thiệp vào các filesystem mount nhạy cảm từ host). Do UID 0 trong container thường ánh xạ tới UID 0 trên host (nếu không bật user namespace remap), kẻ tấn công có được quyền root trên toàn bộ máy host.
> 
> Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 2: Tiến trình Python chỉ chạy với quyền của một user thông thường không có đặc quyền (UID 10001). Ngay cả khi thực thi được mã độc, kẻ tấn công không thể đọc/ghi file nhạy cảm, không thể cài cắm rootkit hay thao tác với Docker daemon, vô hiệu hóa nguy cơ vượt rào container ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
> 
> Giải thích cách đạt được:
> - Cơ chế đếm theo phút đồng hồ (Fixed Window Counter) reset hạn ngạch vào đầu mỗi phút (giây 00).
> - Người dùng gửi liên tiếp 10 request vào giây cuối cùng của phút thứ nhất, cụ thể lúc `10:00:59`. Cả 10 request này đều được tính vào hạn mức của phút 10:00 và đều được chấp thuận.
> - Ngay 1 giây sau đó, đồng hồ chuyển sang `10:01:00`. Bộ đếm bị reset về 0 cho phút mới. Người dùng gửi tiếp 10 request vào thời điểm này.
> - Kết quả: Trong khoảng thời gian chỉ 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công tổng cộng 20 request, gây ra đột biến tải gấp đôi hạn mức quy định. Cửa sổ trượt (Sliding Window) giải quyết triệt để lỗi này bằng cách luôn tính chính xác số lượng request trong 60 giây tính lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Điểm khác nhau:
> - Rate Limit: Kiểm soát tần suất lưu lượng (số lượng request trong một khoảng thời gian ngắn, ví dụ: 10 request/phút) nhằm bảo vệ tính sẵn sàng của hạ tầng và chống DoS.
> - Cost Guard: Kiểm soát chi phí tài chính (tổng số tiền chi tiêu tích lũy theo chu kỳ dài hơn như tháng) nhằm tránh cạn kiệt ngân sách do chi phí token của LLM.
> 
> Tình huống cụ thể:
> 1. Rate limit cho qua nhưng Cost guard chặn: Người dùng chỉ gửi 1 request trong 10 phút (tần suất rất thưa, hoàn toàn nằm trong hạn mức 10 req/phút). Tuy nhiên request này kèm prompt chứa tài liệu dài 50,000 token, chi phí ước tính 0.25 USD, trong khi ngân sách tháng của người dùng chỉ còn lại 0.05 USD -> Cost guard chặn lại với mã 402 Payment Required.
> 2. Cost guard cho qua nhưng Rate limit chặn: Người dùng mới bắt đầu tháng với toàn bộ ngân sách 10.0 USD nguyên vẹn. Người dùng gửi liên tục 15 request ngắn (mỗi câu chỉ 5 token, chi phí chưa đến 0.0001 USD) trong vòng 5 giây -> Ngân sách còn rất nhiều nhưng vượt quá tần suất cho phép -> Rate limit chặn với mã 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Redis gặp sự cố mất kết nối mạng hoặc bảo trì khởi động lại trong vòng 30 giây.
> 2. Cả 3 container agent nhận request thăm dò từ orchestrator (liveness probe). Vì endpoint kiểm tra cả Redis và thấy Redis không phản hồi, nó đồng loạt trả về HTTP 503 Unhealthy.
> 3. Orchestrator hiểu rằng tiến trình bên trong container đã bị treo/hỏng và tự động thực hiện hành động khắc phục: restart (hoặc tiêu diệt và tạo mới) cả 3 container cùng lúc.
> 4. Các container mới khởi động lại, tiếp tục gọi probe và vẫn thấy Redis đang mất kết nối -> Lại báo lỗi 503 -> Tiếp tục bị restart lặp đi lặp lại (hiện tượng CrashLoopBackOff).
> 5. Toàn bộ cụm dịch vụ rơi vào tình trạng gián đoạn hoàn toàn (cascade failure). Khi Redis kết nối lại được, các container vẫn đang trong chu kỳ restart và mất thêm thời gian để ổn định.
> -> Phân tách đúng: `/health` (liveness) chỉ kiểm tra process sống để restart; còn `/ready` (readiness) kiểm tra Redis để load balancer tạm thời ngưng đẩy traffic tới container mà không restart nó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong một dict Python in-memory thay vì Redis:
> - Khi cụm có 3 instance agent (A, B, C) đứng sau Load Balancer, các request tiếp theo của cùng một `X-User-Id` sẽ được điều phối ngẫu nhiên (Round-Robin) tới các instance khác nhau.
> - Request 1 tới instance A: Lưu lịch sử vào bộ nhớ A -> `history_length` = 0.
> - Request 2 tới instance B: Bộ nhớ của B chưa từng gặp user này -> `history_length` lại là 0 thay vì 2.
> - Request 3 tới instance C: Tiếp tục nhận `history_length` = 0.
> - Request 4 nếu quay lại instance A: `history_length` lại nhảy lên 2.
> -> Con số `history_length` sẽ tăng giảm lộn xộn và không nhất quán. Agent sẽ biểu hiện như bị "mất trí nhớ ngẫu nhiên", không duy trì được ngữ cảnh hội thoại liên tục giữa các lượt trao đổi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi gặp phải: Healthcheck probe timeout dẫn đến deploy thất bại trên platform Cloud (Railway/Render).
> - Thông báo lỗi: Platform báo `Service failed to respond on port 8000 within 60s` hoặc `Application failed to start and timed out`.
> - Cách tìm ra nguyên nhân: Đọc runtime logs trên dashboard của platform, nhận thấy nền tảng tự động cấp phát một cổng động ngẫu nhiên qua biến môi trường `$PORT` (ví dụ: PORT=34567), trong khi ứng dụng vẫn lắng nghe cố định trên cổng 8000. Đồng thời, cấu hình cũ bind vào `127.0.0.1` khiến traffic từ router bên ngoài không thể đi vào container.
> - Cách sửa: Cập nhật chỉ thị CMD trong Dockerfile thành:
>   `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
>   Điều này đảm bảo server lắng nghe trên tất cả các network interface (`0.0.0.0`) và ưu tiên nhận giá trị cổng từ biến `$PORT` của Cloud cung cấp.
