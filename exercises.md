# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Thế Việt  Mã học viên: 2A202602594

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu gán giá trị mặc định `"changeme"`, khi deploy lên Cloud mà dev quên thiết lập biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường mà không báo lỗi. Hậu quả là service chạy công khai với khóa bí mật mặc định mà ai cũng đoán được. Bất kỳ ai hoặc bot quét tự động trên Internet cũng có thể gọi API miễn phí qua khóa `"changeme"`, làm rò rỉ dữ liệu hoặc đốt sạch ngân sách LLM (như OpenAI, Anthropic), mà ta chỉ phát hiện khi nhìn thấy hóa đơn tiền triệu. Cơ chế "fail fast" bắt buộc app phải crash ngay từ lúc khởi động với lỗi `ValidationError`, giúp ta phát hiện và bổ sung secret ngay lập tức trước khi bất kỳ request nào được phục vụ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:07:03.456123+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 28, "cost_usd": 0.00012}
```

Hai việc làm được với dòng log JSON mà `print()` thông thường không làm được:
1. **Lọc và truy vấn tự động theo trường (Machine Parsing & Filtering):** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, Railway Logs) có thể parse các trường JSON để lọc chính xác: lọc theo `level == "error"`, tìm kiếm toàn bộ request của một `user_id` cụ thể, hoặc tính tổng chi phí `cost_usd` phát sinh trong ngày. `print()` chỉ in chuỗi text vô cấu trúc nên không thể filter hay group tự động.
2. **Thiết lập cảnh báo (Alerting) và vẽ biểu đồ giám sát (Metrics & Dashboards):** Dựa vào trường `cost_usd` và `tokens_out`, hệ thống monitoring có thể vẽ biểu đồ tiêu thụ token theo thời gian thực và tự động kích hoạt cảnh báo gửi về Slack/Email ngay khi có bất thường (ví dụ: phát hiện chi phí của một request vượt ngưỡng 0.05 USD).

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800MB) bao gồm:
1. **Base image đầy đủ vs Base image slim:** Base image ban đầu `python:3.11` chứa đầy đủ hệ điều hành Debian hoàn chỉnh với nhiều công cụ build, package quản lý hệ thống, man pages, tài liệu và các thư viện C/C++ tiêu chuẩn không cần thiết cho runtime. Trong khi đó, `python:3.11-slim` đã được lược bỏ tối đa các package thừa.
2. **Build dependencies và trình biên dịch:** Ở bản 1 stage, các công cụ build (như `gcc`, `make`, header files...), cache của `pip` và file tạm nằm lại vĩnh viễn trong image. Ở Multi-stage build, stage `builder` thực hiện cài đặt/biên dịch thư viện vào thư mục `/install`, sau đó stage `runtime` chỉ copy các package thành phẩm sang mà hoàn toàn không mang theo compiler hay file rác phát sinh trong quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại:**
  - Docker cache theo từng layer từ trên xuống dưới. Vì chỉ sửa code trong `app/main.py`, file `requirements.txt` không hề thay đổi.
  - Do đó, các layer cài đặt thư viện (`COPY requirements.txt .` và `RUN pip install ...` ở stage builder, cũng như layer `COPY --from=builder ...` và `RUN useradd ...` ở stage runtime) đều được **dùng lại từ cache (CACHED)**.
  - Chỉ các layer từ `COPY app ./app` trở đi mới bị vô hiệu hóa cache và phải chạy lại, giúp thời gian build chỉ mất khoảng 1-2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi sửa dù chỉ một ký tự trong mã nguồn, layer `COPY . .` sẽ bị thay đổi checksum, kéo theo việc Docker hủy toàn bộ cache của tất cả các layer phía sau nó.
  - Khi đó, lệnh `RUN pip install` sẽ bị ép chạy lại từ đầu trong mỗi lần build, khiến Docker phải tải lại toàn bộ dependencies từ Internet, kéo dài thời gian build từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện leo thang đặc quyền khi chạy root:**
  1. Ứng dụng Python xuất hiện một lỗ hổng bảo mật (ví dụ: Remote Code Execution - RCE qua `eval()`, `pickle.loads()`, hoặc Command Injection).
  2. Kẻ tấn công gửi payload độc hại để mở shell hoặc thực thi lệnh tùy ý bên trong container.
  3. Vì container mặc định chạy bằng user `root` (UID 0), kẻ tấn công chiếm toàn quyền root bên trong container (có thể sửa file hệ thống, cài tool độc hại).
  4. Nếu container có mount volume từ host (như Docker socket `/var/run/docker.sock`, thư mục hệ thống của host), hoặc gặp phải lỗ hổng thoát container (Container Breakout / Kernel Privilege Escalation), process của kẻ tấn công có cùng UID 0 trên kernel của máy host, dẫn đến việc kẻ tấn công chiếm quyền điều khiển root toàn bộ máy chủ thật (host).
- **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
  - Lệnh `USER appuser` chuyển tiến trình chạy sang một user không có đặc quyền (UID 10001).
  - Khi kẻ tấn công khai thác được RCE, họ chỉ sở hữu quyền hạn tối thiểu của `appuser`. Họ không thể can thiệp vào tài nguyên hệ thống, không thể chỉnh sửa file của user khác và không có UID 0 để thực hiện các đòn tấn công container breakout leo thang lên root của host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa:** **20 request** trong 2 giây liên tiếp.
- **Giải thích:**
  - Với cách đếm theo phút đồng hồ (Fixed Window), hạn mức được reset về 0 tại giây thứ 00 của mỗi phút.
  - Tại giây `10:00:59`: User gửi liên tiếp **10 request** (đạt mức tối đa cho phép của phút 10:00).
  - Ngay tại giây `10:01:00`: Hệ thống bước sang phút mới và reset bộ đếm request về 0.
  - Tại giây `10:01:01`: User tiếp tục gửi tiếp **10 request** nữa (hợp lệ trong hạn mức của phút 10:01).
  - Như vậy, trong khoảng thời gian chỉ 2 giây (từ `10:00:59` đến `10:01:01`), user đã gửi thành công **20 request** mà không bị chặn, gây đột biến tải gấp đôi cho server. Thuật toán Sliding Window (cửa sổ trượt 60 giây trên Redis ZSET) khắc phục triệt để lỗ hổng này bằng cách luôn tính chính xác số request trong 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt:**
  - `Rate Limit`: Giới hạn **số lượng request** trong khoảng thời gian ngắn (tần suất, ví dụ 10 request/phút) nhằm chống nghẽn và bảo vệ server khỏi bị tấn công từ chối dịch vụ (DoS). Cơ chế này không quan tâm nội dung hay độ dài của request.
  - `Cost Guard`: Giới hạn **tổng chi phí tài chính (USD)** trong khoảng thời gian dài (ngân sách, ví dụ 10 USD/tháng) nhằm bảo vệ ngân sách tài chính của chủ sở hữu dịch vụ khỏi việc bị cạn kiệt tiền do gọi API LLM.
- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  - User chỉ gửi **1 request duy nhất** trong cả giờ (tần suất cực thấp, hoàn toàn thỏa mãn Rate Limit 10 req/phút). Tuy nhiên request đó đính kèm tài liệu khổng lồ với prompt yêu cầu tạo output 100.000 tokens, ước tính chi phí 15 USD. Lúc này Cost Guard sẽ lập tức chặn và trả về lỗi `402 Payment Required` vì vượt hạn mức ngân sách tháng (10 USD).
- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  - User mới chi tiêu 0.05 USD / 10 USD ngân sách tháng. Tuy nhiên, user dùng script gửi liên tục **20 request chỉ trong vòng 3 giây** (mỗi request chỉ là câu chào ngắn tốn vài token, chi phí cực nhỏ). Cost Guard kiểm tra thấy ngân sách còn rất nhiều, nhưng Rate Limiter sẽ lập tức chặn từ request thứ 11 và trả về `429 Too Many Requests` để tránh làm tê liệt server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Giây 0:** Redis gặp sự cố (mạng chập chờn, restart hoặc nghẽn) và mất kết nối trong 30 giây.
2. **Giây 5–10:** Probe định kỳ của Orchestrator (Kubernetes, Docker Swarm hoặc Cloud Platform) gọi kiểm tra endpoint gộp này. Do Redis đang mất kết nối, cả 3 container agent đều đồng loạt trả về lỗi (503 hoặc timeout).
3. **Giây 15:** Vì endpoint này đóng vai trò Liveness (kiểm tra tiến trình còn sống không), Orchestrator cho rằng cả 3 container đều đã bị treo/chết, nên kích hoạt lệnh **kill và restart đồng loạt cả 3 container agent**.
4. **Giây 15–30:** Cả 3 container bị khởi động lại. Trong thời gian này, hệ thống hoàn toàn không còn container nào hoạt động để nhận request $\rightarrow$ 100% người dùng nhận lỗi 502/503.
5. **Giây 30:** Khi Redis bắt đầu phục hồi, các container vừa khởi động lại tiếp tục bị nghẽn (crash loop) do dồn ứ kết nối, khiến một sự cố gián đoạn tạm thời của Redis biến thành thảm họa sập toàn bộ hệ thống (cascading failure).
*(Việc tách riêng `/health` giúp Orchestrator biết tiến trình vẫn sống nên không restart container, trong khi `/ready` báo 503 để Load Balancer tạm thời ngừng chuyển traffic vào cho đến khi Redis kết nối trở lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi dùng Redis (Stateless - thực tế hiện tại):**
  - Cả 3 container agent đều kết nối chung vào một Redis database. Bất kể Load Balancer điều hướng request của user vào container nào (1, 2 hay 3), dữ liệu lịch sử đều được đọc và cập nhật vào cùng một key trên Redis. Do đó, `history_length` sẽ **tăng đều đặn theo từng lượt**: `0, 2, 4, 6, 8...`
- **Nếu lưu trong dict Python (Stateful trong bộ nhớ RAM):**
  - Mỗi container là một tiến trình độc lập với vùng nhớ RAM riêng biệt.
  - Khi Load Balancer phân phối request theo cơ chế xoay vòng (Round-Robin):
    - Lượt 1 vào container 1: `history_length = 0` (lưu vào RAM của container 1).
    - Lượt 2 bị chuyển sang container 2: container 2 chưa từng gặp user này $\rightarrow$ `history_length = 0`! (Agent bị mất trí nhớ).
    - Lượt 3 chuyển sang container 3: `history_length = 0`!
    - Lượt 4 quay lại container 1: `history_length = 2`.
  - Kết quả là `history_length` sẽ nhảy lộn xộn, lúc tăng lúc tụt về 0, làm cho phản hồi của agent bị đứt gãy hoàn toàn ngữ cảnh hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi kết nối Redis khi deploy lên Railway (`REDIS_URL` ban đầu trỏ về `localhost` của máy dev và mục Public Networking báo *"Could not load public networking"*).
- **Thông báo lỗi:**
  - Trên Railway dashboard hiển thị cảnh báo *"Could not load public networking"* và endpoint `/ready` trả về `503 {"status": "not ready", "redis": false}`.
- **Cách tìm ra nguyên nhân:**
  - Mở tab **Deployments / Logs** trên Railway để theo dõi log khởi động của container.
  - Nhận thấy container cố gắng kết nối tới `redis://localhost:6379/` và bị lỗi connection refused. Nguyên nhân là vì bên trong container chạy trên cloud, `localhost` trỏ vào chính container đó chứ không phải service Redis.
- **Cách sửa:**
  - Bấm `+ Add` $\rightarrow$ `Database` $\rightarrow$ `Add Redis` trên Railway để tạo một instance Redis độc lập.
  - Trong tab **Variables** của web service, cấu hình biến `REDIS_URL` sử dụng biến tham chiếu nội bộ của Railway: `${{Redis.REDIS_URL}}`.
  - Nhấn `Deploy` để áp dụng cấu hình và tạo domain công khai trong tab Settings. Sau đó endpoint `/ready` trả về ngay lập tức `200 {"status": "ready", "redis": true}`.
