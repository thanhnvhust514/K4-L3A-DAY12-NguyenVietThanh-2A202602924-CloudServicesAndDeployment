# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Việt Thành  Mã học viên: 2A202602924

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: Khi deploy ứng dụng lên nền tảng cloud (như Railway hoặc Render), nếu developer quên cấu hình biến môi trường `AGENT_API_KEY` trong tab Variables của dự án. Nếu mã nguồn để giá trị mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động thành công và mở cổng public ra Internet. Các bot scanner tự động trên mạng liên tục quét các endpoint `/ask` với những key mặc định phổ biến (`changeme`, `admin`, `secret123`), từ đó chúng có thể thoải mái gửi hàng nghìn request gọi LLM, làm cạn kiệt ngân sách API và tài khoản thẻ của developer mà hệ thống không hề cảnh báo. Ngược lại, nhờ thiết kế "fail fast" không có giá trị mặc định, `pydantic-settings` sẽ ném ngoại lệ `ValidationError` và crash ngay lập tức lúc khởi động. Nền tảng cloud sẽ báo build/deploy thất bại ngay tại chỗ, buộc developer phải vào cấu hình khóa bí mật an toàn trước khi dịch vụ kịp nhận bất kỳ lượt traffic nào từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thực tế thu được từ service:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:54:22.012345+00:00", "user_id": "sv-test", "tokens_in": 92, "tokens_out": 45, "cost_usd": 0.0000408}`
>
> Hai việc làm được với dòng log này mà `print("đã trả lời xong")` không thể làm được:
> 1. **Tổng hợp số liệu và vẽ biểu đồ thời gian thực (Metrics & Aggregation):** Các công cụ quản lý log tập trung (như Datadog, Grafana Loki, CloudWatch, hoặc Railway Logs) có thể tự động phân tích cú pháp JSON để tính tổng chi phí `cost_usd`, đo lường tổng lượng token tiêu thụ theo từng `user_id`, hoặc vẽ đồ thị thống kê lượng request theo từng phút/giờ mà không cần phải viết regex bóc tách văn bản thô dễ lỗi.
> 2. **Thiết lập cảnh báo tự động và truy vết sự cố (Alerting & Tracing):** Có thể cài đặt bộ lọc cảnh báo tự động gửi về Slack/PagerDuty khi trường `cost_usd` của một request vượt quá ngưỡng quy định, hoặc khi có sự cố phát sinh thì lọc chính xác mọi request của một `user_id` cụ thể theo dải thời gian `timestamp` ISO-8601 chuẩn xác đến từng mili-giây.

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
| Multi-stage | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~836 MB) đến từ hai nguyên nhân chính:
> 1. **Khác biệt base image:** Bản 1-stage sử dụng `python:3.11` đầy đủ dựa trên Debian chuẩn, mang theo toàn bộ trình biên dịch C/C++, các công cụ phát triển phần mềm (build tools) và nhiều gói hệ điều hành không cần thiết cho môi trường chạy production. Bản multi-stage chuyển sang sử dụng `python:3.11-slim` đã được lược bỏ tối đa các gói phụ trợ dư thừa.
> 2. **Cơ chế Multi-stage build:** Toàn bộ công đoạn build dependency, cache của pip và các thư viện biên dịch trung gian chỉ tồn tại ở stage `builder`. Khi chuyển sang stage `runtime`, Docker chỉ sao chép kết quả đã cài đặt trong `/install` sang `/usr/local`, hoàn toàn loại bỏ các file tạm, package cache và compiler khỏi image cuối cùng, giúp image gọn nhẹ, bảo mật và deploy nhanh hơn nhiều lần.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:** Khi sửa một ký tự trong `app/main.py`, tất cả các layer phía trước bao gồm: base image, `COPY requirements.txt .`, `RUN pip install...`, và `COPY --from=builder /install /usr/local` đều được dùng lại 100% từ cache (`CACHED`). Chỉ duy nhất layer `COPY --chown=appuser:appuser . .` và các chỉ lệnh phía sau nó mới phải chạy lại. Quá trình build lại diễn ra gần như tức thì (chỉ mất ~1 đến 2 giây).
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`:** Theo cơ chế của Docker, mỗi layer phụ thuộc vào các layer trước đó; nếu một layer bị thay đổi nội dung thì toàn bộ cache của các layer phía sau nó đều bị vô hiệu hóa (cache invalidated). Khi đó, mỗi lần sửa dù chỉ một dấu phẩy trong code, Docker sẽ buộc phải chạy lại lệnh `RUN pip install`, tải lại toàn bộ các thư viện qua mạng Internet và cài đặt lại từ đầu, làm thời gian build kéo dài thêm vài phút mỗi lần thay đổi mã nguồn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - **Chuỗi sự kiện tấn công khi chạy root:**
>   1. Ứng dụng Python chứa một lỗ hổng bảo mật (ví dụ: lỗi Deserialization, SQL Injection dẫn tới RCE, hoặc buffer overflow trong một extension C).
>   2. Kẻ tấn công khai thác thành công lỗ hổng để thực thi mã tùy ý, giành được quyền shell bên trong container.
>   3. Vì container không đổi user nên tiến trình của kẻ tấn công đang chạy với quyền `root` (UID 0).
>   4. Từ quyền `root` trong container, kẻ tấn công khai thác tiếp các lỗ hổng nhân hệ điều hành Linux (kernel exploit như Dirty COW), tận dụng các quyền Linux capabilities mặc định chưa bị tước bỏ, hoặc khai thác Docker socket bị mount để thoát khỏi container (container breakout).
>   5. Do UID 0 bên trong container khớp với UID 0 trên máy chủ Linux host, kẻ tấn công lập tức sở hữu đặc quyền `root` trên chính máy chủ vật lý/host, kiểm soát toàn bộ dữ liệu và các container khác.
> - **Lệnh `USER` cắt đứt chuỗi ở đâu:** Lệnh `USER appuser` (UID 10001) tước bỏ toàn bộ đặc quyền quản trị. Ngay cả khi code Python bị khai thác chiếm shell, kẻ tấn công chỉ có quyền của một user thường: không thể ghi đè file nhạy cảm trong hệ thống, không có quyền can thiệp vào kernel và không thể thực hiện hành vi container escape để leo thang đặc quyền lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - **Số request tối đa trong 2 giây liên tiếp:** **20 request**.
> - **Giải thích cách đạt được:**
>   - Giả sử hạn mức là 10 request/phút theo đồng hồ cố định. Người dùng gửi liên tiếp 10 request vào giây `10:00:59` (giây cuối cùng của phút thứ nhất). Vì phút 10:00 mới chỉ nhận 10 request nên tất cả đều hợp lệ.
>   - Ngay 1 giây sau đó, thời gian chuyển sang `10:01:00`, bộ đếm theo phút cố định tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa ở giây `10:01:00`.
>   - Kết quả: Trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh chịu $10 + 10 = 20$ request mà không vi phạm luật đếm theo phút cố định, tạo ra hiện tượng xung đột tải (traffic spike) gấp đôi hạn mức. Thuật toán Sliding Window 60 giây (cửa sổ trượt) giải quyết triệt để vấn đề này vì nó luôn xét đúng 60 giây thực tế gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Điểm khác nhau:**
>   - **Rate limit:** Giới hạn **tần suất / số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) nhằm ngăn chặn tấn công DDoS, spam làm nghẽn đường truyền và quá tải CPU/RAM của server.
>   - **Cost guard:** Giới hạn **tổng chi phí tài chính tích lũy** (USD hoặc token) trong một chu kỳ dài hạn (ví dụ: $10.0 USD/tháng) nhằm bảo vệ ngân sách, ngăn chặn việc tài khoản bị cạn tiền do các câu hỏi quá dài hoặc độc hại.
> - **Tình huống Rate limit cho qua nhưng Cost guard phải chặn:**
>   Người dùng gửi request đầu tiên trong ngày (tần suất mới chỉ 1 request/phút, hoàn toàn trong hạn mức rate limit), nhưng trong tháng này người dùng đó đã tiêu lũy kế hết $10.0 USD ngân sách quy định $\rightarrow$ Rate limit cho qua, nhưng Cost guard phát hiện vượt ngân sách tháng và chặn lại với mã lỗi `402 Payment Required`.
> - **Tình huống Cost guard cho qua nhưng Rate limit phải chặn:**
>   Người dùng mới bắt đầu chu kỳ tháng, số dư ngân sách còn nguyên $10.0 USD, nhưng gửi liên tiếp 15 request câu hỏi ngắn chỉ trong vòng 5 giây $\rightarrow$ Cost guard cho qua vì tổng chi phí của 15 câu này chỉ tốn vài cent (rất nhỏ so với $10), nhưng Rate limit phát hiện vi phạm tần suất vượt quá 10 request/phút và lập tức chặn các request từ thứ 11 trở đi với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Khi Redis gặp sự cố mất kết nối trong 30 giây, endpoint chung bắt đầu trả về lỗi 503 hoặc timeout do không kết nối được Redis.
> 2. Container orchestrator (như Docker daemon, Kubernetes Kubelet, hoặc Railway Health Monitor) gửi liveness probe định kỳ vào endpoint này và thấy thất bại liên tiếp vượt quá số lần retry cho phép.
> 3. Do xem đây là liveness check, orchestrator kết luận rằng tiến trình container đã bị treo/hỏng và ngay lập tức gửi tín hiệu kill rồi **restart lại toàn bộ cả 3 container agent**.
> 4. Cả 3 container khởi động lại từ đầu, nhưng trong khoảng thời gian đó Redis vẫn chưa phục hồi, dẫn tới việc container vừa bật lên lại tiếp tục kiểm tra Redis thất bại và bị restart tiếp.
> 5. Toàn bộ cụm container rơi vào vòng xoáy khởi động chết liên hoàn (**CrashLoopBackOff / restart storm**), tiêu tốn lượng lớn CPU/RAM của server và làm gián đoạn toàn bộ dịch vụ. Trong khi thực tế, code của container hoàn toàn khỏe mạnh và chỉ cần Load Balancer tạm thời ngừng chuyển tiếp traffic qua cơ chế readiness probe độc lập cho đến khi Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Nếu lịch sử lưu trong dict Python của process:**
>   Khi có 3 instance container chạy song song, Load Balancer (hoặc Nginx) sẽ phân phối các request kế tiếp của cùng một user luân phiên vào Instance A, Instance B và Instance C theo thuật toán round-robin. Vì mỗi instance là một tiến trình độc lập với vùng nhớ RAM hoàn toàn riêng biệt, dict trên Instance A chỉ lưu tin nhắn gửi vào A, không hề biết các tin nhắn gửi vào B và C. Kết quả là `history_length` trong response sẽ thay đổi hỗn loạn và nhảy cóc không liên tục (ví dụ: lượt 1 vào A thấy 0; lượt 2 vào B thấy 0; lượt 3 vào C thấy 0; lượt 4 vào A lại thấy 2 thay vì 6). AI agent sẽ bị "mất trí nhớ từng chặng", không nắm được ngữ cảnh câu hỏi trước đó của người dùng.
> - **Khi lưu trên Redis:** Cả 3 instance đều phi trạng thái (stateless) và cùng trỏ về một cơ sở dữ liệu Redis chung, do đó bất kể request rơi vào instance nào thì `history_length` luôn tăng đều đặn chính xác ($0 \rightarrow 2 \rightarrow 4 \rightarrow 6...$).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Thông báo lỗi gặp phải:**
>   Khi deploy thành công service lên Railway, endpoint `/health` trả về `HTTP 200 OK`, nhưng khi gọi kiểm tra endpoint `/ready` thì nhận thông báo lỗi: `HTTP/1.1 503 Service Unavailable` với nội dung JSON: `{"status":"not ready","redis":false}`.
> - **Cách tìm ra nguyên nhân:**
>   Kiểm tra logic trong `app/main.py` ở hàm `ready()`, ta thấy phản hồi `redis: false` xuất hiện khi `store.ping()` trả về `False`. Mở tab Variables của service web trên dashboard Railway để kiểm tra cấu hình, phát hiện biến `REDIS_URL` ban đầu chưa được liên kết tới node Redis database vừa tạo (nó vẫn đang trỏ về địa chỉ mặc định `localhost:6379`, trong khi bên trong container của Railway thì localhost không có Redis nào chạy cả).
> - **Cách sửa lỗi:**
>   Trên dashboard Railway, vào service Agent $\rightarrow$ chọn tab **Variables** $\rightarrow$ thêm biến `REDIS_URL` và sử dụng cú pháp tham chiếu biến nội bộ của Railway: `${{Redis.REDIS_URL}}` (trỏ trực tiếp vào domain mạng riêng `redis.railway.internal:6379`). Sau khi lưu cấu hình, Railway tự động redeploy lại container, và khi gọi lại endpoint `/ready`, service đã phản hồi thành công: `HTTP/1.1 200 OK {"status":"ready","redis":true}`.
